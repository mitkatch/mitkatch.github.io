---
title: "WiFi Provisioning on Raspberry Pi Zero 2W with Ubuntu — The NetworkManager Way"
date: 2026-03-27
draft: false
tags: ["raspberry-pi", "ubuntu", "networkmanager", "hostapd", "linux", "hvac-vibe"]
description: "Every tutorial assumes Raspbian and wpa_supplicant. This one doesn't. A complete guide to headless WiFi provisioning on Ubuntu cloud-init with NetworkManager, hostapd, and dnsmasq — including every failure mode I hit."
---

The HVAC-Vibe gateway is a headless Raspberry Pi Zero 2W. No screen, no keyboard, no
ethernet. It needs to join a customer's WiFi network on first boot without any manual
SSH configuration. The standard solution is a captive portal: Pi broadcasts its own
access point, user connects their phone, picks a network, enters a password, Pi joins
it and starts working.

Every tutorial I found assumed Raspbian and `wpa_supplicant`. My image is Ubuntu
cloud-init with NetworkManager. Completely different tools, different behavior, and
every one of those tutorials silently fails in ways that are hard to debug on a headless
device.

This post documents what actually works — including ten specific failure modes I hit
and how I resolved each one.

---

## The Setup

**Hardware:** Raspberry Pi Zero 2W  
**OS:** Ubuntu 22.04 cloud-init (not Raspbian)  
**Network manager:** NetworkManager (`nmcli`) — not `wpa_supplicant`  
**Goal:** On first boot, Pi broadcasts `HVAC-Vibe-Setup` AP → user connects phone →
picks their network from a list → enters password → Pi joins that network →
normal operation starts

The key distinction upfront: **if you are running Raspbian, this post is not for you.**
Raspbian uses `wpa_supplicant` and `dhcpcd`. Ubuntu server images use NetworkManager.
They are not interchangeable. Commands that work on one fail silently on the other.

---

## Why Ubuntu Instead of Raspbian

Ubuntu cloud-init gives you a few things that matter for a deployed IoT device:

- `cloud-init` handles first-boot WiFi configuration via `/boot/firmware/network-config`
  — a YAML file writable from Windows without any Linux tools
- NetworkManager handles reconnection, profile persistence, and interface state more
  robustly than `wpa_supplicant` for long-running unattended devices
- The same provisioning approach works on any Ubuntu ARM device, not just Pi

The tradeoff: almost no maker tutorials target it, so you're on your own when things
break.

---

## Architecture

The provisioning flow looks like this:

```text
Boot
 └── hvac-setup.service starts
      └── needs_setup()?
           ├── NO  → show IP on display, exit
           │         hvac-engine starts normally
           └── YES → scan WiFi (while NM owns wlan0)
                     start AP (hostapd + dnsmasq)
                     start Flask web server
                     show QR code on display
                     user connects phone, picks network
                     stop AP, hand wlan0 back to NM
                     connect via nmcli
                     mark setup done
                     hvac-engine starts
```

A few design decisions worth explaining:

**Why pre-scan before starting the AP?** Once `wlan0` switches to AP mode it can no
longer scan for client networks — the radio can only do one thing at a time. Scan
first, cache the results, show them in the browser. This is non-obvious and most
tutorials skip it.

**Why hostapd + dnsmasq instead of NetworkManager's AP mode?** NetworkManager can
create a hotspot, but releasing the interface back to client mode afterward is
unreliable in testing. hostapd gives explicit control over when the AP starts and
stops.

**Why Flask?** A simple HTTP server would work for serving the page, but Flask makes
handling the form POST (network selection + password submission) and running the
connection logic in a background thread straightforward.

---

## Prerequisites

Install the required packages:

```bash
sudo apt update
sudo apt install hostapd dnsmasq python3-flask python3-qrcode network-manager
```

Disable the system dnsmasq service — it binds to port 67 (DHCP) on startup and
blocks our instance from starting:

```bash
sudo systemctl disable dnsmasq
sudo systemctl stop dnsmasq
```

This is not optional. If system dnsmasq is running when you start your own instance,
you get `Address already in use` on port 67 and DHCP never works — phones connect
to the AP but get no IP address and show a spinner indefinitely.

---

## Set Up the USB Serial Console First — Before Anything Else

Before touching any WiFi configuration, set up the USB serial console and verify
it works. This is not optional and it is not a recovery-only tool — it is your
primary debugging interface throughout development.

A headless Pi with broken WiFi is completely inaccessible without an out-of-band
console. I learned this the hard way: the COM port saved the project multiple times
when WiFi provisioning was stuck in a broken state with no way to SSH in and no
display output that explained what was wrong.

Set it up first, test it works, then proceed. You will thank yourself later.

### How It Works

The Pi Zero 2W micro USB data port supports USB gadget mode. When enabled, the Pi
presents itself to a connected Windows or Mac machine as a USB serial (COM) port.
You connect with any terminal emulator — PuTTY on Windows — and get a full Linux
console regardless of network state.

This works even when:
- WiFi is completely broken
- The provisioning script crashed
- NetworkManager is in an unknown state
- The display shows nothing useful

### Enable USB Serial Gadget

These files are on the FAT32 boot partition — writable directly from Windows
before ever booting the Pi, or from the Pi itself after first boot.

**Edit `/boot/firmware/config.txt`** — add this line at the end:

```ini
dtoverlay=dwc2
```

**Edit `/boot/firmware/cmdline.txt`** — this file is a single long line.
Find `rootwait` and add `modules-load=dwc2,g_serial` immediately after it,
separated by a space:

```ini
... rootwait modules-load=dwc2,g_serial quiet ...
```

Do not add a newline — the entire file must remain one line. Adding a newline
here causes a boot failure that looks completely unrelated.

**Enable the getty service** — run this on the Pi:

```bash
sudo systemctl enable getty@ttyGS0.service
sudo systemctl start getty@ttyGS0.service
```

Reboot:

```bash
sudo reboot
```

### Connect from Windows

Connect the micro USB cable to the **data port** on the Pi Zero 2W — not the
power port. The Pi Zero 2W has two micro USB ports: one labelled PWR (power only)
and one labelled USB (data + power). Use the data port.

Open **Device Manager** on Windows. Under **Ports (COM & LPT)** you should see
a new entry appear — something like `USB Serial Device (COM3)`. Note the COM
number.

If nothing appears under Ports, look for an unknown device — you may need to
install a driver. On Windows 10/11 it usually installs automatically.

Open **PuTTY**:
- Connection type: **Serial**
- Serial line: `COM3` (use your actual COM number)
- Speed: `115200`
- Click Open

You should see a Linux login prompt. Log in with your Ubuntu credentials.

### Verify It Works

Once connected, run a quick test to confirm the console is fully functional:

```bash
# Check network state
nmcli dev status

# Check systemd services
systemctl status hvac-setup.service

# Watch live logs
journalctl -f
```

The `journalctl -f` command is your most important debugging tool — it streams
all system logs in real time. When your provisioning script runs and something
goes wrong, the exact error appears here immediately. Without this you are
guessing.

### One Gotcha: Power

The Pi Zero 2W draws enough current during WiFi activity that some USB hubs and
laptop ports can't supply enough power reliably. If the Pi resets unexpectedly
during WiFi operations, power is the first thing to check. Use a dedicated USB
port directly on your machine, not a hub, and use a quality cable.

---

## Step 1: Scan Networks Before Touching the AP

This must happen while NetworkManager still owns `wlan0` — before any AP setup.

```bash
# Force a fresh scan
nmcli dev wifi rescan

# List visible networks with signal strength
nmcli -t -f SSID,SIGNAL,SECURITY dev wifi list
```

Cache these results. Once you start hostapd, `wlan0` is in AP mode and these commands
return nothing.

---

## Step 2: Release wlan0 from NetworkManager

This is the step every Raspbian tutorial omits because it doesn't apply there.
NetworkManager actively manages `wlan0` in client mode. If you start hostapd without
releasing the interface first, NM and hostapd fight over it. The AP may appear to
start but phones can't see it, or they see it but can't connect.

```bash
# Disconnect from any current network
sudo nmcli dev disconnect wlan0

# Tell NetworkManager to stop managing this interface entirely
sudo nmcli dev set wlan0 managed no
```

Verify it worked:

```bash
nmcli dev status
# wlan0 should show: unmanaged
```

---

## Step 3: Assign a Static IP to wlan0

hostapd handles the AP radio but doesn't assign an IP. You need one so dnsmasq
can serve DHCP leases from a known address:

```bash
sudo ip addr flush dev wlan0
sudo ip addr add 192.168.4.1/24 dev wlan0
sudo ip link set wlan0 up
```

---

## Step 4: Start hostapd

Create a minimal hostapd config at `/tmp/hostapd.conf`:

```ini
interface=wlan0
driver=nl80211
ssid=HVAC-Vibe-Setup
hw_mode=g
channel=6
wpa=0
```

`wpa=0` means open network — no password on the setup AP. Acceptable since it's
temporary and serves only a provisioning page.

Start it:

```bash
sudo hostapd -B /tmp/hostapd.conf
```

**Important:** Wait 2 seconds before starting dnsmasq. hostapd needs time to fully
initialize the interface before dnsmasq can bind to it. If dnsmasq starts too early
it fails silently and phones get no DHCP lease.

```bash
sleep 2
```

---

## Step 5: Start dnsmasq

```bash
sudo dnsmasq \
  --no-daemon \
  --interface=wlan0 \
  --dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h \
  --address=/#/192.168.4.1 \
  --log-queries &
```

The `--address=/#/192.168.4.1` line redirects all DNS queries to the Pi's IP —
this is what makes it a captive portal. Any domain a phone tries to look up resolves
to 192.168.4.1, which triggers the captive portal detection on iOS and Android
and pops up the provisioning page automatically.

At this point:
- `HVAC-Vibe-Setup` is visible on a phone
- Connecting gives the phone an IP in the 192.168.4.x range
- Opening any URL redirects to your Flask server

---

## Step 6: Stop the AP and Hand wlan0 Back

When the user submits their network credentials, you need to tear down the AP
before NM can use `wlan0` as a client again.

```bash
# Kill AP processes
sudo pkill hostapd
sleep 2
sudo pkill dnsmasq

# Flush the static IP
sudo ip addr flush dev wlan0

# Return interface to NetworkManager
sudo nmcli dev set wlan0 managed yes
```

**The tricky part:** NM doesn't immediately process the `managed yes` command.
You must poll until it confirms the interface is managed before proceeding:

```bash
# Poll until wlan0 is no longer unmanaged
for i in $(seq 1 20); do
    STATE=$(nmcli -t -f DEVICE,STATE dev status | grep "^wlan0" | cut -d: -f2)
    if [ "$STATE" != "unmanaged" ]; then
        echo "wlan0 is now: $STATE"
        break
    fi
    echo "Waiting for NM to take over wlan0... ($i/20)"
    # Retry the managed command every 5 seconds
    if [ $((i % 5)) -eq 0 ]; then
        sudo nmcli dev set wlan0 managed yes
    fi
    sleep 1
done
```

Skipping this poll is the cause of the most frustrating failure mode: NM is given
the interface back but hasn't finished processing it. The next `nmcli` command
runs against an interface NM thinks is still unmanaged. The command silently fails
or returns "No network with SSID found" even when the network is clearly visible.

---

## Step 7: Connect to the Target Network

After the AP is down and NM owns `wlan0` again, wait 5 seconds for the radio to
reinitialize in client mode, then rescan:

```bash
sleep 5
nmcli dev wifi rescan
```

Verify the target SSID is visible before attempting connection:

```bash
nmcli -t -f SSID dev wifi list | grep "^${TARGET_SSID}$"
```

If it's not visible yet, retry up to 4 times with 5 second gaps. The radio needs
time after switching from AP mode.

**Do not use `nmcli dev wifi connect`** — it is unreliable without a pre-existing
connection profile for the SSID. It fails with "No network with SSID found" or
"key-mgmt property missing" depending on the NM version.

Use `nmcli con add` + `nmcli con up` instead. This explicitly creates a profile
with the correct security settings:

```bash
# Create the connection profile
sudo nmcli con add \
    type wifi \
    con-name "hvac-setup-${TARGET_SSID}" \
    ifname wlan0 \
    ssid "${TARGET_SSID}" \
    wifi-sec.key-mgmt wpa-psk \
    wifi-sec.psk "${PASSWORD}"

# Bring it up
sudo nmcli con up "hvac-setup-${TARGET_SSID}"
```

The `con-name` prefix `hvac-setup-` ensures our profiles never conflict with any
existing NM profiles for the same network.

---

## Step 8: Verify Connection and Get IP

```bash
# Wait for IP assignment (up to 30 seconds)
for i in $(seq 1 30); do
    IP=$(nmcli -t -f IP4.ADDRESS dev show wlan0 | cut -d: -f2 | cut -d/ -f1)
    if [ -n "$IP" ]; then
        echo "Connected: $IP"
        break
    fi
    sleep 1
done
```

Once you have an IP, write it to the cloud-init network config so it persists
across reboots:

```bash
# Store hashed password (never store plaintext)
HASH=$(wpa_passphrase "${TARGET_SSID}" "${PASSWORD}" | grep -v "#psk" | grep "psk=" | cut -d= -f2)

cat > /boot/firmware/network-config << EOF
version: 2
wifis:
  wlan0:
    dhcp4: true
    access-points:
      "${TARGET_SSID}":
        password: "${HASH}"
EOF
```

Mark setup as complete:

```bash
sudo mkdir -p /etc/hvac-vibe
sudo touch /etc/hvac-vibe/wifi-configured
```

---

## Recovery Mechanisms

A headless device needs recovery paths that don't require network access.

**SD card triggers** — create empty files on the `/boot/firmware` partition
(FAT32, writable from Windows Explorer or Mac Finder without any tools):

```ini
hvac-reset-wifi    → forces setup UI on next boot
hvac-restore-wifi  → restores factory WiFi config silently
```

On Windows:
```powershell
# Insert SD card, E: is the boot partition
New-Item E:\hvac-reset-wifi -ItemType File
```

**USB serial console** — if you followed the setup section at the top of this post,
this is already working. Connect the data USB port to Windows, open PuTTY on the
COM port at 115200 baud, and you have a full console regardless of network state.
This is the most reliable recovery path — use it first.

**But the actual saver** was restore.sh running on live COM connection every time need to back to my wifi:
```bash
#!/bin/bash
echo Delete ALL home-wifi duplicates first
nmcli con show | grep "home-wifi" | awk '{print $2}' | while read uuid; do
     sudo nmcli con delete "$uuid"
     echo delete "$uuid"
done
#
echo Return wlan0 to NM
sudo nmcli dev set wlan0 managed yes
sleep 3
#
echo Add fresh profile
sudo nmcli con add \
   type wifi \
   ifname wlan0 \
   con-name "home-wifi" \
   ssid "YUOUR_WIFI_NETWORK" \
   wifi-sec.key-mgmt wpa-psk \
   wifi-sec.psk "YOUR_psk_generated"
#
sudo nmcli con up "home-wifi"

```
to generate your psk run and use only code ,like "c6757**:
```bash
wpa_passphrase "YOUR-WIFI" "your_text_password" | grep -v '#' | grep psk=

```
---

## Complete Sequence Reference

```
1. scan_networks()           ← while NM owns wlan0, BEFORE AP
2. pkill dnsmasq             ← kill any system instance
3. nmcli dev disconnect wlan0
4. nmcli dev set wlan0 managed no
5. ip addr add 192.168.4.1/24 dev wlan0
6. hostapd -B hostapd.conf
7. sleep 2
8. dnsmasq (with captive portal redirect)
9. Flask serves provisioning page with cached network list

--- user submits credentials ---

10. pkill hostapd
11. sleep 2
12. pkill dnsmasq
13. ip addr flush dev wlan0
14. nmcli dev set wlan0 managed yes
15. poll until not unmanaged (up to 20s, retry managed every 5s)
16. sleep 5                  ← radio reinitialize
17. nmcli dev wifi rescan
18. retry until SSID visible (4x, 5s gaps)
19. nmcli con add (explicit profile with wpa-psk)
20. nmcli con up
21. poll for IP
22. write /boot/firmware/network-config
23. touch /etc/hvac-vibe/wifi-configured
24. systemd starts hvac-engine
```

---

## What I Learned

- **Ubuntu cloud-init uses NetworkManager, not wpa_supplicant.** Every command that
  works in a Raspbian tutorial needs to be rewritten.
- **NM and hostapd fight over wlan0.** You must explicitly release the interface
  before starting the AP and verify managed state is restored before connecting.
- **`nmcli dev wifi connect` is unreliable.** Use `con add` + `con up` with explicit
  security settings.
- **Scan before AP mode.** The radio cannot be AP and client simultaneously.
- **Timing matters more than you expect.** The `sleep 2` after hostapd, the `sleep 5`
  after AP teardown, the NM managed state poll — skipping any of these produces
  intermittent failures that are nearly impossible to debug on a headless device.
- **Set up the USB serial console before anything else.** `journalctl -f` over a
  COM port is the only way to see what is actually happening on a headless device
  with broken WiFi. It saved this project multiple times. Do not skip it.
- **Always have multiple recovery paths.** SD card trigger files, USB serial console,
  and a restore script — build all of them in before you need them.

---

## Full Implementation

The complete Python implementation (Flask server, AP control, WiFi connection logic,
display integration) is in the HVAC-Vibe repository:

[github.com/mitkatch/hvacvibe-hub](https://github.com/mitkatch/hvacvibe-hub) — branch `pygame03`, directory `hvac-setup/`
