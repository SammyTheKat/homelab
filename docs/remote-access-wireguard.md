# Remote access with WireGuard on OPNsense

## Problem

Needed secure remote access to the home LAN from a laptop, a phone, and several handheld gaming devices — without exposing management dashboards to the public internet.

## Design

- WireGuard server runs on the OPNsense router itself (Dell OptiPlex SFF).
- Each device gets its own peer config (unique keys, TODO: note allowed-IPs scheme).
- One UDP port forwarded/allowed on WAN; everything else stays closed.

## Why WireGuard over alternatives

- Lower overhead and simpler config than OpenVPN/IPsec.
- Kernel-level on modern systems; painless clients on Android, iOS, Windows, Linux.
- Combined with the paid static IP, peers have a stable endpoint with no DDNS dependency.

## What broke / lessons learned

TODO: add real examples, e.g.:
- Handshake failures and what caused them
- Roaming between Wi-Fi and cellular (WireGuard handles this gracefully — worth noting)
- Any handheld-specific client quirks

## TODO

- [ ] Document peer provisioning steps (how a new device gets added)
- [ ] Note OPNsense WireGuard plugin version and any custom firewall rules
- [ ] Redacted example peer config (no private keys)
