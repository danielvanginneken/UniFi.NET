# UniFi.Protect.Client.Extensions.DependencyInjection

Dependency-injection integration for [UniFi.Protect.Client](https://www.nuget.org/packages/UniFi.Protect.Client),
registering the client through `IHttpClientFactory` for pooled, long-lived HTTP connections. Part of
the [UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family.

```bash
dotnet add package UniFi.Protect.Client.Extensions.DependencyInjection
```

## Usage

```csharp
using Microsoft.Extensions.DependencyInjection;
using UniFi.Protect.Client;

builder.Services.AddUniFiProtectClient(
    ProtectClientOptions.ForLocalConsole("192.168.1.1", apiKey,
        allowUntrustedCertificate: false, pinnedCertificateSha256: "AB:CD:…"));
// or: ProtectClientOptions.ForCloudConnector(consoleId, apiKey)

// then inject it:
public sealed class CameraService(UniFiProtectClient protect)
{
    public Task<byte[]> SnapshotAsync(string id) => protect.Cameras.GetSnapshotAsync(id);
}
```

`AddUniFiProtectClient` returns the `IHttpClientBuilder`, so you can chain resilience/logging
handlers (e.g. Polly). TLS certificate pinning and trust settings from the options are applied to the
underlying primary handler, matching the standalone client.

## Documentation

Full docs and source: **https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
