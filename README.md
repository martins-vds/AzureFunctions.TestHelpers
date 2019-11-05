[![build](https://ci.appveyor.com/api/projects/status/ee41yv4jpp40xj7d?svg=true)](https://ci.appveyor.com/project/riezebosch/azurefunctions-testhelpers/branch/master)
[![nuget](https://img.shields.io/nuget/v/AzureFunctions.TestHelpers.svg)](https://www.nuget.org/packages/AzureFunctions.TestHelpers/)

# AzureFunctions.TestHelpers ⚡

Host and invoke Azure Functions from a test by combining the bits and pieces of
the [WebJobs SDK](https://docs.microsoft.com/en-us/azure/app-service/webjobs-sdk-how-to),
[Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-overview)
and [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-concepts)
and adding some convenience classes and extension methods on top.


## Configure Services for Dependency Injection

```c#
mock = Substitute.For<IInjectable>();
host = new HostBuilder()
    .ConfigureWebJobs(builder => builder
        .UseWebJobsStartup<Startup>()
        .ConfigureServices(services => services.Replace(ServiceDescriptor.Singleton(mock))))
    .Build();
```

Register and replace services that are injected into your functions.
Include `Microsoft.Azure.Functions.Extensions` in your test project to [enable dependency injection](https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection)!

## HTTP Triggered Functions

Invoke a regular http triggered function:

```c#
[Fact]
public static async Task HttpTriggeredFunctionWithDependencyReplacement()
{
    // Arrange
    var mock = Substitute.For<IInjectable>();
    using (var host = new HostBuilder()
        .ConfigureWebJobs(builder => builder
            .AddHttp()
            .ConfigureServices(services => services.AddSingleton(mock)))
        .Build())
    {
        await host.StartAsync();
        var jobs = host.Services.GetService<IJobHost>();

        // Act
        await jobs.CallAsync(nameof(DemoInjection), new Dictionary<string, object>
        {
            ["request"] = new DummyHttpRequest()
        });

        // Assert
        mock
            .Received()
            .Execute();
    }
}
```

### Dummy HTTP Request

Because you can't invoke an HTTP-triggered function without a request you can now use a `DummyHttpRequest`.

```c#
await jobs.CallAsync(nameof(DemoInjection), new Dictionary<string, object>
{
    ["request"] = new DummyHttpRequest()
});
```

You can also set all kinds of regular settings on the request when needed:

```c#
var request = new DummyHttpRequest
{
    Scheme = "http",
    Host = new HostString("dummy"),
    Headers = {["Authorization"] = $"Bearer {token}"}
};
```

### HTTP Response

You capture the result(s) of http-triggered functions via the `options.SetResponse` 
callback on the `AddHttp` extension method:

```c#
// Arrange
object response = null;
using (var host = new HostBuilder()
    .ConfigureWebJobs(builder => builder
        .AddHttp(options => options.SetResponse = (request, o) => response = o))
    .Build())
{
    await host.StartAsync();
    var jobs = host.Services.GetService<IJobHost>();

    // Act
    await jobs.CallAsync(nameof(DemoInjection), new Dictionary<string, object>
    {
        ["request"] = new DummyHttpRequest()
    });
}

// Assert
response
    .Should()
    .BeOfType<OkResult>();
```

## Durable Functions

Invoke a time-triggered durable function:

```c#
[Fact]
public static async Task DurableFunction()
{
    // Arrange
    var mock = Substitute.For<IInjectable>();
    using (var host = new HostBuilder()
        .ConfigureWebJobs(builder => builder
            .AddTimers()
            .AddDurableTaskInTestHub()
            .AddAzureStorageCoreServices()
            .ConfigureServices(services => services.AddSingleton(mock)))
        .Build())
    {
        await host.StartAsync();
        var jobs = host.Services.GetService<IJobHost>();

        // Act
        await jobs.CallAsync(nameof(Starter), new Dictionary<string, object>
        {
            ["timerInfo"] = new TimerInfo(new WeeklySchedule(), new ScheduleStatus())
        });

        await jobs
            .Ready()
            .ThrowIfFailed()
            .Purge();

        // Assert
        mock
            .Received()
            .Execute();
    }
}
```

You'll have to [configure Azure WebJobs Storage](#azure-storage-account) to be able to run durable functions!

### Add Durable Task in TestHub

Add and configure using [the durable task extensions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-webjobs-sdk#webjobs-sdk-3x) and
use a random generated hub name to start with a clean history.

```c#
host = new HostBuilder()
    .ConfigureWebJobs(builder => builder
        .AddDurableTaskInTestHub()
        .AddDurableTaskInTestHub(options => options.MaxQueuePollingInterval = TimeSpan.FromSeconds(2))
        .AddAzureStorageCoreServices()
    .Build();
```

## Managing Durable Tasks

```c#
await jobs
    .Ready(TimeSpan.FromSeconds(30))
    .ThrowIfFailed()
    .Purge();
```

*BREAKING:* In `v2` the `WaitForOrchestrationsCompletion` is replaced with `Wait().ThrowIfFailed().Purge()`.

### Ready

Invoke function to monitor orchestrations status and wait for all to complete.

### Throw if Failed
 
Invoke function to validate if all orchestrations succeeded.

### Purge

Invoke function to purge the history of current execution.

## Azure Storage Account

You need an azure storage table to store the state of the durable functions.
The only two options currenty are [Azure](#option-1-azure) and the [Azure Storage Emulator](#option-2-azure-storage-emulator).

### Option 1: Azure

Just copy the [connection string from your storage account](https://docs.microsoft.com/en-us/azure/storage/common/storage-configure-connection-string#view-and-copy-a-connection-string),
works everywhere.

### Option 2: Azure Storage Emulator

Set the connection string to `UseDevelopmentStorage=true`. Unfortunately works only on Windows. See this [blog](https://zimmergren.net/azure-devops-unit-tests-storage-emulator-hosted-agent/)
on how to enable the storage emulator in an Azure DevOps pipeline.

### Option 3: Azurite

Unfortunately, `azurite@v2` doesn't work with the current version of durable functions,
and `azurite@v3` doesn't have the [required features (implemented yet)](https://github.com/Azure/Azurite#azurite-v3).

### Set the Storage Connection String

The storage connection string setting [is required](https://docs.microsoft.com/en-us/azure/app-service/webjobs-sdk-how-to#host-connection-strings).

#### Option 1: with an environment variable

Set the environment variable `AzureWebJobsStorage`. Hereby you can also overwrite the configured connection from [option 2](#option-2-from-a-configuration-file) on your local dev machine.

#### Option 2: from a configuration file

Include an `appsettings.json` in your test project:

```json
{
  "AzureWebJobsStorage": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...==;EndpointSuffix=core.windows.net"
}
```

and make sure it is copied to the output directory:

```xml
<ItemGroup>
    <None Include="appsettings.json">
        <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
</ItemGroup>
```