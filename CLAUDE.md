# JuicePass Proxy

Fork (v0.5.1) — UDP MITM proxy for JuiceBox EV chargers. Intercepts charger↔EnelX traffic, publishes to MQTT with Home Assistant auto-discovery.

## Key Files
- `juicepassproxy.py` — entry point, CLI args
- `juicebox_mitm.py` — UDP proxy logic
- `juicebox_mqtthandler.py` — MQTT + HA discovery
- `juicebox_message.py` — protocol message parsing

## Run
```bash
# Docker (recommended) — port 8047/UDP
docker build -t juicepassproxy .
docker run -p 8047:8047/udp juicepassproxy

# Manual
python3 juicepassproxy.py --juicebox_host <IP> --mqtt_host <host>
```

Python 3.10+. Deps: paho-mqtt, dnspython, pyyaml, telnetlib3.
