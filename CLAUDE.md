# Colour Installation — Project Briefing

This project is a distributed art installation that captures the average colour of live camera
feeds from remote locations around the world, and displays those colours in real time at a
central installation hub.

---

## How it works

1. **Edge nodes** — people at remote locations open a webpage on any device (phone, laptop, etc),
   point their camera at the sky, and hit Start. The page captures frames from the camera,
   averages all pixels down to a single RGB value, and sends it as a small JSON packet
   to an MQTT broker.

2. **MQTT broker** — acts as a message routing layer between edge nodes and the hub. Each
   location publishes to its own topic: `installation/colour/{location_id}`.

3. **Hub** — a local Node.js server that subscribes to all location topics, stores data in
   SQLite, and serves a live display page. Runs on the operator's machine only — not in git,
   not publicly accessible.

---

## Repo contents

### `index.html`
The edge capture page. Hosted on GitHub Pages, opened on any phone or laptop — no install
required. Key behaviours:
- Requests camera access via `getUserMedia` (rear camera preferred on mobile)
- Captures a frame every N seconds (configurable, default 10s)
- Averages all pixel RGB values using a `<canvas>` element
- Sends a JSON packet to the MQTT broker via MQTT.js over WebSocket
- Config fields: Location ID (city/country format), Interval, Broker URL
- Collapsible Quick Start Guide panel accessible from the header
- Colour history ring around the centre swatch — clockwise arc segments, one per capture,
  up to 36 samples, oldest evicted when full

**Packet format sent by each node:**
```json
{
  "location_id": "london, uk",
  "r": 142,
  "g": 178,
  "b": 201,
  "hex": "#8EB2C9",
  "timestamp": "2026-05-17T14:23:01.000Z",
  "timezone": "Europe/London"
}
```

### `capture_edge.py`
Alternative Python edge node for devices that can run Python (Raspberry Pi, laptops).
Same job as `index.html` but runs in a terminal. Uses OpenCV for camera capture.
Supports both MQTT and WebSocket transports. Config at top of file.

---

## Hub (local only — gitignored)

Lives in `hub/`. Not committed to the repo. Run with `npm start` from the `hub/` directory.

### `hub/server.js`
Node.js server that:
- Subscribes to `installation/colour/#` on the MQTT broker
- Writes every packet to SQLite (`colours.db`) — `latest` table (one row per node) and
  `history` table (full log)
- Serves `hub.html` at `http://localhost:3000`
- Serves `display.html` at `http://localhost:3000/display?node=LOCATION_ID`
- Pushes live updates to connected browsers via Server-Sent Events (SSE)
- API endpoints: `GET /api/colours`, `GET /api/history`

### `hub/hub.html`
Operator dashboard. Shows one card per active node:
- Square colour swatch (live-updating)
- Historical colour strip along the right edge — last 10 colours, oldest top, newest bottom
- Location ID, hex value, timestamp in the edge device's local timezone
- Hide button (×) on hover — persists across refreshes via localStorage
- "N hidden" toggle in header to reveal and restore hidden nodes
- Display button (⤢) — opens the full-screen display page for that node in a new tab

### `hub/display.html`
Full-screen colour display for a single node. Designed to be sent to a dedicated screen.
- URL: `http://localhost:3000/display?node=LOCATION_ID`
- Pure colour fill, live-updating via SSE
- Bottom bar: location ID (left), hex code (centre), local time at edge (right)
- Time ticks live using the edge device's timezone
- Historical colour strip on right edge — last 10 colours, seeded from DB on load
- Amber pulsing reconnection indicator appears if hub connection drops, auto-clears on reconnect

---

## Transport: MQTT over WebSocket

Browsers cannot use raw MQTT (TCP), so `index.html` uses **MQTT over WebSocket** via the
[MQTT.js](https://github.com/mqttjs/MQTT.js) library. This is fully compatible with standard
MQTT brokers — the hub sees normal MQTT messages.

**Default broker for testing:** `wss://broker.hivemq.com:8884/mqtt` (HiveMQ free public broker)

For production, swap for a private broker (e.g. self-hosted Mosquitto, HiveMQ Cloud,
or CloudMQTT). The broker URL field is editable on the page.

---

## Deployment

`index.html` is hosted on **GitHub Pages**. Any push to `main` goes live automatically.
Live URL: `https://AxlOzzy.github.io/Whats-the-Sky-Where-You-Are`

The hub is **local only** — run `npm start` inside `hub/` on the operator's machine.
The `hub/` directory is gitignored.

---

## Physical installation setup

The hub machine runs `npm start` and connects to a **GL.iNet travel router** (recommended)
which creates a dedicated local WiFi network for the installation space.

Each display screen connects to the same WiFi and opens a browser pointed at:
```
http://192.168.8.100:3000/display?node=LOCATION_ID
```
(where `192.168.8.100` is the hub machine's static local IP on the GL.iNet network)

**Screen options (mix and match):**
- **Dumb screen + Fire Stick** — Silk browser, navigate to URL, fullscreen. Disable sleep:
  Settings → Display & Sounds → Sleep → Never
- **Smart TV** — built-in browser (Samsung Tizen, LG WebOS). Disable auto-sleep in TV settings.
- **Android tablet/phone** — Chrome. Use Android Screen Pinning to lock the tab.
- **iPad/iPhone** — Safari. Use Guided Access (triple-click) to lock to the display page.
- **Old laptop** — Chromium fullscreen (F11). Keep plugged in.
- **Raspberry Pi + any HDMI screen** — Chromium kiosk mode, auto-boots to URL on startup.

**Hub machine static IP (Mac):**
System Preferences → Network → Advanced → TCP/IP → Configure IPv4: Manually
- IP: `192.168.8.100`, Subnet: `255.255.255.0`, Router: `192.168.8.1`

**Key risks to test before show day:**
- Devices falling asleep mid-show (disable all auto-lock/auto-sleep)
- Devices losing WiFi (travel router keeps the network stable and isolated)
- Mixed screen resolutions — display page is pure colour fill so it adapts fine; text
  labels are fixed-position and may need checking on unusual aspect ratios

---

## Key decisions made so far

- Processing happens at the **edge** — only 3 numbers + metadata sent per capture, not images.
- Webpage approach chosen so any participant can join by opening a link — no install required.
- MQTT chosen over plain WebSocket for robustness — handles reconnection, queuing, scales to
  many simultaneous nodes.
- Hub is local-only (not hosted) because data only needs capturing during active sessions,
  and keeping it local avoids auth complexity and hosting costs.
- Hub display pages (`/display?node=X`) are designed to be opened on separate screens at the
  physical installation — one URL per node, full-screen colour.
- Timezone auto-detected on the edge device (`Intl.DateTimeFormat`) and included in every
  packet so the hub can show local time at each remote location.
- Display screens can be any browser-capable device (Fire Stick, smart TV, tablet, laptop,
  Raspberry Pi) — chosen for flexibility with recycled/mixed hardware at the venue.
- A dedicated travel router (GL.iNet) is used at the installation so the hub network is
  isolated and reliable, independent of venue WiFi.
