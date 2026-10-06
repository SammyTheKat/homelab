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

**The WireGuard outage that ended in a full rebuild.** After an OPNsense update, WireGuard peers suddenly couldn't connect — no remote access at all. Instead of rolling back the update, I started troubleshooting by deleting the WireGuard configuration and re-adding it from scratch… and couldn't get it working again. At that point I decided: I built this once, I can build it again. I wiped and rebuilt the entire OPNsense box from the ground up in about an hour or two — and it worked.

Two lessons from that day:

1. **Rebuilding from scratch is a skill.** Doing the full router build a second time, from memory and in under two hours, proved I actually understood the configuration instead of having followed a guide once. That's worth more than the original build.
2. **Change management on critical infrastructure.** I work from home 50% of the time — when the router is down, I'm down. Deleting a working config to troubleshoot was the wrong first move; rolling back the update would have restored service in minutes. Now the rule is: back up the OPNsense configuration before any update, and rollback is always step one. (TODO: confirm config backups are automated.)

TODO: more stories as they happen — e.g. a peer that won't handshake, roaming quirks on the handhelds.

## TODO

- [ ] Document peer provisioning steps (how a new device gets added)
- [ ] Note OPNsense WireGuard plugin version and any custom firewall rules
- [ ] Redacted example peer config (no private keys)
