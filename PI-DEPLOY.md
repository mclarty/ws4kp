# Raspberry Pi deployment — WeatherStar 4000+ with warning break-in

This fork of [netbymatt/ws4kp](https://github.com/netbymatt/ws4kp) adds a WeatherStar-style **severe weather interrupt**: when NWS issues a Warning for the selected location the playlist is covered by a red full-screen bulletin with scrolling text and an attention tone, then a red lower-display-line crawl stays up while the warning is active.

This is a nostalgia recreation. It is **not** a substitute for NOAA Weather Radio, WEA, or a battery radio during real severe weather.

## Hardware

| Item | Notes |
| --- | --- |
| Raspberry Pi 4 or 5 (2 GB+) | Pi 3B+ works but is tighter on memory |
| 16 GB+ microSD (or SSD) | Raspberry Pi OS Lite 64-bit is enough |
| Ethernet or reliable Wi-Fi | Alerts refresh from `api.weather.gov` |
| HDMI to the TV | Use kiosk mode in Chromium |

## 1. Base OS

```bash
sudo apt update
sudo apt install -y git curl ca-certificates
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v   # 20+
```

## 2. Install this fork

```bash
sudo useradd -r -m -s /usr/sbin/nologin ws4kp || true
sudo mkdir -p /opt/ws4kp
sudo chown "$USER":"$USER" /opt/ws4kp
git clone https://github.com/mclarty/ws4kp.git /opt/ws4kp
cd /opt/ws4kp
npm ci --omit=dev
```

If `npm ci` complains about a lockfile mismatch, use `npm install --omit=dev`.

## 3. Environment / kiosk permalink

Create `/opt/ws4kp/.env`. Keys use the `WSQS_` prefix described in the upstream README (hyphens become underscores):

```bash
WS4KP_PORT=8080
# Celebration / Osceola County example — generate your own permalink in the UI and convert it
WSQS_kiosk=true
WSQS_scanLines=true
WSQS_warningBreakIn=true
WSQS_current-weather=true
WSQS_latest-observations=true
WSQS_hourly-graph=true
WSQS_local-forecast=true
WSQS_extended-forecast=true
WSQS_radar=true
WSQS_hazards=true
WSQS_latLonQuery=Celebration%2C+FL
```

Test a fake warning without waiting on weather:

`http://<pi-ip>:8080/?demoWarning=true&kiosk=true`

## 4. systemd service

`/etc/systemd/system/ws4kp.service`:

```ini
[Unit]
Description=WeatherStar 4000+
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/ws4kp
EnvironmentFile=-/opt/ws4kp/.env
ExecStart=/usr/bin/node /opt/ws4kp/index.mjs
Restart=always
RestartSec=5
User=ws4kp
Group=ws4kp

[Install]
WantedBy=multi-user.target
```

```bash
sudo chown -R ws4kp:ws4kp /opt/ws4kp
sudo systemctl daemon-reload
sudo systemctl enable --now ws4kp
sudo systemctl status ws4kp
```

Open `http://<pi-ip>:8080` from any browser on the LAN.

## 5. Optional: Chromium kiosk on the Pi desktop

If the Pi itself drives the TV:

```bash
sudo apt install -y chromium-browser unclutter
```

`/home/pi/.config/autostart/ws4kp-kiosk.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=WS4KP Kiosk
Exec=chromium-browser --kiosk --noerrdialogs --disable-infobars --check-for-update-interval=31536000 http://127.0.0.1:8080/?kiosk=true&warningBreakIn=true
```

Disable screen blanking in `raspi-config` → Display.

## 6. Optional: Docker on the Pi

```bash
cd /opt/ws4kp
docker build -t ws4kp:local .
docker run -d --name ws4kp --restart unless-stopped \
  -p 8080:8080 --env-file /opt/ws4kp/.env \
  ws4kp:local
```

The published `ghcr.io/netbymatt/ws4kp` image does **not** include this fork. Build locally from this repo.

## 7. Updating

```bash
cd /opt/ws4kp
sudo systemctl stop ws4kp
git pull
npm ci --omit=dev
sudo systemctl start ws4kp
```

## Settings added in this fork

| Query / setting | Default | Meaning |
| --- | --- | --- |
| `warningBreakIn` | `true` | Interrupt the playlist on a new Warning |
| `demoWarning` | `false` | Inject a sample Tornado Warning |

Watches and advisories still appear in the existing Hazards product and red LDL when they rotate through. Only **Warnings** (and Extreme severity) trigger the full-screen break-in.
