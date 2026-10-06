# Sharing Jellyfin with friends over a static IP

## Problem

Friends wanted access to the Jellyfin library without installing VPN clients or dealing with extra setup.

## Design

- Paid static IP from the ISP → single OPNsense port forward → Jellyfin.
- Each friend gets their own Jellyfin account (TODO: confirm per-user libraries / restrictions).
- Admin keeps full LAN access over WireGuard instead of the public port.

## Trade-offs

| Choice | Pro | Con |
|---|---|---|
| Port forward vs. VPN for friends | Zero setup on their end | One public port to watch |
| Port forward vs. reverse proxy + auth | Simpler, fewer moving parts | No SSO / extra auth layer in front |

## What broke / lessons learned

TODO: real examples, e.g.:
- Remote transcode performance vs. home upload bandwidth
- A friend's client that wouldn't direct-play
- Any port-scan / unwanted-login-attempt observations

## TODO

- External port intentionally omitted from this repo — no reason to publish it
- [ ] Confirm Jellyfin requires login (no anonymous access)
- [ ] Consider noting OPNsense IDS/IPS or rate limiting on that rule
