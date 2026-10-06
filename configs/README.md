# configs/

Sanitized service configurations. **No secrets live here.**

## Rules

1. Strip every credential, API key, private key, public IP, DDNS hostname, and WireGuard key before committing.
2. Prefer `.example` files showing structure with `REDACTED` placeholders.
3. If a config can't be sanitized without losing its meaning, document the *shape* of it in the relevant `docs/` write-up instead.

## Planned contents

- [ ] `truenas/` — app configs / compose equivalents (redacted)
- [ ] `opnsense/` — NAT + firewall rule exports (redacted), WireGuard peer template (no keys)
- [ ] `jellyfin/` — server settings notes
