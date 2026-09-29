# JuiceBox Local Mode + InfluxDB + Grafana Design

**Date:** 2026-03-17
**Status:** Approved

## Overview

Run juicepassproxy in full local mode (EnelX servers are offline), publish JuiceBox telemetry to the existing `farm-sensors` InfluxDB/Grafana stack via MQTT + Telegraf.

## Goals

- Intercept JuiceBox UDP traffic locally — no dependency on EnelX cloud
- Publish JuiceBox metrics into the existing `sensors` InfluxDB bucket
- Visualize charging state, power, energy, and electrical data in Grafana

## Architecture

```
JuiceBox (UDP)
     │
     │  DNS override: juicenet-udp-prod3-usa.enelx.com → host IP
     ▼
juicepassproxy (standalone container, port 8047/UDP)
     │  ignore_enelx=true → acts as server, no EnelX forwarding
     │  MQTT publish:
     │    homeassistant/+/+/+/config  (HA discovery, retained)
     │    hmd/+/+/+/state             (state updates)
     ▼
farm-mosquitto (port 1883)
     ▼
farm-telegraf (MQTT consumer additions)
     ▼
farm-influxdb (org: bloom-and-grow, bucket: sensors, measurement: juicebox)
     ▼
farm-grafana (new provisioned dashboard)
```

## MQTT Topic Structure

`ha-mqtt-discoverable==0.13.1` uses **two separate topic prefixes**:

- `homeassistant/` — HA auto-discovery **config** messages only (retained JSON payloads)
- `hmd/` — all **state** topic updates

State topic format: `hmd/<component>/<device_name>/<entity_name>/state`

Entity names are sanitized by `clean_string()`: every character outside `[A-Za-z0-9_-]` becomes a hyphen. Spaces, parentheses, and slashes all become `-`.

**Known sanitized entity names** (device_name = `JuiceBox`):

| Entity (code name) | Sanitized topic segment | Component |
|--------------------|------------------------|-----------|
| Status | `Status` | sensor |
| Current | `Current` | sensor |
| Current Rating | `Current-Rating` | sensor |
| Max Current(Offline/Device) | `Max-Current-Offline-Device-` | sensor |
| Max Current(Offline/Wanted) | `Max-Current-Offline-Wanted-` | number |
| Max Current(Online/Device) | `Max-Current-Online-Device-` | sensor |
| Max Current(Online/Wanted) | `Max-Current-Online-Wanted-` | number |
| Frequency | `Frequency` | sensor |
| Energy (Lifetime) | `Energy--Lifetime-` | sensor |
| Energy (Session) | `Energy--Session-` | sensor |
| Temperature | `Temperature` | sensor |
| Voltage | `Voltage` | sensor |
| Power | `Power` | sensor |
| Power Factor | `Power-Factor` | sensor |
| Last Debug Message | `Last-Debug-Message` | sensor |
| Act as Server | `Act-as-Server` | switch |

Example full state topic: `hmd/sensor/JuiceBox/Power/state`

**Verify on first startup:**
```bash
mosquitto_sub -h <host-ip> -t 'hmd/#' -v | head -60
```
Confirm the exact topic strings before writing the Telegraf consumer config. Note especially the trailing/doubled hyphens on some entity names.

## Component 1: juicepassproxy Standalone Container

**Location:** `/home/dallen/juicepassproxy/docker-compose.yml`

**Key environment variables:**

| Variable | Value | Notes |
|----------|-------|-------|
| `JUICEBOX_HOST` | `<juicebox-lan-ip>` | Required: satisfies startup validation AND telnet ID discovery on first run |
| `IGNORE_ENELX` | `true` | JPP acts as server; no traffic forwarded to EnelX |
| `MQTT_HOST` | `<host-lan-ip>` | Host machine LAN IP — mosquitto is a separate compose stack |
| `MQTT_PORT` | `1883` | |
| `MQTT_USER` | _(if Mosquitto auth enabled)_ | Check `config/mosquitto/mosquitto.conf` for `allow_anonymous` |
| `MQTT_PASS` | _(if Mosquitto auth enabled)_ | |
| `LOG_LOC` | `/log` | Writes rotated logs to the log volume |

**Startup validation note:** `juicepassproxy.py` lines 401–405 require either `--enelx_ip` OR `--juicebox_host` regardless of `--ignore_enelx`. `JUICEBOX_HOST` satisfies this check. `ENELX_IP` is not needed.

**Restart budget:** `MAX_JPP_LOOP = 10` (from `const.py`). If Mosquitto is unreachable at JPP startup, JPP will exhaust this budget and exit permanently. `farm-sensors` stack must be running before juicepassproxy starts. Use `restart: unless-stopped` to recover from transient failures; document startup order.

**Volumes:**
- `./config:/config` — persists JuiceBox ID, current_rating, and current set values
- `./log:/log` — 14-day rotating logs (from `const.py:DAYS_TO_KEEP_LOGS`)

**Port:** `8047:8047/udp` exposed on host.

**`act_as_server` switch:** Initialised `ON` by default. Enables JPP to respond to JuiceBox status messages with command messages, maintaining the current control loop without EnelX.

## Component 2: UniFi DNS Override

In UniFi Network → Settings → DNS Overrides, add:

```
juicenet-udp-prod3-usa.enelx.com → <host-machine-lan-ip>
```

No `--update_udpc` or JuiceBox telnet manipulation needed. The JuiceBox's existing UDPC config points to this hostname; the DNS override silently redirects UDP traffic to the proxy.

**Verify JuiceBox port:** `telnet <juicebox-ip> 2000`, then `list` — check the `UDPC` line. Typically `8047`.

## Component 3: Telegraf MQTT Consumer Additions

**Location:** `/home/dallen/farm-sensors/config/telegraf/telegraf.conf` (append)

Three new consumer blocks — each needs a unique `client_id` (existing consumers use `telegraf-farm-sensors` and `telegraf-farm-json`).

### Numeric sensor consumer
- `client_id = "telegraf-juicebox-numeric"`
- `servers = ["tcp://mosquitto:1883"]` (Telegraf is inside the farm-sensors compose network)
- Topics: `hmd/sensor/JuiceBox/+/state` and `hmd/number/JuiceBox/+/state`
- `data_format = "value"`, `data_type = "float"`
- `topic_tag = "topic"` — full topic stored as tag, then regex processor extracts `entity`
- `Act-as-Server` is a `switch` component and never matches these topic subscriptions
- `Status` and `Last-Debug-Message` are `sensor` components and will be matched — exclude them via `tagdrop` after the regex processor (see below)
- Tags: `device=juicebox`
- Measurement: `juicebox`

### String consumer (Status only)
- `client_id = "telegraf-juicebox-string"`
- `servers = ["tcp://mosquitto:1883"]`
- Topic: `hmd/sensor/JuiceBox/Status/state`
- `data_format = "string"`
- Tags: `device=juicebox`, `entity=Status`
- Measurement: `juicebox`

### Regex processor and tagdrop (tag extraction + exclusion)

Extract `entity` tag from topic, then drop string-valued entities from the numeric measurement:

```toml
[[processors.regex]]
  namepass = ["juicebox"]
  [[processors.regex.tags]]
    key = "topic"
    pattern = "^hmd/[^/]+/JuiceBox/([^/]+)/state$"
    replacement = "${1}"
    result_key = "entity"

[[processors.filter]]
  namepass = ["juicebox"]
  [processors.filter.tagdrop]
    entity = ["Status", "Last-Debug-Message"]
```

The `namepass = ["juicebox"]` guard prevents these processors from interfering with existing Hubitat/farm sensor metrics in the same Telegraf instance.

### Flux query pattern

```flux
// Numeric entity (example: Power)
from(bucket: "sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "juicebox" and r.entity == "Power")

// Lifetime energy in kWh
from(bucket: "sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "juicebox" and r.entity == "Energy--Lifetime-")
  |> map(fn: (r) => ({r with _value: r._value / 1000.0}))

// Status (string field)
from(bucket: "sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "juicebox" and r.entity == "Status")
```

## Component 4: Grafana Dashboard

**Location:** `/home/dallen/farm-sensors/config/grafana/provisioning/dashboards/json/juicebox.json`

Provisioned automatically (the provider scans the `json/` subdirectory — confirmed in `dashboards.yml:13`). Available within 30 seconds of Grafana startup.

### Panels

| Panel | Type | `entity` tag value | Unit | Notes |
|-------|------|--------------------|------|-------|
| Status | Stat (text) | `Status` | — | Green=Charging, Yellow=Plugged In, Grey=Unplugged, Red=Error |
| Power | Gauge + time series | `Power` | W | |
| Current | Time series | `Current` | A | |
| Voltage | Time series | `Voltage` | V | Flat ~240V |
| Session Energy | Stat | `Energy--Session-` | Wh | Resets per charge |
| Lifetime Energy | Stat | `Energy--Lifetime-` | kWh | `÷ 1000` in Flux |
| Temperature | Time series | `Temperature` | °F | |
| Frequency | Gauge | `Frequency` | Hz | Nominal 60 Hz |
| Max Current (Online) | Stat | `Max-Current-Online-Device-` | A | |
| Max Current (Offline) | Stat | `Max-Current-Offline-Device-` | A | |

## Implementation Order

1. Deploy juicepassproxy container; verify it starts and the JuiceBox ID is discovered (check `./config/` for stored ID)
2. Subscribe `mosquitto_sub -h <host-ip> -t 'hmd/#' -v` — confirm all entity state topics and their exact names
3. Adjust entity topic names in this spec if observed topics differ from the sanitized names above
4. Add Telegraf consumer blocks to `telegraf.conf`; restart `farm-telegraf`
5. Verify data in InfluxDB: `task query` in farm-sensors, filter by `_measurement == "juicebox"`
6. Write `juicebox.json` Grafana dashboard and place in the `json/` provisioning directory
7. Verify dashboard loads in Grafana within 30 seconds

## Out of Scope

- Current control / setting online amperage via Grafana (read-only dashboard)
- Multiple JuiceBox instances
- HA MQTT discovery config cleanup (retained messages)
- TLS on MQTT connection
- Encrypted JuiceBox protocol (`v09e`, `v08`) — not supported by JPP
