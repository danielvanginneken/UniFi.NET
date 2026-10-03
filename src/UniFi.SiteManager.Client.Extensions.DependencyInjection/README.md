# UniFi.SiteManager.Client.Extensions.DependencyInjection

Dependency-injection integration for [UniFi.SiteManager.Client](https://www.nuget.org/packages/UniFi.SiteManager.Client),
registering the client through `IHttpClientFactory` for pooled, long-lived HTTP connections. Part of
the [UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family.

```bash
dotnet add package UniFi.SiteManager.Client.Extensions.DependencyInjection
```

## Usage

```csharp
using Microsoft.Extensions.DependencyInjection;

builder.Services.AddUniFiSiteManagerClient(apiKey);   // API key from unifi.ui.com
// or with options: AddUniFiSiteManagerClient(SiteManagerClientOptions.Create(apiKey))

// then inject it:
public sealed class FleetService(UniFiSiteManagerClient siteManager)
{
    public Task<SiteManagerPage<Host>> HostsAsync() => siteManager.Hosts.ListAsync();
}
```

`AddUniFiSiteManagerClient` returns the `IHttpClientBuilder`, so you can chain resilience/logging
handlers (e.g. Polly).

## Documentation

Full docs and source: **https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
