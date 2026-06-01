# Thread Border Router on Raspberry Pi Zero 2W with XIAO nRF52840

## Overview

This guide covers adding a Thread network interface to a Raspberry Pi Zero 2W using a Seeed XIAO nRF52840 as the Radio Co-Processor (RCP). The XIAO connects to the Pi via UART pins and runs OpenThread RCP firmware, while the Pi runs `otbr-agent` as the Thread Border Router.

```
XIAO nRF52840 (RCP firmware)
    ↕ UART @ 1Mbaud (4 wires)
Raspberry Pi Zero 2W (otbr-agent)
    ↕ WiFi
Your application / MQTT broker
```

**Target use case:** Indoor sensor mesh where Thread nodes push data to the Pi over IPv6 UDP. No LAN bridging required.

---

## Hardware

- Raspberry Pi Zero 2W (Debian Trixie / 64-bit)
- Seeed XIAO nRF52840
- 4x dupont jumper wires (female)
- USB-C cable (to power XIAO and provide UART path during development)

---

## Part 1 — Build RCP Firmware for XIAO

### Prerequisites (Windows, nRF Connect SDK v3.2.1+)

- VS Code with nRF Connect for VS Code extension
- nRF Connect SDK v3.2.1 installed

### Project Setup

Create a new application based on the coprocessor sample:

```
nRF Connect → Create Application → Browse Samples → openthread/coprocessor
Board: xiao_ble
```

### prj.conf

```kconfig
# OpenThread RCP core
CONFIG_OPENTHREAD=y
CONFIG_OPENTHREAD_COPROCESSOR=y
CONFIG_OPENTHREAD_COPROCESSOR_RCP=y

# Nordic RCP library
CONFIG_OPENTHREAD_NORDIC_LIBRARY_RCP=y

# Keep GPIO on for UART
CONFIG_GPIO=y

# Storage
CONFIG_PM_PARTITION_SIZE_SETTINGS_STORAGE=0x8000

# Disable RTT
CONFIG_USE_SEGGER_RTT=n
```

### Board Overlay — boards/xiao_ble.overlay

This is critical — without this file the firmware has no UART assigned for spinel and will not respond.

```dts
/ {
    chosen {
        zephyr,ot-uart = &uart0;
    };
};

&uart0 {
    status = "okay";
    current-speed = <1000000>;
};
```

> **Note:** `uart0` on xiao_ble maps to P1.11 (TX) and P1.12 (RX) per `xiao_ble-pinctrl.dtsi`. These are the pins labeled **TX** and **RX** on the physical board (D6/D7).

### Verify Overlay Was Applied

Before flashing, confirm `ot-uart` is in the built device tree:

```powershell
Select-String -Path "C:\path\to\project\build\coprocessor\zephyr\zephyr.dts" -Pattern "ot-uart"
```

You should see:
```
zephyr,ot-uart = &uart0;   /* from xiao_ble.overlay */
```

If nothing is returned — the overlay was not picked up. Check the project directory and rebuild.

### Flash to XIAO

1. Double-tap the reset button on XIAO — RGB LED pulses, drive appears as `XIAO-SENSE`
2. Drag `build/coprocessor/zephyr/zephyr.uf2` onto the drive
3. Drive disconnects automatically — XIAO reboots into RCP firmware
4. Verify: XIAO appears as COM port (e.g. COM11) — LED off = RCP running correctly

---

## Part 2 — Prepare Raspberry Pi

### Enable SPI1 (if display HAT is present)

If a Waveshare or similar HAT occupies SPI0, enable SPI1 for other peripherals.

Edit `/boot/firmware/config.txt` (Debian Trixie uses this path, not `/boot/config.txt`):

```ini
[all]
enable_uart=1
dtparam=spi=on
dtoverlay=waveshare35b-v2       # your display overlay
dtoverlay=spi1-1cs              # enable SPI1
dtoverlay=disable-bt            # free ttyAMA0 from Bluetooth
extra_transpose_buffer=2
dtoverlay=dwc2
```

> **Important:** On Debian Trixie, `/boot/firmware/config.txt` is the active config. Edits to `/boot/config.txt` are ignored. Verify with `dtoverlay -l` after reboot.

Reboot and verify:

```bash
ls /dev/spidev*      # spidev0.0, spidev0.1 (display), spidev1.0 (free)
ls /dev/ttyAMA*      # ttyAMA0 (free for RCP)
```

### Free ttyAMA0 from agetty

By default, systemd runs a login terminal on ttyAMA0. This blocks otbr-agent from opening the port.

```bash
sudo systemctl mask serial-getty@ttyAMA0
sudo systemctl stop serial-getty@ttyAMA0

# Verify it's free
sudo lsof /dev/ttyAMA0
# Should return nothing
```

### Load Required Kernel Module

```bash
sudo modprobe mac802154
```

Make it persistent:

```bash
echo "mac802154" | sudo tee -a /etc/modules
```

---

## Part 3 — Wire XIAO to Pi

### Pin Connections

```
XIAO nRF52840          Pi Zero 2W GPIO Header
─────────────          ──────────────────────
TX  (D6 / P1.11)  →    Pin 10 (GPIO15 / RXD0)
RX  (D7 / P1.12)  →    Pin 8  (GPIO14 / TXD0)
GND               →    Pin 6  (GND)
VBUS (5V)         →    Pin 2  (5V)
```

> **Critical notes:**
> - TX → RX, RX → TX (crossed, not straight through)
> - Power XIAO from VBUS (5V), not 3V3 — the 3V3 pin on XIAO is an OUTPUT from the onboard regulator, not an input
> - If display HAT covers GPIO header, only pins 1–10 are accessible from the exposed corner

### Verify UART Loopback (Pi side only, no XIAO)

Before connecting XIAO, verify Pi UART works:

```bash
sudo apt install python3-serial -y

# Bridge Pin 8 to Pin 10 with a wire, then:
sudo python3 -c "
import serial
s = serial.Serial('/dev/ttyAMA0', 1000000, timeout=1)
s.write(b'hello')
print(s.read(10))
"
# Should print: b'hello'
```

---

## Part 4 — Build OTBR on Linux (Cross-Compile for Pi)

The Pi Zero 2W has insufficient RAM to compile OTBR natively. Cross-compile on a Linux machine (Ubuntu/Mint).

### Install Cross-Compile Tools

```bash
sudo apt update
sudo apt install -y git cmake ninja-build python3 \
  gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
  libdbus-1-dev libavahi-client-dev \
  libboost-dev libboost-filesystem-dev \
  libboost-system-dev libnetfilter-queue-dev \
  pkg-config libgtest-dev libsystemd-dev
```

### Clone OTBR

```bash
cd ~
git clone https://github.com/openthread/ot-br-posix.git
cd ot-br-posix
git submodule update --init --recursive
```

### Create Toolchain File

```bash
mkdir -p ~/ot-br-posix/cmake/toolchain

cat > ~/ot-br-posix/cmake/toolchain/aarch64.cmake << 'EOF'
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
EOF
```

### Patch Multicast Routing (Required for RPi Kernel)

The RPi kernel does not include `ip6mr` (IPv6 multicast routing). OTBR's `InitMulticastRouterSock()` calls `VerifyOrDie` which crashes the agent. Patch it to fail gracefully:

Edit `~/ot-br-posix/src/host/posix/multicast_routing_manager.cpp`, find `InitMulticastRouterSock()` and replace the two `VerifyOrDie` calls:

**Before:**
```cpp
mMulticastRouterSock = SocketWithCloseExec(AF_INET6, SOCK_RAW, IPPROTO_ICMPV6, kSocketBlock);
VerifyOrDie(mMulticastRouterSock != -1, "Failed to create socket");
VerifyOrDie(0 == setsockopt(mMulticastRouterSock, IPPROTO_IPV6, MRT6_INIT, &one, sizeof(one)),
            "Failed to enable multicast forwarding");
```

**After:**
```cpp
mMulticastRouterSock = SocketWithCloseExec(AF_INET6, SOCK_RAW, IPPROTO_ICMPV6, kSocketBlock);
if (mMulticastRouterSock == -1) {
    otbrLogWarning("Multicast routing not available - skipping");
    return;
}
if (0 != setsockopt(mMulticastRouterSock, IPPROTO_IPV6, MRT6_INIT, &one, sizeof(one))) {
    otbrLogWarning("Failed to enable multicast forwarding - skipping");
    close(mMulticastRouterSock);
    mMulticastRouterSock = -1;
    return;
}
```

> **Why this is safe:** Multicast routing is only needed for bridging Thread traffic to the LAN. For a Pi that acts as the data sink (sensors push UDP to Pi directly), unicast routing is sufficient and multicast routing is never used.

### Build

```bash
mkdir -p ~/ot-br-posix/build && cd ~/ot-br-posix/build

cmake .. \
  -GNinja \
  -DCMAKE_TOOLCHAIN_FILE=../cmake/toolchain/aarch64.cmake \
  -DOTBR_BORDER_ROUTING=ON \
  -DOTBR_REST=OFF \
  -DOTBR_WEB=OFF \
  -DBUILD_TESTING=OFF \
  -DCMAKE_DISABLE_FIND_PACKAGE_Systemd=TRUE \
  -DLIBSYSTEMD_LIBRARIES="" \
  -DLIBSYSTEMD_FOUND=FALSE \
  -DCMAKE_BUILD_TYPE=Release

ninja -j4
```

### Copy to Pi

```bash
scp ~/ot-br-posix/build/src/agent/otbr-agent user@hvacvibe.local:~/
```

### Install on Pi

```bash
sudo mv ~/otbr-agent /usr/local/sbin/otbr-agent
sudo chmod +x /usr/local/sbin/otbr-agent
```

---

## Part 5 — Run and Test

### Start otbr-agent

```bash
sudo otbr-agent -v -I wpan0 -B wlan0 \
  --vendor-name "OTBR" \
  --model-name "RaspberryPi" \
  spinel+hdlc+uart:///dev/ttyAMA0?uart-baudrate=1000000
```

Successful output looks like:
```
[NOTE]-AGENT---: Running 0.3.0-thread-reference-...
[INFO]-APP-----: Radio Co-processor version: OPENTHREAD/ncs-thread-reference-...; Zephyr; ...
```

### Form a Thread Network

```bash
sudo ot-ctl dataset init new
sudo ot-ctl dataset commit active
sudo ot-ctl ifconfig up
sudo ot-ctl thread start
sleep 10
sudo ot-ctl state
```

Expected: `leader`

### Verify Network

```bash
sudo ot-ctl state        # leader
sudo ot-ctl ipaddr       # list IPv6 addresses
sudo ot-ctl dataset active -x   # full dataset (save this for sensor nodes)
sudo ot-ctl networkkey   # network key
sudo ot-ctl channel      # active channel
sudo ot-ctl panid        # PAN ID
```

---

## Part 6 — Make it Persistent (systemd service)

Create `/etc/systemd/system/otbr-agent.service`:

```ini
[Unit]
Description=OpenThread Border Router Agent
After=network.target

[Service]
ExecStart=/usr/local/sbin/otbr-agent -I wpan0 -B wlan0 \
  --vendor-name "OTBR" \
  --model-name "RaspberryPi" \
  spinel+hdlc+uart:///dev/ttyAMA0?uart-baudrate=1000000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable otbr-agent
sudo systemctl start otbr-agent
sudo systemctl status otbr-agent
```

---

## Troubleshooting

### ttyAMA0 not appearing
- Confirm `dtoverlay=disable-bt` is in `/boot/firmware/config.txt` (not `/boot/config.txt`)
- Reboot and check `ls /dev/ttyAMA*`

### agetty holding ttyAMA0
```bash
sudo lsof /dev/ttyAMA0       # check what's holding it
sudo systemctl mask serial-getty@ttyAMA0
sudo systemctl stop serial-getty@ttyAMA0
```

### otbr-agent exits immediately (exit code 5)
- Check `sudo lsof /dev/ttyAMA0` — another process may own it
- Check `sudo systemctl status serial-getty@ttyAMA0`

### "No response from RCP during initialization"
- Wrong baud rate — use `uart-baudrate=1000000`
- Wrong firmware flashed — verify `ot-uart` is in built `zephyr.dts`
- Wrong pins — confirm TX→RX, RX→TX crossing
- XIAO powered from 3V3 pin (wrong) — use VBUS (5V)

### "Failed to communicate with RCP" at other baud rates
- Only 1000000 baud gives meaningful error — other rates time out silently
- This is expected behavior; always use 1000000

### otbr-agent crashes at multicast routing init
- RPi kernel lacks `ip6mr` module — apply the multicast routing patch above
- Verify with: `find /lib/modules/$(uname -r) -name "*mroute*"`

### wpan0 interface missing
```bash
sudo modprobe mac802154
```

---

## Part 7 — ot-ctl Reference

`ot-ctl` is the command line interface to control `otbr-agent` while it's running. It connects via a local Unix socket to the running daemon.

```
ot-ctl  ←→  Unix socket  ←→  otbr-agent  ←→  XIAO RCP  ←→  Thread network
```

Think of it like `iwconfig`/`ip` for WiFi — your Thread control plane CLI.

### Network State
```bash
sudo ot-ctl state              # leader / router / child / detached
sudo ot-ctl dataset active -x  # full network dataset (save for sensor nodes)
sudo ot-ctl channel            # current channel (11-26)
sudo ot-ctl panid              # PAN ID
sudo ot-ctl networkkey         # network encryption key
```

### Node Management
```bash
sudo ot-ctl ipaddr             # Pi's Thread IPv6 addresses
sudo ot-ctl neighbor list      # connected Thread neighbors
sudo ot-ctl router list        # Thread routers in network
sudo ot-ctl child list         # child nodes attached to Pi
```

### Diagnostics
```bash
sudo ot-ctl ping <ipv6addr>    # ping a Thread node
sudo ot-ctl counters           # packet counters
sudo ot-ctl rss                # received signal strength
```

### Network Formation
```bash
sudo ot-ctl dataset init new   # generate new network credentials
sudo ot-ctl dataset commit active
sudo ot-ctl ifconfig up
sudo ot-ctl thread start
sudo ot-ctl thread stop
```

### Install ot-ctl

`ot-ctl` is built alongside `otbr-agent` in the cross-compile step. Copy it to Pi the same way:

```bash
# On Linux build machine
scp ~/ot-br-posix/build/src/agent/ot-ctl user@hvacvibe.local:~/

# On Pi
sudo mv ~/ot-ctl /usr/local/bin/ot-ctl
sudo chmod +x /usr/local/bin/ot-ctl
```

---

## Key Learnings

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| `/boot/config.txt` ignored | Debian Trixie uses `/boot/firmware/config.txt` | Edit correct file |
| No `/dev/ttyAMA0` | BT not disabled | `dtoverlay=disable-bt` |
| ttyAMA0 busy | agetty login terminal | `systemctl mask serial-getty@ttyAMA0` |
| RCP no response | Wrong firmware (missing overlay) | Verify `ot-uart` in zephyr.dts |
| RCP no response | XIAO powered from 3V3 output pin | Power from VBUS (5V) |
| otbr-agent exits (code 5) | ip6mr missing from RPi kernel | Patch multicast_routing_manager.cpp |
| OTBR build OOM on Pi | Only 512MB RAM | Cross-compile on Linux x86 |
