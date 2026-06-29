# Vinyl Switch — Full "Going Live" Guide (every step, in depth)

This documents **exactly** how the Vinyl Switch system was taken from "runs on my laptop"
to **live on the public internet, free**, with ESP32 devices able to connect from **any
network**.

---

## 0. The final architecture (what we ended up with)

```
   ┌─────────────────────┐         TLS 8883          ┌──────────────────────┐
   │  ESP32 (Booth B)     │  ───────────────────────► │                      │
   │  on ANY 2.4GHz Wi-Fi │ ◄─────────────────────── │   HiveMQ Cloud       │
   └─────────────────────┘     machine/<id>/cmd       │   (managed MQTT      │
                                machine/<id>/status    │    broker, free)     │
   ┌─────────────────────┐         TLS 8883          │                      │
   │  Render server       │  ───────────────────────► │                      │
   │  (Node: web + API)   │ ◄─────────────────────── │                      │
   │  public website      │                            └──────────────────────┘
   └─────────────────────┘
           ▲
           │ https
           │
     Customers / staff browser  →  https://vinyl-switch.onrender.com
```

**Key idea:** both the ESP32 *and* the server **dial OUT** to a broker in the cloud
(HiveMQ). Nobody has to accept *incoming* connections, so it doesn't matter that the
laptop/phone is behind CGNAT or has no public IP. Free, stable, works everywhere.

The live pieces:
- **Broker:** HiveMQ Cloud — `0ce62721ce2a45b7813e91b9632b75ae.s1.eu.hivemq.cloud:8883`
- **Server + website:** Render — `https://vinyl-switch.onrender.com`
- **Code:** GitHub — `https://github.com/Dinesh-engineeringhub/vinyl-switch`

---

## 1. Why this design? (the problems we hit and ruled out)

We tried the obvious things first; each failed for a concrete reason. Understanding these
explains *why* the cloud-broker design is the right one.

1. **Same Wi-Fi router (local):** works, but only if both devices are on the *same* network.
   - The JioFiber router only broadcast **5 GHz**, and **the ESP32 supports only 2.4 GHz**,
     so the ESP32 couldn't even join it.
2. **Phone hotspot (Redmi):** the ESP32 *can* use it (2.4 GHz), but the phone had
   **client isolation** — connected devices can't talk to each other (proved with a failed
   `ping` + no ARP entry). So the laptop server and ESP32 couldn't reach each other locally.
3. **Tunnels (Pinggy/ngrok/serveo/bore):** expose a local server to the internet.
   - Free tunnels **drop constantly** (Pinggy reset every few minutes) and the **address
     changes** on each restart (so you'd re-flash every device).
   - ngrok TCP needs a **credit card**; `bore` got **flagged by antivirus**.
4. **Why a tunnel was even needed:** the laptop is behind **CGNAT** (carrier-grade NAT).
   `tracert` showed carrier-private hops (`10.x`, `172.31.x`), and the "public IP" is shared
   by many customers — so **port-forwarding is impossible**.
5. **Free cloud VM (Oracle/AWS/GCP):** gives a real public IP, but **all require a credit
   card** for identity verification.

**The insight:** instead of *hosting* a broker (which needs a public IP), use a **ready-made
broker in the cloud** and have **both sides connect out to it**. That's **HiveMQ Cloud** —
free, no card, no public IP needed. Then host the *website* on **Render** (free Node host).

---

## 2. Prerequisites

| Thing | Why | Notes |
|-------|-----|-------|
| HiveMQ Cloud account | the MQTT broker | free, email only, no card |
| GitHub account | store the code for Render | free |
| Render account | run the server publicly | free, sign in with GitHub |
| Node.js + the project | the server code | already had it |
| PlatformIO + USB cable | flash the ESP32 | already installed |
| An ESP32 on 2.4 GHz Wi-Fi with internet | the device | uses the phone hotspot here |

---

## 3. Part A — HiveMQ Cloud (the broker)

1. Go to **https://console.hivemq.cloud** → **Sign up** (email; no card).
2. **Create cluster** → choose the **Serverless (Free)** plan → it provisions in ~1 min.
3. Open the cluster → note the **Cluster URL** and **port 8883** (TLS):
   - `0ce62721ce2a45b7813e91b9632b75ae.s1.eu.hivemq.cloud`
4. Go to the **Access Management** (a.k.a. Credentials) tab → **Add** a credential:
   - Username: `Dineshv`
   - Password: `Dinesh@2912`
   - Permission: **Publish and Subscribe**
5. These credentials are used by **both** the server and every ESP32.

---

## 4. Part B — Server code changes (embedded broker → MQTT client)

Originally the server ran its *own* MQTT broker (Aedes). For the cloud design it must instead
**connect to HiveMQ as a client**.

**4.1 `src/config.js`** — added the broker connection settings:
```js
mqttHost: process.env.MQTT_HOST || 'localhost',
mqttPort: Number(process.env.MQTT_PORT || 8883),
mqttUsername: process.env.MQTT_USERNAME || '',
mqttPassword: process.env.MQTT_PASSWORD || '',
mqttTls: process.env.MQTT_TLS !== 'false',   // HiveMQ requires TLS
```

**4.2 `src/mqtt.js`** — rewritten to use the `mqtt` library as a client:
- `mqtt.connect('mqtts://<host>:8883', { username, password, reconnectPeriod: 3000 })`
- on connect → `subscribe('machine/+/status')`
- on message → update the DB (`online`, `relay_state`, `last_seen`) — same as before
- `turnRelayOn/Off()` → `client.publish('machine/<id>/cmd', …)`
- (the embedded-broker code and device password auth were removed)

**4.3 `src/index.js`** — `startMqttBroker()` → `startMqttClient()`.

**4.4 `package.json`** — moved `mqtt` to dependencies, removed `aedes`.

**4.5 Bonus reliability fix (reconnect-resume):** in the status handler, if a device comes
back **online with the relay OFF** while a booking is still **active**, the server re-sends
the ON command for the remaining time — so a device reboot mid-session auto-resumes.

---

## 5. Part C — Firmware changes (plain TCP → TLS)

HiveMQ Cloud is **TLS-only on port 8883**, so the ESP32 must use a secure client.

In `firmware/vinyl_switch_esp32/vinyl_switch_esp32.ino`:
```cpp
#include <WiFiClientSecure.h>      // added
...
const char* MQTT_HOST = "0ce62721ce2a45b7813e91b9632b75ae.s1.eu.hivemq.cloud";
const int   MQTT_PORT = 8883;      // TLS
const char* MQTT_USER = "Dineshv";
const char* MQTT_PASS = "Dinesh@2912";
...
WiFiClientSecure net;              // was WiFiClient
PubSubClient mqtt(net);
...
// in setup(), before mqtt.setServer():
net.setInsecure();                 // TLS encryption (no cert pinning, simplest)
mqtt.setBufferSize(512);           // headroom for the TLS CONNECT + creds
```
`WIFI_SSID` / `WIFI_PASS` stay set to whatever 2.4 GHz Wi-Fi the device is on.

---

## 6. Part D — GitHub (so Render can read the code)

1. `git init` inside `software/server`, set user name/email.
2. **`.gitignore`** already excludes `node_modules/`, `data/`, and **`.env`** — so the HiveMQ
   password is **never** committed.
3. `git add -A && git commit -m "…"`.
4. Create an **empty repo** on github.com (`vinyl-switch`).
5. `git remote add origin <url>` then `git push -u origin main`.
6. Make the repo **Public** (Settings → Danger Zone → Change visibility) so Render's free
   "Public Git Repository" connector can read it (or connect GitHub via OAuth for private).

---

## 7. Part E — Render (public hosting)

1. **https://render.com** → sign in **with GitHub** (no card).
2. **New + → Web Service**.
3. **Source:** "Public Git Repository" → paste `https://github.com/Dinesh-engineeringhub/vinyl-switch`
   (or "Git Provider → GitHub" if the repo is private).
4. **Settings:**
   - Name: `vinyl-switch`
   - Region: Singapore
   - Branch: `main`
   - Build Command: `npm install`
   - Start Command: `node src/seed.js && node src/index.js`  *(seed creates demo machines, then start)*
   - Instance Type: **Free**
5. **Environment Variables** (pasted via "Add from .env" — secrets live here, not in the repo):
   ```
   MQTT_HOST=0ce62721ce2a45b7813e91b9632b75ae.s1.eu.hivemq.cloud
   MQTT_PORT=8883
   MQTT_USERNAME=Dineshv
   MQTT_PASSWORD=Dinesh@2912
   SESSION_MINUTES=60
   OPEN_HOUR=0
   CLOSE_HOUR=24
   SLOT_INTERVAL_MINUTES=10
   BOOKING_DAYS=3
   GRACE_MINUTES=15
   ADMIN_TOKEN=1a2b3c4d5e
   ```
   (Do **not** set `PORT` — Render sets it automatically; the app reads `process.env.PORT`.)
6. **Deploy Web Service** → wait ~2-3 min → live at **https://vinyl-switch.onrender.com**.

Verify:
- `https://vinyl-switch.onrender.com/` → booking app loads
- `https://vinyl-switch.onrender.com/api/locations` → returns the seeded locations
- `https://vinyl-switch.onrender.com/admin.html` → admin (token `1a2b3c4d5e`)

---

## 8. Part F — Flash the ESP32

From `firmware/vinyl_switch_esp32/`:
```
pio run -t upload          # builds + flashes over COM4
pio device monitor         # watch the serial log @115200
```
After flashing, **press the EN/RST button** on the board once (the Wi-Fi radio sometimes
needs a clean power-cycle right after flashing).

Expected serial output:
```
[wifi] connected, IP 10.35.252.x
[mqtt] connecting... connected      ← TLS handshake with HiveMQ succeeded
```

---

## 9. Part G — Verify end-to-end

1. ESP32 online → **Booth B shows 🟢 online** on `…/admin.html`.
2. Book a session on the website → get a 6-digit code.
3. Enter the code → the server publishes ON to HiveMQ → ESP32 lights the relay (GPIO 2 LED).
4. At the end time (or admin "force off") → relay turns OFF.
5. Reboot the ESP32 mid-session → it reconnects → server auto-re-sends ON (resume fix).

---

## 10. Key concepts (plain English)

- **MQTT** — a lightweight messaging system: devices *publish* messages to "topics" and
  *subscribe* to topics. A **broker** is the post office in the middle.
- **Broker vs client** — originally our server *was* the post office (needed a public address).
  Now HiveMQ is the post office, and both server and ESP32 are just *customers* who walk in.
- **TLS (port 8883)** — encryption, so the username/password and commands aren't sent in the
  clear over the internet.
- **CGNAT** — your ISP shares one public IP among many homes, so no one on the internet can
  "call" your router directly. Outbound connections still work — which is why dialing OUT to a
  cloud broker is the fix.
- **Why Render for the website** — Netlify only serves static files; our app is a *running*
  Node program with a database, which needs a Node host like Render.

---

## 11. Costs & limits (all free tiers)

| Service | Free tier limit | Caveat |
|---------|------------------|--------|
| HiveMQ Cloud | 100 connections, small data cap | fine for many devices |
| Render Web Service | free instance | **sleeps after ~15 min idle** (cold start ~30s); SQLite **resets on redeploy** |
| GitHub | unlimited public/private repos | — |

**Render sleep workaround:** a free uptime pinger (e.g. cron-job.org) hitting the URL every
~10 min keeps it awake. For permanent booking data, attach a Render Disk (paid) or a hosted DB.

---

## 12. Troubleshooting cheat-sheet

| Symptom | Cause | Fix |
|---------|-------|-----|
| ESP32 stuck `connecting to <wifi>....` | post-flash Wi-Fi init / 5 GHz network | press EN/RST; ensure it's a **2.4 GHz** network |
| `mqtt … rc=-2` | can't reach broker (TCP) | check host/port; check internet |
| `mqtt … rc=4/5` | bad MQTT username/password | match HiveMQ credentials exactly |
| device online but website can't reach it | (not an issue now — all via HiveMQ) | — |
| machines missing on Render | DB reset on redeploy | start command runs `seed.js`; or add via admin |
| Render site slow first hit | free instance was asleep | normal; ~30s cold start |
