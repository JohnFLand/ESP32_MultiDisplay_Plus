# ESP32_MultiDisplay

HTTP-controlled message display for ESP32-S3 with 3.5" capacitive touch screen. Displays messages, clock, weather, calendar, and astronomical times (sun & moon) with full automation support via JSON API.

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Features](#features)
- [Quick Start](#quick-start)
- [Web Interface](#web-interface)
- [API Reference](#api-reference)
  - [Endpoints Summary](#endpoints-summary)
  - [Message Endpoints](#message-endpoints)
  - [Message Profiles](#message-profiles)
  - [Clock Configuration](#clock-configuration)
  - [Schedules](#schedules)
  - [Control Endpoint](#control-endpoint)
  - [System Management](#system-management)
- [JSON Format Reference](#json-format-reference)
  - [Message JSON](#message-json)
  - [Message Profile JSON](#message-profile-json)
  - [Clock Configuration JSON](#clock-configuration-json)
  - [Schedules JSON](#schedules-json)
  - [Control Endpoint JSON](#control-endpoint-json)
- [GET Format Reference](#get-format-reference)
- [Examples](#examples)
  - [Message Examples](#message-examples)
  - [Message Profile Examples](#message-profile-examples)
  - [Clock Examples](#clock-examples)
  - [Control Examples](#control-examples)
  - [Schedules Examples](#schedules-examples)
- [Font Guide](#font-guide)
- [Time and Date Formats](#time-and-date-formats)
- [Advanced Features](#advanced-features)
  - [Message Priority System](#message-priority-system)
  - [Brightness Settings](#brightness-settings)
  - [Touch Controls](#touch-controls)
  - [Pixel Shift (Anti Burn-In)](#pixel-shift-anti-burn-in)
  - [Message History](#message-history)
  - [Weather & Calendar](#weather--calendar)
    - [Local Air Pressure](#local-air-pressure)
  - [Astro Times (Sun & Moon)](#astro-times-sun--moon)
- [Testing](#testing)
  - [HTTP Test Suite](#http-test-suite)
  - [Test Inject Endpoint](#test-inject-endpoint)
  - [Visual Verification Checklist](#visual-verification-checklist)
- [Troubleshooting](#troubleshooting)
- [Changelog](#changelog)
- [License](#license)

---

## Hardware Requirements

**Device:** ESP32-S3 with 3.5" AXS15231B Capacitive Touch Display

**Specifications:**
- **Processor:** ESP32-S3
- **Flash:** 16MB
- **PSRAM:** 8MB
- **Display:** 3.5" 320×480 capacitive touch
- **Driver:** AXS15231B

**Development Environment:**
- **IDE:** Arduino IDE 2.3.7
- **Board:** ESP32S3 Dev Module
- **Partition:** Custom partition scheme (partitions.csv)

---

## Features

### Display & Text
- Receive text and display parameters via HTTP POST (JSON) or GET requests
- **22 Roboto fonts** (6pt to 48pt)
- Text wrapping control
- **Text rotation** (0°, 90°, 180°, 270°)
- **Text alignment** (horizontal: left/center/right, vertical: top/middle/bottom)
- **Scrolling** (horizontal: left/right, vertical: up/down, speed 1-9)
- **Colors** - Named colors or hex (#RRGGBB) for background and text
- **Flash mode** - Swap background/text colors at specified interval
- **Pulse mode** - Brightness pulsing effect
- **Message expiration** - Automatically return to clock after specified seconds
- **Priority levels** - "normal" and "high" priority with override protection
- **Scrollable message history** (configurable size, persistent storage)

### Clock & Display
- **Real-time clock** synced via NTP
- Configurable fonts, colors, and time/date formats
- **Pixel shifting** to prevent screen burn-in
- **Screensaver** with auto-dim after inactivity
- Touch to wake from screensaver or return to clock
- **Night mode** - Auto-dim based on sunset/sunrise times, with three distinct brightness levels and a separate night color profile for the clock face (configurable time, date, background, and suntimes colors)
- **Sunrise/sunset and moonrise/moonset** displayed on Clock and Weather screens using computed astronomical data

### Weather
- Fetches current conditions and 3-day forecast from Open-Meteo (no API key required)
- **Local air pressure** — optionally fetches averaged sensor readings from a Hubitat Maker API endpoint every 5 minutes; local readings are shown without a prefix while Open-Meteo model estimates are shown with a `~` prefix; falls back to Open-Meteo when local data is absent or stale (>30 min). Local pressure can also be pushed to the device at any time via `POST /control {"localpressure": NNN}`

### Astronomical Calculations
- **Sun screen** — all twilight times (astronomical, nautical, civil) at dawn and dusk, plus sunrise, solar noon, sunset, and day length; all computed locally from your latitude/longitude
- **Moon screen** — moonrise and moonset times, illumination percentage, phase angle, and a rendered phase graphic with phase name
- Times computed using Jean Meeus algorithms (same as the WeekCalendar JavaScript app); accurate to within a few minutes for any location worldwide
- DST-aware: UTC offset is derived at runtime from the device's POSIX timezone string, so times are always correct regardless of season
- Computed sun times also replace Open-Meteo API sun times on the Clock and Weather screens, providing greater accuracy and eliminating a network dependency for that data

### System
- **Message Profiles** - Save and recall up to 10 named sets of display attributes (0–9), usable from POST commands and the Scheduler
- **Scheduled actions** - Trigger brightness changes, screen switches, or device reboot
- **Status page** with uptime and message history
- **Persistent settings** storage (survives power cycles)
- **mDNS support** - Access via `http://<DEVICE_NAME>.local`
- **Web debug logging** - Real-time log viewer (if enabled)
- **OTA firmware updates** - Update over WiFi

---

## Quick Start

1. **Install required library:** In Arduino Library Manager, search for and install
   **PubSubClient** by Nick O'Leary (version 2.8 or later). This enables the MQTT
   subscription client that receives weather, pressure, and time from the broker.
2. **Upload the sketch** to your ESP32-S3 device
3. **Configure WiFi and broker IP** in `secrets.h`:
   ```cpp
   #define WIFI_SSID        "your_network"
   #define WIFI_PASSWORD    "your_password"
   #define MQTT_BROKER_IP   "192.168.1.x"   // IP of the Elecrow MQTT broker
   // Optional: local air pressure from Hubitat Maker API (averaged across sensors)
   #define HUBITAT_PRESSURE_URL "http://192.168.1.x/apps/api/NNN/devices/all?access_token=..."
   ```
   > **Location is no longer set in `secrets.h`.** After flashing, open
   > `http://YOUR_IP/locationconfig` and enter your latitude, longitude, display name,
   > and timezone. Defaults (San Diego, CA) are used until configured. Settings are
   > saved to NVS and survive reboots.
4. **Navigate to** `http://YOUR_IP` for web interface
5. **Send a test message:**
   ```bash
   curl -X POST http://YOUR_IP/message \
     -H "Content-Type: application/json" \
     -d '{"text":"Hello World","bgcolor":"blue","textcolor":"white"}'
   ```
6. **View the clock:**
   ```bash
   curl http://YOUR_IP/clock
   ```

---

## Web Interface

Access the device via web browser:

- **`http://YOUR_IP/`** - Interactive message form + quick reference
- **`http://YOUR_IP/status`** - Device status & message history
- **`http://YOUR_IP/logs`** - Real-time debug logs (if enabled)
- **`http://YOUR_IP/clockconfig`** - Clock configuration page
- **`http://YOUR_IP/locationconfig`** - Location & timezone configuration
- **`http://YOUR_IP/control`** - Device controls

### MQTT Data Client

Each CYD subscribes to three retained MQTT topics published by the Elecrow broker:

| Topic | Data received |
|-------|---------------|
| `multidisplay/weather` | Current conditions + daily forecast (replaces Open-Meteo HTTP fetch) |
| `multidisplay/pressure` | Local air pressure from Hubitat (replaces direct Hubitat HTTP fetch) |
| `multidisplay/time` | UTC epoch + POSIX TZ string (broker RTC backup when internet is down) |

**Fallback behavior:** If the broker goes offline or a topic goes stale, the CYD
automatically falls back to its own HTTP fetches — no manual intervention needed.
The transition is seamless in both directions.

**Temperature unit:** The broker and all CYDs must use the same temperature unit
(Celsius or Fahrenheit). A unit mismatch causes the CYD to skip the MQTT weather
payload and use its own HTTP fetch instead.

**Status:** The `/status` page shows the current MQTT broker connection state.

**mDNS Access:**
If configured, you can also use: `http://Display35-1.local` (replace with your device name)

**Note:** mDNS may not work in some automation platforms like Hubitat Rule Machine. Use the IP address instead.

### Root Page — JSON & GET Preview

The root page (`/`) includes a **live JSON and GET preview** panel below the message form. As you fill in the form fields (text, colors, font, scroll, effects, etc.), both preview boxes update automatically to show exactly what the resulting API call will look like:

- **JSON preview** — a ready-to-copy JSON body for `POST /message`
- **GET preview** — a ready-to-copy query string for `GET /message?...`

Each preview has a **Copy** button. Only non-default values appear in the output, keeping the previews concise. This makes it easy to capture the exact parameters for use in Hubitat Rule Machine or any other HTTP client without manually constructing the payload.

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
| `/clockstart` | GET/POST | Navigate to clock (alias) |
| `/clockconfig` | GET/POST | Configure clock settings |
| `/locationconfig` | GET/POST | Configure location & timezone |
| `/weather` | GET/POST | Navigate to weather screen |
| `/calendar` | GET/POST | Navigate to calendar screen |
| `/schedules` | GET/POST | Configure scheduled actions |
| `/control` | GET/POST | Universal control endpoint |
| `/status` | GET | Device status (HTML) |
| `/status.json` | GET | Device status (JSON) |
| `/time` | GET | Current time |
| `/logs` | GET | Debug logs (if enabled) |
| `/otapassword` | GET/POST | OTA password configuration |
| `/reboot` | GET/POST | Reboot device (keep settings) |
| `/resetsettings` | GET/POST | Reset settings + reboot |
| `/resetdevice` | GET/POST | Reset settings + reboot (immediate) |
| `/update` | GET/POST | OTA firmware update |
| `/testinject` | GET/POST | Inject test state for night mode testing (requires `ENABLE_TEST_MODE true` in `User_Setup.h`; **disabled by default**) |

---

### Message Endpoints

#### `/message` - Main Message API

**Send a message via JSON POST:**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello World",
    "bgcolor": "blue",
    "textcolor": "white",
    "font": 24,
    "expirationtime": 30
  }'
```

**Send a message via GET:**
```bash
curl "http://YOUR_IP/message?text=Hello&bgcolor=blue&textcolor=white"
```

#### `/msg` — Simple Message (GET only)
Compatibility endpoint accepting query parameters only. Same parameters as `/message` GET.

---

### Message JSON

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `text` | string | — | Message text to display |
| `bgcolor` | string | `"white"` | Background color (named or #RRGGBB) |
| `textcolor` | string | `"black"` | Text color (named or #RRGGBB) |
| `font` | int | `30` | Font size in points (see [Font Guide](#font-guide)) |
| `rotation` | int | `0` | Rotation: 0, 90, 180, 270 |
| `halign` | string | `"center"` | Horizontal alignment: left/center/right |
| `valign` | string | `"middle"` | Vertical alignment: top/middle/bottom |
| `wrap` | bool | `true` | Enable text wrapping |
| `brightness` | int | `255` | Message brightness 0–255 |
| `expirationtime` | int | `0` | Seconds until auto-return to clock (0 = no expiration) |
| `priority` | string | `"normal"` | Priority: normal or high |
| `scrolldir` | string | `"none"` | Scroll direction: none/left/right/up/down |
| `scrollspeed` | int | `3` | Scroll speed 1 (slow) to 9 (fast) |
| `flash` | bool | `false` | Enable flash effect (swaps bg/text colors) |
| `flashinterval` | int | `1000` | Flash interval in ms (500–60000) |
| `pulse` | bool | `false` | Enable pulse brightness effect |
| `pulseduration` | int | `2000` | Pulse cycle duration in ms (500–600000) |
| `retrieveprofile` | int | — | Apply saved profile 0–9 before other params |
| `saveprofile` | int | — | Save current params to profile 0–9 |
| `clearprofile` | int | — | Clear saved profile 0–9 |

---

### Message Profiles

Save and recall up to 10 sets of display attributes (profiles 0–9). Profiles persist across reboots.

**Save a profile (with a message):**
```json
POST /message
{
  "text": "Alert!",
  "bgcolor": "red",
  "textcolor": "white",
  "font": 36,
  "saveprofile": 0
}
```

**Save a profile (no message — just stores the profile):**
```json
POST /message
{
  "bgcolor": "red",
  "textcolor": "white",
  "font": 36,
  "saveprofile": 0
}
```

**Use a saved profile:**
```json
POST /message
{
  "text": "Motion detected",
  "retrieveprofile": 0
}
```

**Clear a profile:**
```json
POST /message
{"clearprofile": 0}
```

---

### Clock Configuration

#### `GET /clockconfig` — Configuration Page

Returns the interactive HTML clock configuration page.

#### `POST /clockconfig` — Save Clock Settings

Accepts JSON or form POST.

**Clock Configuration JSON fields:**

| Field | Type | Description |
|-------|------|-------------|
| `timeformat` | string | strftime format for time (e.g. `"%l:%M %P"`) |
| `dateformat` | string | strftime format for date (e.g. `"%A  %Y-%m-%d"`) |
| `timefont` | int | Font size for time display |
| `datefont` | int | Font size for date display |
| `timecolor` | string | Time text color |
| `datecolor` | string | Date text color |
| `bgcolor` | string | Clock background color |
| `clksuntimescolor` | string | Sunrise/sunset text color on clock |
| `nighttimecolor` | string | Time text color during night mode |
| `nightdatecolor` | string | Date text color during night mode |
| `nightbgcolor` | string | Background color during night mode |
| `nightsuntimescolor` | string | Suntimes color during night mode |
| `labelfont` | int | Font for screen labels |
| `msgnoticefont` | int | Font for message notices/history |
| `weathertempfnt` | int | Font for weather temperature |
| `weatherhumfnt` | int | Font for weather humidity |
| `weatherpresfnt` | int | Font for weather pressure |
| `labelcolor` | string | Label text color |
| `msgnoticecolor` | string | Message notice text color |
| `rotation` | int | Screen rotation: 0, 90, 180, 270 |
| `brightness` | int | Screen brightness 0–255 |
| `screensaverdim` | int | Screensaver brightness 0–255 |
| `screensavertimeout` | int | Screensaver timeout in ms (0 = disabled) |
| `clockenable` | bool | Enable/disable the clock |
| `clksuntimesenabled` | bool | Show sunrise/sunset times on clock |
| `nightmodeenable` | bool | Enable night mode auto-dimming |
| `nightbrightness` | int | Night mode quiescent brightness 0–255 |
| `daybrightness` | int | Night mode touch-wake brightness 0–255 |
| `nightwakedur` | int | Touch-wake duration in seconds |
| `sunsetoffset` | int | Minutes offset from sunset (negative = before) |
| `sunriseoffset` | int | Minutes offset from sunrise (negative = before) |
| `tempcelsius` | bool | Display temperature in Celsius |

---

### Schedules

Up to 10 scheduled actions (0–9). Persisted across reboots.

#### Supported Actions

| Action | Description |
|--------|-------------|
| `brightness` | Set screen brightness |
| `clock` | Switch to clock screen |
| `weather` | Switch to weather screen |
| `calendar` | Switch to calendar screen |
| `message` | Display a message |
| `reboot` | Reboot the device |

#### Timing Modes

**Absolute** — fires at a fixed time each day:
```json
{"mode": "absolute", "hour": 22, "minute": 0}
```

**Relative** — fires offset from sunrise or sunset:
```json
{"mode": "relative", "reference": "sunset", "offset": -30}
```
`offset` is in minutes; negative = before the reference time.

#### `POST /schedules` — Update a Schedule

```bash
curl -X POST http://YOUR_IP/schedules \
  -H "Content-Type: application/json" \
  -d '{
    "index": 0,
    "enabled": true,
    "mode": "absolute",
    "hour": 22,
    "minute": 0,
    "action": "brightness",
    "brightness": 80
  }'
```

**Schedule JSON fields:**

| Field | Type | Description |
|-------|------|-------------|
| `index` | int | Schedule slot 0–9 |
| `enabled` | bool | Enable/disable this schedule |
| `mode` | string | `"absolute"` or `"relative"` |
| `hour` | int | Hour (0–23) for absolute mode |
| `minute` | int | Minute (0–59) for absolute mode |
| `reference` | string | `"sunrise"` or `"sunset"` for relative mode |
| `offset` | int | Minutes offset (±) for relative mode |
| `action` | string | Action to perform (see table above) |
| `brightness` | int | Target brightness 0–255 (for brightness action) |
| `message` | string | Message text (for message action) |

---

### Control Endpoint

`POST /control` (JSON) or `GET /control?param=value`

#### Boolean Settings (POST JSON or GET with `true`/`false`/`toggle`)

| Key | Description |
|-----|-------------|
| `clockdark` | Clock dark mode |
| `weatherdark` | Weather dark mode |
| `calendardark` | Calendar dark mode |
| `menudark` | Menu dark mode |
| `historydark` | Message history dark mode |
| `astrodark` | Astro screen dark mode |
| `pixelshift` | Pixel shift (anti burn-in) |
| `touchenable` | Enable/disable touch input |
| `weathercycle` | Auto-cycle to weather screen |
| `calendarcycle` | Auto-cycle to calendar screen |
| `clockenable` | Enable/disable the clock |
| `normoverride` | Normal message override mode |

#### Numeric Settings

| Key | Type | Description |
|-----|------|-------------|
| `wxshowdur` | int (sec) | Weather screen display duration |
| `wxcycleint` | int (sec) | Weather cycle interval |
| `calshowdur` | int (sec) | Calendar screen display duration |
| `calcycleint` | int (sec) | Calendar cycle interval |
| `hpalternate` | int (sec) | High-priority message alternation interval |
| `normoverridedur` | int (sec) | Normal override duration |
| `pulsemaxbri` | int 0–255 | Pulse effect maximum brightness |
| `pulseminbri` | int 0–255 | Pulse effect minimum brightness |
| `maxmsgage` | int (hours) | Max message history age (0 = disabled) |
| `localpressure` | int (hPa) | Push a local air pressure reading (800–1200 hPa). Updates the weather screen immediately if it is currently shown. |

#### Temperature Display

| Key | Value | Description |
|-----|-------|-------------|
| `tempactual` | `true` | Show actual temperature |
| `tempactual` | `false` | Show apparent (feels-like) temperature |
| `tempactual` | `"alternate"` | Alternate between actual and apparent |

#### Navigation

| Key | Value | Description |
|-----|-------|-------------|
| `clock` | `true` | Switch to clock screen |
| `weather` | `true` | Switch to weather screen |
| `calendar` | `true` | Switch to calendar screen |

#### History / Messaging

| Key | Value | Description |
|-----|-------|-------------|
| `clear` | `true` | Clear all messages and queue |
| `clearhistory` | `true` | Clear message history only |

#### Clock Reset

| Key | Value | Description |
|-----|-------|-------------|
| `clockresetcolors` | `"dark"` or `"light"` | Reset clock colors to default palette |
| `clockresetall` | `"dark"` or `"light"` | Reset all clock settings to defaults |

#### System

| Key | Value | Description |
|-----|-------|-------------|
| `reboot` | `true` | Reboot device (keeps settings) |
| `resetsettings` | `true` | Clear settings and reboot |
| `resetdevice` | `true` | Clear settings and reboot (alias) |

---

### System Management

#### `/reboot` — Reboot Device
```bash
curl http://YOUR_IP/reboot
```

#### `/resetsettings` — Reset All Settings
```bash
curl -X POST http://YOUR_IP/resetsettings
```

#### `/update` — OTA Firmware Update (Web)
Navigate to `http://YOUR_IP/update` in a browser to upload a new firmware `.bin` file.

#### `/otapassword` — Set OTA Password
```bash
curl -X POST http://YOUR_IP/otapassword \
  -H "Content-Type: application/json" \
  -d '{"password": "mypassword"}'
```

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
  "pulsemaxbri": 255,
  "pulseminbri": 64,
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
  "brightness": 80
}
```

---

## GET Format Reference

Most parameters available via JSON POST can also be sent as GET query parameters:

```
GET /message?text=Hello&bgcolor=blue&textcolor=white&font=24&expirationtime=30
GET /control?clockdark=true&pixelshift=toggle
GET /clockconfig?brightness=200&nightmodeenable=true
```

Named colors: `red`, `green`, `blue`, `white`, `black`, `yellow`, `cyan`, `magenta`, `orange`, `purple`, `pink`, `brown`, `gray`, `lightgray`, `darkgray`, `dark_green`

Hex colors: `#RRGGBB` (e.g. `#FF8800`)

---

## Examples

### Message Examples

**Simple centered message:**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Doorbell","bgcolor":"orange","textcolor":"black","font":36,"expirationtime":10}'
```

**Scrolling message:**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Breaking news ticker scrolling left","scrolldir":"left","scrollspeed":4}'
```

**Flashing alert:**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"ALERT","bgcolor":"red","textcolor":"white","flash":true,"flashinterval":500,"priority":"high"}'
```

**Pulsing notification:**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Package Arrived","pulse":true,"pulseduration":3000}'
```

**High-priority message (persists until dismissed):**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Smoke Alarm","bgcolor":"red","textcolor":"white","priority":"high"}'
```

### Message Profile Examples

**Save alert profile:**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"bgcolor":"red","textcolor":"white","font":36,"flash":true,"flashinterval":500,"saveprofile":1}'
```

**Use alert profile:**
```bash
curl -X POST http://YOUR_IP/message \
  -H "Content-Type: application/json" \
  -d '{"text":"Water Leak Detected","retrieveprofile":1,"priority":"high"}'
```

### Clock Examples

**Set time font and color:**
```bash
curl -X POST http://YOUR_IP/clockconfig \
  -H "Content-Type: application/json" \
  -d '{"timefont":48,"timecolor":"#00FFFF","datefont":16,"datecolor":"white"}'
```

**Enable night mode:**
```bash
curl -X POST http://YOUR_IP/clockconfig \
  -H "Content-Type: application/json" \
  -d '{"nightmodeenable":true,"nightbrightness":80,"daybrightness":160,"sunsetoffset":-30,"sunriseoffset":15}'
```

**Set 24-hour time format:**
```bash
curl -X POST http://YOUR_IP/clockconfig \
  -H "Content-Type: application/json" \
  -d '{"timeformat":"%H:%M","dateformat":"%A  %Y-%m-%d"}'
```

### Control Examples

**Toggle dark mode on clock:**
```bash
curl -X POST http://YOUR_IP/control \
  -H "Content-Type: application/json" \
  -d '{"clockdark":true}'
```

**Set all screens to dark mode:**
```bash
curl -X POST http://YOUR_IP/control \
  -H "Content-Type: application/json" \
  -d '{"clockdark":true,"weatherdark":true,"calendardark":true,"menudark":true,"historydark":true,"astrodark":true}'
```

**Navigate to weather:**
```bash
curl -X POST http://YOUR_IP/control \
  -H "Content-Type: application/json" \
  -d '{"weather":true}'
```

**Clear message history:**
```bash
curl -X POST http://YOUR_IP/control \
  -H "Content-Type: application/json" \
  -d '{"clearhistory":true}'
```

**Reset clock to dark mode defaults:**
```bash
curl -X POST http://YOUR_IP/control \
  -H "Content-Type: application/json" \
  -d '{"clockresetcolors":"dark"}'
```

### Schedules Examples

**Dim at 10pm (absolute):**
```bash
curl -X POST http://YOUR_IP/schedules \
  -H "Content-Type: application/json" \
  -d '{"index":0,"enabled":true,"mode":"absolute","hour":22,"minute":0,"action":"brightness","brightness":80}'
```

**Brighten at 7am (absolute):**
```bash
curl -X POST http://YOUR_IP/schedules \
  -H "Content-Type: application/json" \
  -d '{"index":1,"enabled":true,"mode":"absolute","hour":7,"minute":0,"action":"brightness","brightness":200}'
```

**Dim 30 minutes before sunset:**
```bash
curl -X POST http://YOUR_IP/schedules \
  -H "Content-Type: application/json" \
  -d '{"index":2,"enabled":true,"mode":"relative","reference":"sunset","offset":-30,"action":"brightness","brightness":50}'
```

---

## Font Guide

### Available Fonts

Font codes equal the point size. All fonts are Roboto Regular, covering codepoints 32–255 (full Latin-1).

| Code | Size | Typical Use |
|------|------|-------------|
| 6 | 6pt | Smallest — minimal labels |
| 8 | 8pt | Small labels ⭐ labelFont |
| 10 | 10pt | Compact text ⭐ msgNoticeFont, astroTextFont |
| 12 | 12pt | Small body text |
| 14 | 14pt | — |
| 16 | 16pt | General text, readable ⭐ suntimesFont, historyTextFont |
| 18 | 18pt | — |
| 20 | 20pt | Medium headlines ⭐ astroHeaderFont |
| 22 | 22pt | — |
| 24 | 24pt | Large headlines ⭐ weatherTempFont/weatherHumidFont |
| 26 | 26pt | — |
| 28 | 28pt | — |
| 30 | 30pt | **Default message font** |
| 32 | 32pt | — |
| 34 | 34pt | — |
| 36 | 36pt | Large messages |
| 38 | 38pt | — |
| 40 | 40pt | — |
| 42 | 42pt | — |
| 44 | 44pt | — |
| 46 | 46pt | — |
| 48 | 48pt | Maximum size ⭐ clockTimeFont |

**Recommendations:**
- **Messages:** 24pt or 30pt
- **Clock time:** 48pt
- **Clock date:** 16pt
- **Clock footer label:** 8pt
- **Unread message notice:** 10pt
- **Weather temp/humidity:** 24pt
- **Astro screen titles and arrows:** 20pt — `astroHeaderFont`
- **Astro screen data rows, footer, phase name:** 10pt — `astroTextFont`

### Font Files and Character Encoding

The Roboto font files live in the `Fonts/` subdirectory of the sketch and are included by `RobotoFonts.h`. The files are named `Roboto_Regular<size>pt8b.h` — the `8b` suffix indicates **Latin-1 (8-bit) coverage**, meaning every character from codepoint 32 (space) through 255 (ÿ) is present in the glyph table.

**Important:** Arduino GFX does **not** decode UTF-8. Its `print()` function passes each byte directly to the font table as a character index. This means:

- ✅ **`\xB0`** — correct way to print a degree symbol. This is a direct single-byte index to glyph 176 (U+00B0) in the Latin-1 table.
- ❌ **`\xC2\xB0`** — wrong. This is the UTF-8 multibyte encoding for U+00B0, but GFX renders it as two separate glyphs: `Â` (194) followed by `°` (176).

The same rule applies to any Latin-1 character above 127. Use the single-byte form directly in `snprintf` format strings:

```cpp
snprintf(buf, sizeof(buf), "%.0f\xB0", angle);       // "180°"
snprintf(buf, sizeof(buf), "%.1f\xB0""C", temp);     // "21.5°C"  (the "" separates \xB0 from C)
snprintf(buf, sizeof(buf), "%d\xB5s", microseconds); // "42µs"
```

The `""` between `\xB0` and a following letter is required when the next character is a valid hex digit (0–9, A–F) or a letter that would extend the escape sequence.

**Regenerating the fonts:** If you need to rebuild the font files (e.g. after obtaining a new TTF), use the included `ttf_to_gfxfont.py` script:

```
pip install freetype-py
python ttf_to_gfxfont.py Roboto-Regular.ttf 0 --sizes 6 8 10 12 14 16 18 20 22 24 26 28 30 32 34 36 38 40 42 44 46 48 --outdir Fonts/
```

This produces all 22 sizes covering codepoints 32–255. Do not use `--last 127` as that reproduces the old ASCII-only `7b` behaviour and loses the extended characters.

---

## Time and Date Formats

### Time Format Codes

| Code | Description | Example |
|------|-------------|---------|
| `%l` | Hour without leading zero (1-12) | 9 |
| `%I` | Hour with leading zero (01-12) | 09 |
| `%H` | Hour in 24-hour format (00-23) | 21 |
| `%M` | Minutes with leading zero (00-59) | 45 |
| `%S` | Seconds with leading zero (00-59) | 30 |
| `%P` | Lowercase am/pm | am |
| `%p` | Uppercase AM/PM | AM |

**Examples:**

| Format | Result |
|--------|--------|
| `%l:%M %P` | 9:45 am |
| `%I:%M %p` | 09:45 AM |
| `%H:%M` | 21:45 |
| `%l:%M:%S %P` | 9:45:30 am |

---

### Date Format Codes

**Day:**

| Code | Description | Example |
|------|-------------|---------|
| `%d` | Day with leading zero (01-31) | 03 |
| `%e` | Day without leading zero (1-31) | 3 |
| `%a` | Abbreviated weekday | Mon |
| `%A` | Full weekday | Monday |

**Month:**

| Code | Description | Example |
|------|-------------|---------|
| `%m` | Month number with leading zero (01-12) | 11 |
| `%b` | Abbreviated month | Nov |
| `%B` | Full month | November |

**Year:**

| Code | Description | Example |
|------|-------------|---------|
| `%Y` | 4-digit year | 2025 |
| `%y` | 2-digit year | 25 |

**Examples:**

| Format | Result |
|--------|--------|
| `%Y-%m-%d` | 2025-11-03 |
| `%m/%d/%Y` | 11/03/2025 |
| `%B %d, %Y` | November 03, 2025 |
| `%A, %B %e, %Y` | Monday, November 3, 2025 |

---

## Advanced Features

### Message Priority System

**Two priority levels:**
- **`normal`** - Standard messages (default)
- **`high`** - Priority messages that can't be overridden

**How it works:**

1. **Normal messages:** Display immediately if no message is active; added to queue if high priority message is showing
2. **High priority messages:** Cannot be overridden by normal messages (unless override enabled); multiple high priority messages alternate every 10 seconds; remain until manually cleared or expired
3. **Normal override mode** (configurable in User_Setup.h): When enabled, normal messages temporarily interrupt high priority; display for configured duration, then high priority returns

---

### Brightness Settings

The display uses three independent brightness levels:

| Setting | `User_Setup.h` constant | `/clockconfig` param | When it applies |
|---------|------------------------|---------------------|-----------------|
| **Screen Brightness** | `DEFAULT_SCREEN_BRIGHTNESS` | `brightness` | All screens when Night Mode is **not** active |
| **Night Brightness** | `DEFAULT_NIGHTMODE_BRIGHTNESS` | `nightbrightness` | Quiescent brightness during Night Mode |
| **Night Awake Brightness** | `DEFAULT_NIGHTMODE_AWAKE_BRIGHTNESS` | `daybrightness` | Brightness when display is touched during Night Mode |

**Night Mode period transitions:**
- At **sunset** (±offset): Screen Brightness → Night Brightness
- **Touch during Night Mode** (on any screen): Night Brightness → Night Awake Brightness for the configured wake duration, then back to Night Brightness
- At **sunrise** (±offset): Night Brightness → Screen Brightness

---

### Touch Controls

**Touch interface can be disabled via `/control?touchenable=false`**

**Clock screen:**
- First touch during night mode → brightens display for wake duration (does not open menu)
- Touch (day mode or second touch at night) → opens Menu screen

**Menu screen:**
- Tap a button → navigate to that screen

**Message screen (active message):**
- Tap anywhere → dismiss current message and return to clock

**Message History screen:**
- Swipe up → next message
- Swipe down → previous message
- Tap ▲/▼ arrows (right edge) → next/previous message
- Hold **«** button (lower-left, 500ms) → return to clock
- Hold **✕** button (lower-right, 500ms) → clear all history
- Long press anywhere open (2s, excluding control zones) → return to clock
- When queue is empty → any touch returns to clock immediately

**Weather screen:**
- Tap anywhere → return to clock

**Calendar screen:**
- Tap ◄ triangle (left header) → previous month
- Tap ► triangle (right header) → next month
- Tap Month/Year label (center header) → return to today's month
- Tap below header → return to Clock

**Astro Times screen:**

Both the Sun and Moon pages share the same five-zone touch layout:

| Zone | Location | Action |
|------|----------|--------|
| Left arrow `<` | Header row, left 90 px | Go back one day |
| Title ("Sun" or "Moon") | Header row, centre | Reset to today's date |
| Right arrow `>` | Header row, right 90 px | Go forward one day |
| Footer ("Tap here for Moon/Sun") | Bottom ~25 px | Switch between Sun and Moon pages |
| Data area | Everything else | Return to Clock |

**All screens — night mode:** When night mode is active and the display is dimmed, the first touch (anywhere) brightens the display to Night Awake Brightness without performing any navigation action. A second touch performs the normal action for that screen.

---

### Pixel Shift (Anti Burn-In)

When enabled, the display content is shifted by 1–2 pixels every few minutes. This prevents static elements (clock digits, labels) from burning into the panel over time.

**Enable/disable:**
```json
POST /control
{"pixelshift": true}
```

---

### Message History

Received messages are stored in a scrollable history (capacity configured in `User_Setup.h`). The history screen shows one message at a time with a progress bar indicating position.

- Navigate with swipe gestures or the ▲/▼ arrows (right edge)
- Hold **«** (lower-left) 500ms to return to clock
- Hold **✕** (lower-right) 500ms to clear all history
- The **«** and **✕** buttons are only shown when history contains messages
- Unread messages are marked; viewing a message marks it as read
- Messages older than `maxmsgage` hours are purged automatically (0 = disabled)
- History is also viewable via the `/status` endpoint

Set history age limit:
```bash
curl -X POST http://YOUR_IP/control \
  -H "Content-Type: application/json" \
  -d '{"maxmsgage":24}'
```

---

### Weather & Calendar

**Weather screen:**
- Shows temperature (actual and feels-like), humidity, air pressure (local and from service), weather description, and icon.
- Daily temperature highs and lows (actual and feels-like), amount of precipitation ("Wet"), and chance of precipitation.
- Navigate: `GET /weather` or `POST /control {"weather": true}`

**Calendar screen:**
- Shows the current month with today's date highlighted
- Navigate: `GET /calendar` or `POST /control {"calendar": true}`

**Interactive calendar navigation:**

| Touch zone | Action |
|------------|--------|
| ◄ triangle (left edge of header) | Go to previous month |
| ► triangle (right edge of header) | Go to next month |
| Month/Year label (center of header) | Return to today's month |
| Anywhere below the header | Return to Clock |

**Dark mode:**
```json
POST /control
{
  "weatherdark": true,
  "calendardark": true
}
```

**Temperature display mode** (control via `/control tempactual`):
- `true` — actual temperature
- `false` — apparent (feels-like) temperature
- `"alternate"` — cycles between actual and apparent

#### Local Air Pressure

When `HUBITAT_PRESSURE_URL` is defined in `secrets.h`, the device fetches averaged air pressure readings from a local Hubitat Maker API endpoint every 5 minutes. This supplements the Open-Meteo model pressure with readings from physical sensors on your LAN.

The endpoint must return JSON in the following format:
```json
{"hpa": 1013.5, "sensors": 2, "age_seconds": 30}
```

**Display behaviour:**
- When fresh local data is available (received within the last 30 minutes), it is shown on the weather screen as a plain number (e.g. `1013.5 hPa`).
- Open-Meteo pressure is always shown with a `~` prefix (e.g. `~1012.3 hPa`) to distinguish it as a model estimate.
- When no `HUBITAT_PRESSURE_URL` is configured, or when local data has gone stale, the weather screen falls back to Open-Meteo pressure only.
- By default (`apAlternate: true` in `User_Setup.h`), the display alternates between the local reading and the Open-Meteo estimate every 5 seconds. Set `DEFAULT_AP_ALTERNATION_MODE = false` to always show the local reading.

**Pushing a reading from Hubitat instead of polling:**

If you prefer Hubitat to push pressure data to the device rather than having the device poll, use `POST /control` with the `localpressure` key:

```bash
curl -X POST http://YOUR_IP/control \
  -H "Content-Type: application/json" \
  -d '{"localpressure": 1016}'
```

The value must be an integer in hPa between 800 and 1200. The device accepts the push, updates its local pressure state, and refreshes the weather screen immediately if it is currently displayed. Both polling and push can be used together — whichever updates the value most recently wins.

Auto-cycle behaviour (weather and calendar screens cycle automatically from the clock screen at configurable intervals) is controlled via `weathercycle`, `calendarcycle`, `wxshowdur`, `wxcycleint`, `calshowdur`, `calcycleint` in `/control`.

---

### Astro Times (Sun & Moon)

The Astro Times screen is accessed from the **Menu** (tap the clock face to open the 6-button menu, then tap **Astro Times**). It shows two sub-pages, toggled by tapping the footer row.

#### Sun Page

Displays all solar events for today in chronological order:

| Event | Description |
|-------|-------------|
| Astro. Dawn | Sun 18° below horizon — sky starts to lighten |
| Nautical Dawn | Sun 12° below horizon |
| Civil Dawn | Sun 6° below horizon — outdoor activities possible |
| Sunrise | Upper limb of sun touches the horizon |
| Solar Noon | Sun at maximum elevation for the day |
| Sunset | Upper limb of sun leaves the horizon |
| Civil Dusk | Sun 6° below horizon |
| Nautical Dusk | Sun 12° below horizon |
| Astro. Dusk | Sun 18° below horizon — astronomical dark |
| Day Length | Duration from sunrise to sunset |

All times are displayed in the device's configured clock time format (12-hour or 24-hour, matching the clock face).

#### Moon Page

| Field | Description |
|-------|-------------|
| Moonrise | Time the moon rises above the horizon |
| Moonset | Time the moon sets below the horizon |
| Illumination | Percentage of the visible disk that is lit |
| Phase Angle | 0° = New Moon, 180° = Full Moon |
| Phase graphic | Rendered disk showing current illumination |
| Phase name | New Moon, Waxing Crescent, First Quarter, Waxing Gibbous, Full Moon, Waning Gibbous, Last Quarter, or Waning Crescent |

If the moon does not rise or set on a given day, the field shows "No rise", "No set", or "Always up" as appropriate.

#### Clock and Weather Screens

The sunrise/sunset and moonrise/moonset times shown on the Clock and Weather screens use the same locally-computed astronomical data. The Clock screen shows a contextual two-line display:

- **Before sunrise:** Sunrise time and Moonrise time (line 1) · Sunset time and Moonset time (line 2)
- **After sunrise, before sunset:** Sunset time and Moonset time (line 1) · Tomorrow's sunrise and moonrise (line 2)
- **After sunset:** Tomorrow's sunrise and moonrise (line 1) · Tomorrow's sunset and moonset (line 2)

Moon rise/set lines are omitted if the moon has no rise or set event for that day.

#### Astronomical Calculation Engine

Calculations are performed locally on the ESP32 using the Jean Meeus *Astronomical Algorithms* method, ported from the WeekCalendar JavaScript application (SunMoonInfo.js / DayInfo.js). The source files are `AstroCalc.h` and `AstroCalc.cpp`.

**Accuracy:** Typically within 1–2 minutes of published tables for mid-latitude locations.

**Required configuration:** Set latitude, longitude, and POSIX TZ string via the
**📍 Location** page (`http://YOUR_IP/locationconfig`). These values are stored in
NVS and loaded at boot.

**Important:** Longitude must use standard cartographic convention (western longitudes are negative,
e.g. `-117.16` for San Diego). This is the same value used by the Open-Meteo weather API.

**DST handling:** The UTC offset is derived at runtime by comparing local and UTC wall clock values from the same Unix timestamp. This automatically reflects DST transitions without any additional configuration.

**Caching:** Today's and tomorrow's astronomical data are computed once per calendar day and cached. The cache is refreshed automatically at the first draw after midnight. When browsing other dates via the header arrows, data for the viewed date is computed on demand and cached separately from the today/tomorrow cache.

#### Dark Mode

The Astro Times screen has its own independent dark mode flag, defaulting to dark:

```json
POST /control
{"astrodark": true}
```

The dark/light colors are defined in `User_Setup.h` as `DEFAULT_ASTRO_*_COLOR` constants.

---

## Testing

The sketch includes support for three levels of testing: an automated HTTP test suite, a test injection endpoint for state-dependent behavior, and a manual visual checklist.

---

### HTTP Test Suite

**`test_multidisplay.py`** is a Python script that runs against a live device and exercises all HTTP endpoints systematically.

#### Requirements

- Python 3.10 or newer
- `requests` library: `pip install requests`

#### Running the Suite

```
python test_multidisplay.py --ip 192.168.1.x
```

Run a single group:
```
python test_multidisplay.py --ip 192.168.1.x --group messages
```

Available groups: `endpoints`, `messages`, `effects`, `priority`, `profiles`, `clockconfig`, `control`, `schedules`, `nightmode`, `edge`

---

### Test Inject Endpoint

> **Note:** `ENABLE_TEST_MODE` is set to `false` by default. The `/testinject` endpoint is compiled out and unavailable unless you set it to `true` in `User_Setup.h` for a development build.

The `/testinject` endpoint allows automated tests to inject artificial sun times and force an immediate night mode evaluation. Compiled in only when `ENABLE_TEST_MODE true` is set in `User_Setup.h`.

#### GET /testinject — Read Current State

```bash
curl http://YOUR_IP/testinject
```

**Response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `nightModeActive` | bool | Whether night mode is currently active |
| `nightModeEnabled` | bool | Whether night mode is enabled in settings |
| `nightModeWakeActive` | bool | Whether a touch-wake is currently in effect |
| `nightModeBrightness` | int | Configured night brightness (0–255) |
| `nightModeAwakeBrightness` | int | Configured brightness when touched during night mode |
| `screenBrightness` | int | Current screen brightness |
| `sunsetOffset` | int | Configured sunset offset in minutes |
| `sunriseOffset` | int | Configured sunrise offset in minutes |
| `sunriseHour` | int | Current sunrise hour |
| `sunriseMinute` | int | Current sunrise minute |
| `sunsetHour` | int | Current sunset hour |
| `sunsetMinute` | int | Current sunset minute |
| `sunTimesValid` | bool | Whether sun times are considered valid |
| `currentScreen` | int | Active screen: 0=Clock, 1=Menu, 2=Message, 3=Messages, 4=Weather, 5=Calendar, 6=Astro, 7=About |

#### POST /testinject — Inject Test Values

| Field | Type | Description |
|-------|------|-------------|
| `sunsethour` | int | Override `sunsetHour` |
| `sunsetminute` | int | Override `sunsetMinute` |
| `sunrisehour` | int | Override `sunriseHour` |
| `sunriseminute` | int | Override `sunriseMinute` |
| `sunrisehourtomorrow` | int | Override `tomorrowSunriseHour` |
| `sunriseminutetomorrow` | int | Override `tomorrowSunriseMinute` |
| `setsuntimesvalid` | bool | Set `sunTimesValid` |
| `forcenightcheck` | bool | Reset debounce timer and call `updateNightMode()` immediately |

#### Example: Force Night Mode Active

```bash
curl -X POST http://YOUR_IP/testinject \
  -H "Content-Type: application/json" \
  -d '{
    "sunsethour": 6,
    "sunsetminute": 0,
    "sunrisehour": 2,
    "sunriseminute": 0,
    "setsuntimesvalid": true,
    "forcenightcheck": true
  }'
```

---

### Visual Verification Checklist

**`visual_checklist.md`** is a structured checklist for manual display verification covering all screens, message effects, night mode transitions, touch controls, auto-cycle behavior, screensaver, schedules, and persistence across reboots.

---

## Troubleshooting

### Text Issues

**Text not wrapping?** Enable `wrap: true` in message JSON.

**Text cut off?** Try smaller font, enable wrapping, or reduce text length.

**Colors not showing?** Use named colors ("red", "blue") or hex ("#FF0000"). Check brightness isn't 0.

---

### Display Issues

**Flash not working?** Check `flashinterval` (recommended: 500-5000ms). Ensure `flash: true`.

**Scroll too fast/slow?** Adjust `scrollspeed` (1-9).

**Display too dim?** Increase brightness to 255. Check night mode and screensaver aren't active.

**Weather screen shows `~` prefix on pressure reading?** The `~` prefix indicates the value is from the Open-Meteo model estimate. A reading without the prefix is from your local sensors via `HUBITAT_PRESSURE_URL`. If you expect a local reading but see `~` only, verify `HUBITAT_PRESSURE_URL` is defined in `secrets.h`, that the Hubitat endpoint is reachable from the device, and that it returns `{"hpa": ...}` JSON. Check the `/logs` page for fetch error messages.

---

### Clock Issues

**Clock not showing?** Enable via `/clockconfig?clockenable=true` or `POST /control {"clockenable": true}`.

**Clock colors wrong?** Reset: `POST /control {"clockresetcolors": "dark"}`.

**Time wrong?** Check WiFi (NTP sync requires WiFi). Verify the **POSIX TZ String** on the Location config page (`http://YOUR_IP/locationconfig`). Check `/time` endpoint.

---

### Astro Times Issues

**Times off by an hour?** Verify the **POSIX TZ String** on the Location config page matches your timezone and DST rules. Example for US Pacific: `PST8PDT,M3.2.0,M11.1.0`.

**Solar noon or sunrise/sunset grossly wrong (many hours off)?** Verify that the **Longitude** on the Location config page uses standard cartographic convention — western longitudes should be **negative** (e.g., `-117.16` for San Diego). A positive longitude for a western location will produce incorrect results.

**Moon phase graphic appears all dark or all bright?** This is expected behavior for New Moon (0%) or Full Moon (100%). The graphic renders correctly for all intermediate phases.

**"Calculating..." shown on Astro screen?** The astronomical data has not been computed yet. This normally resolves within a second of first opening the Astro screen. If it persists, check that NTP time is synced (the calculation requires a valid date).

---

### Network Issues

**mDNS not working?** Use IP address instead of `.local` name. Check device IP on serial monitor during boot.

**Can't connect to WiFi?** Verify SSID and password in `secrets.h`. Check WiFi signal. Ensure 2.4GHz WiFi.

**Weather or astro data wrong location?** Open `http://YOUR_IP/locationconfig` and verify your latitude, longitude, and timezone are correctly set.

**HTTP requests failing?** Check device is on same network. Verify correct IP. Try `http://YOUR_IP/time` first.

---

### OTA Update Issues

**Update fails?** Ensure stable WiFi connection. Use correct .bin file. Verify OTA password if set.

---

### Compilation Issues

**Board not found?** Ensure ESP32 board package installed in Arduino IDE. Select "ESP32S3 Dev Module".

**Partition errors?** Verify `partitions.csv` is in sketch folder. Select custom partition scheme in Tools menu.

**Memory errors?** Reduce `LOG_BUFFER_SIZE` in `User_Setup.h`. Disable unused features.

**Library conflicts?** If you see errors like `expected type-specifier before 'Arduino_ESP32QSPI'`, add an explicit include: `#include "databus/Arduino_ESP32QSPI.h"` after the `Arduino_GFX_Library.h` include. This can occur when multiple versions of the GFX library are installed.

---

## Changelog

### v2.0.8 — 2026-05-23

**Bug fixes and hardening (static audit pass):**

- **`ENABLE_ARDUINO_OTA` flag now fully controls Arduino OTA.** Previously the flag only guarded the OTA password refresh; `ArduinoOTA.begin()`, its callbacks, and `ArduinoOTA.handle()` ran unconditionally. All Arduino OTA setup and polling are now wrapped in `#if ENABLE_ARDUINO_OTA`. Web OTA (`/update`) is unaffected and always available.
- **`ENABLE_TEST_MODE` defaults to `false`.** The `/testinject` endpoint can alter runtime state without authentication. It is now compiled out by default; set `ENABLE_TEST_MODE true` in `User_Setup.h` only for development builds.
- **Scheduled screen changes now go through `changeScreen()`.** Previously the scheduler set `currentScreen` directly and called draw functions, bypassing the screensaver reset, brightness handling, rotation, and auto-cycle timer logic in `changeScreen()`. Scheduled clock/weather/calendar actions now behave identically to manual navigation.
- **Scheduler POST input validation.** Hour, minute, and brightness are now clamped to valid ranges; action, mode, and reference strings are whitelist-checked. Malformed direct API POSTs can no longer store out-of-range values.
- **Removed unused `showWeatherScreen()` / `showCalendarScreen()` / `showClockScreen()` wrappers** from `TouchInput.ino`. These were one-line `changeScreen()` wrappers with no call sites.
- **`resetSettings()` in `Settings.ino` documented as unused** but retained as the canonical consolidation point for future reset logic.
- **`getWeatherIcon()` and `getWeatherIconColor()` in `WeatherCalendar.ino` marked as superseded** — the bitmap icon path (`getWeatherIconArrays()`) replaced them; retained for reference.
- **PSRAM helper utilities** (`isLowMemory`, `PSRAMStringBuffer`, `WeatherIconCache`, etc.) marked as intentionally retained for future use.
- **Comment and documentation fixes:** wrong hardware name in README and `partitions.csv` corrected; stale OTA handler location comment in `HTML.cpp` updated; `User_Setup.h` typos (`"Non volitile"`, `"calender"`) fixed.

---

## License

**Dedicated to the public domain in 2025.**

Unless required by applicable law or agreed to in writing, this software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.

---

**Author:** John Land
**Last Updated:** 2026-05-23
**Version:** 2.0.8
