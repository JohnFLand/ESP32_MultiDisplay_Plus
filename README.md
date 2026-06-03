# ESP32_MultiDisplay_MQTT_Broker

HTTP- and MQTT-controlled message display for ESP32-S3 with capacitive touch screen. Displays messages, clock, weather, calendar, and astronomical times (sun & moon) with full automation support via JSON HTTP API and MQTT. Also operates as a general-purpose MQTT broker and forwards display messages in parallel to a configurable list of secondary CYD (Cheap Yellow Display) boards.

> This is the **Elecrow Advance 5″ + MQTT Broker** variant. The earlier `ESP32_MultiDisplay_Elecrow_Advance` (v2.x) sketch is the HTTP-only predecessor.

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [What's New in v3.0](#whats-new-in-v30)
- [Features](#features)
- [Quick Start](#quick-start)
- [Sketch Folder Structure](#sketch-folder-structure)
- [Web Interface](#web-interface)
- [API Reference](#api-reference)
  - [Endpoints Summary](#endpoints-summary)
  - [Message Endpoints](#message-endpoints)
  - [Message Profiles](#message-profiles)
  - [Clock Configuration](#clock-configuration)
  - [Schedules](#schedules)
  - [Control Endpoint](#control-endpoint)
  - [System Management](#system-management)
- [MQTT Reference](#mqtt-reference)
  - [Broker Details](#broker-details)
  - [Topic Routing](#topic-routing)
  - [Display Message Topics](#display-message-topics)
  - [Screen Navigation Topics](#screen-navigation-topics)
  - [Device Control Topics](#device-control-topics)
  - [Hubitat Integration Topics](#hubitat-integration-topics)
  - [Hubitat MQTT Display Publisher Driver](#hubitat-mqtt-display-publisher-driver)
- [CYD Display Forwarding](#cyd-display-forwarding)
  - [displays.json](#displaysjson)
  - [LittleFS Upload](#littlefs-upload)
- [Local Air Pressure](#local-air-pressure)
- [JSON Format Reference](#json-format-reference)
- [GET Format Reference](#get-format-reference)
- [Examples](#examples)
- [Font Guide](#font-guide)
- [Time and Date Formats](#time-and-date-formats)
- [Advanced Features](#advanced-features)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Hardware Requirements

**Device:** Elecrow CrowPanel Advance 5.0″ HMI ESP32-S3

> ⚠️ **Important:** This is the *Advance* model (ESP32-S3-WROOM-1-N16R8), not the basic/TN Elecrow 5″. The two boards have completely different pin assignments, backlight circuits, and I2C buses. Do not use this sketch with the non-Advance Elecrow 5″.

**Specifications:**
- **Processor:** ESP32-S3-WROOM-1-N16R8
- **Flash:** 16MB
- **PSRAM:** 8MB OPI
- **Display:** 5.0″ 800×480 IPS RGB parallel (ST7262-class panel)
- **Touch:** GT911 capacitive (direct I2C on SDA=15, SCL=16, address 0x5D)
- **Backlight:** STC8H1K28 MCU at I2C address 0x30 — 16 hardware brightness steps (0x00–0x10)
- **Serial:** CH340K UART chip (not native USB)

**Development Environment:**
- **IDE:** Arduino IDE 2.x
- **Board:** ESP32S3 Dev Module
- **USB CDC On Boot:** **Disabled** — the board uses a CH340K UART chip. If left enabled, all sketch serial output is silently lost.
- **Board Package:** v3.x (tested with v3.3.7)
- **PSRAM:** OPI PSRAM
- **Flash:** 16MB, custom partition scheme (`partitions.csv`)

**Required Libraries (install via Library Manager):**
- `GFX Library for Arduino` (Arduino_GFX_Library) by Moon On Our Nation
- `sMQTTBroker` by Vyacheslav Shryaev
- `ArduinoJson` by Benoit Blanchon
- `ArduinoOTA` (built into ESP32 Arduino core) — Arduino IDE wireless upload
- `Update` (built into ESP32 Arduino core) — web browser OTA upload

---

## What's New in v3.0

### MQTT Broker

The device now runs a full **MQTT 3.1.1 broker** on port 1883. Any MQTT client on the network can connect to it — Hubitat, MQTT Explorer, Node-RED, ESP32 sensors, Tasmota devices, or any other standards-compliant client. The broker is general-purpose; only messages on the `display/*` and `device/*` topics trigger sketch-specific behaviour.

### Parallel CYD Display Forwarding

When a display message or clear command arrives via MQTT, the sketch simultaneously forwards it as HTTP POST to all configured CYD (secondary) displays. All CYDs receive the message in parallel — the total latency is the time to reach the *slowest* single display rather than the *sum* of all displays. CYD addresses are stored in `data/displays.json` on LittleFS and can be updated without reflashing.

### Hubitat MQTT Integration

A companion Groovy driver (`MQTT_Display_Publisher.groovy`) lets Hubitat publish display messages and control commands via MQTT with a single `publishMessage(body)` or `publishControl(json)` call — replacing the MultiPOST Sender's sequential HTTP approach. The Hubitat **MQTT Export Integration** (built-in beta, firmware 2.5+) is used to push device attribute changes (including air pressure sensor readings) to the broker automatically.

### Local Air Pressure via MQTT

Local air pressure is now received **event-driven** from the Hubitat MQTT Export Integration rather than polled via HTTP. The device subscribes to Hubitat device attribute events and updates the pressure value the moment Hubitat reports a change — typically within 1–2 seconds. The legacy HTTP Maker API fetch runs every 30 minutes as a backup in case MQTT is unavailable.

---

## Features

### Display & Text
- Receive text and display parameters via HTTP POST (JSON) or GET, or via MQTT
- **28 Roboto fonts** (6pt to 72pt)
- Text wrapping, rotation (0°/90°/180°/270°), and alignment (left/center/right, top/middle/bottom)
- **Scrolling** (left/right/up/down, speed 1–9)
- **Colors** — named colors or hex (#RRGGBB)
- **Flash mode** — swap background/text colors at specified interval
- **Pulse mode** — brightness pulsing effect
- **Message expiration** — auto-return to clock after specified seconds
- **Priority levels** — normal and high with override protection
- **Scrollable message history** (configurable size, persistent storage)

### Clock & Display
- Real-time clock synced via NTP; RTC backup (PCF8563)
- Configurable fonts, colors, and time/date formats
- **Pixel shifting** to prevent screen burn-in
- **Screensaver** with auto-dim after inactivity
- **Night mode** — auto-dim based on sunset/sunrise with three brightness levels and a separate night color profile
- **Sunrise/sunset and moonrise/moonset** computed locally using Jean Meeus algorithms

### Weather
- Current conditions and 3-day forecast from Open-Meteo (no API key required)
- Local air pressure from Hubitat MQTT Export Integration (event-driven push); HTTP Maker API fallback every 30 minutes
- Local readings shown without prefix; Open-Meteo estimates shown with `~` prefix
- Automatic weather screen refresh when fresh local pressure arrives via MQTT

### MQTT Broker
- Standard MQTT 3.1.1 broker on port 1883
- No authentication required (configurable)
- General-purpose — all topics pass through to subscribers; only `display/*` and `device/*` topics trigger local actions
- Full `device/control` and `device/clockconfig` command sets via MQTT (parity with HTTP API)
- Parallel HTTP forwarding to CYD displays on message/clear topics
- Local air pressure updates from Hubitat MQTT Export Integration

### System
- **Message Profiles** — save and recall up to 10 named display attribute sets
- **Scheduled actions** — brightness, screen switches, messages, reboot at absolute or sunrise/sunset-relative times
- **Status page** with uptime and message history
- **Persistent settings** storage
- **Device naming** — auto-named from MAC or customized via `secrets.h`
- **mDNS** — `http://<DEVICE_NAME>.local`
- **OTA firmware updates** — Arduino IDE network port or browser (`/update`)
- **Web debug logging** — real-time log viewer

---

## Quick Start

1. Copy `secrets.h.EXAMPLE` to `secrets.h` and fill in your WiFi credentials (and any other
   device-specific values, such as the Hubitat URL or MQTT fallback display list):
   ```cpp
   #define WIFI_SSID      "your_network"
   #define WIFI_PASSWORD  "your_password"
   // Optional: HTTP backup for local air pressure (Hubitat Maker API)
   #define HUBITAT_PRESSURE_URL "http://192.168.1.x/apps/api/NNN/devices/all?access_token=..."
   // MQTT CYD fallback list (used only if data/displays.json is absent)
   #define MQTT_DISPLAYS_FALLBACK R"([
     { "name":"CYD Living Room", "messageUrl":"http://x.x.x.x/message", "clearUrl":"http://x.x.x.x/clear" }
   ])"
   ```
   > **Location is no longer set in `secrets.h`.** After flashing, open
   > `http://YOUR_IP/locationconfig` in a browser and enter your latitude, longitude,
   > display name, and timezone. The defaults (San Diego, CA) will be used until you
   > configure your own values. Settings are saved to NVS and survive reboots.
2. In Arduino IDE set `Tools → USB CDC On Boot → Disabled` before flashing.
3. Edit `data/displays.json` with your actual CYD IP addresses.
4. Flash the sketch via USB, then upload `data/displays.json` via `Tools → Upload LittleFS`.
5. Configure Hubitat MQTT Export Integration to connect to the device IP on port 1883 (no credentials).
6. Install `MQTT_Display_Publisher.groovy` as a Hubitat driver and create a virtual device using it.
7. Send a test message:
   ```bash
   curl -X POST http://YOUR_IP/message \
     -H "Content-Type: application/json" \
     -d '{"text":"Hello World","bgcolor":"blue","textcolor":"white"}'
   ```

---

## Sketch Folder Structure

```
ESP32_MultiDisplay_MQTT_Broker/
  ESP32_MultiDisplay_MQTT_Broker.ino   Main sketch
  MQTTBroker.cpp                       MQTT broker + CYD forwarder (separate translation unit)
  MQTTBroker.h                         MQTT broker public API
  MQTTBroker.ino                       Bridge wrappers (no class definitions)
  AdminHandlers.ino                    HTTP endpoint handlers
  ClockDisplay.ino                     Clock rendering and night mode
  ElecrowHW.ino                        Hardware abstraction (display, touch, backlight)
  MessageEngine.ino                    Message queue, priority, effects
  Network.ino                          WiFi, NTP, mDNS, SNR watchdog
  Scheduler.ino                        Scheduled actions
  Settings.ino                         NVS persistence
  TouchInput.ino                       Touch handling and screen navigation
  UIDrawing.ino                        Screen drawing functions
  WeatherCalendar.ino                  Weather, calendar, local pressure fetch
  HTML.cpp / HTML.h                    Web UI and control endpoint handlers
  HelpFunctions.cpp / HelpFunctions.h  JSON helpers, color parsing, utilities
  AstroCalc.cpp / AstroCalc.h          Astronomical calculations
  PSRAM_Helpers.h                      PSRAMCanvas and RGBPanelGFXBridge
  WebDebugLog.cpp / WebDebugLog.h      Web debug logging
  RobotoFonts.h                        Roboto font data
  Colors.h                             Color constants
  DeviceInfo.h                         Device identity helpers
  User_Setup.h                         All configuration constants
  secrets.h                            WiFi credentials, MQTT fallback (not in repo)
  secrets.h.EXAMPLE                    Template — copy to secrets.h and set WiFi credentials
  secrets.h.EXAMPLE                    Template for secrets.h
  partitions.csv                       Custom 16MB partition layout
  data/
    displays.json                      CYD display registry (uploaded to LittleFS)
```

> **Why MQTTBroker.cpp instead of .ino?**
> Arduino IDE's prototype generator scans all `.ino` files and generates forward declarations for cross-file function calls. A class definition that inherits from an external library type (`sMQTTBroker`) in a `.ino` file defeats the generator, causing cascade "not declared" errors for every function in the entire sketch. Moving the class to a `.cpp` file (compiled as a separate translation unit, never processed by the generator) resolves this. `MQTTBroker.ino` contains only simple wrapper functions with no class definitions.

---

## Web Interface

- **`http://YOUR_IP/`** — interactive message form + quick reference + live JSON/GET preview
- **`http://YOUR_IP/status`** — device status & message history
- **`http://YOUR_IP/clockconfig`** — clock configuration
- **`http://YOUR_IP/locationconfig`** — location & timezone configuration
- **`http://YOUR_IP/control`** — device controls
- **`http://YOUR_IP/update`** — OTA firmware update
- **`http://YOUR_IP/logs`** — real-time debug logs (if enabled)

---

## API Reference

### Endpoints Summary

| Endpoint | Methods | Purpose |
|----------|---------|---------|
| `/` | GET | Web UI / documentation |
| `/message` | GET/POST | Send message (main API) |
| `/msg` | GET | Simple message (query params only) |
| `/clear` | GET/POST | Clear messages, show clock |
| `/clock` | GET/POST | Navigate to clock screen |
| `/clockconfig` | GET/POST | Configure clock settings |
| `/locationconfig` | GET/POST | Configure location & timezone |
| `/weather` | GET/POST | Navigate to weather screen |
| `/calendar` | GET/POST | Navigate to calendar screen |
| `/astro` | GET/POST | Navigate to Astro screen |
| `/schedules` | GET/POST | Configure scheduled actions |
| `/control` | GET/POST | Universal control endpoint |
| `/status` | GET | Device status (HTML) |
| `/status.json` | GET | Device status (JSON) |
| `/time` | GET | Current time |
| `/logs` | GET | Debug logs (if enabled) |
| `/otapassword` | GET/POST | OTA password configuration |
| `/reboot` | GET/POST | Reboot device |
| `/resetsettings` | GET/POST | Reset settings + reboot |
| `/resetdevice` | GET/POST | Reset settings + reboot (alias) |
| `/update` | GET/POST | OTA firmware update |

### MQTT Data Broadcast Topics

After each successful fetch, the broker publishes retained messages on three topics.
CYD devices subscribe to receive live data without making their own internet or LAN requests.

| Topic | Retained | Published when | Payload |
|-------|----------|---------------|---------|
| `multidisplay/weather` | Yes | After each Open-Meteo fetch (~10 min) | JSON — temp, apparent temp, humidity, surface pressure, weather code, daily forecast, celsius flag, unix timestamp |
| `multidisplay/pressure` | Yes | After each Hubitat pressure fetch/receive | `{"hpa": 1013.2}` |
| `multidisplay/time` | Yes | Every 60 seconds | `{"epoch": 1234567890, "tz": "PST8PDT,M3.2.0,M11.1.0"}` |

CYDs fall back to their own HTTP fetches automatically if a topic goes stale (broker offline, WiFi interrupted). No configuration is needed on the broker side beyond normal operation.

---

### Message Endpoints

#### `/message` — Main Message API

```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Hello World","bgcolor":"blue","textcolor":"white","font":36,"expirationtime":30}'
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `text` | string | — | Message text |
| `bgcolor` | string | `"white"` | Background color (named or #RRGGBB) |
| `textcolor` | string | `"black"` | Text color |
| `font` | int | `30` | Font size (see [Font Guide](#font-guide)) |
| `rotation` | int | `0` | 0, 90, 180, 270 |
| `halign` | string | `"center"` | left / center / right |
| `valign` | string | `"middle"` | top / middle / bottom |
| `wrap` | bool | `true` | Enable text wrapping |
| `brightness` | int | `5` | 0–5 |
| `expirationtime` | int | `0` | Seconds until auto-return to clock (0 = no expiry) |
| `priority` | string | `"normal"` | normal / high |
| `scrolldir` | string | `"none"` | none / left / right / up / down |
| `scrollspeed` | int | `3` | 1 (slow) – 9 (fast) |
| `flash` | bool | `false` | Flash effect |
| `flashinterval` | int | `1000` | Flash interval ms (500–60000) |
| `pulse` | bool | `false` | Pulse brightness effect |
| `pulseduration` | int | `2000` | Pulse cycle ms (500–600000) |
| `retrieveprofile` | int | — | Apply saved profile 0–9 before other params |
| `saveprofile` | int | — | Save current params to profile 0–9 |
| `clearprofile` | int | — | Clear saved profile 0–9 |

---

### Message Profiles

Save and recall up to 10 sets of display attributes (profiles 0–9). Persist across reboots.

```json
POST /message
{ "bgcolor":"red","textcolor":"white","font":48,"flash":true,"saveprofile":1 }
```
```json
POST /message
{ "text":"Water Leak","retrieveprofile":1,"priority":"high" }
```

---

### Clock Configuration

`POST /clockconfig` accepts JSON with any combination of:

| Field | Type | Description |
|-------|------|-------------|
| `timeformat` | string | strftime format for time (e.g. `"%l:%M %P"`) |
| `dateformat` | string | strftime format for date |
| `timefont` | int | Time font size |
| `datefont` | int | Date font size |
| `timecolor` | string | Time text color |
| `datecolor` | string | Date text color |
| `bgcolor` | string | Clock background color |
| `clksuntimescolor` | string | Sunrise/sunset color on clock |
| `nighttimecolor` | string | Time color during night mode |
| `nightdatecolor` | string | Date color during night mode |
| `nightbgcolor` | string | Background during night mode |
| `nightsuntimescolor` | string | Suntimes color during night mode |
| `labelfont` | int | Screen label font |
| `msgnoticefont` | int | Message notice font |
| `weathertempfnt` | int | Weather temperature font |
| `weatherhumfnt` | int | Weather humidity font |
| `weatherpresfnt` | int | Weather pressure font |
| `labelcolor` | string | Label text color |
| `msgnoticecolor` | string | Message notice color |
| `rotation` | int | Screen rotation: 0/90/180/270 |
| `brightness` | int | Screen brightness 0–5 |
| `screensaverdim` | int | Screensaver brightness 0–5 |
| `screensavertimeout` | int | Screensaver timeout ms (0 = disabled) |
| `clockenable` | bool | Enable/disable clock |
| `clksuntimesenabled` | bool | Show sunrise/sunset on clock |
| `nightmodeenable` | bool | Enable auto-dim night mode |
| `nightbrightness` | int | Night quiescent brightness 0–5 |
| `daybrightness` | int | Night touch-wake brightness 0–5 |
| `nightwakedur` | int | Touch-wake duration seconds |
| `sunsetoffset` | int | Minutes offset from sunset (negative = before) |
| `sunriseoffset` | int | Minutes offset from sunrise |
| `tempcelsius` | bool | Display temperature in Celsius |

---

### Schedules

Up to 10 scheduled actions (slots 0–9), persisted across reboots.

**Actions:** `brightness`, `clock`, `weather`, `calendar`, `message`, `reboot`

**Timing modes:**
- `"absolute"` — fires at a fixed time daily (`hour`, `minute`)
- `"relative"` — fires offset from `"sunrise"` or `"sunset"` (`offset` in minutes, negative = before)

```json
POST /schedules
{ "index":0,"enabled":true,"mode":"relative","reference":"sunset","offset":-30,"action":"brightness","brightness":3 }
```

---

### Control Endpoint

`POST /control` (JSON) or `GET /control?param=value`

#### Boolean / Toggle Settings

| Key | Description |
|-----|-------------|
| `clockdark` | Clock dark mode |
| `weatherdark` | Weather dark mode |
| `calendardark` | Calendar dark mode |
| `menudark` | Menu dark mode |
| `historydark` | Message history dark mode |
| `astrodark` | Astro screen dark mode |
| `pixelshift` | Pixel shift anti burn-in |
| `touchenable` | Touch input enable |
| `weathercycle` | Auto-cycle to weather |
| `calendarcycle` | Auto-cycle to calendar |
| `clockenable` | Clock enable |
| `normoverride` | Normal message override mode |

Values: `true`, `false`, or `"toggle"`.

#### Numeric Settings

| Key | Unit | Description |
|-----|------|-------------|
| `wxshowdur` | seconds | Weather screen display duration |
| `wxcycleint` | seconds | Weather cycle interval |
| `calshowdur` | seconds | Calendar display duration |
| `calcycleint` | seconds | Calendar cycle interval |
| `hpalternate` | seconds | High-priority alternation interval |
| `normoverridedur` | seconds | Normal override duration |
| `pulsemaxbri` | 0–5 | Pulse maximum brightness |
| `pulseminbri` | 0–5 | Pulse minimum brightness |
| `maxmsgage` | hours | Max message history age (0 = disabled) |
| `localpressure` | hPa int | Push local air pressure reading (800–1200) |

#### Temperature Display

| Key | Value | Description |
|-----|-------|-------------|
| `tempactual` | `true` | Show actual temperature |
| `tempactual` | `false` | Show apparent temperature |
| `tempactual` | `"alternate"` | Alternate actual and apparent |

#### Navigation

```json
{ "clock":true }    { "weather":true }    { "calendar":true }
```

#### History / Messaging

```json
{ "clear":true }    { "clearhistory":true }
```

#### Clock Reset

```json
{ "clockresetcolors":"dark" }    { "clockresetall":"light" }
```

#### System

```json
{ "reboot":true }    { "resetsettings":true }    { "resetdevice":true }
```

---

### System Management

```bash
curl http://YOUR_IP/reboot
curl -X POST http://YOUR_IP/resetsettings
```

OTA: navigate to `http://YOUR_IP/update` in a browser and upload a `.bin` file.

---

## MQTT Reference

### Broker Details

| Property | Value |
|----------|-------|
| Protocol | MQTT 3.1.1 |
| Port | 1883 (plain TCP) |
| Credentials | None required |
| Max clients | ~20–30 simultaneous (limited by ESP32-S3 PSRAM heap) |
| Persistence | None — subscriptions and retained messages are not stored across reboots |

The broker is **general-purpose**. Topics other than `display/*` and `device/*` pass through to subscribers without triggering any local action. Any standard MQTT client (MQTT Explorer, Node-RED, Home Assistant, PubSubClient, etc.) can connect and use arbitrary topics.

### Topic Routing

| Topic | Direction | Description |
|-------|-----------|-------------|
| `display/message` | → broker | Display locally + forward to all CYDs |
| `display/clear` | → broker | Clear locally + forward clear to all CYDs |
| `display/message/elecrow` | → broker | Display locally only |
| `display/message/cyd` | → broker | Forward to all CYDs only |
| `display/message/device/<name>` | → broker | Forward to one named CYD |
| `display/clear/device/<name>` | → broker | Clear one named CYD |
| `display/screen` | → broker | Navigate to a screen |
| `device/control` | → broker | Control commands (JSON) |
| `device/clockconfig` | → broker | Clock configuration (JSON) |
| `hubitat/<uuid>/devices/<id>` | → broker | Hubitat device state (from MQTT Export Integration) |

### Display Message Topics

The message payload is the same JSON format accepted by `POST /message`:

```json
{
  "text": "Motion detected",
  "bgcolor": "red",
  "textcolor": "white",
  "font": 36,
  "priority": "high",
  "expirationtime": 30
}
```

Plain text payloads are also accepted and displayed as-is with default styling.

**Topic | Effect:**
- `display/message` — shows on this display + sends to all CYDs in parallel
- `display/message/elecrow` — shows on this display only
- `display/message/cyd` — sends to all CYDs only (no local display)
- `display/message/device/CYD Bookcase` — sends to that one CYD only

> **Note:** The device name in the topic must exactly match the `"name"` field in `displays.json` including capitalisation and any underscores.

**Clear topics:**
- `display/clear` — clears this display + all CYDs
- `display/clear/device/CYD Garage` — clears one CYD only

### Screen Navigation Topics

Publish to `display/screen` with a plain text payload:

```
display/screen   →   weather
display/screen   →   calendar
display/screen   →   clock
```

### Device Control Topics

`device/control` accepts the same JSON keys as `POST /control`:

```json
{ "clockdark": "toggle" }
{ "pixelshift": true, "maxmsgage": 24 }
{ "reboot": true }
{ "weather": true }
{ "clearhistory": true }
{ "localpressure": 1013 }
```

`device/clockconfig` accepts the same JSON keys as `POST /clockconfig`:

```json
{ "timefont": 60, "timecolor": "#00FFFF", "nightmodeenable": true }
```

### Hubitat Integration Topics

When Hubitat's **MQTT Export Integration** is connected to this broker, it publishes device states to:

```
hubitat/<hub-uuid>/devices/<device-id>
```

The payload is a JSON object containing the device's full attribute list. Example for a pressure sensor (device 1024):

```json
{
  "id": 1024,
  "name": "VD Air Pressure Average",
  "attributes": [
    { "name": "pressure", "value": "1013.2", "unit": "hPa" },
    ...
  ]
}
```

The broker's `onEvent()` handler extracts the `pressure` attribute value and updates the local pressure display automatically. **No Hubitat rule is required** — publishing happens automatically whenever the device's pressure attribute changes.

**Setup:**
1. In Hubitat go to **Integrations → Add Built-In Integration → MQTT Export Integration**
2. Disable "Use built-in MQTT service"
3. Set host to the device IP, port 1883, no credentials
4. Add your pressure sensor to the **read only devices** list
5. Set `HUBITAT_PRESSURE_DEVICE_ID` in `User_Setup.h` to match the Hubitat device ID
6. Save and reconnect

> **Beta note:** The MQTT Export Integration may publish `homeassistant/` discovery topics even when that option is disabled (known Hubitat beta bug). These are filtered out silently by the sketch and have no effect.

### Hubitat MQTT Display Publisher Driver

`MQTT_Display_Publisher.groovy` is a Hubitat virtual device driver that publishes display messages and control commands to this broker. Install it via **Drivers Code → New Driver**, create a virtual device using it, and configure it with the broker IP.

**Rule Machine commands:**
- `publishMessage(body)` — publish JSON to `display/message` (all displays)
- `publishClear()` — publish to `display/clear`
- `publishToGroup(group, body)` — publish to `display/message/elecrow` or `display/message/cyd`
- `publishControl(json)` — publish to `device/control`

---

## CYD Display Forwarding

When a message arrives on `display/message` or `display/clear`, the broker spawns one FreeRTOS task per configured CYD display. All HTTP POSTs fire simultaneously — latency is determined by the slowest single display, not the sum.

**Retry behaviour:** 3 total attempts per display (initial + 2 retries), 500ms between retries. HTTP 2xx and 3xx responses are treated as success (some CYD sketches respond to `/clear` with a 303 redirect).

### displays.json

Stored in the `data/` folder and uploaded to LittleFS. Edit this file and re-upload via LittleFS uploader to change CYD addresses without reflashing.

```json
[
  { "name":"CYD_Bookcase",
    "messageUrl":"http://192.168.1.78/message",
    "clearUrl":  "http://192.168.1.78/clear" },
  { "name":"CYD_Closet",
    "messageUrl":"http://192.168.1.79/message",
    "clearUrl":  "http://192.168.1.79/clear" },
  { "name":"CYD_Garage",
    "messageUrl":"http://192.168.1.70/message",
    "clearUrl":  "http://192.168.1.70/clear" },
  { "name":"CYD_Desk",
    "messageUrl":"http://192.168.1.71/message",
    "clearUrl":  "http://192.168.1.71/clear" }
]
```

> **Important:** The file must be a valid JSON array — it must start with `[` and end with `]`. If the file is absent or invalid, the fallback list from `MQTT_DISPLAYS_FALLBACK` in `secrets.h` is used.

If `displays.json` is not present on LittleFS, the `MQTT_DISPLAYS_FALLBACK` constant defined in `secrets.h` is used. If that is also absent, a single-entry emergency fallback defined in `MQTTBroker.cpp` is used. The Serial Monitor reports which source loaded at boot.

### LittleFS Upload

To update `displays.json` without reflashing:

1. Install the **arduino-littlefs-upload** plugin from `https://github.com/earlephilhower/arduino-littlefs-upload/releases` — place the `.vsix` file in `C:\Users\<you>\.arduinoIDE\plugins\` and restart Arduino IDE
2. Edit `data/displays.json`
3. Close the Serial Monitor
4. Press **Ctrl+Shift+P** → **Upload LittleFS to Pico/ESP8266/ESP32**

---

## Local Air Pressure

Local air pressure is displayed on the weather screen alongside the Open-Meteo model estimate.

**Display convention:**
- `1013.2 hPa` — local sensor reading (no prefix)
- `~1012.8 hPa` — Open-Meteo model estimate (`~` prefix)
- Alternates between local and Open-Meteo every few seconds (configurable)

**Primary source — MQTT (event-driven):**
Hubitat MQTT Export Integration publishes attribute changes for the pressure virtual device. The broker receives the update immediately and refreshes the weather screen if it is currently displayed.

**Backup source — HTTP Maker API (polled):**
`HUBITAT_PRESSURE_URL` in `secrets.h` defines a Hubitat Maker API endpoint that returns:
```json
{ "hpa": 1013.5, "sensors": 2, "age_seconds": 30 }
```
This is polled every 30 minutes, but only fires if MQTT hasn't delivered a fresh reading recently. When MQTT delivers a value, the 30-minute timer resets.

**Push from Hubitat rule:**
A pressure value can also be pushed at any time via `device/control`:
```json
{ "localpressure": 1013 }
```
Or via HTTP: `POST /control {"localpressure": 1013}`

**Boot behaviour:**
At boot, `fetchWeather()` runs immediately. If Hubitat hasn't yet reconnected to the broker (typically takes 10–30 seconds after broker restart), the weather screen shows Open-Meteo pressure with the `~` prefix. When MQTT delivers the local pressure, the weather screen refreshes automatically if it is currently visible.

---

## JSON Format Reference

### Control Endpoint JSON

```json
POST /control
{
  "clockdark": true,
  "weatherdark": false,
  "pixelshift": true,
  "wxshowdur": 15,
  "wxcycleint": 300,
  "pulsemaxbri": 5,
  "pulseminbri": 2,
  "maxmsgage": 24,
  "tempactual": "alternate"
}
```

### Schedules JSON

```json
POST /schedules
{
  "index": 1,
  "enabled": true,
  "mode": "relative",
  "reference": "sunset",
  "offset": 0,
  "action": "brightness",
  "brightness": 3
}
```

---

## GET Format Reference

Most POST parameters are also accepted as GET query params:

```
GET /message?text=Hello&bgcolor=blue&font=36&expirationtime=30
GET /control?clockdark=true&pixelshift=toggle
GET /clockconfig?brightness=5&nightmodeenable=true
```

**Named colors:** `red`, `green`, `blue`, `white`, `black`, `yellow`, `cyan`, `magenta`, `orange`, `purple`, `pink`, `brown`, `gray`, `lightgray`, `darkgray`, `dark_green`

**Hex colors:** `#RRGGBB` (e.g. `#FF8800`)

---

## Examples

### Message Examples

```bash
# Simple centered message with expiry
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Doorbell","bgcolor":"orange","textcolor":"black","font":48,"expirationtime":10}'

# Scrolling ticker
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Breaking news scrolling left","scrolldir":"left","scrollspeed":4}'

# High-priority flashing alert
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"SMOKE ALARM","bgcolor":"red","textcolor":"white","flash":true,"flashinterval":500,"priority":"high"}'
```

### MQTT Examples (via MQTT Explorer or any MQTT client)

```
Topic:   display/message
Payload: {"text":"Front door opened","bgcolor":"blue","textcolor":"white","expirationtime":15}

Topic:   display/screen
Payload: weather

Topic:   device/control
Payload: {"clockdark":"toggle"}

Topic:   device/control
Payload: {"reboot":true}

Topic:   display/message/device/CYD_Bookcase
Payload: {"text":"Garage door open","bgcolor":"orange","textcolor":"black"}

Topic:   display/clear/device/CYD_Desk
Payload: (any or empty)
```

### Schedules Examples

```bash
# Dim at 10pm
curl -X POST http://YOUR_IP/schedules \
  -H "Content-Type: application/json" \
  -d '{"index":0,"enabled":true,"mode":"absolute","hour":22,"minute":0,"action":"brightness","brightness":3}'

# Brighten at sunrise
curl -X POST http://YOUR_IP/schedules \
  -H "Content-Type: application/json" \
  -d '{"index":1,"enabled":true,"mode":"relative","reference":"sunrise","offset":0,"action":"brightness","brightness":5}'

# Daily reboot at 3am
curl -X POST http://YOUR_IP/schedules \
  -H "Content-Type: application/json" \
  -d '{"index":2,"enabled":true,"mode":"absolute","hour":3,"minute":0,"action":"reboot"}'
```

---

## Font Guide

All fonts are Roboto Regular. Font code equals point size.

| Code | Typical Use |
|------|-------------|
| 6–12 | Status labels, fine print |
| 14–20 | Notices, weather body, dates |
| 24–28 | Menu buttons, weather values |
| 30 | **Default message font** |
| 32–48 | Large messages |
| 60 | **Default clock time font** |
| 66–72 | Full-screen display |

Valid codes: 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 32, 36, 40, 44, 48, 54, 60, 66, 72

---

## Time and Date Formats

| Code | Meaning | Example |
|------|---------|---------|
| `%l` | Hour 1–12 | 9 |
| `%H` | Hour 00–23 | 09 |
| `%M` | Minutes | 45 |
| `%P` | am/pm | am |
| `%A` | Full weekday | Sunday |
| `%B` | Full month | November |
| `%d` | Day 01–31 | 03 |
| `%Y` | 4-digit year | 2025 |

Defaults: `"%l:%M %P"` → `9:45 am` / `"%A  %Y-%m-%d"` → `Sunday  2025-11-23`

---

## Advanced Features

### Message Priority System

**Normal** — displays immediately if no high-priority message is active; queued otherwise.

**High** — interrupts normal messages. Multiple high-priority messages alternate at a configurable interval. Persists until expiry or screen tap.

**Normal Override** — when enabled, a normal message can temporarily push aside a high-priority message for a configurable duration, then the high-priority message returns.

---

### Touch Controls

**Clock screen:** Touch opens Menu (first touch during night mode brightens display only).

**Message screen:** Tap dismisses current message.

**Message History:** Swipe up/down or tap ▲/▼ to navigate. Hold **«** (500ms) to return to clock. Hold **✕** (500ms) to clear history.

**Weather screen:** Tap returns to clock.

**Calendar screen:** Tap ◄/► to change month; tap title to return to current month; tap data area for clock.

**Astro screen:** Tap ◄/► to change date; tap title to toggle Sun/Moon page; tap footer for clock.

---

### Pixel Shift (Anti Burn-In)

Shifts display content by 1–2 pixels every 5 minutes to prevent static element burn-in. Enable: `POST /control {"pixelshift":true}`.

---

### Message History

Scrollable history of received messages. Navigate by swipe or arrows. Messages older than `maxmsgage` hours are automatically purged (0 = disabled).

---

### Weather & Calendar

Weather from Open-Meteo API — no API key required. Shows current conditions, 3-day forecast, and astronomical sun times. Calendar shows current month with today highlighted.

---

### Astro Times (Sun & Moon)

Computed locally using Jean Meeus algorithms from the latitude, longitude, and timezone configured via the **📍 Location** page (`/locationconfig`).

**Sun page:** all twilight times, solar noon, day length.
**Moon page:** moonrise/set, illumination, phase name and graphic.

Browse by date with ◄/► arrows. Accurate to within a few minutes worldwide.

---

## Testing

### HTTP Test Suite

`test_multidisplay.py` exercises all HTTP endpoints against a live device.

```bash
pip install requests
python test_multidisplay.py --ip 192.168.1.x
python test_multidisplay.py --ip 192.168.1.x --group messages
```

Groups: `endpoints`, `messages`, `effects`, `priority`, `profiles`, `clockconfig`, `control`, `schedules`, `nightmode`, `edge`

### MQTT Testing

Use **MQTT Explorer** (`https://mqtt-explorer.com`) to:
- Monitor all topics flowing through the broker in real time
- Publish test messages to any topic
- Verify Hubitat device state publications
- Confirm CYD forwarding (watch Serial Monitor for `MQTT Fwd [...] OK`)

---

## Troubleshooting

### MQTT Issues

**No clients connecting?** Verify broker IP in client configuration. Confirm port 1883. Check Serial Monitor for `MQTT: broker started on port 1883`.

**Hubitat MQTT Export Integration shows "not connected"?** Verify the device IP hasn't changed (set a DHCP reservation in your router). Confirm no credentials are needed. Try Save and reconnect.

**Flood of `homeassistant/` topics in Serial Monitor?** Known Hubitat beta bug — HA discovery traffic is published even when that option is disabled. These are filtered and have no effect. Report to Hubitat community.

**CYD forwarding failing with HTTP 303?** Normal — some CYD sketches redirect after `/clear`. The sketch treats 2xx and 3xx as success. No action needed.

**CYD forwarding failing with HTTP -1?** The CYD is offline or unreachable. Check its IP in `displays.json`.

---

### Display Issues

**Weather screen shows `~` prefix only?** Open-Meteo pressure is showing because local pressure hasn't arrived yet. Wait 10–30 seconds after boot for Hubitat to reconnect and publish. If it never arrives, check that the pressure device is added to Hubitat MQTT Export Integration and that `HUBITAT_PRESSURE_DEVICE_ID` in `User_Setup.h` matches the Hubitat device ID. Check Serial Monitor for `MQTT: local pressure updated`.

**Text not wrapping?** Enable `wrap: true` in message JSON.

**Display too dim?** Increase brightness to 5. Check night mode and screensaver are not active.

---

### Clock Issues

**Clock not showing?** `POST /control {"clockenable":true}`.

**Time wrong?** Verify the **POSIX TZ String** on the Location config page (`http://YOUR_IP/locationconfig`). Check `http://YOUR_IP/time`.

---

### Serial Issues

**No output after flashing?** Set `Tools → USB CDC On Boot → Disabled`. The CH340K UART appears as "USB-SERIAL CH340K" in Device Manager. Seeing ROM boot lines but nothing after is the telltale sign of CDC being enabled.

---

### Compilation Issues

**"not declared in scope" cascade for every function?** A class definition in a `.ino` file is defeating Arduino's prototype generator. Ensure `MQTTBroker.ino` contains no class definitions — the broker class must stay in `MQTTBroker.cpp`.

**Partition errors?** Verify `partitions.csv` is in the sketch folder and `Tools → Partition Scheme → Custom` is selected.

---

### OTA Issues

**Web OTA fails?** Use `http://YOUR_IP/update`. Ensure stable WiFi. Correct `.bin` file. Verify OTA password if set.

---

## License

**Dedicated to the public domain in 2025.**

Distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND.

---

**Author:** John Land
**Last Updated:** 2026-05-23
**Version:** 3.1-EL-MQTT
