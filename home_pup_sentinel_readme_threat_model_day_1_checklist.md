# HomePup Sentinel

**Privacy‑first home NVR + dog behavior analytics** (local‑only, modular like SimpliSafe, but with open components).

> Goal: Track a dog’s location, posture, and routines; generate heatmaps and events; run a secure home NVR; never send video/audio to third‑party clouds. Optimized for Raspberry Pi 5 + PoE IP cameras, with optional treat/buzzer actuators.

---

## Table of Contents
- [Why this project](#why-this-project)
- [Key features](#key-features)
- [System architecture](#system-architecture)
- [Bill of materials (BOM)](#bill-of-materials-bom)
- [Software stack](#software-stack)
- [Security & privacy](#security--privacy)
- [Threat model](#threat-model)
- [Quickstart (Day‑1 checklist)](#quickstart-day1-checklist)
- [Network/VLAN setup primer](#networkvlan-setup-primer)
- [Rules DSL (examples)](#rules-dsl-examples)
- [Data model (sketch)](#data-model-sketch)
- [Dashboard views](#dashboard-views)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Why this project
- **Own your data**: Replace closed camera clouds with local recording, encryption at rest, and your VPN for remote access.
- **For dogs & humans**: Passive CV + audio to learn routines: where/when/how long, barks, couch visits, crate time, naps.
- **CS learning path**: Algorithms (tracking, HMM), OS (pipelines, queues), Architecture (ARM perf), Compilers (mini DSL).

---

## Key features
- **Local NVR**: RTSP/ONVIF ingest → segmented MP4 to NAS/ZFS; retention & export.
- **Dog analytics**: YOLO‑tiny + ByteTrack; room homography; posture heuristics; occupancy heatmaps; event timeline.
- **Audio events**: Bark/whine baseline via mel‑spectrogram + tiny CNN (or RMS heuristic to start).
- **Zones & rules**: YAML/DSL (e.g., couch/quiet hours). Actuators: buzzer, treat wheel, smart plug.
- **Modular tiles**: Add cameras, mic nodes, mmWave presence, BLE collar tag.
- **Security‑first**: IoT VLAN, mTLS via local CA, no camera egress to internet, optional WireGuard for off‑LAN access.

---

## System architecture
```
           +-------------------+                 +-----------------------+
           |   Admin Client    |  WireGuard/TLS  |   Dashboard (HTTPS)   |
           |  (LAN or VPN)     |<--------------->|  Streamlit/Caddy      |
           +---------+---------+                 +-----------+-----------+
                     |                                       |
                     |                                       |
                +----v----+                         +--------v--------+
                |  MQTT   |  mTLS topics            |  CV/Rules Core  |
                | broker  |<----------------------->|  (Pi 5)         |
                +----+----+                         +----+------------+
                     ^                                   |
                     |                                   |
    +----------------+--------------+                    |
    |                               |                    |
+---+----+    +----------------+    +------+        +----v----+
| Cams  |--RTSP-->  NVR/Recorder |--> Disk |        | Actuators|
| (PoE) |    +----------------+    +------+        | (GPIO)  |
+---+----+                                          +----+----+
    |                                                    |
    | ONVIF Health                                       |
    |                                                    |
+---v-------+     +-----------------+               +----v-----+
| Mic Node  |-->  audio/* topics    |               | Sensors  |
| (Pi Zero) |     (mTLS MQTT)       |               | (mmWave, |
+-----------+     +-----------------+               | reed, BLE)
                                                   +----------+
```

**Notes**
- CV/Rules Core subscribes to camera frame metadata and audio topics; writes events and tracks to DB; controls actuators.
- NVR writes full‑res streams to disk/NAS independent of CV pipeline (so analytics can drop frames under load without losing video).

---

## Bill of materials (BOM)
- **Compute**: Raspberry Pi 5 (8GB) + USB3 SSD (>=500GB) or NAS; PoE+ HAT optional.
- **Cameras**: 2–4 ONVIF/RTSP PoE (e.g., Reolink/Dahua); 2.8–4mm lens.
- **Audio**: Pi Zero 2 W + I2S mic (SPH0645/INMP441) for bark node.
- **Networking**: PoE switch; router with VLANs (pfSense/opnSense/UniFi).
- **Actuators**: SG90/MG996R servo for treat wheel; piezo buzzer; 5V supply.
- **Optional sensors**: LD2410 mmWave, ESP32 reed sensors, BLE tag for collar.

---

## Software stack
- **OS**: Raspberry Pi OS (64‑bit), systemd services.
- **CV**: OpenCV, ONNXRuntime or TensorRT (if available), YOLO‑tiny.
- **Tracking**: ByteTrack/DeepSORT (Hungarian + IOU), Kalman filter.
- **Audio**: librosa/torch for mel‑spec; tiny CNN or heuristic.
- **Broker**: Mosquitto (mTLS), ACL per device/topic.
- **Web**: Streamlit dashboard behind Caddy (TLS, auth via passkeys/WebAuthn).
- **DB/Store**: SQLite (MVP) → Postgres (scaling); Parquet exports.
- **NVR**: FFmpeg/Go2RTC/Frigate‑style segmenter.
- **PKI**: step‑ca for internal CA; cert rotation scripts.
- **VPN**: WireGuard (router or Pi).

---

## Security & privacy
- **Zero egress** for cameras/IoT VLAN; block DNS/HTTP(S) outbound; allow only RTSP to recorder and mTLS MQTT to broker.
- **Mutual TLS** everywhere (devices have client certs); short‑lived certs, CRL on compromise.
- **Least privilege** ACLs: device can only publish/subscribe its topics.
- **Hardening**: SSH keys only, disable password auth, unattended‑upgrades, fail2ban, AppArmor profiles for FFmpeg/CV.
- **At rest**: LUKS on Pi SSD; server‑side encryption on NAS; backups encrypted with restic/borg.
- **Remote access**: WireGuard to LAN (no port‑forwarded dashboard).
- **Privacy modes**: presence‑based redaction; blackout zones; schedule‑based retention.

---

## Threat model
**Assets**
- A1: Live and recorded video/audio (private).
- A2: Event metadata (locations, routines, schedules).
- A3: Control plane (rules, actuators).
- A4: Credentials/keys (CA root, client certs, VPN keys).

**Adversaries**
- Adv‑E: External internet attacker scanning home IPs.
- Adv‑N: Nearby network intruder (joins Wi‑Fi or LAN jack).
- Adv‑V: Vendor cloud collecting telemetry.
- Adv‑P: Casual physical intruder (steals Pi/SSD).

**Attack surfaces**
- S1: Camera firmware calling home; ONVIF exposure.
- S2: MQTT broker/web dashboard.
- S3: CV/NVR processes (FFmpeg, model loaders).
- S4: Backup/VPN endpoints.

**Mitigations**
- M1: VLAN isolation; egress blocks; no UPNP; camera admin password rotated; ONVIF creds distinct.
- M2: mTLS with step‑ca; per‑topic ACL; dashboard behind Caddy + passkeys; rate‑limit + fail2ban.
- M3: AppArmor; run under non‑root; read‑only FS for capture nodes; auto‑update security; health probes.
- M4: LUKS disk; encrypted backups; WireGuard keys rotated; offsite secrets stored in password manager.

**Assumptions & residual risk**
- Router/VLAN correctly configured; users keep CA root offline; physical access cannot be fully prevented.

---

## Quickstart (Day‑1 checklist)
**0) Prep**
- Flash Pi OS (64‑bit), set hostname `sentinel`, enable SSH (keys only). Update + reboot.
- Install Caddy, Mosquitto, FFmpeg, Python deps; create `sentinel` user.

**1) Network & VLAN**
- Create VLAN10=Trusted, VLAN20=IoT. Place cameras/mic/ESP on VLAN20; Pi on VLAN10.
- Block internet egress for VLAN20 (deny all except RTSP→Pi IP, MQTT→Pi IP).

**2) PKI**
- Install step‑ca on Pi; generate root (store offline) + intermediate.
- Issue certs: `mqtt-broker`, `dashboard`, `cv-core`, `nvr`, `mic-node-1`, `cam-bridge-*`.

**3) MQTT**
- Configure Mosquitto with mTLS (listener on LAN only). Create ACLs (publish/subscribe per topic prefix).

**4) NVR**
- Add RTSP URLs; set segment size (e.g., 1 min); retention policy; write to `/srv/nvr` (or NAS).

**5) CV/Tracking**
- Run detector wrapper (YOLO‑tiny); enable ByteTrack; calibrate homography (4 floor points) per camera.
- Verify event logging to SQLite (`events`, `tracks`, `zones`).

**6) Dashboard**
- Bring up Streamlit behind Caddy TLS; enable passkeys; show live view, timeline, daily heatmap.

**7) Audio Node**
- Bring up Pi Zero mic node; publish `audio/<node>/spectrum` + `audio/<node>/bark`.

**8) Rules v0**
- Add `couch_guardian` and `quiet_hours` rules; wire GPIO buzzer/treat servo; test on dummy triggers.

**9) Backups**
- Configure restic to encrypted remote (e.g., rclone to cloud bucket you control); schedule via systemd timer.

**10) Hardening & Docs**
- Enable unattended‑upgrades; fail2ban; AppArmor; write runbook + threat model; record cert rotation steps.

---

## Network/VLAN setup primer
- **UniFi**: Create VLAN20 (IoT), SSID `IoT‑LAN` with VLAN20; set “Block internet” policy; device group rules allow RTSP/MQTT to Pi.
- **pfSense/opnSense**: Add OPT interface (VLAN20) with block rules; allowlist Pi IP as destination on ports 1883/8883 (MQTT TLS) and 554 (RTSP) only.

**ASCII diagram**
```
[Internet]
    |
 [Router]
  |     \
VLAN10  VLAN20 (blocked egress)
  |           \
 [Pi 5]       [PoE Switch]--[Cams]
  |                            \
 [NAS]                        [Mic Node]
```

---

## Rules DSL (examples)
```yaml
rules:
  - name: Couch Guardian
    when: zone("couch").occupied && duration > 1.5s && time.between("08:00","22:00")
    then:
      - actuator.beep()
      - log.event("couch_violation")

  - name: Quiet Hours
    when: audio.bark_rate(window=10s) > 3 && time.after("22:30")
    then:
      - actuator.click()
      - notify.mobile("Bark burst detected")

  - name: Training — Sit Treat
    when: posture == sit && duration >= 2s && session.training
    then:
      - actuator.treat(1)
      - log.metric("sit_reward")
```

---

## Data model (sketch)
**events**(id, ts, type, cam_id, zone_id, score, payload_json)

**tracks**(track_id, ts, cam_id, x_room, y_room, posture, speed)

**zones**(zone_id, name, polygon, cam_id)

**audio_events**(id, ts, node_id, bark_score, rms)

**health**(svc, ts, level, message)

---

## Dashboard views
- **Live**: camera feed + dog bbox/track trail; current zone.
- **Timeline**: events (couch, crate, door, barks) with filters.
- **Heatmaps**: occupancy per room/day/time; export PNG/CSV.
- **Routines**: median nap windows, alone‑time calm score.
- **Security**: sensor states, camera health, last motion, storage space.

---

## Roadmap
- Person re‑ID or collar AprilTag for fast identity lock.
- HMM for behavior states; anomaly alerts (outlier path entropy).
- Two‑cam triangulation for jump height/speed.
- Quantized on‑device model; compare CPU vs NEON/SIMD.
- Privacy enhancements: per‑room redaction schedules; automatic face‑blur for humans.
- Postgres + Timescale for scalable time‑series.

---

## Contributing
PRs welcome. Please include:
- Reproducible benchmarks (fps, CPU, RAM) on Pi 5.
- Security review notes for any new service or network change.

---

## License
TBD (e.g., MIT for code, CC‑BY‑SA for docs).



---

## Deployment: docker-compose (starter)
Create a `deploy/` folder with these files.

**deploy/.env**
```
# Edit these
DOMAIN=sentinel.local
PI_PUBLIC_IP=192.168.10.50
TRUSTED_NET=192.168.10.0/24
IOT_NET=192.168.20.0/24
NVR_DIR=/srv/nvr
DATA_DIR=/srv/homepup
CA_DIR=/srv/pki
POSTGRES_DIR=/srv/pg
``` 

**deploy/docker-compose.yml**
```yaml
version: "3.9"
name: homepup-sentinel
services:
  mqtt:
    image: eclipse-mosquitto:2
    container_name: mqtt
    restart: unless-stopped
    ports: ["8883:8883"]  # TLS only
    volumes:
      - ./mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf:ro
      - ${CA_DIR}/mqtt:/mosquitto/certs:ro
      - ${DATA_DIR}/mosquitto:/mosquitto/data
      - ${DATA_DIR}/mosquitto/log:/mosquitto/log
    networks: [core]

  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_DB: homepup
      POSTGRES_USER: homepup
      POSTGRES_PASSWORD: change-me
    volumes:
      - ${POSTGRES_DIR}:/var/lib/postgresql/data
    networks: [core]

  nvr:
    image: alexxit/go2rtc:latest
    container_name: nvr
    restart: unless-stopped
    ports: ["8554:8554", "1984:1984"]
    volumes:
      - ./go2rtc.yaml:/config/go2rtc.yaml:ro
      - ${NVR_DIR}:/recordings
    networks: [core]

  cvcore:
    build: ../cv  # or image: ghcr.io/you/homepup-cv:latest
    container_name: cvcore
    restart: unless-stopped
    environment:
      MQTT_URL: mqtts://mqtt:8883
      DB_URL: postgresql://homepup:change-me@postgres:5432/homepup
    volumes:
      - ${DATA_DIR}/models:/models:ro
      - ${DATA_DIR}/config:/config:ro
      - ${DATA_DIR}/cache:/cache
    depends_on: [mqtt, postgres]
    networks: [core]

  dashboard:
    build: ../dashboard
    container_name: dashboard
    restart: unless-stopped
    environment:
      DB_URL: postgresql://homepup:change-me@postgres:5432/homepup
    networks: [core]

  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    environment:
      DOMAIN: ${DOMAIN}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ${CA_DIR}/caddy:/certs:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on: [dashboard]
    networks: [core]

volumes:
  caddy_data: {}
  caddy_config: {}

networks:
  core: {}
```

**deploy/Caddyfile** (local TLS via internal CA)
```
{
  auto_https off
}

https://{$DOMAIN} {
  tls /certs/dashboard.crt /certs/dashboard.key
  reverse_proxy dashboard:8501
}
```

**deploy/mosquitto/mosquitto.conf**
```
listener 8883 0.0.0.0
cafile /mosquitto/certs/ca.crt
certfile /mosquitto/certs/mqtt.crt
keyfile /mosquitto/certs/mqtt.key
require_certificate true
use_identity_as_username true
allow_anonymous false

persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log

include_dir /mosquitto/config/conf.d
```

**deploy/go2rtc.yaml** (example; replace with your RTSP URLs)
```yaml
streams:
  living: rtsp://user:pass@192.168.20.11:554/stream1
  hallway: rtsp://user:pass@192.168.20.12:554/stream1
record:
  path: /recordings
  segments: 60s
  clean:
    keep_days: 7
```

**Build context hints**
- `../cv/Dockerfile`: minimal Python runtime with OpenCV + onnxruntime; run `python -m cv.main`.
- `../dashboard/Dockerfile`: Python + Streamlit; run `streamlit run app.py --server.address 0.0.0.0`.

Run:
```bash
cd deploy && docker compose --env-file ./.env up -d --build
```

---

## Systemd (bare‑metal alternative)
Place these in `/etc/systemd/system/` and `systemctl daemon-reload`.

**homepup-mqtt.service**
```
[Unit]
Description=Mosquitto MQTT (TLS)
After=network-online.target

[Service]
ExecStart=/usr/sbin/mosquitto -c /etc/mosquitto/mosquitto.conf
User=mosquitto
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**homepup-nvr.service**
```
[Unit]
Description=HomePup NVR (go2rtc)
After=network-online.target

[Service]
ExecStart=/usr/local/bin/go2rtc -config /etc/go2rtc.yaml
WorkingDirectory=/srv/nvr
User=homepup
Restart=always

[Install]
WantedBy=multi-user.target
```

**homepup-cv.service**
```
[Unit]
Description=HomePup CV Core
After=network-online.target mosquitto.service postgresql.service

[Service]
Environment=MQTT_URL=mqtts://localhost:8883
Environment=DB_URL=postgresql://homepup:change-me@localhost:5432/homepup
ExecStart=/usr/bin/python3 -m cv.main
WorkingDirectory=/srv/homepup/cv
User=homepup
Restart=always

[Install]
WantedBy=multi-user.target
```

**homepup-dashboard.service**
```
[Unit]
Description=HomePup Dashboard
After=network-online.target

[Service]
ExecStart=/usr/bin/streamlit run app.py --server.address 0.0.0.0 --server.port 8501
WorkingDirectory=/srv/homepup/dashboard
User=homepup
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**restic-backup.service** and **restic-backup.timer**
```
[Unit]
Description=Encrypted backup via restic

[Service]
Type=oneshot
Environment=RESTIC_REPOSITORY=rclone:myremote:homepup
Environment=RESTIC_PASSWORD=change-me
ExecStart=/usr/bin/restic backup /srv/homepup /srv/nvr
```
```
[Unit]
Description=Nightly restic backup

[Timer]
OnCalendar=02:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:
```bash
sudo systemctl enable --now homepup-mqtt homepup-nvr homepup-cv homepup-dashboard restic-backup.timer
```

---

## Starter repo scaffold
```
homepup-sentinel/
  cv/
    main.py
    detector.py
    tracker.py
    homography.py
    rules_runtime.py
    requirements.txt
  dashboard/
    app.py
    requirements.txt
  deploy/
    .env
    docker-compose.yml
    Caddyfile
    go2rtc.yaml
    mosquitto/
      mosquitto.conf
  scripts/
    enroll_device.sh
    rotate_certs.sh
    backup.sh
  docs/
    THREAT_MODEL.md
    NETWORKING.md
```

**cv/main.py (stub)**
```python
import time
# TODO: connect to MQTT, DB, run inference loop
if __name__ == "__main__":
    print("CV core starting…")
    while True:
        time.sleep(5)
```

**dashboard/app.py (stub)**
```python
import streamlit as st
st.set_page_config(page_title="HomePup Sentinel")
st.title("HomePup Sentinel")
st.write("Live view, timeline, and heatmaps coming soon.")
```

