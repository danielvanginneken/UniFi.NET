# UniFi.Mobility.Client

A .NET client for the [UniFi Mobility API](https://developer.ui.com/mobility/) — the account-wide
cloud API (`api.ui.com`) for mobile routing devices (UMR): workspaces, devices, clients, and device
configuration. Cloud-only. Part of the [UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family.
Zero third-party dependencies.

```bash
dotnet add package UniFi.Mobility.Client
```

## Quick start

```csharp
using UniFi.Mobility.Client;

using var client = new UniFiMobilityClient(apiKey);   // API key with the `mobility` scope

var workspaces = await client.Workspaces.ListAsync();
foreach (var workspace in workspaces.Data)
{
    var devices = await client.Devices.ListAsync(workspace.WorkspaceId);
    foreach (var device in devices.Data)
        Console.WriteLine($"{device.Name} [{device.Model}] {device.State}");
}
```

Resources: `Workspaces` (list + admins) and `Devices` (list, detail, clients, and
`UpdateName`/`UpdateNetwork`/`UpdateWireless` — the writes need a `write:mobility` key). Fully typed,
including device detail (WAN, cellular, WiFi, VPN, subscription, GPS). List endpoints return a
`MobilityPage<T>` (offset/limit). Non-2xx responses throw
`UniFi.Mobility.Client.Http.UniFiMobilityException`. Rate limited to 100 requests/minute per key.

## Documentation

Full docs, other clients (Network, Protect, Site Manager), and source:
**https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
