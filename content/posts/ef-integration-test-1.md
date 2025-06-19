---
title: "Integration testing with EF Core, part 1"
description: How to run integration test with Entity Framework Core part 1, SQL docker container
date: 2021-04-06T16:25:33Z
draft: false
tags: ["efcore", "testing", "dotnet", "docker"]
series: ["Ef Core Integration testing"]
---

In this mini series I will go through some challenges and the solutions I applied in implementing integration testing with EF Core and SQL Server running on Docker.

EF Core has been out for a while now (according to [Wikipedia](https://en.wikipedia.org/wiki/Entity_Framework) it's been released on 27/6/2016) and since day one it had support for an in [memory database provider](https://docs.microsoft.com/en-us/ef/core/providers/in-memory/?tabs=dotnet-core-cli). The aim of the in memory database provider is to simplify testing and if you compare what it takes now to write test against an Entity Framework Core DbContext compared to the old Entity Framework one you can see how much easier it's now compared to the experience we had back then.

I won't go into why the in memory database is not the best bet for integration testing, [Jimmy Bogard](https://twitter.com/jbogard) already did that long time ago.
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">blogged about my thoughts on in-memory databases for testing purposes <a href="https://t.co/OZcEQvdMYH">https://t.co/OZcEQvdMYH</a> tl;dr - avoid. it&#39;s not worth the pain/side effects</p>&mdash; Jimmy Bogard 🍻 (@jbogard) <a href="https://twitter.com/jbogard/status/1240343707758534658?ref_src=twsrc%5Etfw">March 18, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
Long story short: there are several limitations introduced by the in memory provider e.g. it doesn't support transactions, so you may end up having to specialize your test code to work around these limitations.

If the in memory provider does satisfy your needs then this mini series is not for you. If, instead, you want  to run your tests on the infrastructure that matches, as closely as possible, your production environment keep on reading.

Starting from SQL Server 2017 it's possible to run the database engine in a container with Docker, so we can take advantage of this in order to run our integration tests on top of a real SQL Server database.

At the end of this series we will have:

- A throw away SQL Server DB so every test run starts from a clean state
- Integration tests that run on top of a SQL Server running in a docker container
- Running integration tests via command line (useful in a CI environment)
- Testing EF Core migrations (bonus)
- Running (and debugging) integration tests from within the IDE.

>In order to be able to successfully run integration tests that requires a DB connection, we need (stating the obvious) to have a SQL Server database up & running and ready to accept connections. One of the way to achieve this with Docker is via docker-compose.

When I started to implement this my focus was mostly on having the integration tests run during the CI builds so I started creating a docker-compose file for every integration project that needed SQL Server.

>I won't go in the detail of what docker-compose is and what it does, you can find the documentation [here](https://docs.docker.com/compose/)

I used docker compose to spin up SQL Server and a docker image created from my integration test project.

As you probably know, docker-compose has the [depends-on](https://docs.docker.com/compose/startup-order/) feature to control the start-up order, but there's no guarantee over the ready state of the dependency (i.e. your application may start quicker than the DB, and try to connect to the DB container that's not yet ready to accept connections)

In order to wait until SQL Server is up and running we will take advantage of the great [docker-compose-wait](https://github.com/ufoscout/docker-compose-wait) utility.

The Dockerfile for the integration test project looks like this:

```Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:5.0-alpine AS build
WORKDIR /src

# Get connection string argument from docker compose and set it as an environment variable
ARG connection_string
ENV ConnectionStrings__Database=${connection_string}

# Standard docker build
COPY ["tests/Integration.Tests/Integration.Tests.csproj", "Integration.Tests/"]

RUN dotnet restore "Integration.Tests/Integration.Tests.csproj"
COPY . .
WORKDIR "Integration.Tests"

# Restore the dotnet-ef command
RUN dotnet tool restore
RUN dotnet build "Integration.Tests.csproj" -c Release -o /app/build

# Install docker-compose-wait to make sure the db server is up & running before moving on
ADD https://github.com/ufoscout/docker-compose-wait/releases/download/2.5.0/wait /wait
RUN chmod +x /wait

# Wait for sql server and then migrate the db and run tests
CMD /wait && dotnet ef database update --context MyDbContext && dotnet test --no-build
```

The docker-compose file is a very straightforward one that looks like this:

```yml
version: "3"

services: 
    sql-server-db:
        image: mcr.microsoft.com/mssql/server:2019-latest
        ports:
            - "1533:1433"
        environment:
            SA_PASSWORD: "Guess_me"
            ACCEPT_EULA: "Y"
        logging:
            driver: none 

    integrationtests:
        image: integrationtests
        build: 
            context: ../..
            dockerfile: test/Integration.Tests/Dockerfile
            args: 
                connection_string: Data Source=sql-server-db;User Id=sa;Password=Guess_me
        environment:
            WAIT_HOSTS: sql-server-db:1433
        depends_on:
            - sql-server-db

```

So every time the integration tests container starts, we wait until the SQL Server is ready to accept connections, run EF core migrations and run `dotnet test`. Starting from a clean state every time may be a bit slow but it adds, as a bonus the ability to test migrations.

The last piece I added to make it easier to run tests locally, is just a run-tests.cmd file to run `docker-compose` with `--abort-on-container-exit` . It looks like this

```cmd
@echo off
REM Run Docker compose build and stops after the container exits
docker-compose up --build --abort-on-container-exit
REM Removes volumes, networks and images
docker-compose down
```

All of these files (Dockerfile, docker-compose and `run-tests.cmd`) lives in the integration tests directory.

This will gives us 4 of the 5 points outlined above, the only downside is that running and debugging from within Visual Studio doesn't work yet and this will be the subject of the next post.
