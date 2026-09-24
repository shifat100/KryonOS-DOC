# KryonOS Documentation

**Version:** 1.0 (based on provided source)  
**Platform:** ESP32 (Arduino framework)  
**Display:** 240x320 TFT (TFT_eSPI)  
**Storage:** LittleFS (internal) + SD Card (SPI)  
**Runtime:** Duktape JavaScript engine  

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [File System Layout](#file-system-layout)
4. [Building & Flashing](#building--flashing)
5. [User Interface](#user-interface)
   - [Launcher](#launcher)
   - [Settings](#settings)
   - [App Installer](#app-installer)
   - [App Store](#app-store)
   - [Help Center](#help-center)
   - [Web Server Manager](#web-server-manager)
6. [Creating Apps](#creating-apps)
   - [App Structure](#app-structure)
   - [app.json Format](#appjson-format)
   - [main.js Basics](#mainjs-basics)
7. [JavaScript API Reference](#javascript-api-reference)
   - [System Object](#system-object)
   - [FS Object](#fs-object)
   - [Global Constants](#global-constants)
8. [Example Apps](#example-apps)
9. [Web File Manager](#web-file-manager)
10. [Kernel & Runtime](#kernel--runtime)
11. [Settings Details](#settings-details)
12. [Troubleshooting](#troubleshooting)
13. [Appendix](#appendix)

---

## Overview

KryonOS is a lightweight, touch‑friendly operating system for ESP32 microcontrollers. It provides:

- A graphical launcher with system apps and user‑installed JavaScript apps.
- A JavaScript runtime (Duktape) for running apps from SD card or internal flash.
- An App Store (online) and App Installer (offline) for distributing and installing apps.
- WiFi connectivity, web‑based file manager, and NTP time sync.
- Settings for WiFi, touch calibration, app management, time/region, and system updates.

---

## Architecture

### Core Components

| Component | Description |
|-----------|-------------|
| `main.cpp` | State machine, touch handling, main loop |
| `LauncherUI` | Home screen, app scanning, launching apps |
| `SettingsUI` | WiFi, calibration, app management, time, about, updates |
| `InstallerUI` | SD/Internal file browser, app installation |
| `AppStoreUI` | Online app store browser |
| `HelpCenterUI` | Offline & online help articles |
| `WebServerAppUI` | Web server status & toggle |
| `HarixKernel` | Duktape heap management, JS execution |
| `JSBindings` | C++ functions exposed to JavaScript |
| `FileSystem` | Unified API for LittleFS & SD |
| `WebManager` | WiFi connection, Async web server, file manager API |
| `TimeManager` | NTP sync, timezone, manual time |
| `TouchCalibrator` | Touch screen calibration routine |
| `MyKeyboard` | On‑screen QWERTY keyboard |

### State Machine

The system uses a global `int currentState` to switch between screens. States are defined in `main.cpp`:

```cpp
#define STATE_LAUNCHER          0
#define STATE_SETTINGS          1
#define STATE_RUN_APP           2
#define STATE_INSTALLER         3
#define STATE_CALIBRATOR        4
#define STATE_WEB_APP           5
#define STATE_SETTINGS_WIFI     6
#define STATE_SETTINGS_ABOUT    7
#define STATE_SETTINGS_APPS     8
#define STATE_SETTINGS_TIME     9
#define STATE_SETTINGS_TIME_MANUAL 10
#define STATE_UPDATER_BOOT      11
#define STATE_UPDATER_MANUAL    12
#define STATE_APP_STORE         13
#define STATE_HELP_CENTER       14
```

Transitions are triggered by touch events or internal logic (e.g., after app exit).

---

## File System Layout

Two file systems are supported:

- **LittleFS** – internal flash, mounted as `/local/`
- **SD Card** – external SPI, mounted as `/sd/`

Paths in the API always start with `/local/` or `/sd/`.

### Standard Directories

| Path | Purpose |
|------|---------|
| `/local/apps/` | Installed apps (LittleFS) |
| `/sd/apps/` | Installed apps (SD) |
| `/local/nowifi.txt` | If present, WiFi is disabled at boot |
| `/local/web_on.txt` | If present, web server starts at boot |
| `/local/config_install_sd.txt` | If present, default install location is SD |
| `/sd/wifi.txt` or `/local/wifi.txt` | WiFi credentials (`SSID\nPassword`) |
| `/local/config_time.txt` | Time settings (`TZ|24H|NTP`) |
| `/local/touch_cal_p.bin` | Touch calibration data |
| `/tmp_download/` | Temporary downloads (App Store, Help Center) |

### App Folder Structure

```
/…/apps/MyApp/
├── app.json      # App metadata
└── main.js       # JavaScript code
```

Legacy single‑file apps (`*.js`) are also supported.

---

## Building & Flashing

### Prerequisites

- Arduino IDE or PlatformIO with ESP32 board support.
- Required libraries:
  - `TFT_eSPI`
  - `ArduinoJson`
  - `ESPAsyncWebServer` (for web manager)
  - `LittleFS`, `SD`, `SPI`, `WiFi`, `HTTPClient`
- **Duktape 2.7.0** source files: `duktape.c`, `duktape.h`, `duk_config.h`  
  *`duktape.c` is mandatory; without it the JS runtime will not link.*

### PlatformIO (`platformio.ini`)

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
lib_deps =
    bodmer/TFT_eSPI@^2.5.0
    bblanchon/ArduinoJson@^6.21.0
    me-no-dev/ESPAsyncWebServer@^1.2.3
build_flags =
    -D KRYONOS_VERSION=\"1.0.0\"
    -D KRYONOS_API_LEVEL=1
build_src_filter = +<*> +<Runtime/duktape.c>
```

### Arduino IDE

- Place all source files in the sketch folder.
- Ensure `duktape.c` is in the same folder or a subfolder included in the build.

### Configuration Macros

Define in `platformio.ini` or a global header:

```cpp
#define KRYONOS_VERSION "1.0.0"
#define KRYONOS_API_LEVEL 1
```

---

## User Interface

### Launcher

- Shows `[ SYSTEM ]` header, then system apps: **App Store**, **App Installer**, **Settings**, **Help Center**.
- Then `[ APPS ]` header, followed by user‑installed apps.
- Touch an item to launch.
- Footer: `UP | SEL | DN` for navigation.
- Scans `/local/apps/` and `/sd/apps/` for folders containing `app.json`.

### Settings

Main menu with buttons:

1. **WiFi Options** – enable/disable WiFi, forget network, start web server.
2. **Touch Calibrator** – runs `calibrateTouch()`.
3. **Manage Apps** – list installed apps, uninstall, move between SD/LFS, toggle default install location.
4. **Time & Region** – NTP on/off, timezone selection, 12/24h format, manual time.
5. **About Device** – storage info, free heap, version, reset app data (format LittleFS).
6. **System Updates** – check for OTA updates from GitHub.

### App Installer

- File browser for `/local/` and `/sd/`.
- Detects app packages (folders with `app.json`) and shows `[Type] Name`.
- Tap an app package → shows info dialog (version, author, description/changelog).
- **Install** copies the folder to `/local/apps/` or `/sd/apps/` (depending on default install location).
- Checks for API compatibility, syntax errors in `main.js`, and author conflicts.
- If app already exists, prompts to overwrite.
- Legacy `.js` files can be run directly or installed as single‑file apps.

### App Store

- Requires WiFi.
- Fetches `index.json` from GitHub:  
  `https://raw.githubusercontent.com/Haris16-code/KryonOS-AppStore/main/index.json`
- `index.json` format:
  ```json
  {
    "categories": {
      "Category Name": "https://…/category.json",
      …
    }
  }
  ```
- Each category file contains:
  ```json
  {
    "apps": {
      "appid": {
        "meta": "https://…/app.json",
        "app": "https://…/main.js"
      }
    }
  }
  ```
- Displays categories, then apps. Tap app → downloads meta, shows details, then **DOWNLOAD** installs.
- Also includes **Check For Apps Update** category that compares installed app versions with remote versions.

### Help Center

- **Offline Help** – built‑in categories and articles.
- **Online Help** – fetches `help/index.json` from GitHub:
  ```json
  {
    "categories": [
      { "name": "Category", "url": "https://…/articles.json" }
    ]
  }
  ```
- Article file format:
  ```json
  {
    "articles": [
      { "title": "Title", "url": "https://…/article.json" }
    ]
  }
  ```
- Article content file:
  ```json
  { "content": "Article text…" }
  ```

### Web Server Manager

- Shows WiFi status, IP address, and whether the web server is running.
- Button to toggle web server on/off (requires reboot).
- Web server serves a file manager at `http://<IP>/`.

---

## Creating Apps

### App Structure

```
MyApp/
├── app.json
└── main.js
```

### `app.json` Format

```json
{
  "name": "My App",
  "packageName": "com.example.myapp",
  "version": "1.0.0",
  "api": 1,
  "author": "Your Name",
  "type": "App",
  "category": "Utility",
  "description": "What this app does.",
  "changelog": "Initial release"
}
```

- `packageName`: lowercase, dot‑separated, no spaces (e.g., `com.example.myapp`).
- `api`: minimum required API level (≤ `KRYONOS_API_LEVEL`).
- `type`: shown in installer (e.g., `App`, `Game`).
- `category`: arbitrary string.

### `main.js` Basics

- Runs in a Duktape ECMAScript environment.
- Has access to global `System` and `FS` objects.
- No `setInterval`/`setTimeout` – use `while` loops with `System.delay()`.
- To exit, the OS automatically handles the top‑right `X` button (touch area x ≥ 200, y ≤ 40). Alternatively, break the loop on a touch event.

**Minimal example:**

```javascript
System.fillScreen(BLACK);
System.setTextColor(WHITE);
System.drawString("Hello KryonOS!", 20, 60, 4);

while (true) {
  var t = System.getTouch();
  if (t.touched) break;
  System.delay(50);
}
```

---

## JavaScript API Reference

### System Object

#### GPIO

| Method | Description |
|--------|-------------|
| `System.gpio.pinMode(pin, mode)` | Set pin mode |
| `System.gpio.digitalWrite(pin, value)` | Write digital value |
| `System.gpio.digitalRead(pin)` | Read digital value → int |
| `System.gpio.analogRead(pin)` | Read analog value → int |
| `System.gpio.analogWrite(pin, value)` | Write PWM value |
| `System.gpio.pulseIn(pin, state, timeout)` | Measure pulse duration → uint |

**Constants:** `System.gpio.OUTPUT`, `INPUT`, `INPUT_PULLUP`, `HIGH`, `LOW`.

#### Display – Drawing

| Method | Parameters |
|--------|------------|
| `System.fillScreen(color)` | Fill entire screen |
| `System.fillRect(x, y, w, h, color)` | Filled rectangle |
| `System.drawRect(x, y, w, h, color)` | Rectangle outline |
| `System.drawLine(x0, y0, x1, y1, color)` | Line |
| `System.drawPixel(x, y, color)` | Single pixel |
| `System.drawCircle(x, y, r, color)` | Circle outline |
| `System.fillCircle(x, y, r, color)` | Filled circle |
| `System.drawTriangle(x0,y0,x1,y1,x2,y2,color)` | Triangle outline |
| `System.fillTriangle(x0,y0,x1,y1,x2,y2,color)` | Filled triangle |
| `System.drawRoundRect(x, y, w, h, r, color)` | Rounded rectangle outline |
| `System.fillRoundRect(x, y, w, h, r, color)` | Filled rounded rectangle |
| `System.drawFastVLine(x, y, h, color)` | Vertical line |
| `System.drawFastHLine(x, y, w, color)` | Horizontal line |
| `System.drawBMP(path, x, y)` | Draw BMP image from file (24/16/32‑bit) |

#### Display – Text

| Method | Description |
|--------|-------------|
| `System.drawString(text, x, y, font)` | Draw text (font 1,2,4,6,7,8) |
| `System.setTextColor(fg[, bg])` | Set text foreground & optional background |
| `System.setTextSize(size)` | Multiply text size |

#### Display – Utility

| Method | Description |
|--------|-------------|
| `System.color(r, g, b)` | Convert RGB888 → RGB565 |
| `System.screenWidth()` | Returns display width (240) |
| `System.screenHeight()` | Returns display height (320) |

#### Touch Input

| Method | Returns |
|--------|---------|
| `System.getTouch()` | `{ x, y, touched }` |

> Touch in the top‑right corner (x ≥ 200, y ≤ 40) triggers a hidden OS exit.

#### System Utilities

| Method | Description |
|--------|-------------|
| `System.millis()` | Milliseconds since boot |
| `System.micros()` | Microseconds since boot |
| `System.delay(ms)` | Delay (also runs GC) |
| `System.delayMicroseconds(us)` | Microsecond delay |
| `System.print(msg)` | Print to Serial |
| `System.getTemperature()` | ESP32 internal temperature (°C) |
| `System.hasTemperatureSensor()` | Boolean |
| `System.getInfo()` | Object: `totalRAM`, `freeRAM`, `minFreeRAM`, `maxAllocRAM`, `cpuFreqMHz`, `chipModel`, `chipCores`, `chipRevision`, `flashSize`, `uptimeMs` |
| `System.restart()` | Reboot ESP32 |
| `System.getTime()` | Formatted time string |
| `System.getSeconds()` | Seconds (0‑59) |
| `System.getDate()` | Formatted date string |
| `System.getYear()` | Year |
| `System.getMonth()` | Month (1‑12) |
| `System.getDay()` | Day of month |
| `System.getTimezone()` | Timezone string (e.g., `UTC-5`) |
| `System.getOSVersion()` | KryonOS version string |
| `System.getAPILevel()` | API level integer |
| `System.getIPAddress()` | IP address string |
| `System.isWiFiActive()` | Boolean |
| `System.prompt(msg, initial)` | On‑screen keyboard input → string |

#### Double Buffering (Sprite)

| Method | Description |
|--------|-------------|
| `System.createSprite(w, h)` | Create sprite (returns bool) |
| `System.deleteSprite()` | Delete sprite |
| `System.pushSprite(x, y)` | Push sprite to screen |
| `System.bindSprite(enable)` | Redirect drawing to sprite |

### FS Object

| Method | Description |
|--------|-------------|
| `FS.readTextFile(path)` | Read file → string (null if not exists) |
| `FS.writeTextFile(path, content)` | Write file → bool |
| `FS.appendTextFile(path, content)` | Append → bool |
| `FS.deleteFile(path)` | Delete → bool |
| `FS.renameFile(old, new)` | Rename → bool |
| `FS.exists(path)` | Check existence → bool |
| `FS.listDir(path)` | List directory → array of strings |
| `FS.mkdir(path)` | Create directory → bool |
| `FS.rmdir(path)` | Remove directory → bool |
| `FS.isDirectory(path)` | → bool |
| `FS.isFile(path)` | → bool |
| `FS.getFileSize(path)` | → size in bytes |
| `FS.getTotalSpace(drive)` | `"/sd"` or `"/local"` → total bytes |
| `FS.getUsedSpace(drive)` | → used bytes |
| `FS.getFreeSpace(drive)` | → free bytes |
| `FS.getFileMD5(path)` | → MD5 hex string |
| `FS.mountSD()` | Mount SD → bool |
| `FS.unmountSD()` | Unmount SD |

### Global Constants

Available colors: `BLACK`, `WHITE`, `RED`, `GREEN`, `BLUE`, `YELLOW`, `CYAN`, `MAGENTA`, `ORANGE`, `DARKGREY`.

---

## Example Apps

### 1. Hello Kryon

**app.json**
```json
{
  "name": "Hello Kryon",
  "packageName": "com.demo.hello",
  "version": "1.0.0",
  "api": 1,
  "author": "Kryon",
  "type": "App",
  "category": "Demo",
  "description": "Simple hello world app."
}
```

**main.js**
```javascript
System.fillScreen(BLACK);
System.setTextColor(GREEN);
System.drawString("Hello KryonOS!", 20, 60, 4);
System.setTextColor(WHITE);
System.drawString("OS: " + System.getOSVersion(), 20, 120, 2);
System.drawString("API: " + System.getAPILevel(), 20, 140, 2);
System.drawString("Free RAM: " + Math.floor(System.getInfo().freeRAM / 1024) + " KB", 20, 160, 2);
System.drawString("Tap to exit", 20, 200, 2);

while (true) {
  var t = System.getTouch();
  if (t.touched) break;
  System.delay(50);
}
```

### 2. LED Blink (GPIO 2)

**main.js**
```javascript
var LED_PIN = 2;
System.gpio.pinMode(LED_PIN, System.gpio.OUTPUT);
var state = 0;

function draw() {
  System.fillScreen(BLACK);
  System.setTextColor(YELLOW);
  System.drawString("GPIO LED Blink", 20, 20, 4);
  System.setTextColor(WHITE);
  System.drawString("Pin: " + LED_PIN, 20, 70, 2);
  System.drawString(state ? "State: ON" : "State: OFF", 20, 100, 2);
  System.drawString("Tap to toggle", 20, 140, 2);
}

draw();

while (true) {
  var t = System.getTouch();
  if (t.touched) {
    state = !state;
    System.gpio.digitalWrite(LED_PIN, state ? System.gpio.HIGH : System.gpio.LOW);
    draw();
    System.delay(200);
  }
  System.delay(20);
}
```

### 3. Digital Clock

**main.js**
```javascript
while (true) {
  System.fillScreen(BLACK);
  System.setTextColor(CYAN);
  System.drawString(System.getTime(), 20, 80, 4);
  System.setTextColor(WHITE);
  System.drawString(System.getDate(), 20, 140, 2);
  System.drawString("TZ: " + System.getTimezone(), 20, 170, 2);
  System.delay(1000);

  var t = System.getTouch();
  if (t.touched) break;
}
```

### 4. Temperature Monitor

**main.js**
```javascript
if (!System.hasTemperatureSensor()) {
  System.fillScreen(BLACK);
  System.setTextColor(RED);
  System.drawString("No temp sensor found!", 20, 100, 2);
} else {
  while (true) {
    var temp = System.getTemperature();
    System.fillScreen(BLACK);
    System.setTextColor(ORANGE);
    System.drawString("ESP32 Temperature", 10, 20, 2);
    System.setTextColor(WHITE);
    System.drawString(temp.toFixed(1) + " C", 20, 80, 4);
    System.delay(500);

    var t = System.getTouch();
    if (t.touched) break;
  }
}
```

### 5. File Browser

**main.js**
```javascript
var currentPath = "/local";
var files = [];

function loadDir() {
  files = FS.listDir(currentPath);
}

function draw() {
  System.fillScreen(BLACK);
  System.setTextColor(GREEN);
  System.drawString("Files: " + currentPath, 10, 10, 2);
  System.setTextColor(WHITE);
  var y = 40;
  for (var i = 0; i < files.length && i < 10; i++) {
    var display = files[i];
    if (FS.isDirectory(files[i])) display = "[D] " + files[i];
    System.drawString(display, 10, y, 2);
    y += 20;
  }
  System.drawString("Tap to exit", 10, 280, 2);
}

loadDir();
draw();

while (true) {
  var t = System.getTouch();
  if (t.touched) break;
  System.delay(50);
}
```

### 6. Simple Notepad

**main.js**
```javascript
var filePath = "/local/note.txt";

function draw() {
  System.fillScreen(BLACK);
  System.setTextColor(YELLOW);
  System.drawString("Simple Notepad", 20, 20, 2);
  System.setTextColor(WHITE);
  var content = FS.readTextFile(filePath);
  if (!content) content = "(empty)";
  var lines = content.split("\n");
  var y = 50;
  for (var i = 0; i < lines.length && i < 8; i++) {
    System.drawString(lines[i], 10, y, 2);
    y += 18;
  }
  System.drawString("Tap to edit", 10, 280, 2);
}

draw();

while (true) {
  var t = System.getTouch();
  if (t.touched) {
    var text = System.prompt("Enter text", FS.readTextFile(filePath));
    FS.writeTextFile(filePath, text);
    draw();
    System.delay(200);
  }
  System.delay(50);
}
```

### 7. Drawing Pad

**main.js**
```javascript
System.fillScreen(BLACK);
System.setTextColor(GREEN);
System.drawString("Draw! Top-right X to exit", 10, 5, 2);

var lastX = -1;
var lastY = -1;
var color = GREEN;

while (true) {
  var t = System.getTouch();
  if (t.touched) {
    if (lastX >= 0) {
      System.drawLine(lastX, lastY, t.x, t.y, color);
    }
    lastX = t.x;
    lastY = t.y;
  } else {
    lastX = -1;
    lastY = -1;
  }
  System.delay(10);
}
```

### 8. System Info Dashboard

**main.js**
```javascript
var info = System.getInfo();

System.fillScreen(BLACK);
System.setTextColor(GREEN);
System.drawString("System Info", 10, 10, 2);

System.setTextColor(WHITE);
var y = 40;
System.drawString("Chip: " + info.chipModel, 10, y, 2); y += 18;
System.drawString("Cores: " + info.chipCores, 10, y, 2); y += 18;
System.drawString("CPU: " + info.cpuFreqMHz + " MHz", 10, y, 2); y += 18;
System.drawString("Flash: " + Math.floor(info.flashSize / 1024 / 1024) + " MB", 10, y, 2); y += 18;
System.drawString("Free RAM: " + Math.floor(info.freeRAM / 1024) + " KB", 10, y, 2); y += 18;
System.drawString("Max Alloc: " + Math.floor(info.maxAllocRAM / 1024) + " KB", 10, y, 2); y += 18;
System.drawString("Uptime: " + Math.floor(info.uptimeMs / 1000) + " s", 10, y, 2); y += 18;
System.drawString("WiFi: " + (System.isWiFiActive() ? System.getIPAddress() : "OFF"), 10, y, 2); y += 18;
System.drawString("SD Free: " + Math.floor(FS.getFreeSpace("/sd") / 1024) + " KB", 10, y, 2); y += 18;
System.drawString("LFS Free: " + Math.floor(FS.getFreeSpace("/local") / 1024) + " KB", 10, y, 2); y += 18;

while (true) {
  var t = System.getTouch();
  if (t.touched) break;
  System.delay(100);
}
```

---

## Web File Manager

When the web server is enabled, navigating to `http://<ESP_IP>/` serves a file manager UI.

### API Endpoints

| Method | Endpoint | Parameters | Description |
|--------|----------|------------|-------------|
| GET | `/api/list` | `dir=/littlefs/path` or `/sd/path` | List directory → JSON array `[{name, type, size}]` |
| GET | `/api/edit` | `path=…` | Get file content as text |
| POST | `/api/edit` | `path=…`, `content=…` | Write file content |
| GET | `/api/download` | `path=…` | Download file |
| DELETE | `/api/delete` | `path=…` | Delete file/folder |
| POST | `/api/create` | `path=…`, `type=folder|file` | Create new folder/file |
| POST | `/api/rename` | `oldPath=…`, `newPath=…` | Rename file/folder |
| POST | `/api/upload` | multipart form with `data` field | Upload file(s) |

All paths must start with `/littlefs` or `/sd` (mapped to LittleFS and SD respectively).

---

## Kernel & Runtime

- **Duktape 2.7.0** – lightweight JavaScript engine.
- Heap is created on demand in `HarixKernel::runFile()` and destroyed after app exit.
- Memory allocation functions (`my_alloc`, `my_realloc`, `my_free`) handle out‑of‑memory gracefully.
- Syntax checking runs in a separate FreeRTOS task (`syntaxCheckTask`) with 16 KB stack.
- `JSBindings::init()` registers all C functions into the global scope.
- Errors are caught and displayed with a red screen; `OS_EXIT` error is used for the hidden exit button.

### Out‑of‑Memory Handling

When allocation fails, a red screen appears with instructions to disable WiFi to free RAM. A touch on the top‑right `X` reboots the device.

---

## Settings Details

### WiFi Options

- **WiFi: ON/OFF** – toggles presence of `/local/nowifi.txt`. Reboot required.
- **Forget Network** – deletes `wifi.txt` from SD/LittleFS.
- **Start Web Server** – sets `/local/web_on.txt` and reboots.

### Touch Calibrator

Runs `tft.calibrateTouch()`, saves 5‑point calibration to `/local/touch_cal_p.bin`.

### Manage Apps

- Lists apps from `/local/apps/` and `/sd/apps/`.
- Toggle default install location (SD vs LittleFS) via `/local/config_install_sd.txt`.
- Uninstall (deletes folder/files).
- Move app between storages (copies then deletes source).

### Time & Region

- **NTP Sync** – enable/disable SNTP.
- **Region** – select from 26 timezones.
- **Format** – 12h/24h.
- **Set Manual Time** – only when NTP is off.

### About Device

- Shows LittleFS & SD total/used/free.
- Free heap.
- Version.
- **Reset App Data** – formats LittleFS (erases all apps and settings).

### System Updates

- Checks GitHub for `update.json`.
- Displays version, changelog, and installation guide.
- Silent check on boot if WiFi connected.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| **JS app crashes with "Out Of Ram"** | Disable WiFi from Settings → WiFi Options to free ~60 KB RAM. |
| **App Store fails to load** | Ensure WiFi is connected and the GitHub raw URL is accessible. |
| **SD card not detected** | Check SPI wiring: SCK=14, MISO=26, MOSI=13, CS=15. Format as FAT32. |
| **Touch not responding** | Run Touch Calibrator from Settings. |
| **App won't install** | Verify `app.json` is valid, `packageName` is lowercase dot‑separated, `api` ≤ OS API level. |
| **Syntax error on app install** | Check `main.js` for JavaScript syntax errors; the installer shows the error. |
| **Web server not starting** | Ensure WiFi is ON, web server is ON (`/local/web_on.txt` exists). |
| **`duktape.c` missing** | Add Duktape 2.7.0 source to the project; it is required for linking. |

---

## Appendix

### Constants

| Name | Value | Description |
|------|-------|-------------|
| `KRYONOS_VERSION` | `"1.0.0"` | OS version string |
| `KRYONOS_API_LEVEL` | `1` | API level for app compatibility |

### State Constants (main.cpp)

| State | Value |
|-------|-------|
| `STATE_LAUNCHER` | 0 |
| `STATE_SETTINGS` | 1 |
| `STATE_RUN_APP` | 2 |
| `STATE_INSTALLER` | 3 |
| `STATE_CALIBRATOR` | 4 |
| `STATE_WEB_APP` | 5 |
| `STATE_SETTINGS_WIFI` | 6 |
| `STATE_SETTINGS_ABOUT` | 7 |
| `STATE_SETTINGS_APPS` | 8 |
| `STATE_SETTINGS_TIME` | 9 |
| `STATE_SETTINGS_TIME_MANUAL` | 10 |
| `STATE_UPDATER_BOOT` | 11 |
| `STATE_UPDATER_MANUAL` | 12 |
| `STATE_APP_STORE` | 13 |
| `STATE_HELP_CENTER` | 14 |

### File System Paths

| Path | Description |
|------|-------------|
| `/local/apps/` | Installed apps on LittleFS |
| `/sd/apps/` | Installed apps on SD |
| `/local/nowifi.txt` | Disable WiFi |
| `/local/web_on.txt` | Enable web server |
| `/local/config_install_sd.txt` | Default install to SD |
| `/sd/wifi.txt` or `/local/wifi.txt` | WiFi credentials |
| `/local/config_time.txt` | Time preferences |
| `/local/touch_cal_p.bin` | Touch calibration |
| `/tmp_download/` | Temporary downloads |

---

*End of documentation.*
