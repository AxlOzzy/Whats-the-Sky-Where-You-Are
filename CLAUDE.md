# Colour Installation — Project Briefing

This project is a distributed art installation that captures the average colour of live camera
feeds from remote locations around the world, and displays those colours in real time at a
central installation hub.

---

## How it works

1. **Edge nodes** — people at remote locations open a webpage on any device (phone, laptop, etc),
   point their camera at their surroundings, and hit Start. The page captures frames from the
   camera, averages all pixels down to a single RGB value, and sends it as a small JSON packet
   to an MQTT broker.

2. **MQTT broker** — acts as a message routing layer between edge nodes and the hub. Each
   location publishes to its own topic: `installation/colour/{location_id}`.

3. **Hub** (not yet built) — subscribes to all location topics on the MQTT broker and displays
   the incoming colours at the physical installation.

---

## Repo contents

### `index.html`
The edge capture page. Designed to be hosted on GitHub Pages and opened on any phone or laptop —
no install required. Key behaviours:
- Requests camera access via `getUserMedia` (rear camera preferred on mobile)
- Captures a frame every N seconds (configurable, default 10s)
- Averages all pixel RGB values using a `<canvas>` element
- Sends a JSON packet to the MQTT broker via MQTT.js over WebSocket
- Config fields on the page: Location ID, Interval, Broker URL

**Packet format sent by each node:**
```json
{
  "location_id": "iceland-01",
  "r": 142,
  "g": 178,
  "b": 201,
  "hex": "#8EB2C9",
  "timestamp": "2026-05-17T14:23:01+00:00"
}
```

### `capture_edge.py`
An alternative Python edge node script for devices that can run Python (Raspberry Pi, laptops).
Does the same job as `index.html` but runs in a terminal. Uses OpenCV for camera capture.
Supports both MQTT and WebSocket transports. Config is at the top of the file.

---

## Transport: MQTT over WebSocket

Browsers cannot use raw MQTT (TCP), so `index.html` uses **MQTT over WebSocket** via the
[MQTT.js](https://github.com/mqttjs/MQTT.js) library. This is fully compatible with standard
MQTT brokers — the hub sees normal MQTT messages.

**Default broker for testing:** `wss://broker.hivemq.com:8884/mqtt` (HiveMQ free public broker)

For production, swap this for a private broker (e.g. self-hosted Mosquitto, HiveMQ Cloud,
or CloudMQTT). The broker URL field is editable on the page so nodes can be pointed at a
new broker without a code change.

---

## Deployment

`index.html` is hosted on **GitHub Pages**. Any change pushed to `main` goes live automatically.
The live URL follows the pattern: `https://{username}.github.io/{repo-name}`

---

## What still needs building

- **Hub display** — a page or application that subscribes to `installation/colour/#` on the
  MQTT broker and displays live colour panels for each active location. This is the next thing
  to build. It should show one colour swatch per location, update in real time as packets arrive,
  and ideally show the location ID and last-updated timestamp alongside each colour.

---

## Key decisions made so far

- Processing happens at the **edge** (not the hub) to keep bandwidth minimal — only 3 numbers
  are sent per capture, not images.
- The webpage approach was chosen over Python/native apps so that any participant anywhere
  can join by just opening a link — no install, no technical knowledge required.
- MQTT was chosen over plain WebSocket for robustness — it handles reconnection, message
  queuing, and scales easily to many simultaneous nodes.
