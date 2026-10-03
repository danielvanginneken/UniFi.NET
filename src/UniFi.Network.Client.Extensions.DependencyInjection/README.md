# UniFi.Network.Client.Extensions.DependencyInjection

Dependency-injection integration for [UniFi.Network.Client](https://www.nuget.org/packages/UniFi.Network.Client),
registering the client through `IHttpClientFactory` for pooled, long-lived HTTP connections. Part of
the [UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family.

```bash
dotnet add package UniFi.Network.Client.Extensions.DependencyInjection
```

## Usage

```csharp
using Microsoft.Extensions.DependencyInjection;
using UniFi.Network.Client;

// e.g. in Program.cs
builder.Services.AddUniFiNetworkClient(
    UniFiClientOptions.ForLocalConsole("192.168.1.1", apiKey,
        allowUntrustedCertificate: false, pinnedCertificateSha256: "AB:CD:…"));

// or resolve options from configuration / other services:
builder.Services.AddUniFiNetworkClient(sp =>
{
    var cfg = sp.GetRequiredService<IConfiguration>();
    return UniFiClientOptions.ForCloudConnector(cfg["UniFi:ConsoleId"]!, cfg["UniFi:ApiKey"]!);
});
```

Then just inject it:

```csharp
public sealed class GuestService(UniFiNetworkClient unifi)
{
    public Task<...> ListClientsAsync(Guid siteId) => unifi.Clients.ListAsync(siteId);
}
```

`AddUniFiNetworkClient` returns the `IHttpClientBuilder`, so you can chain resilience/logging
handlers (e.g. Polly). TLS certificate pinning and trust settings from the options are applied to the
underlying primary handler, matching the standalone client.

## Documentation

Full docs and source: **https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
