# UniFi.Network.Client

A .NET client for the [UniFi Network API](https://developer.ui.com/network/), targeting either a
local console directly or the Site Manager cloud connector — no VPN required. Part of the
[UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family. Zero third-party dependencies.

```bash
dotnet add package UniFi.Network.Client
```

## Quick start

```csharp
using UniFi.Network.Client;

// Local console (LAN). Consoles use a self-signed cert — pin it rather than disabling validation:
var options = UniFiClientOptions.ForLocalConsole(
    "192.168.1.1", apiKey,
    allowUntrustedCertificate: false,
    pinnedCertificateSha256: "AB:CD:…");   // SHA-256 thumbprint of the console cert
// or remotely, without VPN:
// var options = UniFiClientOptions.ForCloudConnector(consoleId, apiKey);

using var client = new UniFiNetworkClient(options);

var sites = await client.Sites.ListAsync();
foreach (var site in sites.Data)
{
    var devices = await client.Devices.ListAsync(site.Id);
    foreach (var device in devices.Data)
        Console.WriteLine($"{device.Name} [{device.Model}] {device.State}");
}
```

Covers sites, devices, clients, networks, WiFi, vouchers, firewall/ACL/DNS policies, and more.
List endpoints return a `PagedResult<T>` (offset/limit). Deep, frequently-extended resources read as
`JsonElement` and write as `JsonObject`; everything else is fully typed. Non-2xx responses throw
`UniFi.Network.Client.Http.UniFiApiException`.

Generate an API key at [unifi.ui.com](https://unifi.ui.com) or on your console.

## Documentation

Full docs, other clients (Protect, Site Manager, Mobility), and source:
**https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
