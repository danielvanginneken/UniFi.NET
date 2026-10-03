# UniFi.Protect.Client

A .NET client for the [UniFi Protect API](https://developer.ui.com/protect/) — cameras, sensors,
lights, sirens, and the rest of the Protect device fleet — over both a local console and the Site
Manager cloud connector. Part of the [UniFi.NET](https://github.com/danielvanginneken/UniFi.NET) family.
Zero third-party dependencies.

```bash
dotnet add package UniFi.Protect.Client
```

## Quick start

```csharp
using UniFi.Protect.Client;

var options = ProtectClientOptions.ForLocalConsole("192.168.1.1", apiKey,
    allowUntrustedCertificate: false, pinnedCertificateSha256: "AB:CD:…");
// or: ProtectClientOptions.ForCloudConnector(consoleId, apiKey);
using var client = new UniFiProtectClient(options);

var info = await client.Meta.GetInfoAsync();
Console.WriteLine($"Protect {info.ApplicationVersion}");

foreach (var camera in await client.Cameras.ListAsync())
    Console.WriteLine($"{camera.Name} [{camera.ModelKey}] {camera.State}");

var snapshot = await client.Cameras.GetSnapshotAsync(cameraId, highQuality: true); // byte[]
```

Resources cover the full fleet: `Cameras`, `Lights`, `Sensors`, `Chimes`, `Viewers`, `Bridges`,
`Fobs`, `Nvrs`, `Speakers`, `Sirens`, `Relays`, `LinkStations`, `AlarmHubs`, `LiveViews`,
`ArmProfiles`, `Users`, `UlpUsers`, `Files`, `AlarmManager`, and `Subscriptions` — plus camera
actions (RTSPS streams, snapshots, talkback, PTZ) and siren/relay/alarm-hub control. Identity fields
are strongly typed; deep settings blobs read as `JsonElement` and update via a `JsonObject` PATCH.
`Subscriptions` streams the WebSocket feeds as `IAsyncEnumerable<JsonElement>`. Non-2xx responses
throw `UniFi.Protect.Client.Http.UniFiProtectException`.

Generate an API key at [unifi.ui.com](https://unifi.ui.com) or on your console.

## Documentation

Full docs and source: **https://github.com/danielvanginneken/UniFi.NET**

MIT licensed.
