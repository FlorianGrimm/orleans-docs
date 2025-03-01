---
title: Clients
---

# Clients

A client allows non-grain code to interact with an Orleans cluster.
Clients allow application code to communicate with grains and streams hosted in a cluster.
There are two ways to obtain a client, depending on where the client code is hosted: in the same process as a silo, or in a separate process.
This article will discuss both options, starting with the recommended option: co-hosting the client code in the same process as the grain code.

## Co-hosted clients

If the client code is hosted in the same process as the grain code, then the client can be directly obtained from the hosting application's dependency injection container.
In this case, the client communicates directly with the silo it is attached to and can take advantage of the extra knowledge that the silo has about the cluster.

This provides several benefits, including reducing network and CPU overhead as well as decreasing latency and increasing throughput and reliability.
The client utilizes the silo's knowledge of the cluster topology and state and does not need to use a separate gateway.
This avoids a network hop and serialization/deserialization round trip.
This therefore also increases reliability, since the number of required nodes in between the client and the grain is minimized.
If the grain is a [stateless worker grain](~/docs/grains/stateless_worker_grains.md) or otherwise happens to be activated on the silo which the client is hosted in, then no serialization or network communication needs to be performed at all and the client can reap additional performance and reliability gains.
Co-hosting client and grain code also simplifies deployment and application topology by eliminating the need for two distinct application binaries to be deployed and monitored.

There are also detractors to this approach, primarily that the grain code is no longer isolated from the client process.
Therefore, issues in client code, such as blocking IO or lock contention causing thread starvation can affect the performance of grain code.
Even without code defects like the abovementioned, *noisy neighbor* effects can result simply by having the client code execute on the same processor as grain code, putting additional strain on CPU cache and additional contention for local resources in general.
Additionally, identifying the source of these issues is now more difficult because monitoring systems cannot distinguish what is logically client code from grain code.

Despite these detractors, co-hosting client code with grain code is a popular option and the recommended approach for most applications.
To elaborate, the abovementioned detractors are minimal in practice for the following reasons:

* Client code is often very *thin*, for example translating incoming HTTP requests into grain calls, and therefore the *noisy neighbor* effects are minimal and comparable in cost to the otherwise required gateway.
* In the event that a performance issue arises, the typical workflow for a developer involves tools such as CPU profilers and debuggers, which are still effective in quickly identifying the source of the issue despite having both client and grain code executing in the same process. In other words, metrics become more coarse and less able to precisely identify the source of an issue, but more detailed tools are still effective.

### Obtaining a client from a host

If hosting using the [.NET Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host), the client will be available in the host's [dependency injection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) container automatically and can be injected into services such as [ASP.NET controllers](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/actions) or [`IHostedService`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice) implementations.

Alternatively, a client interface such as `IGrainFactory` or `IClusterClient` can be obtained from either `IHost` or `ISiloHost`:

``` C#
var client = host.Services.GetService<IClusterClient>();
await client.GetGrain<IMyGrain>(0).Ping();
```

## External clients

Client code can run outside of the Orleans cluster where grain code is hosted.
Hence, an external client acts as a connector or conduit to the cluster and to all grains of the application.

![](~/images/frontend_cluster.png)

Usually, clients are used on the frontend web servers to connect to an Orleans cluster that serves as a middle tier with grains executing business logic.
In a typical setup, a frontend web server:
* Receives a web request
* Performs necessary authentication and authorization validation
* Decides which grain(s) should process the request
* Uses Grain Client to make one or more method call to the grain(s)
* Handles successful completion or failures of the grain calls and any returned values
* Sends a response for the web request

### Initialization of Grain Client

Before a grain client can be used for making calls to grains hosted in an Orleans cluster, it needs to be configured, initialized, and connected to the cluster.

Configuration is provided via  `ClientBuilder` and a number of supplemental option classes that contain a hierarchy of configuration properties for programmatically configuring a client.

More information can be in the [Client Configuration guide](configuration_guide/client_configuration.md).

Example of a client configuration:

```csharp

var client = new ClientBuilder()
    // Clustering information
    .Configure<ClusterOptions>(options =>
    {
        options.ClusterId = "my-first-cluster";
        options.ServiceId = "MyOrleansService";
    })
    // Clustering provider
    .UseAzureStorageClustering(options => options.ConnectionString = connectionString)
    // Application parts: just reference one of the grain interfaces that we use
    .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(IValueGrain).Assembly))
    .Build();

```

Lastly, we need to call `Connect()` method on the constructed client object to make it connect to the Orleans cluster. It's an asynchronous method that returns a `Task`. So we need to wait for its completion with an `await` or `.Wait()`.

```csharp

await client.Connect();

```

### Making Calls to Grains

Making calls to grain from a client is really no different from [making such calls from within grain code](~/docs/grains/index.md).
The same `GetGrain<T>(key)` method, where `T` is the target grain interface, is used in both cases [to obtain grain references](~/docs/grains/index.md#grain-reference).
The slight difference is in through what factory object we invoke `GetGrain`.
In client code we do that through the connected client object.

``` csharp
IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
Task t = player.JoinGame(game)
await t;
```

A call to a grain method returns a `Task` or a`Task<T>` as required by the [grain interface rules](~/docs/grains/index.md).
The client can use the `await` keyword to asynchronously await the returned `Task` without blocking the thread, or in some cases the `Wait()` method to block the current thread of execution.

The major difference between making calls to grains from client code and from within another grain is the single-threaded execution model of grains.
Grains are constrained to be single-threaded by the Orleans runtime, while clients may be multi-threaded.
Orleans does not provide any such guarantee on the client side, and so it is up to the client to manage its own concurrency using whatever synchronization constructs are appropriate for its environment – locks, events, `Tasks`, etc.

### Receiving Notifications

There are situations in which a simple request-response pattern is not enough, and the client needs to receive asynchronous notifications.
For example, a user might want to be notified when a new message has been published by someone that she is following.

[Observers](~/docs/grains/observers.md) is one such mechanism that enables exposing client side objects as grain-like targets to get invoked by grains.
Calls to observers do not provide any indication of success or failure, as they are sent as one-way best effort message.
So it is a responsibility of the application code to build a higher level reliability mechanism on top of observers where necessary. 

Another mechanism that can be used for delivering asynchronous messages to clients is [Streams](~/docs/streaming/index.md). Streams expose indications of success or failure of delivery of individual messages, and hence enable reliable communication back to the client.

### Client Connectivity

There are two scenarios in which a cluster client can experience connectivity issues:

* When the `IClusterClient.Connect` method is called initially.
* When making calls on grain references which were obtained from a connected cluster client.

In the first case, the `Connect` method will throw an exception to indicate what went wrong. This is typically (but not necessarily) a `SiloUnavailableException`. If this happens, the cluster client instance is unusable and should be disposed. A retry filter function can optionally be provided to the `Connect` method which could, for instance, wait for a specified duration before making another attempt. If no retry filter is provided, or if the retry filter returns `false`, the client gives up for good.

If `Connect` returns successfully, the cluster client is guaranteed to be usable until it is disposed. This means that even if the client experiences connection issues, it will attempt to recover indefinitely. The exact recovery behavior can be configured on a `GatewayOptions` object provided by the `ClientBuilder`, e.g.:

```csharp
var client = new ClientBuilder()
    // ...
    .Configure<GatewayOptions>(opts => GatewayListRefreshPeriod = TimeSpan.FromMinutes(10)) // Default is 1 min.
    .Build();
```

In the second case, where a connection issue occurs during a grain call, a `SiloUnavailableException` will be thrown on the client side. This could be handled like so:

``` csharp
IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);

try
{
    await player.JoinGame(game);
}
catch (SiloUnavailableException)
{
    // Lost connection to the cluster...
}
```

The grain reference is not invalidated in this situation; the call could be retried on the same reference later, when a connection might have been re-established.

### Dependency Injection

The recommended way to create an external client in a program that uses the .NET Generic Host is to inject an `IClusterClient` singleton instance via dependency injection, which can then be accepted as a constructor parameter in hosted services, ASP.NET controllers, etc.

[!NOTE]
When co-hosting an Orleans silo in the same process that will be connecting to it, it is *not* necessary to manually create a client; Orleans will automatically provide one and manage its lifetime appropriately.

When connecting to a cluster in a different process (e.g. on a different machine), a common pattern is to create a hosted service like this:

```csharp
public class ClusterClientHostedService : IHostedService
{
    public IClusterClient Client { get; }

    public ClusterClientHostedService(ILoggerProvider loggerProvider)
    {
        Client = new ClientBuilder()
            // Appropriate client configuration here, e.g.:
            .UseLocalhostClustering()
            .ConfigureLogging(builder => builder.AddProvider(loggerProvider))
            .Build();
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // A retry filter could be provided here.
        await Client.Connect();
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await Client.Close();

        Client.Dispose();
    }
}
```

The service is then registered like this:

```csharp
public class Program
{
    static Task Main()
    {
        return new HostBuilder()
            .ConfigureServices(services =>
            {
                services.AddSingleton<ClusterClientHostedService>();
                services.AddSingleton<IHostedService>(sp => sp.GetService<ClusterClientHostedService>());
                services.AddSingleton<IClusterClient>(sp => sp.GetService<ClusterClientHostedService>().Client);
                services.AddSingleton<IGrainFactory>(sp => sp.GetService<ClusterClientHostedService>().Client);
            })
            .ConfigureLogging(builder => builder.AddConsole())
            .RunConsoleAsync();
    }
}
```

At this point, an `IClusterClient` instance could be consumed anywhere that dependency injection is supported, such as in an ASP.NET controller:

```csharp
public class HomeController : Controller
{
    readonly IClusterClient _client;

    public HomeController(IClusterClient client) => _client = client;

    public IActionResult Index()
    {
        var grain = _client.GetGrain<IMyGrain>();
        var model = grain.GetModel();

        return View(model);
    }
}
```

### Example

Here is an extended version of the example given above of a client application that connects to Orleans, finds the player account, subscribes for updates to the game session the player is part of with an observer, and prints out notifications until the program is manually terminated.

```csharp
namespace PlayerWatcher
{
    class Program
    {
        /// <summary>
        /// Simulates a companion application that connects to the game
        /// that a particular player is currently part of, and subscribes
        /// to receive live notifications about its progress.
        /// </summary>
        static void Main(string[] args)
        {
            RunWatcher().Wait();
            // Block main thread so that the process doesn't exit.
            // Updates arrive on thread pool threads.
            Console.ReadLine();
        }

        static async Task RunWatcher()
        {
            try

            {
            var client = new ClientBuilder()
                // Clustering information
                .Configure<ClusterOptions>(options =>
                {
                    options.ClusterId = "my-first-cluster";
                    options.ServiceId = "MyOrleansService";
                })
                // Clustering provider
                .UseAzureStorageClustering(options => options.ConnectionString = connectionString)
                // Application parts: just reference one of the grain interfaces that we use
                .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(IValueGrain).Assembly))
                .Build();

                // Hardcoded player ID
                Guid playerId = new Guid("{2349992C-860A-4EDA-9590-000000000006}");
                IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
                IGameGrain game = null;

                while (game == null)
                {
                    Console.WriteLine("Getting current game for player {0}...", playerId);

                    try
                    {
                        game = await player.GetCurrentGame();
                        if (game == null) // Wait until the player joins a game
                        {
                            await Task.Delay(5000);
                        }
                    }
                    catch (Exception exc)
                    {
                        Console.WriteLine("Exception: ", exc.GetBaseException());
                    }
                }

                Console.WriteLine("Subscribing to updates for game {0}...", game.GetPrimaryKey());

                // Subscribe for updates
                var watcher = new GameObserver();
                await game.SubscribeForGameUpdates(
                    await client.CreateObjectReference<IGameObserver>(watcher));

                Console.WriteLine("Subscribed successfully. Press <Enter> to stop.");
            }
            catch (Exception exc)
            {
                Console.WriteLine("Unexpected Error: {0}", exc.GetBaseException());
            }
        }
    }

    /// <summary>
    /// Observer class that implements the observer interface. Need to pass a grain reference to an instance of this class to subscribe for updates.
    /// </summary>
    class GameObserver : IGameObserver
    {
        // Receive updates
        public void UpdateGameScore(string score)
        {
            Console.WriteLine("New game score: {0}", score);
        }
    }
    }
}
```
