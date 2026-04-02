---
title: "Having a look at the new WebJobs support in App Service for Linux"
description: In this step by step guide, you will see how to create and deploy a WebJob in Azure App Service for Linux using .NET
date: 2024-10-03T08:18:59+02:00
draft: false
tags: [azure, dotnet, linux, webjobs]
type: "featured"
---

In this blog post, I want to explore one of the MS Build 2024 announcements that stood out to me. The reason it caught my eye is that WebJobs, which were previously only available for Windows-based App Service instances, now have expanded support. After reviewing the rather sparse documentation on Microsoft Learn, I decided to document the setup process for my own reference and to share with others who might find it helpful.

## What are WebJobs

Let's start with a quick reminder of what WebJobs are, here's the definition from the Microsoft Learn documentation:

{{<lead>}}
WebJobs is a feature of Azure App Service that enables you to run a program or script in the same instance as a web app. All app service plans support WebJobs. There's no extra cost to use WebJobs.
You can use the Azure WebJobs SDK with WebJobs to simplify many programming tasks.
{{</lead>}}

> [!WARNING]
> Please note that at the time of writing, October 2024, WebJobs for Linux is still in **preview**, so what I'm writing here may not be accurate when the functionality reaches GA.

## Types of WebJobs

Azure App Service supports two different types of WebJobs, those are:

- Continuous
- Triggered

Continuous WebJobs are very similar to an Azure Functions, they're always running and the WebJob SDK can be used to simplify the creation of WebJobs. Thsse WebJobs can be used, for exmaple, to respond to Azure Service Bus messages, a file uploaded to an Azure Storage Account, and so on.

Triggered WebJobs can be started manually or on a schedule and ca be useful to run a job at a specific time, e.g. to run some heavy tasks during the night.

The WebJobs SDK is very similar to the Azure Functions SDK, it provides triggers and bindings - input and output ones. This is no conincidence, in fact the .NET Azure Functions SDK is based on the WebJobs SDK one.

## Linux WebJobs

As mentioned above, until the MS Build conference in 2024, WebJobs were only available on a Windows-based App Service.

> [!NOTE]
> Here's the blog post from the App Service team announcing the availability of Linux WebJobs [https://azure.github.io/AppService/2024/04/04/Public-Preview-Sidecars-Webjobs.html](https://azure.github.io/AppService/2024/04/04/Public-Preview-Sidecars-Webjobs.html)

Let's now look at how to create and deploy a .NET WebJob on a Linux-based App Service.

With App Service on Linux we can choose to deploy using code (e.g. the output of dotnet publish) or using a container, so let's have a look at how to use WebJobs in both cases.

You can find all the code for this blog post, including the Terraform IaC and the GitHub actions, in this repository:

{{< github repo="ilmax/webjobs-linux" >}}

### Linux code

Since App Service is used to deploy web applications, I started with the classic ASP.NET Core _webapi_ template and then I created a console application following the steps shown in the WebJobs documentation [here](https://learn.microsoft.com/en-us/azure/app-service/webjobs-create?tabs=linuxcode).

Here's the full continuous WebJob code:

```csharp
class Program
{
    static async Task Main()
    {
        var builder = new HostBuilder();
        builder.ConfigureWebJobs(b =>
        {
            b.AddAzureStorageCoreServices();
            b.AddAzureStorageQueues();
        });
        builder.ConfigureLogging((context, b) =>
        {
            b.AddConsole();
        });
        
        using var host = builder.Build();
        await host.RunAsync();
    }
}

public class Functions
{
    public static void ProcessQueueMessage([QueueTrigger("webjobs-linux-queue")] string message, ILogger logger)
    {
        logger.LogInformation($"Continuous Job received message: {message}");
    }
}
```

The sample WebJob has a trigger on an Azure Storage Queue and echoes the message to the logger, so I've created and used this as a continuous WebJob plus a simple console application that I will use as a triggered one.

Here's the full code of the triggered webjob:

```csharp
Console.WriteLine("Hello World, from the triggered webjob");
```

To seamlessly deploy the WebJob alongside the application, I use a bit of MSBuild magic, I add a target on the ASP.NET Core web application to publish the WebJob to the specific folder where the App Service expects it.

These folders differ depending on the WebJob type:

- Continuous: \site\wwwroot\app_data\Jobs\Continuous\<job_name>
- Triggered: \site\wwwroot\app_data\Jobs\Triggered\<job_name>

This is achieved using this little MSBuild script:

```xml
  <Target Name="PostpublishScript" AfterTargets="Publish" Condition="$(IsDockerBuild) == 'false'">
    <Exec Command="dotnet publish ../WebJobContinuous/WebJobContinuous.csproj -o $(PublishDir)app_data/Jobs/Continuous/webjob-continuous"/>
    <Exec Command="dotnet publish ../WebJobTriggered/WebJobTriggered.csproj -o $(PublishDir)app_data/Jobs/Triggered/webjob-triggered"/>
  </Target>
```

With this code in place in the webapi project file, deployment is now as easy as deploying the webapi.

> [!NOTE]
> For the sake of completeness, I would also like to mention this [suggested approach](https://learn.microsoft.com/en-us/azure/app-service/webjobs-dotnet-deploy-vs) to deploying WebJobs, but haven't looked into it yet.

After following all the steps and defined in the MS Learn documentation [here](https://learn.microsoft.com/en-us/azure/app-service/webjobs-sdk-get-started#create-a-console-app) on how to create a continuous WenJob using the WebJob SDK, I've had to:

- Set `WEBSITES_ENABLE_APP_SERVICE_STORAGE` to true to enable WebJobs on Linux, otherwise the Azure portal will show the following error: `Remote storage is required for WebJob operations.` and the rest of the UI is completely greyed out as shown in the image below

    {{<figure src="remote-storage.png" alt="Linux App Service WebJob disabled" caption="*Linux App Service WebJob disabled*">}}

- Add a .sh file to every WebJob to start the WebJob, and make sure this file it's included in the dotnet publish output for the WebJob project, despite The Microsoft Learn documentation article saying you don't need an sh file, I couldn't get it to work without it.

    Here's the content of one the .sh file:

    ```sh
    dotnet ./WebJobContinuous.dll
    ```

With these small changes, I successfully deployed my WebApi to the Linux App Service using the Rider right-click deploy (You can see the full GitHub workflow in the repository linked above as well).
As expected, both WebJobs are up and running as shown here below:

{{<figure src="webjobs-code.png" alt=".NET WebJob running on Linux App Service" caption="*.NET WebJob running on Linux App Service*">}}

### Linux container

For the container approach, I'm using the same projects, so I've just added a dockerfile to the WebApi.

Deploying the Linux Container WebApi is quite simple but, due to the fact that we are deploying a container, we need a container registry. Azure provides the Azure Container Registry service so I went ahead and deployed one.

After adding a regular dockerfile to the project, all I had to do was build the docker image, push the docker image to the Azure Container registry and configure App Service to pull and use the image from the Azure Container Registry.

For the WebJobs the deployment needs to be done as a separate step, so this makes it a bit more complicated than the [Linux code](#linux-code) option, essentially before (or after) deploying the container, we need to deploy the webjobs into the app_data/jobs/{job type}/{job name} folder.

After some trial and error approches, I was finally able to deploy the WebJobs using this GitHub Action:

```yml
      - name: Publish WebJobs
        run: |
          dotnet publish -c Release -o ./publish/app_data/Jobs/Triggered/webjob-triggered ./WebJobTriggered/WebJobTriggered.csproj
          dotnet publish -c Release -o ./publish/app_data/Jobs/Continuous/webjob-continuous ./WebJobContinuous/WebJobContinuous.csproj

      - name: Deploy WebJobs
        uses: azure/webapps-deploy@v3
        with: 
          app-name: ${{ env.SITE_NAME }}
          package: './publish'
```

As in for the case above, I had to set the `WEBSITES_ENABLE_APP_SERVICE_STORAGE` environment variables to enable WebJobs and I was able to get everything up and running.

{{<figure src="webjobs-container.png" alt=".NET WebJob running on Linux App Service" caption="*.NET WebJob running on Linux App Service*">}}

## Gotchas

Along the way (or in my previous experiences with WebJobs) I came across several pain points so I decided to mention them here, in no particular order:

- You can't use the WebJob SDK for a triggered WebJob, the process will be killed with error code 137, I couldn't find any examples of a triggered job in the WebJob SDK repo either.
- Running a Docker container in App Service is not as simple/intuitive as it should be, I had several errors like the following:

    `Container <container name> didn't respond to HTTP pings on port: 8080, failing site start. See container logs for debugging.`

- Making sure the container starts is also quite unintuitive, especially if you're configuring the App Service from the Azure Portal, after you've set the image and tag, you also need to also browse to the website url, otherwise the App Service won't even pull the container 🤷‍♂️.
- If you try to configure a not-exitistent image (e.g. because you forgot to push the image to the ACR), it's quite complicated to get the App Service to try to pull the image again after the first failure (I wish there was a button to force an image pull and another to simply start of the container).
- It's not obvious to know when a continuous WebJob is not running/failed to start, unless you check on the WebJobs page, you have no other visual clues in the Azure portal. This is especially tricky if the deployment succeeds but the job fails to start, in this situation the only viable option to ensure the deployment succeeded and the WebJob is running is to check the status e.g. via `az webapp webjob continuous list -n webapp-name -g resource-group-name --query "[?contains(name, 'name of the webjob')].status" -o tsv`, I would rather have my deployment fail instead.
- WebJobs can't expose HTTP endpoints, so they can't implement or participate in health checks, which means that if a WebJob stops working for whatever reason, it's not easy to figure out the WebJob is not running anymore.
- The built-in logs of continuous WebJob are limited, so without some additional work you will only see a very limited amount of log, if you add the `Microsoft.Extensions.Logging.AzureAppServices` and configure the logging option by calling the `AddAzureWebAppDiagnostics();` method, the logging will be mixed between the WebApi and the WebJob. This can be solved by using the ApplicationInsights logger, but ApplicationInsights doesn't show log messages in real time and has a typical delay of several minutes.
- Scaling of WebJobs has little to no control, you can run one or more instances depending on the number of instances of the App Service, you can't scale the WebJob separately from the App Service. This makes sense of course but it also limits the flexibility and therefore the applicability of the technology.
- Another major annoyance, at least for me, is that the team in responsible for the WebJob SDK is not very responsive on the SDK repository. When logging an issue you may or may not get a response from a team member. To prove this point, I decided to use the [issue-metrics](https://github.com/github/issue-metrics) action on the WebJobs SDK repository and the results can be seen in the image below, which is quite underwhelming to be honest.

    {{<figure src="issue-metrics.png" alt="Issue metrics for the Azure/azure-webjobs-sdk over last 12 months" caption="*Issue metrics for the Azure/azure-webjobs-sdk over last 12 months*">}}

As you can sees, some issues are related to App Services running containers, while others are more specific to WebJobs. I have found a fair amount of issues/inconveniences when working with WebJobs.

## Conclusions

As you can see, we were able to configure and run WebJobs with both Linux code and Linux containers. Given the very poor user experience, especially when using the WebJob SDK, I suggest you look at Container Apps with ASP.NET Core and Dapr (if you're wondering why not Azure Functions, it's because I really dislike the Azure Functions SDK and try to avoid it as much as possible :) ).

Using Dapr eliminates the need to learn the WebJob SDK and instead relies on the well-known ASP.NET Core one, Azure Container Apps (or AKS), especially when building messaging applications, allowing you to easily scale based on the message processing throughput, something that just isn't possible to achieve with WebJobs.

If you found this article useful, please leave a thumbs up.

Till the next time!
