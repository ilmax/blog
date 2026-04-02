---
title: "Dotnet Nuget Why, your new best friend to troubleshoot transitive dependencies issues"
date: 2024-09-03T14:52:48+02:00
description: Let's look at what's dotnet nuget why is, how to install it and how we can use it to detect transitive vulnerabilities
draft: false
tags: [dotnet]
---

Recently, a new dotnet CLI command was introduced without too much fanfare, so I thought it was worth writing a few lines about it.
This new command is very useful to debug transient dependency issues, the command is `dotnet nuget why` and it helps figure out why a transitive package is referenced.

This command is available starting with .NET SDK version 8.0.4xx, so ensure you have at least that version installed. If not, you can download it [here](https://get.dot.net).

> [!TIP]
> You can access the command's documentation on the [Microsoft website](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-why)

You can verify if you have a compatible SDK using the following commands:

```sh
dotnet --info
```

The output of this command will display, among other details, all the installed dotnet SDKs on the machine along with their respective versions

or

```sh
dotnet --list-sdks
```

## Usage

The command usage looks like the following:

```sh
dotnet nuget why <PROJECT|SOLUTION> <PACKAGE> [-f|--framework <FRAMEWORK>]
```

The output looks is similar to what you can see here below:

```console
dotnet nuget why ConsoleApplication.sln System.Collections.Immutable
Project 'ConsoleApplication' has the following dependency graph(s) for 'System.Collections.Immutable':

  [net8.0]
   │
   └─ BenchmarkDotNet (v0.13.12)
      ├─ Microsoft.CodeAnalysis.CSharp (v4.1.0)
      │  └─ Microsoft.CodeAnalysis.Common (v4.1.0)
      │     └─ System.Collections.Immutable (v5.0.0)
      └─ Microsoft.Diagnostics.Runtime (v2.2.332302)
         └─ System.Collections.Immutable (v5.0.0)
```

If the project under analysis targets multiple frameworks, there's an option to specify which framework to look at using the **-f** flag as shown below:

```console
dotnet nuget why ConsoleApplication.sln System.Collections.Immutable -f net6.0
Project 'ConsoleApplication' has the following dependency graph(s) for 'System.Collections.Immutable':

  [net6.0]
   │
   └─ BenchmarkDotNet (v0.13.12)
      ├─ Microsoft.CodeAnalysis.CSharp (v4.1.0)
      │  └─ Microsoft.CodeAnalysis.Common (v4.1.0)
      │     └─ System.Collections.Immutable (v5.0.0)
      └─ Microsoft.Diagnostics.Runtime (v2.2.332302)
         └─ System.Collections.Immutable (v5.0.0)
```

## Inspect transitive vulnerabilities

This tool becomes very useful especially when there's a vulnerable transitive dependency to investigate.

It would be useful to have a flag that shows all vulnerable packages in a given project, but that's not currently available. However, you can achieve this with a bit of Linux shell gynmastics as follows:

```sh
dotnet list package --vulnerable --include-transitive --format json | grep id | cut -d':' -f2 | sed 's/"\(.*\)".*/\1/' | xargs -I {} dotnet nuget why <project name> {}
```

That's it for today, I hope you find this useful, till the next time!
