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
     │  MQTT publish: homeassistant/+/+/state topics
     ▼
farm-mosquitto (port 1883)
     ▼
farm-telegraf (MQTT consumer additions)
     ▼
farm-influxdb (org: bloom-and-grow, bucket: sensors, measurement: juicebox)
     ▼
farm-grafana (new provisioned dashboard)
```

## Component 1: juicepassproxy Standalone Container

**Location:** `/home/dallen/juicepassproxy/docker-compose.yml`

**Key environment variables:**

| Variable | Value | Notes |
|----------|-------|-------|
| `JUICEBOX_HOST` | `<juicebox-lan-ip>` | Required for startup validation (see note below) and telnet ID discovery on first run |
| `IGNORE_ENELX` | `true` | JPP acts as the server; no traffic forwarded to EnelX |
| `MQTT_HOST` | `<host-lan-ip>` | Host machine LAN IP — mosquitto is a separate compose stack, container-name DNS won't resolve |
| `MQTT_PORT` | `1883` | |
| `MQTT_USER` | _(if auth enabled)_ | Mosquitto `allow_anonymous false` requires this |
| `MQTT_PASS` | _(if auth enabled)_ | |
| `LOG_LOC` | `/log` | Writes rotated logs to the log volume |

**Startup validation note:** `juicepassproxy.py` lines 401–405 require either `--enelx_ip` OR `--juicebox_host` to be set, regardless of `--ignore_enelx`. This check fires before `ignore_enelx` is evaluated. Since `JUICEBOX_HOST` is needed anyway for telnet ID discovery, it satisfies this requirement. `ENELX_IP` is not needed.

**Volumes:**
- `./config:/config` — persists JuiceBox ID, current_rating, and current set values between restarts
- `./log:/log` — rotated daily, 7-day retention

**Port:** `8047:8047/udp` exposed on host

**JuiceBox ID discovery:** On first start, JPP telnets to `JUICEBOX_HOST:2000` to read the device serial. This serial becomes the HA MQTT `unique_id` prefix and is stored in `config/` for subsequent starts.

**`act_as_server` switch:** Initialised `ON` by default in JPP code (`juicebox_mqtthandler.py:403`). This enables JPP to respond to JuiceBox status messages with command messages, maintaining the current control loop without EnelX.

**Startup ordering:** `farm-mosquitto` must be running before juicepassproxy starts. JPP has a fixed restart budget (`MAX_JPP_LOOP`). Use `restart: unless-stopped` and document that `farm-sensors` stack must be started first. A simple solution is to add a `healthcheck` or rely on the `restart` policy to recover.

## Component 2: UniFi DNS Override

In UniFi Network → Settings → DNS Overrides (or under WAN/LAN DNS settings depending on firmware), add a local DNS A record:

```
juicenet-udp-prod3-usa.enelx.com → <host-machine-lan-ip>
```

This silently redirects the JuiceBox's UDP traffic to juicepassproxy. No `--update_udpc` or telnet manipulation of the JuiceBox needed.

**Verify JuiceBox port:** `telnet <juicebox-ip> 2000`, then `list` — check the `UDPC` line. Typically port `8047`.

## Component 3: Telegraf MQTT Consumer Additions

**Location:** `/home/dallen/farm-sensors/config/telegraf/telegraf.conf` (additions appended)

### MQTT topic structure — verify before finalising config

juicepassproxy uses `ha-mqtt-discoverable==0.13.1`. The state topic for each entity is derived from its `unique_id`, which is constructed as `"{juicebox_id} {entity_name}"` (see `juicebox_mqtthandler.py:44`).

**Critical:** Entity names include spaces, parentheses, and forward slashes (e.g. `Max Current(Offline/Device)` — the `/` creates extra MQTT topic levels). The library may sanitize these characters before building topic strings, or it may not.

**Required verification step:** After first container startup, run:
```bash
mosquitto_sub -h <host-ip> -t 'homeassistant/#' -v | head -40
```
Note the actual state topic paths before writing the Telegraf consumer config. The Telegraf topic subscriptions and regex processor patterns below are templates that must be adjusted to match the observed topics.

### Entities and expected data types

| Entity | MQTT component | Value type | Unit |
|--------|---------------|------------|------|
| Status | sensor | string | Charging / Plugged In / Unplugged / Error |
| Current | sensor | float | A |
| Current Rating | sensor | float | A |
| Max Current(Offline/Device) | sensor | float | A |
| Max Current(Offline/Wanted) | number | float | A |
| Max Current(Online/Device) | sensor | float | A |
| Max Current(Online/Wanted) | number | float | A |
| Frequency | sensor | float | Hz |
| Energy (Lifetime) | sensor | float | Wh |
| Energy (Session) | sensor | float | Wh |
| Temperature | sensor | float | °F |
| Voltage | sensor | float | V |
| Power | sensor | float | W |
| Power Factor | sensor | float | — |

### Two consumer blocks required

**Numeric consumer** (`data_format = "value"`, `data_type = "float"`):
- Subscribes to juicebox sensor/number state topics (all except Status)
- `client_id = "telegraf-juicebox-numeric"` (must be unique — existing consumers use `telegraf-farm-sensors` and `telegraf-farm-json`)
- Topic regex processor extracts `entity` tag from the topic path (strip the `{juicebox_id}` prefix and trailing `/state`)
- Tags all metrics: `device=juicebox`
- Measurement: `juicebox`
- The Status topic must be excluded from this consumer to avoid float-parse errors and log noise on every status update. Use a `metric_filter` or `topic_filter` once actual topic strings are known.

**String consumer** (`data_format = "string"`):
- Subscribes to the Status entity state topic only
- `client_id = "telegraf-juicebox-string"`
- Tags: `device=juicebox`, `entity=status`
- Measurement: `juicebox`
- Stores as string field for Grafana text display

### Flux query pattern

```flux
// Numeric entity
from(bucket: "sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "juicebox" and r.entity == "Power")

// Status
from(bucket: "sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "juicebox" and r.entity == "status")
```

Note: The exact `entity` tag value depends on what the regex processor extracts from the actual MQTT topic. This must be confirmed after the verification step above.

## Component 4: Grafana Dashboard

**Location:** `/home/dallen/farm-sensors/config/grafana/provisioning/dashboards/json/juicebox.json`

(The provisioning config at `dashboards.yml` scans the `json/` subdirectory — not the `dashboards/` root.)

Provisioned automatically. File is picked up within `updateIntervalSeconds: 30` of the Grafana container starting, or on the next reload cycle.

### Panels

| Panel | Type | Entity tag | Unit | Notes |
|-------|------|------------|------|-------|
| Status | Stat (text, colour-mapped) | `status` | — | Green=Charging, Yellow=Plugged In, Grey=Unplugged, Red=Error |
| Power | Gauge + time series | `Power` | W | |
| Current | Time series | `Current` | A | |
| Voltage | Time series | `Voltage` | V | Flat ~240V; useful for grid anomaly detection |
| Session Energy | Stat | `Energy (Session)` | Wh | Resets per charge session |
| Lifetime Energy | Stat | `Energy (Lifetime)` | kWh | Divide by 1000 in Flux: `\|> map(fn: (r) => ({r with _value: r._value / 1000.0}))` |
| Temperature | Time series | `Temperature` | °F | |
| Frequency | Gauge | `Frequency` | Hz | Alert threshold if outside 59.5–60.5 |
| Max Current (Online) | Stat | `Max Current(Online/Device)` | A | |
| Max Current (Offline) | Stat | `Max Current(Offline/Device)` | A | |

**Note:** Entity tag values in the table above use the entity names as they appear in the code. These will need to match whatever the Telegraf regex extracts from the actual MQTT topics — adjust after the verification step in Component 3.

## Out of Scope

- Current control / setting online amperage via Grafana (read-only dashboard)
- Multiple JuiceBox instances
- HA MQTT discovery cleanup (retained config messages)
- TLS on MQTT connection
- Encrypted JuiceBox protocol (`v09e`, `v08`) — not supported by JPP

## Implementation Order

1. Deploy juicepassproxy container; verify it starts and the JuiceBox ID is discovered
2. Run `mosquitto_sub` to capture actual MQTT topic strings
3. Write Telegraf consumer config using observed topic patterns
4. Restart `farm-telegraf`; verify data appears in InfluxDB
5. Write and provision Grafana dashboard JSON
