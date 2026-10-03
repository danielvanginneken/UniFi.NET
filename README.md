# UniFi.NET

A .NET client for the [UniFi Network API](https://developer.ui.com/network/) (v10.4.57), targeting
either a local console directly or the Site Manager cloud connector — no VPN required.

It also includes a client for the [UniFi Site Manager API](https://developer.ui.com/site-manager/)
(`api.ui.com`) — the account-wide cloud API for listing hosts, sites, devices, ISP metrics, and
SD-WAN configs across all your consoles. See [Site Manager API](#site-manager-api) below.

It also includes a client for the [UniFi Protect API](https://developer.ui.com/protect/) — cameras,
sensors, lights, and the rest of the Protect device fleet, over both local console and cloud
connector. See [Protect API](#protect-api) below.

And a client for the [UniFi Mobility API](https://developer.ui.com/mobility/) — workspaces and
mobile routing devices (UMR), cloud-only. See [Mobility API](#mobility-api) below.

This repo is meant to grow into a family of UniFi .NET clients (`UniFi.Network.Client`,
`UniFi.SiteManager.Client`, `UniFi.Protect.Client`, and `UniFi.Mobility.Client` today,
`UniFi.Access.Client` potentially later), so each product's client lives under its own namespace
and project.

## Install

```bash
dotnet add package UniFi.Network.Client
dotnet add package UniFi.SiteManager.Client
dotnet add package UniFi.Protect.Client
dotnet add package UniFi.Mobility.Client
```

All four clients are published on [NuGet.org](https://www.nuget.org/packages?q=UniFi.NET). They're
versioned independently — each tracks its upstream API version: `UniFi.Network.Client` **10.4.57**,
`UniFi.Protect.Client` **7.1.87**, `UniFi.SiteManager.Client` **1.0.0**, `UniFi.Mobility.Client`
**1.0.0**.

Each client also has an optional DI integration package,
`UniFi.<Product>.Client.Extensions.DependencyInjection`, for `IHttpClientFactory`-based registration
(see [Dependency injection](#dependency-injection)). They're in the repo and release on their next
tag — not yet on NuGet at the time of writing.

## Usage

Generate an API key at [unifi.ui.com](https://unifi.ui.com) or on your console.

### Local console

Talk directly to a console on your LAN (e.g. a UDM or Cloud Gateway). Local consoles serve a
self-signed certificate, so TLS validation is relaxed by default (`allowUntrustedCertificate: true`).

> **Security:** disabling validation exposes the API key to man-in-the-middle attacks on the local
> network. For anything beyond local testing, **pin the console's certificate** instead — it works
> with self-signed certs but still authenticates the peer:
>
> ```csharp
> var options = UniFiClientOptions.ForLocalConsole(
>     "192.168.1.1", apiKey,
>     allowUntrustedCertificate: false,
>     pinnedCertificateSha256: "AB:CD:…");   // SHA-256 thumbprint of the console cert
> ```
>
> The same `pinnedCertificateSha256` parameter is available on `ProtectClientOptions.ForLocalConsole`
> (and applies to Protect's WebSocket subscriptions too).

```csharp
using UniFi.Network.Client;

var options = UniFiClientOptions.ForLocalConsole("192.168.1.1", apiKey);
using var client = new UniFiNetworkClient(options);

var sites = await client.Sites.ListAsync();
foreach (var site in sites.Data)
{
    var devices = await client.Devices.ListAsync(site.Id);
    foreach (var device in devices.Data)
    {
        Console.WriteLine($"{device.Name} [{device.Model}] {device.State}");
    }
}
```

### Site Manager cloud connector

Reach a console remotely through `api.ui.com`, without VPN. You'll need the console's id, as
shown in Site Manager.

```csharp
var options = UniFiClientOptions.ForCloudConnector(consoleId, apiKey);
using var client = new UniFiNetworkClient(options);
```

### Pagination

List endpoints return a `PagedResult<T>`: `Data`, `Count`, `Limit`, `Offset`, `TotalCount`, and a
`HasMore` helper.

```csharp
var clients = await client.Clients.ListAsync(siteId, offset: 0, limit: 200);
while (true)
{
    foreach (var c in clients.Data) { /* ... */ }
    if (!clients.HasMore) break;
    clients = await client.Clients.ListAsync(siteId, offset: (int)(clients.Offset + clients.Count), limit: 200);
}
```

Most list endpoints also accept a `filter` expression per their documented filterable properties,
e.g. `client.Devices.ListAsync(siteId, filter: "state eq 'ONLINE'")`.

### Polymorphic resources

A handful of resources are typed discriminated unions using `System.Text.Json` polymorphism —
pattern-match to get at type-specific fields:

```csharp
var clients = await client.Clients.ListAsync(siteId);
foreach (var c in clients.Data)
{
    if (c is WiredClientOverview wired)
        Console.WriteLine($"{wired.Name} wired via {wired.UplinkDeviceId}, mac {wired.MacAddress}");
    else if (c is WirelessClientOverview wireless)
        Console.WriteLine($"{wireless.Name} wireless via {wireless.UplinkDeviceId}");
}
```

### Deep/flexible resources

Some resources — firewall policies, firewall zones, ACL rules, DNS policies, traffic matching
lists, WiFi broadcast security config, network DHCP/IPv4/IPv6 config — are themselves deep,
frequently-extended discriminated unions in the underlying API (dozens of nested DTOs). Modeling
every variant 1:1 would mean constant upkeep chasing new filter/protocol types the API adds.
Instead, these resources read as `System.Text.Json.JsonElement` and write as
`System.Text.Json.Nodes.JsonObject`, built directly against the
[OpenAPI spec](https://developer.ui.com/network/v10.4.57/openapi.json):

```csharp
using System.Text.Json.Nodes;

var zone = new JsonObject
{
    ["name"] = "IoT",
    ["networkIds"] = new JsonArray(networkId.ToString()),
};
var created = await client.FirewallZones.CreateAsync(siteId, zone);
```

Everything else — sites, devices, clients, networks (overview), WiFi broadcasts (overview),
vouchers, reference data — is fully typed.

### Errors

Non-2xx responses throw `UniFi.Network.Client.Http.UniFiApiException`, carrying the HTTP status
code plus the API's `code`/`message`/`requestId` fields when present.

```csharp
try
{
    await client.Devices.RestartAsync(siteId, deviceId);
}
catch (UniFiApiException ex)
{
    Console.WriteLine($"{ex.StatusCode} {ex.Code}: {ex.Message}");
}
```

### Dependency injection

Each client has an optional `UniFi.<Product>.Client.Extensions.DependencyInjection` package that
registers it as a typed `HttpClient` via `IHttpClientFactory` (pooled, long-lived connections),
applying the same TLS pinning/trust settings. The client libraries stay dependency-free; only these
packages pull in `Microsoft.Extensions.Http`.

```csharp
using Microsoft.Extensions.DependencyInjection;
using UniFi.Network.Client;

builder.Services.AddUniFiNetworkClient(
    UniFiClientOptions.ForLocalConsole("192.168.1.1", apiKey, pinnedCertificateSha256: pin));

// the others follow the same shape:
builder.Services.AddUniFiProtectClient(ProtectClientOptions.ForCloudConnector(consoleId, apiKey));
builder.Services.AddUniFiSiteManagerClient(apiKey);          // cloud-only, API key
builder.Services.AddUniFiMobilityClient(apiKey);             // cloud-only, `mobility`-scoped key

// then inject the client anywhere:
public sealed class GuestService(UniFiNetworkClient unifi) { /* ... */ }
```

Each `AddUniFi…Client` returns the `IHttpClientBuilder`, so you can chain resilience/logging handlers
(e.g. Polly). All overloads also accept a `Func<IServiceProvider, …Options>` to resolve options from
configuration.

## Site Manager API

`UniFi.SiteManager.Client` targets the account-wide cloud API at `https://api.ui.com`. Unlike the
Network client (which talks to one console), the Site Manager API spans every host on your account.
It only needs an API key generated at [unifi.ui.com](https://unifi.ui.com).

```csharp
using UniFi.SiteManager.Client;

using var client = new UniFiSiteManagerClient(apiKey);

// Hosts (consoles / network servers) on your account
var hosts = await client.Hosts.ListAsync(pageSize: 50);
foreach (var host in hosts.Data)
    Console.WriteLine($"{host.Id} [{host.Type}] {host.IpAddress}");

// Sites across all hosts
var sites = await client.Sites.ListAsync();

// Devices, grouped by host
var deviceGroups = await client.Devices.ListAsync(hostIds: new[] { "host-id" });

// ISP metrics and SD-WAN
var metrics = await client.IspMetrics.GetAsync("5m", duration: "24h");
var sdwan = await client.SdWanConfigs.ListAsync();
```

The available resources are `Hosts`, `Sites`, `Devices`, `IspMetrics`, and `SdWanConfigs`.

### Pagination (cursor-based)

Site Manager list endpoints use cursor pagination, not offset/limit. Each `SiteManagerPage<T>`
carries `Data`, a `NextToken`, and a `HasMore` helper — pass `NextToken` back to fetch the next page:

```csharp
var page = await client.Hosts.ListAsync(pageSize: 50);
while (true)
{
    foreach (var host in page.Data) { /* ... */ }
    if (!page.HasMore) break;
    page = await client.Hosts.ListAsync(pageSize: 50, nextToken: page.NextToken);
}
```

### Flexible fields and errors

Version-dependent blobs (a host's `UserData`/`ReportedState`, a site's `Meta`/`Statistics`, a
device's `Uidb`) are exposed as `System.Text.Json.JsonElement`. Non-2xx responses throw
`UniFi.SiteManager.Client.Http.UniFiSiteManagerException`, carrying the HTTP status plus the API's
`code`/`message`/`traceId` when present.

## Protect API

`UniFi.Protect.Client` targets the UniFi Protect integration API. Like the Network client, it
supports both a **local console** and the **Site Manager cloud connector** — pick a target the
same way:

```csharp
using UniFi.Protect.Client;

var options = ProtectClientOptions.ForLocalConsole("192.168.1.1", apiKey);
// or: ProtectClientOptions.ForCloudConnector(consoleId, apiKey);
using var client = new UniFiProtectClient(options);

var info = await client.Meta.GetInfoAsync();
Console.WriteLine($"Protect {info.ApplicationVersion}");

foreach (var camera in await client.Cameras.ListAsync())
    Console.WriteLine($"{camera.Name} [{camera.ModelKey}] {camera.State}");
```

Resources cover the full device fleet: `Cameras`, `Lights`, `Sensors`, `Chimes`, `Viewers`,
`Bridges`, `Fobs`, `Nvrs`, `Speakers`, `Sirens`, `Relays`, `LinkStations`, `AlarmHubs`,
`LiveViews`, `ArmProfiles`, `Users`, `UlpUsers`, `Files`, `AlarmManager`, and `Subscriptions`,
plus camera actions (RTSPS streams, snapshots, talkback, PTZ) and siren/relay/alarm-hub control.

### Typed reads, JsonObject writes

Protect device responses return plain arrays (no pagination). Identity/overview fields — `Id`,
`Name`, `Mac`, `State`, quality/volume/mode fields, battery status, timestamps — are strongly
typed, while deep, frequently-extended settings blobs (`osdSettings`, `smartDetectSettings`,
`featureFlags`, ...) are exposed as `System.Text.Json.JsonElement`. Update devices by PATCHing a
partial `System.Text.Json.Nodes.JsonObject`:

```csharp
using System.Text.Json.Nodes;

await client.Cameras.UpdateAsync(cameraId, new JsonObject
{
    ["name"] = "Front Door",
    ["osdSettings"] = new JsonObject { ["isNameEnabled"] = true },
});

var snapshot = await client.Cameras.GetSnapshotAsync(cameraId, highQuality: true); // byte[]
```

### Real-time updates

`Subscriptions` streams Protect's WebSocket feeds as `IAsyncEnumerable<JsonElement>` (the event
schemas are numerous and version-dependent, so messages are surfaced as raw JSON):

```csharp
await foreach (var message in client.Subscriptions.SubscribeToEventsAsync(cancellationToken))
    Console.WriteLine(message.GetRawText());
```

### Errors

Non-2xx responses throw `UniFi.Protect.Client.Http.UniFiProtectException`, carrying the HTTP status
plus the API's `error`/`name` fields.

## Mobility API

`UniFi.Mobility.Client` targets the UniFi Mobility API at `https://api.ui.com` — the account-wide
cloud API for mobile routing devices (UMR). Like the Site Manager client, it's **cloud-only** and
needs just an API key (issued with the `mobility` scope) from unifi.ui.com.

```csharp
using UniFi.Mobility.Client;

using var client = new UniFiMobilityClient(apiKey);

var workspaces = await client.Workspaces.ListAsync();
foreach (var workspace in workspaces.Data)
{
    var devices = await client.Devices.ListAsync(workspace.WorkspaceId);
    foreach (var device in devices.Data)
        Console.WriteLine($"{device.Name} [{device.Model}] {device.State}");
}
```

The resources are `Workspaces` (list + admins) and `Devices` (list, detail, clients, and
`UpdateName`/`UpdateNetwork`/`UpdateWireless` — the last three need a `write:mobility` key).
Everything is fully typed, including device detail (WAN, cellular, WiFi, VPN, subscription, GPS).

### Pagination (offset/limit)

Mobility list endpoints return a `MobilityPage<T>` with `Data`, `Total`, `Offset`, `Limit`, and a
`HasMore` helper:

```csharp
var page = await client.Devices.ListAsync(workspaceId, limit: 200, offset: 0);
while (true)
{
    foreach (var device in page.Data) { /* ... */ }
    if (!page.HasMore) break;
    page = await client.Devices.ListAsync(workspaceId, limit: 200, offset: page.Offset + page.Data.Count);
}
```

### Errors

Non-2xx responses throw `UniFi.Mobility.Client.Http.UniFiMobilityException`, carrying the HTTP
status plus the API's `code`/`message`/`traceId`. Note the API is rate limited to 100 requests
per minute per key (`429`, surfaced as `code == "rate_limit"`).

## Project layout

- `src/UniFi.Network.Client` — the Network API library
  - `UniFiClientOptions` — connection targets (local console / cloud connector)
  - `UniFiNetworkClient` — entry point, aggregates resource clients
  - `Http/` — HTTP plumbing (`ApiConnection`, `UniFiApiException`)
  - `Models/` — request/response types
  - `Resources/` — one class per API resource group (Sites, Devices, Clients, Networks, ...)
- `src/UniFi.SiteManager.Client` — the Site Manager API library
  - `SiteManagerClientOptions` — API key / base address
  - `UniFiSiteManagerClient` — entry point, aggregates resource clients
  - `Http/` — HTTP plumbing (`ApiConnection`, `UniFiSiteManagerException`)
  - `Models/` — response types and the `SiteManagerPage<T>` pagination envelope
  - `Resources/` — Hosts, Sites, Devices, IspMetrics, SdWanConfigs
- `src/UniFi.Protect.Client` — the Protect API library
  - `ProtectClientOptions` — connection targets (local console / cloud connector)
  - `UniFiProtectClient` — entry point, aggregates resource clients
  - `Http/` — HTTP plumbing (`ApiConnection`, `UniFiProtectException`)
  - `Models/` — device and response types
  - `Resources/` — one class per device group plus Files, AlarmManager, and Subscriptions
- `src/UniFi.Mobility.Client` — the Mobility API library
  - `MobilityClientOptions` — API key / base address
  - `UniFiMobilityClient` — entry point, aggregates resource clients
  - `Http/` — HTTP plumbing (`ApiConnection`, `UniFiMobilityException`)
  - `Models/` — workspace, device, and client types plus the `MobilityPage<T>` envelope
  - `Resources/` — Workspaces, Devices
- `src/UniFi.<Product>.Client.Extensions.DependencyInjection` (one per client) — optional DI
  integration (`AddUniFi<Product>Client(...)` over `IHttpClientFactory`)
- `samples/UniFi.Network.Client.Sample` — console app demonstrating both connection targets
- `samples/UniFi.SiteManager.Client.Sample` — console app for the Site Manager API
- `samples/UniFi.Protect.Client.Sample` — console app for the Protect API
- `samples/UniFi.Mobility.Client.Sample` — console app for the Mobility API

## Sample app

```bash
export UNIFI_API_KEY=...
export UNIFI_CONSOLE_HOST=192.168.1.1   # or UNIFI_CONSOLE_ID=... for the cloud connector
dotnet run --project samples/UniFi.Network.Client.Sample
```

For the Site Manager API (only an API key is needed):

```bash
export UNIFI_API_KEY=...
dotnet run --project samples/UniFi.SiteManager.Client.Sample
```

For the Protect API (same local/cloud targets as the Network sample):

```bash
export UNIFI_API_KEY=...
export UNIFI_CONSOLE_HOST=192.168.1.1   # or UNIFI_CONSOLE_ID=... for the cloud connector
dotnet run --project samples/UniFi.Protect.Client.Sample
```

For the Mobility API (only an API key with the `mobility` scope is needed):

```bash
export UNIFI_API_KEY=...
dotnet run --project samples/UniFi.Mobility.Client.Sample
```

## Releasing (package security)

Publishing to NuGet.org uses **Trusted Publishing** (OIDC) — there is no long-lived
`NUGET_API_KEY` secret. Each package is versioned and released **independently**: its version lives
in its own `.csproj`, and pushing a per-package tag `<PackageId>-v<version>` (e.g.
`UniFi.Protect.Client-v7.1.87`) runs the `publish-release` job, which packs, signs, and publishes
**only that package** via OIDC. `make release` creates one such tag per project from its csproj
version; `make release PROJECT=src/UniFi.Protect.Client/UniFi.Protect.Client.csproj` releases just
one. (Pushes to `main` still publish `-canary` prereleases of every package to GitHub Packages.)

One-time setup (repository/organization owner):

1. **Trusted Publisher policy** on NuGet.org (Account → Trusted Publishing) with:
   - Package Owner: `DanielvGinneken`
   - Repository Owner: `danielvanginneken`, Repository: `UniFi.NET`
   - Workflow File: `ci-cd.yml`, Environment: `production`
2. A GitHub Actions **environment** named `production` (the release job references it).
3. **Signing certificate** — register the public `.cer` under the NuGet.org organization's
   certificates, and add the signing key to the repo:
   - Secret `SIGNING_CERTIFICATE_BASE64` — base64 of the code-signing `.pfx`
     (`base64 -w0 cert.pfx` on Linux / `base64 -i cert.pfx` on macOS).
   - Secret `SIGNING_CERTIFICATE_PASSWORD` — the `.pfx` password.
   - Optional variable `SIGNING_TIMESTAMPER_URL` — RFC 3161 timestamp server
     (defaults to `http://timestamp.digicert.com`).

Private keys (`*.pfx`, `*.p12`, `*.snk`, `*.key`) are git-ignored and must never be committed —
only ever provide them through repository secrets.

## Status

Four independently-versioned client packages, each built against — and versioned to match — its
upstream API, plus an optional DI package:

| Package | Version | API / role |
|---|---|---|
| `UniFi.Network.Client` | 10.4.57 | [Network](https://developer.ui.com/network/) (local console + cloud connector) |
| `UniFi.Protect.Client` | 7.1.87 | [Protect](https://developer.ui.com/protect/) (local console + cloud connector) |
| `UniFi.SiteManager.Client` | 1.0.0 | [Site Manager](https://developer.ui.com/site-manager/) (cloud) |
| `UniFi.Mobility.Client` | 1.0.0 | [Mobility](https://developer.ui.com/mobility/) (cloud) |
| `UniFi.<Product>.Client.Extensions.DependencyInjection` (×4) | 1.0.0 | DI / `IHttpClientFactory` registration for each client — *not yet published* |

Generated request/response shapes are built against these API versions and may drift from newer
firmware — the linked OpenAPI specs are the source of truth. The client libraries have **zero
third-party dependencies** (the DI package pulls in `Microsoft.Extensions.Http`), and local-console
targets support **TLS certificate pinning** (see the Security note
under [Local console](#local-console)).
