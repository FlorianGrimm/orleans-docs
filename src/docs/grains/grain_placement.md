---
title: Grain Placement
---

# Grain Placement

Orleans ensures that when a grain call is made there is an instance of that grain available in memory on some server in the cluster to handle the request.
If the grain is not currently active in the cluster, Orleans picks one of the servers to activate the grain on.
This is called grain placement.
Placement is also one way that load is balanced: even placement of busy grains helps to even the workload across the cluster.

The placement process in Orleans is fully configurable: developers can choose from a set of out-of-the-box placement policies such as random, prefer-local, and load-based, or custom logic can be configured.
This allows for full flexibility in deciding where grains are created.
For example, grains can be placed on a server close to resources which they need to operate on or close to other grains which they communicate with.
By default, Orleans will pick a random compatible server.

The placement strategy which Orleans uses can be configured globally or per-grain-class.

## In-built placement strategies

## Random placement

A server is randomly selected from the set of compatible servers.

This placement strategy is configured by adding the `[RandomPlacement]` attribute to a grain.

## Local placement

If the local server is compatible, select the local server, otherwise select a random server.

This placement strategy is configured by adding the `[PreferLocalPlacement]` attribute to a grain.

## Hash-based placement

Hash the grain id to a non-negative integer and modulo it with the number of compatible servers.
Select the corresponding server from list of compatible servers ordered by server address.
Note that this is not guaranteed to remain stable as the cluster membership changes.
Specifically, adding, removing, or restarting servers can alter the server selected for a given grain id.
Because grains placed using this strategy are registered in the grain directory, this change in placement decision as membership changes typically does not have a noticeable effect.

This placement strategy is configured by adding the `[HashBasedPlacement]` attribute to a grain.

## Activation-count-based placement

The intention of this placement strategy is to place new grain activations on the least heavily loaded server based on the number of recently busy grains.
It includes a mechanism in which all servers periodically publish their total activation count to all other servers.
The placement director then selects a server which is predicted to have the fewest activations by examining the most recently reported activation count and a making prediction of the current activation count based upon the recent activation count made by the placement director on the current server.
The director selects a number of servers at random when making this prediction, in an attempt to avoid multiple separate servers overloading the same server.
By default, two servers are selected at random, but this value is configurable via `ActivationCountBasedPlacementOptions`.

This algorithm is based on the thesis [*The Power of Two Choices in Randomized Load Balancing* by Michael David Mitzenmacher](https://www.eecs.harvard.edu/~michaelm/postscripts/mythesis.pdf), and is also used in NGINX for distributed load balancing, as described in the article [*NGINX and the "Power of Two Choices" Load-Balancing Algorithm*](https://www.nginx.com/blog/nginx-power-of-two-choices-load-balancing-algorithm/).

This placement strategy is configured by adding the `[ActivationCountBasedPlacement]` attribute to a grain.


## Stateless worker placement

This is a special placement strategy used by [*stateless worker* grains](~/docs/grains/stateless_worker_grains.md).
This operates almost identically to `PreferLocalPlacement` except that each server can have multiple activations of the same grain and the grain is not registered in the grain directory since there is no need.

This placement strategy is configured by adding the `[StatelessWorker]` attribute to a grain.

## Configuring the default placement strategy

Orleans will use random placement unless the default is overridden.
The default placement strategy can be overridden by registering an implementation of `PlacementStrategy` during configuration:

``` C#
siloBuilder.ConfigureServices(services =>
  services.AddSingleton<PlacementStrategy, MyPlacementStrategy>());
```

## Configuring the placement strategy for a grain

The placment strategy for a grain type is configured by adding the appropriate attribute on the grain class.
The relevant attributes are specified in the [in-built placement strategies](#in-built-placement-strategies) section.

## Sample custom placement strategy

First define a class which implements `IPlacementDirector` interface, requiring a single method.
In this example we assume you have a function `GetSiloNumber` defined which will return a silo number given the guid of the grain about to be created.

``` csharp
public class SamplePlacementStrategyFixedSiloDirector : IPlacementDirector
{

    public Task<SiloAddress> OnAddActivation(PlacementStrategy strategy, PlacementTarget target, IPlacementContext context)
    {
        var silos = context.GetCompatibleSilos(target).OrderBy(s => s).ToArray();
        int silo = GetSiloNumber(target.GrainIdentity.PrimaryKey, silos.Length);
        return Task.FromResult(silos[silo]);
    }
}
```
You then need to define two classes to allow grain classes to be assigned to the strategy:

```csharp
[Serializable]
public class SamplePlacementStrategy : PlacementStrategy
{
}

[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public sealed class SamplePlacementStrategyAttribute : PlacementAttribute
{
    public SamplePlacementStrategyAttribute() :
        base(new SamplePlacementStrategy())
        {
        }
}
```
Then just tag any grain classes you want to use this strategy with the attribute:
``` csharp
[SamplePlacementStrategy]
public class MyGrain : Grain, IMyGrain
{
    ...
}
```
And finally register the strategy when you build the SiloHost:
``` csharp
private static async Task<ISiloHost> StartSilo()
{
    ISiloHostBuilder builder = new SiloHostBuilder()
        // normal configuration methods omitted for brevity
        .ConfigureServices(ConfigureServices);

    var host = builder.Build();
    await host.StartAsync();
    return host;
}


private static void ConfigureServices(IServiceCollection services)
{
    services.AddSingletonNamedService<PlacementStrategy, SamplePlacementStrategy>(nameof(SamplePlacementStrategy));
    services.AddSingletonKeyedService<Type, IPlacementDirector, SamplePlacementStrategyFixedSiloDirector>(typeof(SamplePlacementStrategy));
}
```

For a second simple example showing further use of the placement context, refer to the `PreferLocalPlacementDirector` in the [Orleans source repo](https://github.com/dotnet/orleans/blob/master/src/Orleans.Runtime/Placement/PreferLocalPlacementDirector.cs)
