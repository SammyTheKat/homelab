# Sharing Jellyfin with friends over a static IP

## Problem

Friends wanted access to the Jellyfin library without installing VPN clients or dealing with extra setup.

## Design

- Static IP from the ISP → single OPNsense port forward → Jellyfin.
- Each friend gets their own Jellyfin account, created manually — no open registration.
- Passwords are set by the admin and handed over **in person** — nothing sensitive ever crosses email or chat.
- Admin keeps full LAN access over WireGuard instead of the public port.

## Trade-offs

| Choice | Pro | Con |
|---|---|---|
| Port forward vs. VPN for friends | Zero setup on their end | One public port to watch |
| Port forward vs. reverse proxy + auth | Simpler, fewer moving parts | No SSO / extra auth layer in front |

## What broke / lessons learned

**ErsatzTV melted the 9020.** On my second TrueNAS build (the OptiPlex 9020, now the replication target), I ran ErsatzTV in Docker to fake live TV channels and feed them into Jellyfin. One viewer: fine. The moment a second person tuned in, the box fell over — ErsatzTV stitches commercials into the stream, which forces a transcode per viewer, and the 9020's Quick Sync couldn't keep up with two of those at once. That was the moment I stopped trying to make the old hardware work and built the current server around the i7-12700K's UHD 770. Sometimes the fix is just more Quick Sync.

**Friends and family are the real monitoring system.** Sharing Jellyfin with actual humans means being tech support: forgotten passwords and usernames get sorted out in person. Accounts are created by hand and credentials delivered face-to-face — no self-service signup, which is a security feature, not a limitation.

## Notes

- External port intentionally omitted from this repo — no reason to publish it
- **Current posture on that forward:** Jellyfin's own authentication plus hand-created accounts (no self-service signup). No IDS/IPS or rate limiting on the rule today — that's on the roadmap. The OPNsense box has the headroom for Suricata when I get to it, and this doc will get updated when it lands.
