# Zabbix Template: Ubiquiti UniFi API

A comprehensive Zabbix 7.0 template set for monitoring Ubiquiti UniFi infrastructure via both the UniFi Site Manager cloud API and local controller endpoints. Provides unified visibility across gateways, access points, switches, UPS units, redundant power supplies, and UniFi Protect cameras/NVRs.

See [CHANGELOG.md](CHANGELOG.md) for what's changed between versions.

---

## Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Template Architecture](#template-architecture)
- [Polling Architecture](#polling-architecture)
- [Getting Started](#getting-started)
  - [1. Obtain a Cloud API Key](#1-obtain-a-cloud-api-key)
  - [2. Prepare Local Controller Credentials](#2-prepare-local-controller-credentials)
  - [3. Import the Template](#3-import-the-template)
  - [4. Create the Root Host](#4-create-the-root-host)
  - [5. Configure Host Macros](#5-configure-host-macros)
  - [6. Run Discovery](#6-run-discovery)
  - [7. Configure Per-Host Macros](#7-configure-per-host-macros)
- [Macros Reference](#macros-reference)
- [Alarms Reference](#alarms-reference)
- [Site Discovery Filtering](#site-discovery-filtering)
- [WAN Alarm Suppression](#wan-alarm-suppression)
- [Site Outage Alarm Suppression](#site-outage-alarm-suppression)
- [High Availability (Shadow Mode)](#high-availability-shadow-mode)
- [Troubleshooting](#troubleshooting)

---

## Overview

This template set bridges the UniFi Site Manager cloud API (`api.ui.com`) with local UniFi OS controller polling. Discovery is automatic - a single root host with your API key discovers all consoles, devices, SD-WAN configs, and sites, and creates Zabbix hosts for each automatically.

**What gets monitored:**

- UniFi console cloud connectivity, firmware, backup age, and release channel
- Per-WAN uptime (24h rolling), latency, external IP changes, plugged state, and downtime periods
- Site-level statistics (offline devices by type, gateway count, critical notifications, IPS mode, combined WAN uptime)
- SD-WAN tunnel health with proportional severity, config errors, hub/spoke WAN issues
- Local gateway CPU, memory, temperature, WAN status, VPN tunnels, HA/Shadow Mode
- Per-AP radio health (band, channel, utilisation, client count, retries)
- Per-switch port status, errors, PoE power, and TX/RX rates
- Per-SFP port temperature, RX/TX optical power (dBm), voltage, current, and TX/RX fault state
- UPS battery level, runtime, mains state, and outlet relay state
- Redundant Power Supply failover and port redundancy state
- UniFi Protect camera connectivity, recording-gap detection, and firmware state
- UniFi Protect NVR storage utilization, database/disk health, and recording status
- Cloud and local controller health checks for all discovered hosts

---

## Requirements

| Component | Minimum Version |
|-----------|----------------|
| Zabbix Server | 7.0 |
| Zabbix Agent | Not required (all HTTP_AGENT) |
| UniFi OS | 3.x or later |
| UniFi Network App | 8.x or later |

The Zabbix server (or proxy) must have HTTPS access to:
- `api.ui.com` (cloud API)
- Each UniFi console's management IP on port 443 (local polling)

---

## Template Architecture

The YAML file contains ten linked templates. You only assign one manually - everything else is created by LLD discovery.

```
Ubiquiti UniFi API          <- Assign this to a single Zabbix host
  |
  +-- [LLD] Discover Hosts
  |     +-- Ubiquiti UniFi API Host    (one host per console)
  |           +-- [LLD] Discover UAP Devices        --> Ubiquiti UniFi Device + Ubiquiti UniFi UAP
  |           +-- [LLD] Discover USW Devices        --> Ubiquiti UniFi Device + Ubiquiti UniFi USW
  |           +-- [LLD] Discover UPS Devices        --> Ubiquiti UniFi Device + Ubiquiti UniFi UPS
  |           +-- [LLD] Discover RPS Devices        --> Ubiquiti UniFi Device + Ubiquiti UniFi RPS
  |           +-- [LLD] Discover Other Devices      --> Ubiquiti UniFi Device (generic; excludes Protect devices)
  |           +-- [LLD] Discover WAN Interfaces
  |           +-- [LLD] Discover Host WAN Interfaces (uptime/latency via local API)
  |           +-- [LLD] Discover Site Statistics
  |           +-- [LLD] Discover Host Controllers
  |           +-- [LLD] Discover Protect Storage Disks   (item prototypes on the console host itself)
  |           +-- [LLD] Discover Host Protect Cameras    --> Ubiquiti UniFi Protect Camera
  |           +-- [LLD] Discover Host Protect Other      --> Ubiquiti UniFi Protect Device (chimes, etc.)
  |
  +-- [LLD] Discover SD-WAN Configs
        +-- Ubiquiti UniFi SD-WAN      (one host per SD-WAN config)
```

**Ubiquiti UniFi Device** is a shared base template applied alongside all Network device-type templates (UAP/USW/UPS/RPS). It provides cloud-side firmware status, reboot detection, and managed state monitoring. Alarms from this base template apply to all device types. **It is deliberately not applied to Protect cameras/devices** - UniFi Protect isn't part of the Network cloud fleet API this base template queries, so there's no cloud-side data for it to add; Protect device health comes entirely from the local Protect controller instead.

---

## Polling Architecture

```
                    api.ui.com (UniFi Site Manager cloud)
                                    |
        +-------------------------------------------------------+
        |  Root host (1x globally, every 1m)                    |
        |    GET /v1/hosts            -> unifi.api.hosts        |
        |    GET /v1/sd-wan-configs   -> unifi.sdwan.configs    |
        +-------------------------------------------------------+
                                    |
                     (each console independently re-fetches its
                      own record - see limitation below)
                                    v
        +-------------------------------------------------------+
        |  Console host (1x per console, every 1m)               |
        |    GET /v1/hosts/{hostId}    -> unifi.api.hosts        |
        |    GET /v1/devices?hostId=X  -> unifi.host.session |
        |    GET /v1/sites?hostIds=X   -> unifi.api.host.sites   |
        +-------------------------------------------------------+
                                    |
                     (each device independently re-fetches the
                      same device list - see limitation below)
                                    v
        +-------------------------------------------------------+
        |  Per-device host (Ubiquiti UniFi Device template)      |
        |    GET /v1/devices?hostId=X  -> unifi.api.device.raw   |
        +-------------------------------------------------------+


                    Local controller (per console, HTTPS 443)
                                    |
        +-------------------------------------------------------+
        |  Console host (1x per console) - the ONLY thing that   |
        |  logs in under normal operation                        |
        |    POST /api/auth/login  (session token)                |
        |    GET  /v1/devices?hostId=X  (cloud, as before)        |
        |      --> unifi.host.session                        |
        |    every {$UNIFI.SESSION.REFRESH.INTERVAL}, default 10m |
        +-------------------------------------------------------+
                                    |
                     (CALCULATED item reads this item's last
                      stored value via last(//unifi.host.session)
                      - does NOT trigger a new poll of it, so the
                      two schedules are fully independent)
                                    v
        +-------------------------------------------------------+
        |  Console host (1x per console) - own fast schedule,    |
        |  no login of its own unless the cached token is stale  |
        |    GET  /proxy/network/api/s/default/stat/health       |
        |    GET  /proxy/network/api/s/default/stat/device       |
        |    GET  /proxy/network/api/s/default/rest/.../shadow   |
        |    GET  /api/system                                    |
        |      --> unifi.local.all                                |
        |    every {$UNIFI.CONSOLE.POLL.INTERVAL}, default 3m     |
        |    (safe to set much lower, e.g. 10s, for fast WAN/     |
        |    health telemetry - login frequency is unaffected)    |
        +-------------------------------------------------------+
                                    |
                     (token also propagated to every device via
                      {$UNIFI.SESSION.TOKEN} - same mechanism
                      already used for LOCAL.IP/USERNAME/PASSWORD)
                                    v
        +-------------------------------------------------------+
        |  Per-device host (UAP / USW / UPS / RPS template)      |
        |    GET .../stat/device using the cached token - no      |
        |    login, unless the token is rejected (see below)      |
        |      --> unifi.uap.device / usw.device /                |
        |          ups.device / rps.device                        |
        |    every {$UNIFI.LOCAL.POLL.INTERVAL}, default 10s      |
        +-------------------------------------------------------+
                                    |
                     DEPENDENT items only from here down
                                    v
        Per-port / per-radio / per-outlet / per-SFP discovery
        and metrics (switch ports, AP radios, UPS outlets, RPS
        ports) derive from the per-device fetch above - no
        further API calls.
```

**Local controller logins: fixed.** Every device host used to log into the local controller itself, every cycle, and `unifi.local.all` did too, independently - on a console with 12 devices that's 12+ separate logins hitting the same controller, and it actually tripped UniFi's local rate limit in testing (10 concurrent logins to a real console came back `429` on 7 of them). Now there's exactly **one** thing that logs in under normal operation: `unifi.host.session`, on its own slow schedule (`{$UNIFI.SESSION.REFRESH.INTERVAL}`, default 10m). Everything else reads that cached token instead of logging in itself:

- Devices get it via `{$UNIFI.SESSION.TOKEN}`, the same LLD mechanism that already carries `{$UNIFI.LOCAL.IP}`/`{$UNIFI.USERNAME}`/`{$UNIFI.PASSWORD}` from console to device.
- `unifi.local.all` gets it via a `CALCULATED` item formula, `last(//unifi.host.session)`, which reads `unifi.host.session`'s most recently stored value *without* triggering a new poll of it. This is what actually makes fast+independent WAN/health polling possible: `unifi.local.all` runs on its own schedule (`{$UNIFI.CONSOLE.POLL.INTERVAL}`, default 3m, safe to drop to something like `10s` if you want responsive WAN telemetry) with zero effect on how often the console actually logs in. A `DEPENDENT` item couldn't do this - it would tie `unifi.local.all` to `unifi.host.session`'s slow schedule - and there's no way for an item to write a macro back onto its own host, so `CALCULATED` + `last()` is the one native mechanism that decouples the two.

Both consumers fall back to a one-off fresh login, just for that one poll, if their cached token is ever missing or rejected (console reboot, a missed refresh cycle) - self-healing immediately rather than waiting on the next scheduled refresh.

`{$UNIFI.SESSION.REFRESH.INTERVAL}`'s `10m` default is deliberately conservative rather than cut close to the session token's ~2 hour lifetime: the login itself is cheap (O(1) per console regardless of device count), but the exact trigger for UniFi's local rate limiting isn't fully characterized - testing confirmed it's tripped by concurrent logins, but a rolling per-time-window limit hasn't been ruled out, so refreshing every 10 minutes rather than every 1 keeps total login attempts an order of magnitude lower for no real cost (still a 12x safety margin against the token's own expiry, and new/renamed devices - discovered via this same item's cloud fetch - just take up to 10 minutes to show up instead of 1).

All three poll intervals - `{$UNIFI.LOCAL.POLL.INTERVAL}` (per-device), `{$UNIFI.CONSOLE.POLL.INTERVAL}` (`unifi.local.all`), and `{$UNIFI.SESSION.REFRESH.INTERVAL}` (`unifi.host.session`) - flow through the same chain as `{$UNIFI.USERNAME}`/`{$UNIFI.PASSWORD}`: one real default on the root `Ubiquiti UniFi API` template, propagated to every console, and for the per-device one, propagated again from each console to its devices - so any of them can be changed once at the root for everything, or overridden at the console or device level. Unlike the credential macros, every tier this passes through also keeps its own real fallback value (matching the ordinary threshold macros like `{$UNIFI.CPU.USAGE.WARN}`), not just the root: an empty *credential* just fails a login cleanly, but an empty polling *interval* is a hard Zabbix configuration error (`Invalid update interval ""`), so there's always something valid to fall back to for the brief window before a freshly-imported host's discovery chain has fully propagated a real value down to it.

**UniFi Protect rides the same token.** UniFi Protect's local API (`/proxy/protect/api/bootstrap`) accepts the exact same session cookie as the Network API, as long as the account also has Protect permission granted in UniFi OS (a separate, one-time grant - see [Step 2](#2-prepare-local-controller-credentials)). A new `CALCULATED` item, `unifi.protect.bootstrap`, reads the cached token via `last(//unifi.host.session)` (identical mechanism to `unifi.local.all` above) and fetches the Protect bootstrap payload on its own schedule (`{$UNIFI.CONSOLE.POLL.INTERVAL}`) - no separate login, no new credentials. Every Protect camera and NVR-level item then derives from that one fetch, the same way Network devices derive from `unifi.local.all`/`unifi.host.session`. On a console that doesn't run Protect at all, this item detects the resulting 404 and reports an empty result rather than erroring - most consoles in a mixed fleet (pure Network/SD-WAN gateways) will simply show zero Protect hosts with no noise.

**Known limitation: redundant cloud calls per device.** Every device host still fetches the cloud `/v1/devices` list itself (`unifi.api.device.raw`), even though its own console already pulled that exact data. This is a smaller-impact limitation than the local logins above - cloud API calls aren't subject to the same aggressive rate limiting UniFi applies to local logins, so it hasn't caused practical problems the way local logins did. The same structural blocker applies here, with one nuance: `CALCULATED` items *can* read another item's last value without polling it (that's exactly what `unifi.local.all` now does, above) - but only from a host they can name, and the only host-naming options are a literal hostname or an empty host slot meaning "myself." Neither works across the console→device boundary, since a device can't hardcode its own dynamically-discovered console's hostname, and "myself" isn't the console. `DEPENDENT` items have the same problem: the master must be on the same host. The only way to genuinely eliminate this one is turning per-device metrics into LLD item prototypes living on the console host instead of giving each device its own Zabbix host - a real architecture change, parked rather than attempted here.

None of this causes bad alerting. A rate-limited or failed poll (local or cloud) is just skipped rather than misread as "offline" (see [Troubleshooting](#troubleshooting)).

---

## Getting Started

### 1. Obtain a Cloud API Key

The template authenticates to the UniFi Site Manager API using an API key.

1. Log in to [unifi.ui.com](https://unifi.ui.com) with an administrator account
2. In the **left sidebar**, click **API Keys**
3. Click **Create New API Key**
4. Enter a descriptive name (e.g. `Zabbix`)
5. Set an **Expiration**: select **Never Expires** for a long-lived monitoring key, or at minimum **1 Year**
6. Under **Applications**, ensure both **Site Manager** and **UniFi Applications** are checked
7. Under **Sites**, select **All Sites** (or restrict to the specific sites you wish to monitor)
8. Click **Create** and copy the key. It's only shown once, so save it somewhere safe.

> The API key is passed as an `X-API-KEY` header on all requests to `api.ui.com`. It does not grant access to local controller endpoints.

---

### 2. Prepare Local Controller Credentials

Local polling hits each console's UniFi OS controller directly over HTTPS. That is port 443 on a UniFi OS console (UDM, CloudKey, UX); self-hosted UniFi OS Server uses 11443 instead, which you set with `{$UNIFI.LOCAL.PORT}`. A local admin account with **View Only** permissions is sufficient and recommended.

**To create a local read-only account:**

1. Log in to [unifi.ui.com](https://unifi.ui.com) and open the **Network** app for the site, or navigate directly to the console's IP address (e.g. `https://10.1.1.1`). Either way works the same.
2. In the **left sidebar**, click the **People** icon (the group icon near the bottom of the sidebar)
3. Click **Create New** → **Create New User**
4. Enter a **First Name** and **Last Name** for the account (e.g. `Zabbix` / `Monitor`); email isn't required
5. Check the **Admin** checkbox to reveal **Username**, **Password**, and role fields
6. Enter a username (e.g. `zabbix-viewonly`) and a strong password; leave **Super Admin** unchecked
7. Set both role dropdowns to **View Only**
8. Leave **Groups** empty, no group membership is needed
9. In the **Assignments** section, **uncheck** both **Network** and **One-Click VPN**. These control cloud/Fabric app access, which a local monitoring account doesn't need.
10. Click **Create**

Repeat this process for each console you wish to monitor locally, using the same username/password everywhere if that's practical for your environment. The account credentials are entered as Zabbix macros. If every site shares the same credentials, set them **once** on the root host in [Step 5](#5-configure-host-macros) instead of per console; a per-console override in [Step 7](#7-configure-per-host-macros) still works for any site that needs different credentials.

**Permissions required:**
- View Only access to the Network application (devices, health, port stats)
- **If you want Protect (camera/NVR) monitoring**, also set the **Protect** role dropdown (in the same Roles panel as Network) to **View Only**, then click through to actually save it. This is a separate grant from Network access - a console can run Protect with cameras adopted while this account still has zero Protect permission, and the account working fine for Network tells you nothing about whether Protect access is also granted. If `unifi.protect.bootstrap` (see [Troubleshooting](#troubleshooting)) reports `HTTP 403`, this is almost always why.

No write permissions are needed or recommended.

---

### 3. Import the Template

1. In Zabbix, navigate to **Data collection > Templates**
2. Click **Import**
3. Upload `zbx_template_unifi.yaml`
4. Ensure all checkboxes are ticked (Templates, Items, Triggers, etc.)
5. Click **Import**

All ten templates will appear in your template list.

---

### 4. Create the Root Host

Create a single Zabbix host to act as the API entry point. This host does not represent a physical device - it exists solely to run cloud discovery.

1. Navigate to **Data collection > Hosts > Create host**
2. Set:
   - **Host name:** `UniFi Site Manager` (or any name)
   - **Templates:** `Ubiquiti UniFi API`
   - **Host group:** Your preferred group (e.g. `UniFi`)
   - **Interfaces:** None required
3. Click **Add**

---

### 5. Configure Host Macros

On the root host, open the **Macros** tab and set:

| Macro | Value | Notes |
|-------|-------|-------|
| `{$APIKEY}` | `your-api-key-here` | From Step 1. Use **Secret text** type. |

Optional threshold overrides (defaults are suitable for most deployments):

| Macro | Default | Description |
|-------|---------|-------------|
| `{$WAN_UPTIME_WARN}` | `99` | Site-wide combined WAN uptime % for AVERAGE alarm |
| `{$WAN_UPTIME_HIGH}` | `95` | Site-wide combined WAN uptime % for DISASTER alarm |

**If every site shares the same local controller credentials**, set them once now instead of repeating [Step 7](#7-configure-per-host-macros) per console, on this same root host:

| Macro | Value | Notes |
|-------|-------|-------|
| `{$UNIFI.USERNAME}` | `zabbix-viewer` | Local controller read-only username |
| `{$UNIFI.PASSWORD}` | `your-password` | Use **Secret text** type |

Every console discovered from this point on inherits these automatically. To use different credentials for a specific site, override `{$UNIFI.USERNAME}` / `{$UNIFI.PASSWORD}` on that console's own host (Step 7); a host-level value always wins over the inherited one. Setting the default here, on the root *host* rather than a template, means it survives re-importing an updated version of this template in future.

---

### 6. Run Discovery

LLD runs automatically on the configured interval (default: 1 hour). To trigger immediately:

1. Navigate to the root host
2. Go to **Data collection > Discovery rules**
3. Click **Execute now** on **Discover UniFi Hosts** and **Discover SD-WAN Configs**

After a few minutes, new hosts will appear for each discovered console, pre-populated with:

| Auto-populated Macro | Source |
|----------------------|--------|
| `{$HOSTID}` | Console UUID from cloud API |
| `{$UNIFI.LOCAL.IP}` | First private IP from the console's reported addresses |
| `{$DEVICENAME}` | Console display name |

---

### 7. Configure Per-Host Macros

If you set a shared default in [Step 5](#5-configure-host-macros), every discovered console already has working local credentials and you can skip this unless a specific site needs different ones.

To override credentials for one console, open that console host, go to its **Macros** tab, and set:

| Macro | Value | Notes |
|-------|-------|-------|
| `{$UNIFI.USERNAME}` | `zabbix-viewer` | Local controller read-only username |
| `{$UNIFI.PASSWORD}` | `your-password` | Use **Secret text** type |

A host-level value here always takes priority over the root host's default, for just this one console.

Common optional overrides:

| Macro | Default | When to change |
|-------|---------|----------------|
| `{$UNIFI.WAN.LATENCY.WARN}` | `100` (ms) | Raise to 200+ for high-latency links (e.g. Central America sites with 60-80ms base RTT to UK) |
| `{$UNIFI.WAN.ENABLED:WAN2}` | `1` | Set to `0` if WAN2 is not physically connected on this gateway |
| `{$UNIFI.CPU.USAGE.WARN}` | `80` | Adjust per device capability |
| `{$UNIFI.TEMP.WARN}` | `70` | Adjust for deployment environment |
| `{$BACKUP_INTERVAL}` | `8d` | Match your configured UniFi backup schedule |

---

## Macros Reference

### Global (Root Template: Ubiquiti UniFi API)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$APIKEY}` | *(secret)* | UniFi Site Manager API key |
| `{$UNIFI.USERNAME}` | *(empty)* | Default local controller username, propagated to every discovered console - see [Step 5](#5-configure-host-macros) |
| `{$UNIFI.PASSWORD}` | *(secret)* | Default local controller password, propagated to every discovered console |
| `{$UNIFI.LOCAL.POLL.INTERVAL}` | `10s` | Default per-device local poll interval, propagated to every discovered console and then on to every device - override at any tier (root, console, or device). Not a secret, so unlike username/password the real default lives directly on this template rather than requiring a root host override. Safe to poll this fast since no login is attached to this item (see [Polling Architecture](#polling-architecture)) - only the session-token refresh (`{$UNIFI.SESSION.REFRESH.INTERVAL}`, default 10m) actually logs in |
| `{$UNIFI.CONSOLE.POLL.INTERVAL}` | `3m` | Default interval for each console's own combined local poll (`Local Raw (Combined)`, a `CALCULATED` item), propagated to every discovered console - override at the root or per-console. Safe to set much lower (e.g. `10s`) for fast WAN/health telemetry - this item doesn't log in itself, so its speed has no effect on login frequency (see [Polling Architecture](#polling-architecture)) |
| `{$UNIFI.SESSION.REFRESH.INTERVAL}` | `10m` | Default interval for each console's session-token mint + cloud device fetch (`Local Session + Cloud Devices Raw`), propagated to every discovered console - override at the root or per-console. Deliberately conservative rather than cut close to the ~2 hour session token lifetime: the login is cheap either way, but UniFi's local rate limiter is only confirmed to trigger on concurrent logins, not proven safe against a sustained per-minute rate too, so this keeps total login attempts low regardless |
| `{$UNIFI.LOCAL.PORT}` | `443` | Local controller HTTPS port, propagated to every discovered console and then on to every device. `443` on UniFi OS consoles (UDM, CloudKey, UX); set this to `11443` for self-hosted UniFi OS Server. Override at any tier if a fleet is mixed |
| `{$UNIFI.API.SITE}` | `default` | Internal name of the UniFi site to poll, propagated to every discovered console and device. See [Finding your site name](#finding-your-site-name) |
| `{$WAN_UPTIME_WARN}` | `99` | Site-wide combined WAN uptime AVERAGE threshold (%) |
| `{$WAN_UPTIME_HIGH}` | `95` | Site-wide combined WAN uptime DISASTER threshold (%) |
| `{$UNIFI.SITE.EXCLUDE}` | *(empty)* | Comma-separated consoles to exclude from discovery, see [Site Discovery Filtering](#site-discovery-filtering) |

### Per-Console Host (Ubiquiti UniFi API Host)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$APIKEY}` | *(secret)* | Inherited or overridden API key |
| `{$UNIFI.LOCAL.IP}` | *(auto)* | Console management IP - set by discovery |
| `{$UNIFI.LOCAL.PORT}` | `443` | Local controller HTTPS port - inherited from the root host at discovery time, or overridden here. Use `11443` for self-hosted UniFi OS Server |
| `{$UNIFI.USERNAME}` | *(auto)* | Local controller read-only username - inherited from the root host at discovery time, or overridden here |
| `{$UNIFI.PASSWORD}` | *(auto)* | Local controller password - inherited from the root host at discovery time, or overridden here |
| `{$UNIFI.API.AUTH.URI}` | `api/auth/login` | Login endpoint |
| `{$UNIFI.API.AUTH.TOKEN}` | `TOKEN` | Session cookie name |
| `{$UNIFI.API.URI}` | `proxy/network/api/s` | Local API base path, up to but not including the site. Changed in 1.7.0 - it previously held the full path including the site |
| `{$UNIFI.API.SITE}` | `default` | Internal name of the UniFi site to poll. See [Finding your site name](#finding-your-site-name) |
| `{$UNIFI.LOCAL.POLL.INTERVAL}` | *(auto)* | Local poll interval - inherited from the root host's default at discovery time, or overridden here for every device on this console |
| `{$UNIFI.CONSOLE.POLL.INTERVAL}` | *(auto)* | Console combined local poll interval - inherited from the root host's default at discovery time, or overridden here |
| `{$UNIFI.SESSION.REFRESH.INTERVAL}` | *(auto)* | Session-token refresh interval - inherited from the root host's default at discovery time, or overridden here |
| `{$HOSTID}` | *(auto)* | Console UUID from cloud - set by discovery |
| `{$BACKUP_INTERVAL}` | `8d` | Expected maximum time between backups |
| `{$WAN_UPTIME_NOTICE}` | `99.9` | Per-WAN 24h availability WARNING threshold (%), about 1.4 min/day |
| `{$WAN_UPTIME_WARN}` | `99` | Per-WAN 24h availability AVERAGE threshold (%), about 14 min/day |
| `{$WAN_UPTIME_HIGH}` | `98` | Per-WAN 24h availability HIGH threshold (0% fires a separate alarm) |
| `{$UNIFI.WAN.LATENCY.WARN}` | `100` | WAN latency warning threshold (ms) |
| `{$UNIFI.CONSOLE.OFFLINE.GRACE}` | `3m` | How long the console must be continuously cloud-disconnected before "Console lost cloud connectivity" fires - sized to ride out a WAN failover between two uplinks without alarming. Must be longer than the 1m Connection State poll interval |
| `{$UNIFI.WAN.ENABLED}` | `1` | Master WAN alarm switch (1=enabled, 0=suppressed for all WANs) |
| `{$UNIFI.WAN.ENABLED:WAN2}` | `1` | WAN2-specific alarm switch, set to 0 to suppress all WAN2 alarms |
| `{$UNIFI.CPU.USAGE.WARN}` | `80` | CPU usage warning threshold (%) |
| `{$UNIFI.CPU.USAGE.HIGH}` | `90` | CPU usage critical threshold (%) |
| `{$UNIFI.MEM.USAGE.WARN}` | `80` | Memory usage warning threshold (%) |
| `{$UNIFI.MEM.USAGE.HIGH}` | `90` | Memory usage critical threshold (%) |
| `{$UNIFI.TEMP.WARN}` | `70` | Temperature warning threshold (°C) |
| `{$UNIFI.TEMP.HIGH}` | `80` | Temperature critical threshold (°C) |
| `{$UNIFI.PROTECT.API.URI}` | `proxy/protect/api` | Local Protect controller API base path |
| `{$UNIFI.PROTECT.STORAGE.HIGH}` | `99` | Protect recording storage usage % HIGH threshold - deliberately high and single-tier, since continuous recording is designed to run near-full as a matter of course |
| `{$UNIFI.PROTECT.DISK.TEMP.WARN}` | `60` | Protect NVR spinning disk temperature warning threshold (°C) - separate from `{$UNIFI.TEMP.WARN}` since HDD media runs at different temperatures than gateway hardware |
| `{$UNIFI.PROTECT.DISK.TEMP.HIGH}` | `70` | Protect NVR spinning disk temperature critical threshold (°C) |
| `{$UNIFI.PORT.DROPPED.WINDOW}` | `5m` | Time window for the gateway RX/TX dropped-packet rate check (see [Gateway Switch Ports](#gateway-switch-ports-discovered-automatically)) |
| `{$UNIFI.PORT.DROPPED.WARN}` | `1000` | Gateway RX/TX dropped-packet count threshold within `{$UNIFI.PORT.DROPPED.WINDOW}` - starting default, tune to your own link's normal baseline |

### Per-Device (UAP / USW / UPS / RPS)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$UNIFI.UPTIME.WARN}` | `24` | Uptime warning threshold (hours) - alarms if uptime is below this after a reboot |
| `{$UNIFI.PORT.DROPPED.WINDOW}` | `5m` | Time window for the switch RX/TX dropped-packet rate check (USW only) |
| `{$UNIFI.PORT.DROPPED.WARN}` | `1000` | Switch RX/TX dropped-packet count threshold within `{$UNIFI.PORT.DROPPED.WINDOW}` - starting default, tune to your own link's normal baseline (USW only) |
| `{$UNIFI.CPU.USAGE.WARN}` | `80` | CPU warning (%) |
| `{$UNIFI.CPU.USAGE.HIGH}` | `90` | CPU critical (%), UAP/USW only |
| `{$UNIFI.MEM.USAGE.WARN}` | `80` | Memory warning (%) |
| `{$UNIFI.MEM.USAGE.HIGH}` | `90` | Memory critical (%), UAP/USW only |
| `{$UNIFI.TEMP.WARN}` | `55` | Temperature warning (°C), RPS only |
| `{$UNIFI.TEMP.HIGH}` | `65` | Temperature critical (°C), RPS only |
| `{$UNIFI.UPS.BATTERY.WARN}` | `20` | UPS battery warning threshold (%) |
| `{$UNIFI.UPS.BATTERY.HIGH}` | `10` | UPS battery critical threshold (%) |
| `{$UNIFI.UPS.RUNTIME.HIGH}` | `300` | UPS runtime critical threshold (seconds) |

### Per-Protect-Camera / Per-Protect-Device (Ubiquiti UniFi Protect Camera / Ubiquiti UniFi Protect Device)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$UNIFI.UPTIME.WARN}` | `24` | Uptime warning threshold (hours) - alarms if uptime is below this after a reboot |
| `{$UNIFI.SESSION.TOKEN}` | *(auto)* | Cached local controller session token - inherited from the console, same mechanism as UAP/USW/UPS/RPS |
| `{$UNIFI.LOCAL.POLL.INTERVAL}` | *(auto)* | Local poll interval - inherited from the console's default at discovery time, or overridden here |
| `{$UNIFI.PROTECT.API.URI}` | `proxy/protect/api` | Local Protect controller API base path |

### Per-SFP Port (USW, discovered automatically for ports with `sfp_found = true`)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$UNIFI.SFP.TEMP.WARN}` | `70` | SFP module temperature warning threshold (°C). Active 10G SFP+ modules routinely run 60–65°C; 70°C indicates a genuinely elevated condition. |
| `{$UNIFI.SFP.TEMP.HIGH}` | `75` | SFP module temperature critical threshold (°C). Commercial SFP+ modules are typically rated to 70°C; above this degradation risk is high. |
| `{$UNIFI.SFP.RXPOWER.LOW.WARN}` | `-12` | RX optical power warning threshold (dBm) |
| `{$UNIFI.SFP.RXPOWER.LOW.HIGH}` | `-16` | RX optical power critical threshold (dBm). Near receiver sensitivity limit for 10GBase-LR. |
| `{$UNIFI.SFP.TXPOWER.LOW.WARN}` | `-6` | TX laser power warning threshold (dBm) |
| `{$UNIFI.SFP.TXPOWER.LOW.HIGH}` | `-9` | TX laser power critical threshold (dBm). Near minimum spec for 10GBase-LR. |

> **SFP temperature note:** SFP+ modules run significantly hotter than the switch chassis. 60–65°C is normal for an active 10G module in a warm rack. Raise `{$UNIFI.SFP.TEMP.WARN}` to `67` on a per-host basis if you see persistent warnings on healthy modules in a warm environment.

---

## Alarms Reference

Severities follow standard Zabbix conventions: INFO < WARNING < AVERAGE < HIGH < DISASTER.

### Cloud Connectivity and Console Health

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Console lost cloud connectivity | DISASTER | Console state has not been `connected` at any point in the last `{$UNIFI.CONSOLE.OFFLINE.GRACE}` (default 3m), so a brief WAN failover doesn't alarm; recovers on the first `connected` sample |
| Host is blocked by Ubiquiti | HIGH | Console account is suspended or blocked |
| Unadopted UniFi OS devices found | WARNING | One or more UniFiOS devices are pending adoption |
| No backup within expected interval | AVERAGE | Last backup older than `{$BACKUP_INTERVAL}` |
| UniFi OS firmware update available | INFO | A newer firmware version is available |
| UniFi OS version changed | INFO | Firmware version changed (post-upgrade tracking) |
| Console on non-production release channel | INFO | Console is on a beta or testing channel |
| Cloud system logging not enabled | INFO | Cloud log forwarding is disabled |
| Console device state changed | INFO | Device state transitioned (e.g. updating, setup) |

### Gateway - Local Polling

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| HA failover: gateway is now STANDBY | DISASTER | Primary failed; this unit is now serving traffic |
| WAN status is not OK | HIGH | Local WAN health reports a non-ok state |
| Shadow Mode link (eth6) disconnected | HIGH | HA interconnect cable is unplugged |
| Gateway CPU usage critical | HIGH | CPU > `{$UNIFI.CPU.USAGE.HIGH}`% |
| Gateway CPU temperature critical | HIGH | CPU temp > `{$UNIFI.TEMP.HIGH}`°C |
| Gateway board temperature critical | HIGH | Board temp > `{$UNIFI.TEMP.HIGH}`°C |
| Gateway memory usage critical | HIGH | Memory > `{$UNIFI.MEM.USAGE.HIGH}`% |
| Gateway CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Gateway CPU temperature high | WARNING | CPU temp > `{$UNIFI.TEMP.WARN}`°C |
| Gateway board temperature high | WARNING | Board temp > `{$UNIFI.TEMP.WARN}`°C |
| Gateway memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| WAN IP changed | WARNING | Local WAN IP changed from one value to another |
| ISP name changed | INFO | ISP name reported by the local controller changed |
| Shadow Mode peer IP changed | INFO | HA peer management IP changed |
| Local firmware version changed | INFO | Local controller firmware version changed |

### Gateway Switch Ports (discovered automatically)

Most UniFi gateways (UDM/UXG) have a built-in switch, so their physical ports get the same port-level monitoring as a dedicated switch, sourced from the same local controller data as everything else in this section.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Gateway port down | WARNING | Port transitions from up to down after previously being up |
| Gateway port TX errors increasing | WARNING | TX error counter is actively incrementing |
| Gateway port RX errors increasing | WARNING | RX error counter is actively incrementing |
| Gateway port RX dropped packets elevated | WARNING | More than `{$UNIFI.PORT.DROPPED.WARN}` RX drops within `{$UNIFI.PORT.DROPPED.WINDOW}` - not any single increase, since a low steady trickle is normal |
| Gateway port TX dropped packets elevated | WARNING | More than `{$UNIFI.PORT.DROPPED.WARN}` TX drops within `{$UNIFI.PORT.DROPPED.WINDOW}` - not any single increase, since a low steady trickle is normal |
| Gateway port STP blocking | WARNING | Spanning Tree is actively blocking this port to prevent a loop |
| Gateway SFP TX fault active | HIGH | Module reports a transmit fault |
| Gateway SFP RX loss of signal | HIGH | Module isn't detecting a receive signal (the gateway-side equivalent of a switch's RX fault) |
| Gateway SFP temperature critical/high | HIGH/WARNING | SFP module temp exceeds `{$UNIFI.SFP.TEMP.HIGH}` / `{$UNIFI.SFP.TEMP.WARN}` |
| Gateway SFP RX/TX power critical/low | HIGH/WARNING | Optical power exceeds the same `{$UNIFI.SFP.*}` thresholds used on switches |
| Gateway port renamed | INFO | Port name changed in UniFi |
| Gateway SFP module changed | INFO | Module serial number changed, indicating a physical swap |

> Gateways expose SFP module status/metadata (TX fault, RX loss-of-signal, part number, serial, vendor) for any inserted module, DAC or fibre, but optical measurements (temperature, RX/TX power, voltage, current) only for fibre modules, same DAC/fibre split as switches - see the note under [Switches (USW)](#switches-usw). Other per-port items with no alarm: `Satisfaction`, `Anomalies`, `Full Duplex`, `Autoneg`, `Is Uplink`, `Link Down Count`, `MAC Table Count`, `STP Edge Port`, `QoS Mode`, `EEPROM Readable` (SFP).

> **RX/TX dropped-packet tip:** these alarms fire on total growth within `{$UNIFI.PORT.DROPPED.WINDOW}` exceeding `{$UNIFI.PORT.DROPPED.WARN}`, not on any single increase - a low, steady trickle of drops is normal background behavior on a real link (congestion, buffer limits), not a fault. `1000`/`5m` is a starting default, not derived from a large sample of real links; if it fires too often (or not sensitively enough) on a specific port, check that port's RX/TX Dropped item in Latest data to see its actual normal rate and adjust the host-level macro accordingly.

### Per-WAN (discovered automatically via two LLD rules)

WAN uptime and latency are sourced from the **local controller** (`/stat/health`), which provides a true 24-hour rolling window and per-WAN latency readings. Downtime and packet loss period counts are sourced from the **cloud API** (`/v1/sites`), which tracks individual outage events at ~5-minute granularity.

Two sets of WAN items are discovered per gateway:

- **Discover WAN Interfaces** (cloud API): plugged state, enabled state, speed type, IPv4 address. These work even if the local controller is unreachable.
- **Discover Host WAN Interfaces** (local API): 24h uptime, latency, downtime periods, packet loss periods, external IP.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| {WAN} availability is 0% (completely down) | HIGH | 24h availability = 0% and `{$UNIFI.WAN.ENABLED:{WAN}}=1` |
| {WAN} physically unplugged | HIGH | Plugged = false for 3 consecutive checks and `{$UNIFI.WAN.ENABLED:{WAN}}=1` |
| {WAN} sustained downtime (3/3 periods) | HIGH | All 3 of the most recent outage periods show downtime |
| {WAN} 24h availability degraded | AVERAGE | 24h availability < `{$WAN_UPTIME_WARN}`% (~14 min/day down) |
| {WAN} intermittent downtime (2/3 periods) | AVERAGE | 2 of the last 3 outage periods show downtime |
| {WAN} 24h availability slightly below target | WARNING | 24h availability < `{$WAN_UPTIME_NOTICE}`% but ≥ `{$WAN_UPTIME_WARN}`% |
| {WAN} packet loss in 2/3 recent periods | WARNING | Packet loss in 2 or more of the last 3 measurement periods |
| {WAN} latency high | WARNING | 24h average latency > `{$UNIFI.WAN.LATENCY.WARN}` ms |
| {WAN} administratively disabled | WARNING | WAN interface disabled in UniFi config |
| {WAN} IPv4 address changed | WARNING | Local IPv4 address on this WAN changed |
| {WAN} external IP changed | INFO | Public (ISP-visible) IP changed |

> **Latency tip:** For sites where 60–80 ms to the hub is normal (e.g. Central America to UK), raise `{$UNIFI.WAN.LATENCY.WARN}` to `200` or higher on those gateway hosts to avoid false positives.

### Site Statistics (discovered automatically)

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Site has no gateway device | DISASTER | Gateway device count = 0 for two consecutive samples while the console is cloud-connected, all routing is down |
| Site aggregate WAN uptime critical | DISASTER | Combined WAN uptime < `{$WAN_UPTIME_HIGH}`% |
| Site has critical notification(s) in UniFi | AVERAGE | UniFi is reporting one or more critical notifications |
| Site aggregate WAN uptime degraded | AVERAGE | Combined WAN uptime < `{$WAN_UPTIME_WARN}`% |
| Site has 1 offline device | AVERAGE | One device has gone offline |
| Site has 1 offline WiFi device | AVERAGE | One AP has gone offline |
| Site has 1 offline wired device | AVERAGE | One switch/wired device has gone offline |
| Site has multiple offline devices | HIGH | More than one device offline simultaneously |
| Site has multiple offline WiFi devices | HIGH | More than one AP offline simultaneously |
| Site has multiple offline wired devices | HIGH | More than one wired device offline simultaneously |
| Site has offline gateway device(s) | HIGH | One or more gateways are offline |

All seven offline-count alarms require the condition on two consecutive samples (~2-3 minutes with the count items' 2m heartbeat) and are suppressed while the console itself is not cloud-connected - during a site WAN outage or failover these cloud-derived counts describe lost visibility, not real device state. The fast path for "something just died" is the per-device offline alarm (~1 minute); these are the slower site-level rollups (see [Site Outage Alarm Suppression](#site-outage-alarm-suppression)).
| IPS/IDS disabled or not configured | WARNING | Threat management is not active on this site |
| Site has device(s) with firmware updates | INFO | One or more devices have firmware updates queued |

### SD-WAN

Tunnel severity is proportional to actual impact. A dual-WAN spoke losing one WAN drops some tunnels but remains connected via its other WAN, so that's HIGH, not DISASTER. DISASTER only fires when a site is completely cut off or a hub loses all spoke connections.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Spoke(s) completely isolated, all tunnels down | DISASTER | One or more spokes have zero connected tunnels (all WANs lost) |
| Hub(s) have no active spoke connections | DISASTER | A hub has no spokes connected to it at all |
| SD-WAN config errors | HIGH | Configuration has errors that may prevent tunnels establishing |
| Hub WAN internet issues | HIGH | A hub site is reporting WAN internet issues |
| Spoke WAN internet issues | HIGH | A spoke site is reporting WAN internet issues |
| SD-WAN config generation failed | HIGH | Config generation status is not OK |
| SD-WAN tunnel(s) disconnected | HIGH | One or more tunnels are down (site may still have other active paths) |
| SD-WAN config warnings | WARNING | Configuration has warnings |

> **Example:** Guatemala has two WANs (WAN and WAN2), each with a tunnel to the UK hub. If WAN goes down, two tunnels drop → HIGH (`disconnectedTunnels`). WAN2 still has its tunnel, so Guatemala is not isolated. If both WANs go down → zero connected tunnels → DISASTER (`isolatedSpokes`).

### Controllers (discovered automatically)

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Controller not running | HIGH | A UniFi application (network, protect, etc.) has stopped |
| Controller unadopted device(s) found | WARNING | One or more devices are unadopted in this controller |
| Controller update available | INFO | A new software version is available for this application |
| Controller version changed | INFO | Application version changed (post-update tracking) |

### Access Points (UAP)

The following alarms come from the **Ubiquiti UniFi UAP** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Offline - not seen by local controller | HIGH | Device state ≠ 1 (alarms and clears on a single poll, no persistence window) |
| Device offline (cloud) | HIGH | Cloud API reports the device offline while its console still reads online in the same payload - a real single-device failure, not a site-wide outage. Fires on the first offline sample, ~1 minute (see [Site Outage Alarm Suppression](#site-outage-alarm-suppression)) |
| CPU usage critical | HIGH | CPU > `{$UNIFI.CPU.USAGE.HIGH}`% |
| Memory usage critical | HIGH | Memory > `{$UNIFI.MEM.USAGE.HIGH}`% |
| Radio not running | HIGH | Radio state is not RUN |
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Firmware upgrade available | WARNING | Local controller reports an upgrade is available |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold, indicating a recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |
| Firmware version changed | INFO | Firmware version changed |

### Switches (USW)

The following alarms come from the **Ubiquiti UniFi USW** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Offline - not seen by local controller | HIGH | Device state ≠ 1 (alarms and clears on a single poll, no persistence window) |
| Device offline (cloud) | HIGH | Cloud API reports the device offline while its console still reads online in the same payload - a real single-device failure, not a site-wide outage. Fires on the first offline sample, ~1 minute (see [Site Outage Alarm Suppression](#site-outage-alarm-suppression)) |
| CPU usage critical | HIGH | CPU > `{$UNIFI.CPU.USAGE.HIGH}`% |
| Memory usage critical | HIGH | Memory > `{$UNIFI.MEM.USAGE.HIGH}`% |
| SFP TX fault active | HIGH | Module reports a transmit laser fault, likely a failed SFP or broken TX fibre |
| SFP RX fault active | HIGH | Module reports a receive fault, likely a broken RX fibre or failed remote laser |
| SFP temperature critical | HIGH | SFP module temp > `{$UNIFI.SFP.TEMP.HIGH}`°C |
| SFP RX power critical low | HIGH | Received optical power < `{$UNIFI.SFP.RXPOWER.LOW.HIGH}` dBm, near receiver limit |
| SFP TX power critical low | HIGH | Transmit laser output < `{$UNIFI.SFP.TXPOWER.LOW.HIGH}` dBm, laser likely failing |
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Port down | WARNING | Port transitions from up to down after previously being up |
| Port TX errors increasing | WARNING | TX error counter is actively incrementing |
| Port RX errors increasing | WARNING | RX error counter is actively incrementing |
| Port RX dropped packets elevated | WARNING | More than `{$UNIFI.PORT.DROPPED.WARN}` RX drops within `{$UNIFI.PORT.DROPPED.WINDOW}` - not any single increase, since a low steady trickle is normal (congestion/buffer, not physical layer) |
| Port TX dropped packets elevated | WARNING | More than `{$UNIFI.PORT.DROPPED.WARN}` TX drops within `{$UNIFI.PORT.DROPPED.WINDOW}` - not any single increase, since a low steady trickle is normal |
| Port STP blocking | WARNING | Spanning Tree is actively blocking this port to prevent a loop |
| SFP temperature high | WARNING | SFP module temp > `{$UNIFI.SFP.TEMP.WARN}`°C |
| SFP RX power low | WARNING | Received optical power < `{$UNIFI.SFP.RXPOWER.LOW.WARN}` dBm |
| SFP TX power low | WARNING | Transmit laser output < `{$UNIFI.SFP.TXPOWER.LOW.WARN}` dBm |
| Firmware upgrade available | WARNING | Local controller reports an upgrade is available |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold, indicating a recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |
| Port renamed | INFO | Port name changed in UniFi (e.g. auto-named after a newly connected device) |
| SFP module changed | INFO | Module serial number changed, indicating a physical swap rather than a reseat |

> **Port down alarms** resolve immediately if the port comes back up. If the port remains down, the alarm auto-resolves after 24 hours. Operators can also close it manually for ports that are intentionally unused.

> **SFP monitoring** is discovered automatically and split into two tiers. Fault/metadata items (TX/RX fault, compliance, part number, serial, vendor, revision) are created for any port where `sfp_found = true`, including DAC (direct-attach copper) cables. Optical measurements (temperature, RX/TX power, voltage, current) are only created on non-DAC (fibre) modules, since DAC cables have no laser or photodiode and structurally cannot report those values - confirmed live by checking the same physical DAC cable from both ends. Voltage and laser current are logged for trending but have no alarm thresholds by default. All SFP alarms are suppressed if the device is offline. Supported models: any USW with SFP/SFP+ ports (confirmed on USW-Pro-Aggregation `USAGGPRO`).

> **Other per-port items** with no alarm attached, purely for visibility: `Anomalies`, `Full Duplex`, `Autoneg`, `Is Uplink`, `Link Down Count` (cumulative flap counter), `MAC Table Count` (learned MAC addresses on the port), `STP Edge Port`, and `QoS Mode` (surfaces Pro AV/traffic-control profiles such as `aes67_audio`).

### UPS (Uninterruptible Power Supply)

The following alarms come from the **Ubiquiti UniFi UPS** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Offline - not seen by local controller | HIGH | Device state ≠ 1 (alarms and clears on a single poll, no persistence window) |
| Device offline (cloud) | HIGH | Cloud API reports the device offline while its console still reads online in the same payload - a real single-device failure, not a site-wide outage. Fires on the first offline sample, ~1 minute (see [Site Outage Alarm Suppression](#site-outage-alarm-suppression)) |
| Battery level critical | HIGH | Battery < `{$UNIFI.UPS.BATTERY.HIGH}`% |
| Battery runtime critical | HIGH | Remaining runtime < `{$UNIFI.UPS.RUNTIME.HIGH}` seconds |
| Running on battery, mains power failed | HIGH | UPS is in battery mode |
| Battery level low | WARNING | Battery < `{$UNIFI.UPS.BATTERY.WARN}`% |
| Outlet lost power | WARNING | Outlet relay state = off |
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold, indicating a recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |
| Firmware version changed | INFO | Firmware version changed |

### Redundant Power Supply (RPS)

The following alarms come from the **Ubiquiti UniFi RPS** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| RPS delivering 12V, connected device PSU failed | DISASTER | RPS is actively supplying 12V to a device |
| RPS delivering 54V, connected device PSU failed | DISASTER | RPS is actively supplying 54V (PoE) to a device |
| RPS port ACTIVE, primary PSU failed on connected device | DISASTER | RPS port has taken over from a failed PSU |
| Offline - not seen by local controller | HIGH | Device state ≠ 1 (alarms and clears on a single poll, no persistence window) |
| Device offline (cloud) | HIGH | Cloud API reports the device offline while its console still reads online in the same payload - a real single-device failure, not a site-wide outage. Fires on the first offline sample, ~1 minute (see [Site Outage Alarm Suppression](#site-outage-alarm-suppression)) |
| Temperature critical | HIGH | Temp > `{$UNIFI.TEMP.HIGH}`°C |
| Temperature high | WARNING | Temp > `{$UNIFI.TEMP.WARN}`°C |
| RPS port disconnected, redundancy lost | WARNING | Port is not connected; device has no redundant power path |

> **RPS port discovery** only creates items for ports that have a connected device. Ports with a generic name (e.g. `Port 1`, `Port 2`) are filtered out by design, since those ports have nothing plugged in yet. Once a device is connected, the RPS names the port after the device's hostname (e.g. `FRN1CORESW01`), and it's automatically discovered on the next LLD cycle.
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold, indicating a recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |
| Firmware version changed | INFO | Firmware version changed |

### UniFi Protect (Cameras, NVR & Other Devices)

Camera alarms come from the **Ubiquiti UniFi Protect Camera** template; the NVR-level alarms below come from the **Ubiquiti UniFi API Host** template (the same host already used for Network monitoring on that console - Protect's NVR state is added there, not to a separate host). Unlike UAP/USW/UPS/RPS, there is no cloud-side base template applied to Protect devices - Protect isn't part of the Network cloud fleet API, so there's no "offline (cloud)" cross-check available here.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Camera offline - not seen by Protect | HIGH | Camera state is not `CONNECTED` |
| Protect detected a recording gap | HIGH | Protect's own gap detector (`isMissingRecordingDetected`) has flagged missed recording - already accounts for the camera's configured mode, so it won't false-fire on motion-only/scheduled cameras |
| Protect database corruption detected | HIGH | NVR reports its recording database state as not `healthy` |
| Protect recording is globally disabled | HIGH | NVR-wide recording toggle is off - no camera on the console is recording regardless of its own settings |
| Protect storage critical | HIGH | Recording storage utilization > `{$UNIFI.PROTECT.STORAGE.HIGH}`% - deliberately high (99% default): continuous recording is designed to run near-full and auto-overwrite the oldest footage, so normal steady-state is commonly mid-to-high 90s and not itself a problem |
| Protect disk not healthy / state not normal | HIGH | Per-disk health or state reported by the NVR is not the expected value |
| Protect disk temperature critical | HIGH | Disk temperature > `{$UNIFI.PROTECT.DISK.TEMP.HIGH}`°C |
| Protect disk array capability not ok | HIGH | The underlying disk/RAID array's own capability check (independent of Protect's own storage accounting) is not `ok` |
| Camera recordings paused | WARNING | Recording explicitly paused for this camera - check the paused-reason item |
| Protect camera license capacity not ok | WARNING | NVR's camera license/capacity state is not `ok` |
| Protect disk has bad sector(s) | WARNING | Reallocated/bad sectors reported on an NVR disk |
| Protect disk temperature high | WARNING | Disk temperature > `{$UNIFI.PROTECT.DISK.TEMP.WARN}`°C |
| Protect reports camera(s) offline (NVR-side count) | WARNING | NVR's own aggregate offline-camera count is above zero - a belt-and-suspenders check alongside the per-camera trigger above |
| Firmware upgrade available | WARNING | Local Protect controller reports an upgrade is available |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold, indicating a recent reboot |
| Protect hard drive not officially compatible | INFO | NVR reports the installed drive as unsupported (`hddNotCompatible`) - a compatibility/support notice, not an indicator of an active fault; cross-check the per-disk Health/State/Bad Sectors alarms above for actual drive problems |
| Protect hard drive state changed | INFO | Safety net: fires on any change to the drive compatibility/health value, even ones not yet covered by a dedicated alarm |
| Protect disk action in progress | INFO | NVR is actively performing an operation on a disk (e.g. a RAID rebuild) - check that disk's Rebuild Progress/Estimate items |
| Protect disk serial number changed | INFO | A disk's reported serial number changed, indicating a physical drive swap - same idiom as the existing SFP module serial-change alarm |
| SSH access enabled | INFO | Often a legitimate, intentional support toggle - informational only |

> **Only one denylisted `hardDriveState` value is currently alarmed** (`hddNotCompatible`) - no confirmed "healthy" string has been observed to build a full allowlist against. The INFO change-tracking alarm above ensures any other value is still visible even before it gets a dedicated trigger.

> **Other per-camera/per-disk items with no alarm**, purely for visibility: camera `Link Speed` (phyRate - mirrors the no-alarm treatment of satisfaction/channel-utilization items elsewhere in this template), `Is Recording`, `Recording Mode`, `Poor Network Flag`, `IR/Night Mode`, `Last Seen`, `Last Ring`; NVR `Storage Allocation Mode`, `Camera License Utilization`; per-disk `State Reason`, `Rebuild Progress`, `Rebuild Estimate`, `Size`, `Power-On Hours`. `Disk Array Total/Used/Available` (from the underlying filesystem, distinct from Protect's own recording-space accounting above) are also data-only today - the two views measure slightly different things and can disagree by a few percent.

> **Retention estimates (`estimatedHqRetentionDays`/`estimatedLqRetentionDays`) are deliberately not monitored.** UniFi Protect's own UI shows an "Estimated N Days" figure per quality tier, but confirmed live (including a fresh poll) that both underlying bootstrap fields are `null` regardless - the UI's number is computed client-side from raw rate/capacity stats using a formula that isn't exposed by any endpoint tested. Guessing at that formula risks showing a confidently wrong number, which is worse than showing none - `Disk Array Available`/`Protect Storage Available` above are reliable, always-populated proxies for capacity awareness instead.

> **"Discover Host Protect Other"** covers chimes today, and will pick up sensors/lights/sirens/viewers automatically if you adopt them later - same alarm set as above, minus anything recording-specific (those device types have no recording concept).

---

## Site Discovery Filtering

The UniFi Site Manager API does not currently enforce an API key's configured "Sites" restriction on the `GET /v1/hosts` and `GET /v1/sites` endpoints. Both always return every console on the account, regardless of which sites the key is scoped to in the UniFi UI. Since the template relies on these endpoints for host discovery, a restricted API key will still cause every console on the account to be discovered.

`{$UNIFI.SITE.EXCLUDE}` is a client-side workaround: a comma-separated list of consoles to drop before host prototypes are created. Each entry can be any of:

- The console hostname, as shown in Site Manager (e.g. `MyConsole`)
- The full host ID, copied from the console's URL at `unifi.ui.com/consoles/<host-id>/network/default` (e.g. `AABBCCDDEEFF00000000000000000000000000000000000000000000112233:1234567890`)
- Just the ID portion before the colon (e.g. `AABBCCDDEEFF00000000000000000000000000000000000000000000112233`)

Forms can be mixed in one value, e.g. `MyConsole,AABBCCDDEEFF00000000000000000000000000000000000000000000112233`.

Set this on the root host (**Data collection > Hosts > Ubiquiti UniFi API > Macros**), then re-run discovery. It's empty by default, so nothing is filtered unless configured.

---

## WAN Alarm Suppression

For gateways where a WAN port is intentionally unused, set a host-level context macro to suppress both the unplugged and 0% uptime alarms for that WAN:

| Macro | Value | Effect |
|-------|-------|--------|
| `{$UNIFI.WAN.ENABLED:WAN2}` | `0` | Suppresses all WAN2 alarms (uptime and plugged) |
| `{$UNIFI.WAN.ENABLED:WAN1}` | `0` | Suppresses all WAN1 alarms (not recommended on a primary) |
| `{$UNIFI.WAN.ENABLED}` | `0` | Suppresses WAN alarms for all WANs on this host |

**To add a host macro:**
1. Go to **Data collection > Hosts** and open the console host
2. Click the **Macros** tab
3. Add the macro name and value, then click **Update**

Changes take effect on the next trigger evaluation (within one poll cycle).

---

## Finding your site name

`{$UNIFI.API.SITE}` is the site's **internal** name, which appears in every local API path. It is not the label shown in the UniFi interface — that's a separate field, and the two are frequently different.

Nearly every install calls its first site `default`, which is why this macro can normally be left alone. A site created by hand gets a generated identifier instead, something like `a1b2c3d4`, and renaming it in the interface only changes the display label. Replacing or deleting the stock default site leaves that generated name in place permanently.

If local items fail with `api.err.NoSiteContext`, this is why. To find your site name, log in to the console and open:

```
https://<console-ip>/proxy/network/api/self/sites
```

```json
{"meta":{"rc":"ok"},"data":[{"name":"default","desc":"Default","role":"admin", ...}]}
```

Use the **`name`** field. Ignore `desc` (the display label), `_id`, `external_id` and `anonymous_id` — none of those appear in an API path.

If `data` comes back empty, that's a different problem: the account can sign in to UniFi OS but holds no role inside the Network application, which is a separate permission grant from console access.

Set `{$UNIFI.API.SITE}` once on the root host and discovery propagates it to every console and device.

---

## Site Outage Alarm Suppression

When a site loses its WAN (or fails over between two WANs), the console disconnects from the Ubiquiti cloud and the cloud API marks **every device at that site** `offline` - not because the devices went down, but because nothing can report on them any more. Without suppression this fans out into one HIGH alert per AP and switch, followed minutes later by the actual WAN/tunnel alert, and then a matching flood of recovery emails when the WAN comes back.

The templates treat "the cloud lost sight of the site" and "a device actually died" as different conditions:

- **Per-device "is offline" (cloud)** fires on a dedicated `Offline State` item whose preprocessing reads both the device's status *and* its console's status from the *same* `/v1/devices` payload (the console lists itself there with `isConsole: true`), and resolves them to one value: `0` = online, `1` = device offline while the console still reads online (a genuine single-device failure - alarm), `2` = device offline but the console is offline too (the cloud can't see the site at all, so per-device statuses are meaningless - stay silent). Because the distinction is made atomically inside one item from one payload, there is no cross-item or cross-host polling race and **no confirmation delay: a genuine failure alerts on the first offline sample (~1 minute)**, while a site outage or WAN failover - however brief - resolves to `2` and never alarms. A problem that was already open when the site went dark is held open rather than flap-closed (the trigger recovers only on state `0`), and a device that stays offline after the site comes back alerts within a poll cycle.
- **Site-level offline counts** (offline devices / WiFi / wired / gateway, no-gateway) live on the console host itself, so they gate directly on the console's `Connection State` item, plus a two-consecutive-samples requirement (~2-3 minutes with the count items' 2m heartbeat) that closes the sub-minute race between the counts and the state item, which come from different master items on independent poll schedules. These are aggregate rollups - the per-device alarm above is the fast path.
- **"Console lost cloud connectivity"** (DISASTER) only fires after the console has been disconnected for the whole of `{$UNIFI.CONSOLE.OFFLINE.GRACE}` (default `3m`), so a 1-2 minute WAN failover doesn't alarm at all. It recovers on the first `connected` sample.
- **Local-controller polling** (per-device state, Protect cameras) needs no gating: when the site is unreachable the poll itself fails, the items go unsupported, and the triggers simply hold their last state.

The net result for a sustained site outage is two meaningful alerts - the console DISASTER (after the grace period) plus whatever WAN/SD-WAN monitoring you have - instead of one per device. For a brief WAN failover, nothing fires.

---

## High Availability (Shadow Mode)

For deployments using UniFi HA (two gateway units in Shadow Mode):

- **HA failover: gateway is now STANDBY** fires at DISASTER severity when the primary unit fails and this unit takes over active routing
- **Shadow Mode link (eth6) physically disconnected** fires at HIGH when the HA interconnect cable is unplugged, meaning redundancy is lost even if the primary is still running
- **Shadow Mode peer IP changed** fires at INFO if the management IP of the standby unit changes unexpectedly
- Trigger dependencies suppress downstream device alarms when the console itself goes offline

No additional configuration is required - HA status is detected automatically from the local controller's Shadow Mode API.

---

## Troubleshooting

**Discovery runs but no hosts are created**
- Verify `{$APIKEY}` is set correctly on the root host
- Check the Zabbix server can reach `api.ui.com`
- Check the `unifi.api.hosts` item for errors under **Monitoring > Latest data**

**Local items show connection errors or timeouts**
- Confirm `{$UNIFI.LOCAL.IP}` is populated on the console host (set automatically by discovery)
- Confirm the Zabbix server/proxy can reach that IP on port 443
- Check `{$UNIFI.USERNAME}` and `{$UNIFI.PASSWORD}` are set and the account exists locally on that console
- Some consoles require the account to have logged in via the UI at least once before API auth succeeds
- Brief timeouts (10s) are expected if the console temporarily loses management connectivity; these clear automatically

**WAN uptime or latency shows 0 / incorrect values**
- Both items are sourced from the local controller (`/stat/health`). If the local controller is unreachable, they fall back to safe defaults (100% / 0ms) until the connection recovers
- Confirm `Local Health Raw` is collecting successfully under **Monitoring > Latest data**

**WAN2 alarms on a single-WAN gateway**
- Set host macro `{$UNIFI.WAN.ENABLED:WAN2}=0`, see [WAN Alarm Suppression](#wan-alarm-suppression)

**False reboot alarms on newly-discovered devices**
- The startup time trigger requires the stored timestamp to change by more than 5 minutes, filtering polling jitter
- If still noisy on first discovery, acknowledge and close; the alarm only fires on genuine changes thereafter

**Port error alarms that won't clear automatically**
- Port TX/RX error triggers use cumulative counters and are marked `manual_close`
- Acknowledge and close the alarm in Zabbix once the underlying issue is resolved

**HTTP 429 errors from the local controller**
- Device hosts no longer log in themselves (see [Polling Architecture](#polling-architecture)), so this should be rare. The only thing that logs in under normal operation is `Local Session + Cloud Devices Raw` (`unifi.host.session`), on its own `{$UNIFI.SESSION.REFRESH.INTERVAL}` schedule (default 10m). `unifi.local.all` doesn't log in routinely at all - it only does a one-off fresh login itself if its cached token turns out to be rejected
- A rate-limited or failed poll is simply skipped rather than misread as the device being down, so an occasional `429` is not something you need to act on

**`unifi.protect.bootstrap` reports HTTP 403**
- The account has Network access but not Protect access - these are two separate permission grants in UniFi OS. Open the console's admin panel, find the account used for `{$UNIFI.USERNAME}`, and check the **Protect** role dropdown specifically (not just the overall Admin Privileges dropdown) is set to **View Only** or higher, then click through to actually save it. It's possible for the dropdown to visually show a value that hasn't been persisted server-side - if the account still 403s after saving, reload the admin page and confirm the setting held
- This item logs a specific message naming the missing grant (`Zabbix.log` level 4 / Debug) if you want to confirm this is the cause before changing anything

**No Protect hosts (cameras/NVR items) appear for a console that does run Protect**
- Confirm `{$UNIFI.SESSION.TOKEN}` is populated on the console host - Protect discovery depends on the same cached token as Network device discovery
- Check `unifi.protect.bootstrap` under **Monitoring > Latest data** on the console host for errors
- A console with Protect genuinely not installed will show an empty result here with no error - this is expected, not a fault (see [Polling Architecture](#polling-architecture))

**Camera or chime host also appears as a generic device under a console's inventory**
- Protect devices are excluded from "Discover Host Other Devices" by their `productLine` field (`protect` vs `network`), confirmed against a live cloud API response before this was added. If you see a duplicate, check whether that specific device's `productLine` is actually reported as something other than `protect` - if so, it's worth reporting as an edge case, since every Protect device checked during development reported `protect` consistently regardless of type (camera, doorbell, chime)
- If a device's cached `{$UNIFI.SESSION.TOKEN}` is ever rejected, that device performs one fresh login itself to recover the same cycle - a burst of these across many devices at once would point at the console-level token propagation being broken (check `Local Session + Cloud Devices Raw` under **Monitoring > Latest data**) rather than the per-device rate limit itself
