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

**Key configuration:**
- `IGNORE_ENELX=true` — JPP acts as the server; JuiceBox never contacts EnelX
- `JUICEBOX_HOST=<juicebox-lan-ip>` — used on first run to telnet-query the JuiceBox ID; stored in config for subsequent starts
- `ENELX_IP=54.161.185.130` — provided explicitly to skip DNS resolution (not contacted, required by MITM handler constructor)
- `MQTT_HOST=<host-lan-ip>` — host machine LAN IP; mosquitto is a separate compose stack so container-name DNS won't resolve
- `MQTT_PORT=1883`
- `LOCAL_PORT=8047` — UDP listen port
- `LOG_LOC=/log`
- Port `8047/udp` exposed on host

**Volumes:**
- `./config:/config` — persists JuiceBox ID, current_rating, and set values between restarts
- `./log:/log` — rotated logs

**JuiceBox ID discovery:** On first start, JPP telnets to `JUICEBOX_HOST:2000` to read the device serial (used as HA MQTT unique_id). Stored in `config/`. Subsequent starts skip telnet.

**`act_as_server` switch:** Initialised `ON` by default in JPP code. This enables JPP to respond to JuiceBox status messages with command messages, maintaining the charging current control loop.

## Component 2: UniFi DNS Override

In UniFi Network → Settings → DNS Overrides, add a local DNS record:

```
juicenet-udp-prod3-usa.enelx.com → <host-machine-lan-ip>
```

This silently redirects the JuiceBox's UDP traffic to juicepassproxy without modifying the device. No `--update_udpc` needed.

**Note:** The JuiceBox port is typically `8047`. Verify with: `telnet <juicebox-ip> 2000`, then `list` command — look for the `UDPC` line.

## Component 3: Telegraf MQTT Consumer Additions

**Location:** `/home/dallen/farm-sensors/config/telegraf/telegraf.conf` (additions)

Two new `[[inputs.mqtt_consumer]]` blocks added to the existing config:

### Numeric consumer
Subscribes to all juicepassproxy state topics. Uses `data_format = "value"` / `data_type = "float"` to ingest numeric readings (current, voltage, power, energy, temperature, frequency, power_factor, current_rating, current_max_online, current_max_offline).

- **Topics:** `homeassistant/sensor/+/state`, `homeassistant/number/+/state`
- **Filter:** regex processor tags `device=juicebox` only for topics containing the JuiceBox device name; non-JuiceBox HA topics (if any) are ignored via tag filtering
- **Tags extracted via regex:** `entity` (e.g. `Current`, `Power`, `Voltage`) from the topic path

### String consumer
Separate consumer for the `Status` entity (values: Charging / Plugged In / Unplugged / Error).

- **Topics:** `homeassistant/sensor/<juicebox-id> Status/state` (topic contains "Status")
- **data_format:** `"string"`

### Measurement
Both consumers write to measurement `juicebox` in the existing `sensors` bucket. All metrics share tags: `device=juicebox`, `entity=<name>`.

### Flux query pattern
```flux
from(bucket: "sensors")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "juicebox" and r.entity == "Power")
```

## Component 4: Grafana Dashboard

**Location:** `/home/dallen/farm-sensors/config/grafana/provisioning/dashboards/juicebox.json`

Provisioned automatically at Grafana startup (picked up by existing provisioning config).

### Panels

| Panel | Type | Entity | Unit |
|-------|------|--------|------|
| Status | Stat (text, colour-coded) | status | — |
| Power | Gauge + time series | power | W |
| Current | Time series | current | A |
| Voltage | Time series | voltage | V |
| Session Energy | Stat | energy_session | Wh |
| Lifetime Energy | Stat | energy_lifetime | kWh (÷1000) |
| Temperature | Time series | temperature | °F |
| Frequency | Gauge | frequency | Hz |
| Max Current (Online) | Stat | current_max_online | A |
| Max Current (Offline) | Stat | current_max_offline | A |

**Status colour mapping:**
- Charging → green
- Plugged In → yellow
- Unplugged → grey
- Error → red

## Out of Scope

- Current control / setting online amperage via Grafana (read-only dashboard)
- Multiple JuiceBox instances
- HA MQTT discovery removal/cleanup (existing retained messages)
- TLS on MQTT connection
