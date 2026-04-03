---
title: "HVAC-Vibe Display"
date: 2026-04-03
draft: false
tags: ["raspberry-pi", "pygame", "waveshare", "lcd", "mqqt", "hvac-vibe"]
description: "The display subsystem for HVAC-Vibe runs on a Waveshare 3.5 IPS LCD HAT attached to the Raspberry Pi Zero 2W gateway. It shows live vibration data, environmental readings, and a daily RMS trend chart — all rendered locally via pygame directly to the  framebuffer."

---

## Requirements

**Hardware**
- Raspberry Pi Zero 2W
- Waveshare 3.5" IPS LCD (B) HAT — 480×320, SPI, `fb_ili9486` driver
- MicroSD card (16GB+)

**Software**
- Ubuntu 22.04 (cloud-init image)
- Python 3.11+
- pygame 2.x
- numpy (RGB565 framebuffer conversion)
- paho-mqtt (data feed from hvac-engine)

**Python packages**
```bash
pip install pygame numpy paho-mqtt
```

---

## Installing the Waveshare 3.5" LCD (B) HAT

### 1. Install the driver

Waveshare provides a driver installer script. Clone their repo and run it:

```bash
git clone https://github.com/waveshare/LCD-show.git ~/LCD-show
cd ~/LCD-show
sed -i 's/\r//' LCD35B-show-V2     # fix Windows line endings if present
sudo ./LCD35B-show-V2               # installs driver and reboots automatically
```

After reboot the LCD registers as `/dev/fb0` (primary framebuffer) via the
`fb_ili9486` kernel module.

### 2. Disable the touch controller overlay (GPIO conflict)

The ADS7846 touch overlay shares GPIO17 with the display driver, causing an
initialization conflict. Since hvac-pygame doesn't use touch, disable it:

Edit `/boot/firmware/config.txt` and comment out:

```ini
# dtoverlay=ads7846,speed=2000000,penirq=17,...
```

Reboot for the change to take effect.

### 3. Create a stable udev symlink

The kernel assigns framebuffer numbers non-deterministically at boot — the LCD
can appear as `fb0` or `fb1` depending on driver initialization order. A udev
rule creates a stable `/dev/waveshare` symlink regardless of which number is
assigned:

Create `/etc/udev/rules.d/99-waveshare.rules`:

```ini
SUBSYSTEM=="graphics", KERNEL=="fb*", ATTR{name}=="fb_ili9486", SYMLINK+="waveshare"
```

Reload udev:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
ls -la /dev/waveshare   # should show -> fb0 or -> fb1
```

Then set `fb_device` in `config.py` to `/dev/waveshare` instead of a hardcoded
device path.

### 4. Prevent the OS from claiming the framebuffer on boot

By default the kernel sends boot messages and the getty login prompt to `fb0`,
which overwrites the display before pygame starts. Two things suppress this:

**Suppress kernel console output on the LCD** — add to `/boot/firmware/cmdline.txt`:

```ini
fbcon=map:1 fbcon=font:VGA8x8
```

This redirects the framebuffer console to a non-existent display, leaving `fb0`
free for pygame.

**Suppress getty on fb0** — mask the relevant service:

```bash
sudo systemctl mask getty@tty1.service
```

### 5. Run the service as root

The `fb_ili9486` driver creates the framebuffer device as `root:root 644`. The
udev group rule does not stick because DRM recreates the device node after udev
runs. The simplest fix is to run `hvac-pygame.service` as root:

```ini
[Service]
User=root
ExecStartPre=/bin/bash -c 'until [ -c "$(readlink -f /dev/waveshare)" ]; do sleep 1; done'
ExecStart=/home/mitkatch/hvac-pygame/venv/bin/python main.py
```

The `ExecStartPre` line polls until `/dev/waveshare` resolves to a character
device, handling the race condition where pygame starts before the driver
initializes.

### Framebuffer rendering

pygame's `SDL_VIDEODRIVER=fbcon` was removed in SDL2. The display writes pixels
directly to the framebuffer using `offscreen` SDL mode with a manual flush:

```python
os.environ['SDL_VIDEODRIVER'] = 'offscreen'

def flush_to_fb(surface):
    rotated = pygame.transform.rotate(surface, DISPLAY["rotate"])
    raw = pygame.image.tostring(rotated, 'RGB')
    px  = np.frombuffer(raw, dtype=np.uint8).reshape(-1, 3)
    r, g, b = px[:,0].astype(np.uint16), px[:,1].astype(np.uint16), px[:,2].astype(np.uint16)
    rgb565 = ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3)
    with open(DISPLAY["fb_device"], 'wb') as f:
        f.write(rgb565.astype('<u2').tobytes())
```

Each frame is rendered offscreen, converted from RGB888 to RGB565
(the native format of the LCD controller), then written directly to the
framebuffer device. numpy vectorizes the pixel conversion — a pure Python
loop over 153,600 pixels would be too slow at 5 fps.

---

## pygame vs React Kiosk

Before settling on pygame we evaluated running a Chromium kiosk serving a
React dashboard via a local FastAPI/WebSocket backend.

| | pygame | React + Chromium kiosk |
|---|---|---|
| RAM usage | ~40 MB | ~250–400 MB |
| CPU at idle | <5% | 15–25% |
| Boot to display | ~3s | ~20–30s |
| GPU dependency | None — software render | Requires GPU / DRM |
| Network dependency | MQTT (local) | WebSocket (local HTTP) |
| Offline operation | Full | Full (once loaded) |
| Development speed | Slower — draw calls | Faster — HTML/CSS |
| Mobile access | No | Yes (browser) |
| Pi Zero 2W viability | Excellent | Marginal — OOM risk |

On the Pi Zero 2W with 512MB RAM, Chromium routinely triggered the OOM killer
under moderate load. pygame with direct framebuffer write uses a fraction of the
memory and starts in seconds. The tradeoff is lower-level rendering code — every
label, tile, and chart is drawn with explicit coordinates rather than CSS layout.

The React kiosk approach is still viable for Pi 4 class hardware where memory
and GPU access are not constraints.

---

## Display Design

The display uses an adaptive layout engine that switches between four modes
based on the number of active sensors:

```text
1 sensor  → Full screen: header + 5 metric tiles + daily RMS chart
2 sensors → Split screen: compact row per sensor with sparkline
3-4       → 2×2 grid: tile + mini sparkline per cell
5+        → List view: one row per sensor, 8 visible
```

**Single sensor layout (most common)**

![HVAC-Vibe single sensor display](/images/hvac-display.png)


**Chart features**
- Daily RMS trend for the current day (minute-resolution from SQLite)
- Alarm threshold line in red
- NOW marker showing current time position
- Future region dimmed with a semi-transparent overlay
- Last value annotated at the current point

**Color palette**

```python
C_BG         = (18,  22,  28)   # near-black background
C_ACCENT     = (0,  160, 220)   # cyan — primary labels, HVAC-Vibe brand
C_GREEN      = (0,  200, 100)   # connected / healthy
C_YELLOW     = (255, 190,   0)  # temperature, peak values
C_RED        = (220,  50,  50)  # alarm, disconnected
C_CHART_LINE = (0,  180, 255)   # vibration RMS line
C_NOW_LINE   = (255, 220,  60)  # current time marker
```

---

## Data Feed — MQTT

The display has no direct dependency on BLE, SQLite, or the engine internals.
It consumes data exclusively via MQTT from a local Mosquitto broker, through
a lightweight `MQTTStore` that mirrors the `DataStore` interface.

**Topic subscriptions**

```text
hvac/{gateway_id}/{sensor_id}/status              ← connection, battery, rssi
hvac/{gateway_id}/{sensor_id}/vibration/features  ← rms, peak, dominant_hz
hvac/{gateway_id}/{sensor_id}/environment         ← temp, humidity, pressure
hvac/{gateway_id}/{sensor_id}/alert               ← alarm state changes
```

**Data flow**

```text
nRF52840 sensor
    ↓  BLE
Raspberry Pi Zero 2W
    ↓  hvac-engine (Python/bleak)
    ↓  MQTT publish → Mosquitto (localhost:1883)
    ↓  mqtt_store.py subscribes to hvac/#
    ↓  SensorState objects updated in memory
    ↓  display.py reads store on each frame
    ↓  flush_to_fb() → /dev/waveshare → LCD
```

**`mqtt_store.py`** is a drop-in replacement for the original `data_store.py`.
It subclasses `DataStore` and populates `SensorState` objects from MQTT messages
rather than direct BLE reads. This decouples the display completely from the
data collection layer — the display works identically whether driven by real
sensors, simulation mode, or replayed data.

The `status` topic is published with `retain=True` so the display immediately
shows the last known sensor state on connect, without waiting for the next
BLE burst (which arrives every ~10 seconds).

---

## Taking a Screenshot

To capture what's on the display:

```bash
# On the Pi
sudo cat /dev/fb0 > /tmp/screenshot.raw
```

```powershell
# On Windows — pull the raw file
scp mitkatch@hvacvibe.local:/tmp/screenshot.raw .
```

```python
# Convert RGB565 raw to PNG
from PIL import Image
import numpy as np

data = open("screenshot.raw", "rb").read()
arr  = np.frombuffer(data, dtype=np.uint16).reshape((320, 480))
r = ((arr & 0xF800) >> 11) << 3
g = ((arr & 0x07E0) >> 5)  << 2
b = (arr  & 0x001F)        << 3
Image.fromarray(np.stack([r, g, b], axis=2).astype(np.uint8)).save("display.png")
```
