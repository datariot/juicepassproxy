# JuiceBox Local Mode + InfluxDB + Grafana Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy juicepassproxy in full local mode, pipe JuiceBox telemetry into the existing farm-sensors InfluxDB/Grafana stack via Telegraf.

**Architecture:** juicepassproxy container intercepts JuiceBox UDP traffic (redirected via UniFi DNS override), publishes MQTT state topics under the `hmd/` prefix; Telegraf MQTT consumers subscribe and write to the existing `sensors` InfluxDB bucket under measurement `juicebox`; Grafana dashboard provisioned from the `json/` directory.

**Tech Stack:** Python/Docker (juicepassproxy), Telegraf 1.29, InfluxDB 2.7 (Flux), Grafana 10.2.3, Mosquitto 2, Podman rootless.

---

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `/home/dallen/juicepassproxy/docker-compose.yml` | Create | Standalone container definition |
| `/home/dallen/juicepassproxy/config/` | Create dir | Persisted JuiceBox config (ID, ratings) |
| `/home/dallen/juicepassproxy/log/` | Create dir | Rotating log files |
| `/home/dallen/farm-sensors/config/telegraf/telegraf.conf` | Modify | Add two MQTT consumers + regex processor |
| `/home/dallen/farm-sensors/config/grafana/provisioning/dashboards/json/juicebox.json` | Create | Grafana dashboard |

---

## Chunk 1: juicepassproxy Container

### Task 1: Create directory structure and docker-compose.yml

**Files:**
- Create: `/home/dallen/juicepassproxy/docker-compose.yml`
- Create dirs: `config/`, `log/`

> **Before starting:** You need your JuiceBox LAN IP. Find it via your UniFi Network controller or `arp -a`. Replace `<JUICEBOX_IP>` throughout.

- [ ] **Step 1: Create config and log directories with .gitkeep**

```bash
mkdir -p /home/dallen/juicepassproxy/config
mkdir -p /home/dallen/juicepassproxy/log
touch /home/dallen/juicepassproxy/config/.gitkeep
touch /home/dallen/juicepassproxy/log/.gitkeep
```

The `.gitkeep` files allow git to track the directories before any content exists.

- [ ] **Step 2: Write docker-compose.yml**

Create `/home/dallen/juicepassproxy/docker-compose.yml`:

```yaml
services:
  juicepassproxy:
    build: .
    container_name: juicepassproxy
    restart: unless-stopped
    ports:
      - "8047:8047/udp"
    environment:
      - JUICEBOX_HOST=<JUICEBOX_IP>
      - IGNORE_ENELX=true
      - MQTT_HOST=192.168.1.110
      - MQTT_PORT=1883
      - LOG_LOC=/log
    volumes:
      - ./config:/config
      - ./log:/log

# Note: farm-sensors stack must be running before starting this container.
# JUICEBOX_HOST satisfies startup validation (juicepassproxy.py:401-405)
# and enables telnet ID discovery on first run.
# IGNORE_ENELX=true means JPP acts as server — no EnelX contact.
# MQTT_HOST is the host LAN IP (not localhost; separate compose network).
```

- [ ] **Step 3: Verify docker-compose.yml syntax**

```bash
cd /home/dallen/juicepassproxy && docker compose config
```

Expected: prints the resolved compose config with no errors. (`docker` is aliased to `podman` on this host; `docker compose` uses podman-compose.)

- [ ] **Step 4: Ensure farm-sensors stack is running**

```bash
cd /home/dallen/farm-sensors && docker compose ps
```

Expected: `farm-mosquitto`, `farm-influxdb`, `farm-telegraf`, `farm-grafana` all show `running`.

If not running: `docker compose up -d`

- [ ] **Step 5: Build and start juicepassproxy**

```bash
cd /home/dallen/juicepassproxy && docker compose up -d --build
```

Expected: container starts, no immediate exit.

- [ ] **Step 6: Verify container started and is discovering JuiceBox ID**

```bash
docker logs juicepassproxy --tail 40
```

Expected output includes lines like:
```
INFO      [juicepassproxy] juicebox_id: <some-serial>
INFO      [juicepassproxy] Starting JuiceboxMITM at 0.0.0.0:8047
```

If you see `Exiting: --enelx_ip is not set` — `JUICEBOX_HOST` is missing or wrong.
If you see MQTT connection errors — check that farm-mosquitto is up and `MQTT_HOST=192.168.1.110` is reachable.

- [ ] **Step 7: Verify JuiceBox ID was persisted to config**

```bash
ls /home/dallen/juicepassproxy/config/
cat /home/dallen/juicepassproxy/config/juicepassproxy.yaml
```

Expected: `juicepassproxy.yaml` exists and contains `JUICEBOX_ID: <serial>`.

### Task 2: Configure UniFi DNS override

> This is a manual step in the UniFi Network web UI. No files to edit.

- [ ] **Step 1: Find JuiceBox UDPC port**

```bash
telnet <JUICEBOX_IP> 2000
```

At the telnet prompt, type `list` and press Enter. Look for the `UDPC` line:
```
# 1 UDPC  juicenet-udp-prod3-usa.enelx.com:8047 (...)
```

Note the port (typically `8047`). Type `quit` to exit.

- [ ] **Step 2: Add DNS override in UniFi**

In UniFi Network UI → Settings → DNS Overrides (exact path varies by firmware version; also found under WAN settings or Network → DNS):

Add record:
- **Hostname:** `juicenet-udp-prod3-usa.enelx.com`
- **IP:** `192.168.1.110`

Save and apply. The JuiceBox will resolve the EnelX server hostname to the proxy host.

- [ ] **Step 3: Verify JuiceBox traffic reaching the proxy**

Wait 1–2 minutes for the JuiceBox to send its next UDP update, then check logs:

```bash
docker logs juicepassproxy --tail 20
```

Expected: log lines showing received messages from the JuiceBox, e.g.:
```
INFO  [juicebox_mitm] From JuiceBox: ...
```

If no messages arrive after 5 minutes, the JuiceBox may still be resolving the old IP. Try rebooting the JuiceBox (unplug/replug) to force a fresh DNS lookup.

### Task 3: Verify MQTT topics

- [ ] **Step 1: Subscribe to JuiceBox state topics**

```bash
mosquitto_sub -h 192.168.1.110 -t 'hmd/#' -v -q 1
```

Leave this running for ~30 seconds. Expected output (one line per state update):
```
hmd/sensor/JuiceBox/Power/state 0
hmd/sensor/JuiceBox/Current/state 0.0
hmd/sensor/JuiceBox/Voltage/state 240.1
hmd/sensor/JuiceBox/Status/state Unplugged
hmd/sensor/JuiceBox/Energy--Lifetime-/state 123456
...
```

- [ ] **Step 2: Record actual topic names**

Compare observed topics against the spec table. Specifically note:
- Any entity names that differ from expected sanitized forms
- Confirm `Energy--Lifetime-` and `Energy--Session-` have doubled hyphens
- Confirm `Max-Current-Offline-Device-` and `Max-Current-Online-Device-` have trailing hyphens

If any topics differ, update the topic list in Task 4 Step 2 accordingly before proceeding.

- [ ] **Step 3: Commit container setup**

```bash
cd /home/dallen/juicepassproxy
git add docker-compose.yml config/.gitkeep log/.gitkeep
git commit -m "feat: add standalone docker-compose for juicepassproxy local mode"
```

---

## Chunk 2: Telegraf MQTT Consumers

### Task 4: Add JuiceBox consumers to telegraf.conf

**Files:**
- Modify: `/home/dallen/farm-sensors/config/telegraf/telegraf.conf`

- [ ] **Step 1: Read the existing telegraf.conf to find the append point**

Open `/home/dallen/farm-sensors/config/telegraf/telegraf.conf`. Append the new blocks after the existing `[[processors.regex]]` block (after line ~101).

- [ ] **Step 2: Append JuiceBox consumer blocks**

Add to the end of `/home/dallen/farm-sensors/config/telegraf/telegraf.conf`:

> **Design note:** The spec describes a wildcard (`hmd/sensor/JuiceBox/+/state`) + tagdrop architecture to exclude `Status` and `Last-Debug-Message` from the float consumer. Telegraf has no `[[processors.filter]]` plugin, and tagdrop cannot act on tags set by a downstream processor. The solution here uses an **explicit topic list** instead — subscribing only to the 13 numeric entities we care about. This is functionally equivalent and simpler. Trade-off: if juicepassproxy adds a new numeric entity in a future version, the topic list must be updated manually.

```toml
###############################################################################
#                         JUICEBOX EV CHARGER                                #
###############################################################################

# Numeric metrics from JuiceBox via juicepassproxy
# Explicit topic list (not wildcard) keeps string-valued entities out of the float consumer
[[inputs.mqtt_consumer]]
  servers = ["tcp://mosquitto:1883"]
  client_id = "telegraf-juicebox-numeric"
  qos = 1
  persistent_session = true
  name_override = "juicebox"
  topics = [
    "hmd/sensor/JuiceBox/Current/state",
    "hmd/sensor/JuiceBox/Current-Rating/state",
    "hmd/sensor/JuiceBox/Max-Current-Offline-Device-/state",
    "hmd/sensor/JuiceBox/Max-Current-Online-Device-/state",
    "hmd/sensor/JuiceBox/Frequency/state",
    "hmd/sensor/JuiceBox/Energy--Lifetime-/state",
    "hmd/sensor/JuiceBox/Energy--Session-/state",
    "hmd/sensor/JuiceBox/Temperature/state",
    "hmd/sensor/JuiceBox/Voltage/state",
    "hmd/sensor/JuiceBox/Power/state",
    "hmd/sensor/JuiceBox/Power-Factor/state",
    "hmd/number/JuiceBox/Max-Current-Offline-Wanted-/state",
    "hmd/number/JuiceBox/Max-Current-Online-Wanted-/state",
  ]
  topic_tag = "topic"
  data_format = "value"
  data_type = "float"

  [inputs.mqtt_consumer.tags]
    device = "juicebox"

# Status entity (string value: Charging / Plugged In / Unplugged / Error)
[[inputs.mqtt_consumer]]
  servers = ["tcp://mosquitto:1883"]
  client_id = "telegraf-juicebox-string"
  qos = 1
  persistent_session = true
  name_override = "juicebox"
  topics = [
    "hmd/sensor/JuiceBox/Status/state",
  ]
  topic_tag = "topic"
  data_format = "string"

  [inputs.mqtt_consumer.tags]
    device = "juicebox"
    entity = "Status"

# Extract entity name from topic path for all juicebox metrics
# Pattern: hmd/<component>/JuiceBox/<entity-name>/state -> entity tag
[[processors.regex]]
  namepass = ["juicebox"]

  [[processors.regex.tags]]
    key = "topic"
    pattern = "^hmd/[^/]+/JuiceBox/([^/]+)/state$"
    replacement = "${1}"
    result_key = "entity"
```

- [ ] **Step 3: Restart farm-telegraf to pick up the new config**

```bash
cd /home/dallen/farm-sensors && docker compose restart telegraf
```

Wait 15 seconds, then check for errors:

```bash
docker logs farm-telegraf --tail 30
```

Expected: no `E!` (error) lines. You may see `W!` warnings about persistent sessions on first connect — these are harmless. You should see lines like:
```
[inputs.mqtt_consumer] Connected [telegraf-juicebox-numeric]
[inputs.mqtt_consumer] Connected [telegraf-juicebox-string]
```

If you see `E! [inputs.mqtt_consumer] ... client_id already in use` — another consumer with the same ID is connected. Check that persistent sessions from a previous failed run aren't lingering (restart mosquitto if needed: `docker compose restart mosquitto`).

- [ ] **Step 4: Verify data is reaching InfluxDB**

Wait 60 seconds for the JuiceBox to send a message and Telegraf to flush it. Then run a quick query via the farm-sensors task:

```bash
cd /home/dallen/farm-sensors && task query
```

If the `task query` command doesn't filter for `juicebox`, run directly:

```bash
docker exec farm-influxdb influx query \
  --org bloom-and-grow \
  --token farm-sensors-token \
  '
from(bucket: "sensors")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "juicebox")
  |> limit(n: 10)
'
```

Expected: rows with `_measurement=juicebox`, `device=juicebox`, `entity=Power` (or similar), `_field=value`.

If no results after 2 minutes: check `docker logs juicepassproxy` to confirm the JuiceBox is sending messages.

- [ ] **Step 5: Verify Status field is storing correctly**

```bash
docker exec farm-influxdb influx query \
  --org bloom-and-grow \
  --token farm-sensors-token \
  '
from(bucket: "sensors")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "juicebox" and r.entity == "Status")
  |> last()
'
```

Expected: `_value` is a string: `"Unplugged"`, `"Plugged In"`, `"Charging"`, or `"Error"`.

- [ ] **Step 6: Commit telegraf config**

```bash
cd /home/dallen/farm-sensors
git add config/telegraf/telegraf.conf
git commit -m "feat: add juicebox MQTT consumers to telegraf"
```

---

## Chunk 3: Grafana Dashboard

### Task 5: Create juicebox.json dashboard

**Files:**
- Create: `/home/dallen/farm-sensors/config/grafana/provisioning/dashboards/json/juicebox.json`

- [ ] **Step 1: Write the dashboard JSON**

Create `/home/dallen/farm-sensors/config/grafana/provisioning/dashboards/json/juicebox.json`:

```json
{
  "annotations": { "list": [] },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [
            {
              "options": {
                "Charging":   { "color": "green",  "index": 0, "text": "Charging" },
                "Plugged In": { "color": "yellow", "index": 1, "text": "Plugged In" },
                "Unplugged":  { "color": "blue",   "index": 2, "text": "Unplugged" },
                "Error":      { "color": "red",    "index": 3, "text": "Error" }
              },
              "type": "value"
            }
          ],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "blue", "value": null }] }
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
      "id": 1,
      "options": {
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Status",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Status\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "blue", "value": null },
              { "color": "green", "value": 100 },
              { "color": "yellow", "value": 5000 }
            ]
          },
          "unit": "watt"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 6, "y": 0 },
      "id": 2,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Power",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Power\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\n  |> yield(name: \"mean\")",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }] },
          "unit": "watth"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 12, "y": 0 },
      "id": 3,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Session Energy",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Energy--Session-\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }] },
          "unit": "kwatth"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 0 },
      "id": 4,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Lifetime Energy",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Energy--Lifetime-\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()\n  |> map(fn: (r) => ({r with _value: r._value / 1000.0}))",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }] },
          "unit": "amp"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 0, "y": 4 },
      "id": 5,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Current",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Current\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "green", "value": 110 },
              { "color": "yellow", "value": 250 }
            ]
          },
          "unit": "volt"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 6, "y": 4 },
      "id": 6,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Voltage",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Voltage\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "green", "value": 59.5 },
              { "color": "yellow", "value": 60.5 }
            ]
          },
          "unit": "hertz"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 12, "y": 4 },
      "id": 7,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Frequency",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Frequency\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 100 },
              { "color": "red", "value": 140 }
            ]
          },
          "unit": "fahrenheit"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 4 },
      "id": 8,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "auto"
      },
      "title": "Temperature",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Temperature\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }] },
          "unit": "amp"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 12, "x": 0, "y": 8 },
      "id": 9,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "value_and_name"
      },
      "title": "Max Current — Online (device reported)",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Max-Current-Online-Device-\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }] },
          "unit": "amp"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 12, "x": 12, "y": 8 },
      "id": 10,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "textMode": "value_and_name"
      },
      "title": "Max Current — Offline (device reported)",
      "type": "stat",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Max-Current-Offline-Device-\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> last()",
          "refId": "A"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "smooth",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }] },
          "unit": "watt"
        },
        "overrides": [
          {
            "matcher": { "id": "byName", "options": "Current" },
            "properties": [{ "id": "unit", "value": "amp" }, { "id": "custom.axisPlacement", "value": "right" }]
          }
        ]
      },
      "gridPos": { "h": 8, "w": 24, "x": 0, "y": 12 },
      "id": 11,
      "options": {
        "legend": { "calcs": ["last", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "none" }
      },
      "title": "Power & Current",
      "type": "timeseries",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Power\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\n  |> set(key: \"_field\", value: \"Power\")\n  |> yield(name: \"Power\")",
          "refId": "A"
        },
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Current\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\n  |> set(key: \"_field\", value: \"Current\")\n  |> yield(name: \"Current\")",
          "refId": "B"
        }
      ]
    },
    {
      "datasource": { "type": "influxdb", "uid": "influxdb" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "smooth",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": { "mode": "absolute", "steps": [{ "color": "green", "value": null }] },
          "unit": "volt"
        },
        "overrides": [
          {
            "matcher": { "id": "byName", "options": "Temperature" },
            "properties": [{ "id": "unit", "value": "fahrenheit" }, { "id": "custom.axisPlacement", "value": "right" }]
          }
        ]
      },
      "gridPos": { "h": 8, "w": 24, "x": 0, "y": 20 },
      "id": 12,
      "options": {
        "legend": { "calcs": ["last", "min", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "none" }
      },
      "title": "Voltage & Temperature",
      "type": "timeseries",
      "targets": [
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Voltage\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\n  |> set(key: \"_field\", value: \"Voltage\")\n  |> yield(name: \"Voltage\")",
          "refId": "A"
        },
        {
          "datasource": { "type": "influxdb", "uid": "influxdb" },
          "query": "from(bucket: \"sensors\")\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\n  |> filter(fn: (r) => r._measurement == \"juicebox\" and r.entity == \"Temperature\")\n  |> filter(fn: (r) => r._field == \"value\")\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\n  |> set(key: \"_field\", value: \"Temperature\")\n  |> yield(name: \"Temperature\")",
          "refId": "B"
        }
      ]
    }
  ],
  "refresh": "30s",
  "schemaVersion": 38,
  "tags": ["juicebox", "ev"],
  "templating": { "list": [] },
  "time": { "from": "now-6h", "to": "now" },
  "timepicker": {},
  "timezone": "browser",
  "title": "JuiceBox EV Charger",
  "uid": "juicebox",
  "version": 1
}
```

- [ ] **Step 2: Validate JSON is well-formed**

```bash
jaq . /home/dallen/farm-sensors/config/grafana/provisioning/dashboards/json/juicebox.json > /dev/null && echo "OK"
```

Expected: `OK` with no errors.

- [ ] **Step 3: Verify dashboard loads in Grafana**

Grafana polls the provisioning directory every 30 seconds (`updateIntervalSeconds: 30` in `dashboards.yml`). Either wait 30 seconds or restart Grafana to pick it up immediately:

```bash
cd /home/dallen/farm-sensors && docker compose restart grafana
```

Open Grafana at `http://192.168.1.110:3000` (admin / farmsensors123). Navigate to Dashboards → Farm folder → "JuiceBox EV Charger".

Expected: dashboard loads with all 12 panels. Panels showing current values (stat panels) should display data within 60 seconds of the JuiceBox sending its next message.

- [ ] **Step 4: Smoke-test each panel**

Check that:
- **Status** panel shows a coloured background (Charging / Plugged In / Unplugged / Error)
- **Power** shows a watt value (0 W when unplugged is correct)
- **Lifetime Energy** shows a plausible kWh value (not 0 unless brand new charger)
- **Voltage** shows ~240 V
- **Power & Current** time series has data points
- No panels show "No data" (if they do, check the entity tag value against `mosquitto_sub` output)

- [ ] **Step 5: Commit dashboard**

```bash
cd /home/dallen/farm-sensors
git add config/grafana/provisioning/dashboards/json/juicebox.json
git commit -m "feat: add juicebox grafana dashboard"
```

---

## Troubleshooting

**JuiceBox not sending messages to proxy:**
- Verify UniFi DNS override is active and the JuiceBox is resolving to `192.168.1.110`
- Reboot the JuiceBox (unplug power) to force a fresh DNS lookup and UDP connection
- Check `docker logs juicepassproxy` for any errors

**Telegraf not receiving MQTT messages:**
- Run `mosquitto_sub -h 192.168.1.110 -t 'hmd/#' -v` and confirm topics are arriving
- If topics have different entity names than expected, update the `topics` list in the numeric consumer to match exactly
- Check `docker logs farm-telegraf` for connection or parse errors

**Grafana shows "No data":**
- Query InfluxDB directly to confirm data is there (Task 4 Step 4)
- Check that the `entity` tag value in the Flux query matches exactly what Telegraf stored (case-sensitive, including hyphens)
- Use the InfluxDB Data Explorer at `http://192.168.1.110:8086` to browse the `sensors` bucket and confirm tag values
