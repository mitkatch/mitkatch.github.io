---
title: "HVAC-Vibe: Cloudflare Tunnel Setup Guide"
date: 2026-04-04
draft: false
tags: ["raspberry-pi", "ubuntu", "networkmanager", "hostapd", "linux", "Cloudflare"]
description: "the goal is: Expose the Pi Zero 2W FastAPI backend (`api.hvacvibe.ca`) to the internet via Cloudflare Tunnel — no port forwarding, no static IP required."
---

 

**Final architecture:**

```text
nRF52840 (BLE)
      ↓
Pi Zero 2W  [hvac-engine → SQLite → FastAPI :8000]
      ↓
cloudflared daemon  (outbound tunnel only)
      ↓
Cloudflare Edge (YYZ — Toronto)
      ↓
https://api.hvacvibe.ca  (public internet)
```

---

## Part 1 — Cloudflare Account & Domain

### 1.1 Register domain via Cloudflare Registrar

- Go to `dash.cloudflare.com` → **Domain Registration** → **Register Domains**
- Search and purchase `hvacvibe.ca`
- DNS is automatically configured — no manual nameserver setup required
- A payment receipt is sent by email

### 1.2 Locate the Tunnels section

The Tunnels UI lives in the **main Cloudflare dashboard**, not inside Zero Trust.

**Navigation path:**

```text
dash.cloudflare.com → Networking → Tunnels
```

> **Note:** The Zero Trust sidebar has a "Networks" section but it shows Connectors/Routes/Resolvers — not Tunnels. Do not use the Zero Trust onboarding wizard for this.

### 1.3 Create a tunnel

1. Go to **Networking → Tunnels → Create a tunnel**
2. Choose **Cloudflared** connector type
3. Name the tunnel: `hvacvibe`
4. On the **Setup Environment** step:
   - OS: **Debian**
   - Architecture: **ARM 32-bit**
5. The dashboard generates three commands:
   - **Install cloudflared** — adds Cloudflare apt repo and installs the binary
   - **Install as service** — registers systemd service with token embedded ✅ use this one
   - **Run tunnel (manual)** — for one-off testing only, skip for production

> **Security notice:** The token in these commands grants access to your Cloudflare account. Never share or commit it to version control.

---

## Part 2 — Install cloudflared on the Pi

### 2.1 Run the install commands

SSH into the Pi and run the two commands from the dashboard in order:

```bash
# Step 1: Install cloudflared (copy exact command from dashboard)
# Adds Cloudflare GPG key, apt repo, and installs cloudflared

# Step 2: Install and enable as systemd service
sudo cloudflared service install <your-token-here>
```

### 2.2 Verify service status

```bash
sudo systemctl status cloudflared
journalctl -fu cloudflared.service
```

Watch for `INF Connection established connIndex=0` lines. The Cloudflare dashboard Connection Status panel will flip from "Waiting..." to **Healthy** with a green indicator.

---

## Part 3 — Configure Public Hostname

Once the tunnel shows **Healthy** in the dashboard:

1. Click the tunnel name (`hvacvibe`)
2. Go to the **Public Hostname** tab
3. Click **Add a public hostname**

| Field | Value |
|---|---|
| Subdomain | `api` |
| Domain | `hvacvibe.ca` |
| Path | *(leave empty)* |
| Type | `HTTP` |
| Service URL | `http://localhost:8000` |

Save. This automatically creates a CNAME DNS record pointing `api.hvacvibe.ca` → the tunnel.

### 2.3 Smoke test

From any browser or terminal:

```bash
curl https://api.hvacvibe.ca/docs
```

FastAPI Swagger UI should load, confirming end-to-end connectivity.

---

## Part 4 — Troubleshooting

This section documents every issue encountered during setup and how it was resolved.

---

### Issue 1: `cloudflared tunnel login` callback never completes

**Symptom:**
```text
INF Waiting for login...
INF Waiting for login...
^C
ls ~/.cloudflared/   # empty — no cert.pem generated
```

**Cause:** The browser auth flow works by Cloudflare sending a credential back to a callback URL that `cloudflared` is listening on. If the Pi is behind NAT and the callback URL isn't reachable from Cloudflare's servers, the cert is never delivered.

**Fix:** Skip the CLI login flow entirely. Use the **token-based setup** from the dashboard instead:

```bash
sudo cloudflared service install <token-from-dashboard>
```

No `cert.pem` or `~/.cloudflared/` credentials are needed with the token approach.

---

### Issue 2: Service fails immediately after install

**Symptom:**
```ini
systemctl [start cloudflared.service] returned with error code exit status 1
Job for cloudflared.service failed because the control process exited with error code.
```

**Diagnosis:**
```bash
journalctl -xeu cloudflared.service --no-pager | tail -30
```

**Root cause:** The tunnel had no Public Hostname configured yet. `cloudflared` connected to Cloudflare's network but received zero edge addresses because there were no routes defined.

```text
there are no free edge addresses left to resolve to
```

**Fix:** Add a Public Hostname in the dashboard (Part 3 above) before or immediately after installing the service.

---

### Issue 3: DNS resolution of `argotunnel.com` fails

**Symptom in journal:**
```ini
ERR Failed to initialize DNS local resolver
    error="lookup region1.v2.argotunnel.com: operation was canceled"
ERR initial tunnel connection failed
    error="there are no free edge addresses left to resolve to"
You requested 4 HA connections but I can give you at most 0.
```

**Diagnosis:**
```bash
# Test with explicit DNS server (works fine)
dig region1.v2.argotunnel.com @1.1.1.1
# Returns 10 clean A records

# Test with system DNS (router)
dig region1.v2.argotunnel.com
# Warning: Message parser reports malformed message packet.
```

**Root cause:** The home router (`192.168.1.1`) returns a **malformed DNS response** for `argotunnel.com`. The system resolv.conf was pointing at the router. `cloudflared` uses the system DNS internally for its initial edge lookup, gets a corrupted packet, can't parse any IPs, and reports "no free edge addresses."

```bash
cat /etc/resolv.conf
# nameserver 192.168.1.1   ← router returns malformed packets
```

**Fix:** Override system DNS via NetworkManager to use Cloudflare's resolver:

```bash
# Check active connection name
nmcli con show --active

# Set DNS override (use your actual connection name)
sudo nmcli con mod "home-wifi" ipv4.dns "1.1.1.1,8.8.8.8"
sudo nmcli con mod "home-wifi" ipv4.ignore-auto-dns yes

# Apply without dropping WiFi
sudo systemctl restart NetworkManager

# Verify
cat /etc/resolv.conf
# Should show: nameserver 1.1.1.1
```

**Verify fix:**
```bash
dig region1.v2.argotunnel.com
# Should return 10 clean A records with no malformed packet warning
```

---

### Issue 4: QUIC protocol blocked by router

**Symptom:** Even after DNS fix, tunnel still fails with same edge address error.

**Diagnosis:** `cloudflared` defaults to QUIC (UDP port 7844) which some home routers block.

**Fix:** Force TCP (http2) transport:

```bash
sudo tee /etc/systemd/system/cloudflared.service.d/override.conf << 'EOF'
[Service]
Environment="TUNNEL_DNS_UPSTREAM=https://1.1.1.1/dns-query,https://8.8.8.8/dns-query"
Environment="TUNNEL_TRANSPORT_PROTOCOL=http2"
EOF

sudo systemctl daemon-reload
sudo systemctl restart cloudflared
```

Confirm in journal:
```text
INF Initial protocol http2   ← was "quic" before
INF Connection established connIndex=0
```

---

### Issue 5: WiFi jumps to wrong access point

**Symptom:** After restarting NetworkManager, the Pi connected to `hvac-setup-dlink-7C7A` (the HVAC-Vibe setup AP) instead of the home WiFi.

**Cause:** The `hvac-setup` service was running with `hostapd`/`dnsmasq`, broadcasting its own AP. NetworkManager picked it up automatically.

**Fix:** Stop setup services first, then reconnect to home WiFi:

```bash
sudo systemctl stop hvac-setup
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
sudo nmcli con up "home-wifi"
```

---

### Issue 6: ISP/DSL outage during setup

**Symptom:** Laptop lost internet connectivity while Pi changes were being made, causing concern that the Pi's nmcli commands broke the home network.

**Root cause:** Coincidental DSL line drop — the router's WAN status showed "Internet is Offline" with DSL in "EstablishingLink" state. Router admin page at `192.168.1.1` was still reachable, confirming local WiFi was unaffected.

**Diagnosis:** `nmcli con mod` only modifies the Pi's network profile. It cannot affect DSL line synchronization between the router and ISP.

**Fix:** Power cycle the router/modem. Wait 3–5 minutes for DSL to re-establish. Internet returned without any intervention on the Pi.

---

### Issue 7: `systemd-resolved` not available

**Symptom:**
```bash
sudo systemctl restart systemd-resolved
Failed to restart systemd-resolved.service: Unit systemd-resolved.service not found.
```

**Cause:** This Ubuntu install on the Pi does not include `systemd-resolved`.

**Fix:** Use NetworkManager directly (see Issue 3 fix above) instead of `/etc/systemd/resolved.conf`.

---

### Issue 8: `cloudflared tunnel login` browser tab still open

**Symptom:** An old browser tab from the failed `tunnel login` attempt showed "Authorize Cloudflare Tunnel" with `hvacvibe.ca` listed.

**Resolution:** This tab is harmless — close it. It's the stale OAuth flow from the CLI login attempt. The token-based service install does not use or require this authorization.

---

## Part 5 — Key Commands Reference

```bash
# Service management
sudo systemctl status cloudflared
sudo systemctl restart cloudflared
sudo systemctl stop cloudflared
journalctl -fu cloudflared.service        # live log
journalctl -xeu cloudflared.service --no-pager | tail -50  # recent log

# DNS verification
dig region1.v2.argotunnel.com             # should return clean A records
dig region1.v2.argotunnel.com @1.1.1.1   # test against specific resolver
cat /etc/resolv.conf                       # check current DNS server

# Network
nmcli con show --active                   # current connections
nmcli con show "home-wifi" | grep -i dns  # DNS settings on profile
sudo nmcli con up "home-wifi"             # reconnect to home WiFi

# Cloudflared override config
cat /etc/systemd/system/cloudflared.service.d/override.conf

# Full reinstall sequence
sudo cloudflared service uninstall
sudo cloudflared service install <token>
sudo systemctl start cloudflared
```

---

## Part 6 — Architecture Notes

### Why token-based install instead of `cloudflared tunnel login`

The CLI login flow requires Cloudflare to deliver a credential back to the Pi via a callback URL. Behind NAT this callback is unreachable. The token-based approach embeds credentials in the systemd service unit — no inbound connectivity needed at any point.

### Why DNS matters for cloudflared

`cloudflared` performs its own internal DNS resolution to find Cloudflare's edge IPs (`region1.v2.argotunnel.com` etc.) before establishing the tunnel. If system DNS returns malformed responses, `cloudflared` gets zero usable IPs and reports "no free edge addresses" — which looks like a tunnel or auth problem but is actually a DNS problem.

### Why QUIC may fail on home networks

Cloudflare Tunnel defaults to QUIC (UDP port 7844) for performance. Many home routers block non-standard UDP or perform deep packet inspection that corrupts QUIC. Forcing `http2` uses standard TCP 443 which passes through any router that allows HTTPS.

### DNS change persistence note

The `ipv4.ignore-auto-dns yes` setting on the NetworkManager profile persists across reboots. If the Pi ever switches networks (e.g., to a different WiFi), the static DNS `1.1.1.1` will still be used — which is generally desirable for `cloudflared` reliability.

---

## Part 7 — Next Steps

```text
✅ Domain registered   — hvacvibe.ca
✅ Cloudflare Tunnel   — Healthy, YYZ edge, linux_arm64
✅ FastAPI exposed     — https://api.hvacvibe.ca/docs
⬜ WebSocket endpoint  — /ws/{sensor_id} for live data push
⬜ Hugo dashboard      — Cloudflare Pages deployment
⬜ Wire dashboard      — wss://api.hvacvibe.ca/ws to frontend charts
```

Remaining work: add a FastAPI WebSocket endpoint, build the Hugo dashboard on Cloudflare Pages, and connect the two with a live sensor data stream.
