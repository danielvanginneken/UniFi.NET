# UniFi.SiteManager.Client

A .NET client for the [UniFi Site Manager API](https://developer.ui.com/site-manager/) — the
account-wide cloud API (`api.ui.com`) spanning every console on your account: hosts, sites, devices,
ISP metrics, and SD-WAN configs. Cloud-only. Part of the
[UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family. Zero third-party dependencies.

```bash
dotnet add package UniFi.SiteManager.Client
```

## Quick start

```csharp
using UniFi.SiteManager.Client;

using var client = new UniFiSiteManagerClient(apiKey);   // API key from unifi.ui.com

var hosts = await client.Hosts.ListAsync(pageSize: 50);
foreach (var host in hosts.Data)
    Console.WriteLine($"{host.Id} [{host.Type}] {host.IpAddress}");

var sites = await client.Sites.ListAsync();
var deviceGroups = await client.Devices.ListAsync(hostIds: new[] { "host-id" });
var metrics = await client.IspMetrics.GetAsync("5m", duration: "24h");
var sdwan = await client.SdWanConfigs.ListAsync();
```

Resources: `Hosts`, `Sites`, `Devices`, `IspMetrics`, `SdWanConfigs`. List endpoints use cursor
pagination (`SiteManagerPage<T>` with `NextToken`/`HasMore`). Version-dependent blobs are exposed as
`JsonElement`. Non-2xx responses throw `UniFi.SiteManager.Client.Http.UniFiSiteManagerException`.

## Documentation

Full docs, other clients (Network, Protect, Mobility), and source:
**https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
