# UniFi.Mobility.Client.Extensions.DependencyInjection

Dependency-injection integration for [UniFi.Mobility.Client](https://www.nuget.org/packages/UniFi.Mobility.Client),
registering the client through `IHttpClientFactory` for pooled, long-lived HTTP connections. Part of
the [UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family.

```bash
dotnet add package UniFi.Mobility.Client.Extensions.DependencyInjection
```

## Usage

```csharp
using Microsoft.Extensions.DependencyInjection;

builder.Services.AddUniFiMobilityClient(apiKey);   // API key with the `mobility` scope
// or with options: AddUniFiMobilityClient(MobilityClientOptions.Create(apiKey))

// then inject it:
public sealed class RouterService(UniFiMobilityClient mobility)
{
    public Task<MobilityPage<WorkspaceSummary>> WorkspacesAsync() => mobility.Workspaces.ListAsync();
}
```

`AddUniFiMobilityClient` returns the `IHttpClientBuilder`, so you can chain resilience/logging
handlers (e.g. Polly).

## Documentation

Full docs and source: **https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
