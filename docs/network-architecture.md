# Network architecture

## Overview

Single public static IP → OPNsense router (Dell OptiPlex small form factor) → home LAN.

Remote access is handled by **WireGuard running directly on OPNsense** — no publicly exposed reverse proxy, which keeps the attack surface to one UDP port. Friends access Jellyfin through a single OPNsense port forward.

## Design decisions

| Decision | Why |
|---|---|
| WireGuard on the router instead of a reverse proxy | Smaller attack surface; no public web dashboard to harden; native OPNsense support with per-peer config |
| Static IP from ISP | Stable endpoint for WireGuard peers and the Jellyfin port forward; no DDNS moving parts |
| qBittorrent routed through TorGuard VPN | Torrent traffic never touches the home IP; ISP sees only encrypted VPN traffic |
| Single Jellyfin port forward | Pragmatic sharing for a handful of friends; one TCP port, not a whole dashboard |

## Traffic flow

- **Road-warrior devices** (laptop, phone, handheld gaming systems): WireGuard tunnel into OPNsense → full LAN access, including TrueNAS apps and OPNsense admin.
- **Friends**: `static-ip:8096` (TODO: confirm port) → OPNsense port forward → Jellyfin. No VPN client needed on their end.
- **Torrents**: qBittorrent → TorGuard tunnel → internet. Bound to the VPN interface so a dropped tunnel kills traffic instead of leaking it (TODO: confirm kill-switch/binding config).

## What broke / lessons learned

TODO: add 2–3 real stories. Good candidates:
- A WireGuard peer that wouldn't handshake (key mismatch? firewall rule order?)
- Jellyfin buffering for a remote friend (transcode settings? upload bandwidth cap?)
- qBittorrent leaking or stalling when the VPN dropped

## Hardening notes

- OPNsense admin UI is **not** exposed to WAN (TODO: confirm).
- Jellyfin has authentication required for all users (TODO: confirm).
- TODO: consider fail2ban or OPNsense intrusion detection for the forwarded port.

## TODO

- [ ] Fill in LAN subnet(s), VLANs if any
- [ ] Confirm Jellyfin external port
- [ ] Confirm qBittorrent VPN binding / kill switch behavior
- [ ] Screenshot or export of OPNsense NAT/firewall rules (redacted)
