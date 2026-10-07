# War stories

Operating notes from things that broke and what fixed them. The goal is
honest incident documentation: what happened, what was tried, what the
root cause was, and what changed so it doesn't recur.

## 2026-10-06 — TrueNAS SCALE installer USB wouldn't boot in UEFI mode

**Setup:** Dell OptiPlex 9020 mini tower, fresh TrueNAS SCALE install.
Installer ISO written to a flash drive with Rufus in GPT mode.

**Symptom:** Booting the UEFI USB entry dropped to a GRUB prompt with an
"unknown filesystem" error.

**Root cause:** Rufus wrote the ISO in "ISO Image mode", which rearranges
the filesystem layout of TrueNAS installer images. GRUB in UEFI mode
couldn't parse it. This is a known Rufus/TrueNAS quirk.

**Fix:** Re-flashed the same stick in **DD Image mode** (raw byte-for-byte
write). UEFI boot entry worked immediately.

**Lesson:** For TrueNAS install media, always pick DD Image mode when
Rufus asks. Also: set the Dell BIOS to UEFI and disable Secure Boot
before booting — TrueNAS SCALE doesn't do Secure Boot.

---

## 2026-10-06 — Static IP change on the replica box failed with a middleware error

**Setup:** Newly installed TrueNAS replica node, reachable via DHCP at
`192.168.4.63`. Changing it to static `192.168.4.123/24` via the web UI.

**Symptom:** The "Register Default Gateway" dialog asked for the gateway.
After entering values, the save failed with:

```
[EFAULT] '192.168.4.1' is not reachable from any interface on the system.
```

**Root cause (layered):**
1. The gateway field was initially filled with the box's own IP
   (`192.168.4.123`) — a machine can't be its own gateway. The correct
   value was the router: `192.168.4.1`.
2. Even with the correct value, the error persisted because the earlier
   failed edit had rolled back the interface config — the netmask wasn't
   applied, so the box believed `192.168.4.1` was on a different network.

**Fix:** Re-applied the interface edit cleanly with the netmask explicitly
set to `/24`, confirmed the gateway was already `192.168.4.1`, skipped the
gateway dialog, and confirmed the 60-second test change at
`http://192.168.4.123`.

**Lesson:** When TrueNAS says the gateway is unreachable, check the
interface netmask before the gateway value — the error usually means the
box can't see the gateway's subnet at all, which is a mask problem, not a
gateway problem.

---

## 2026-10-07 — Replica box unreachable over WireGuard: the static gateway never took effect

**Setup:** Dell OptiPlex 9020 replica node, TrueNAS SCALE 25.04.2.6, static
`192.168.4.123/24`. Same-subnet replication from the primary (`.122` →
`.123`) was working fine.

**Symptom:** The replica was unreachable from WireGuard clients (phone,
laptop) — no ping, no web UI — while every LAN device reached it without
issue.

**Root cause:** The box had no default route at all. `ip route` showed only
the link route (`192.168.4.0/24 dev eno1 ... src 192.168.4.123`) — the
`192.168.4.1` gateway from the previous day's static-IP config never
actually took effect. LAN traffic never needs a gateway, and replication is
`.122` → `.123` on the same subnet, so nothing noticed. WireGuard clients
live on the tunnel subnet, so replies from `.123` had to be routed back
through OPNsense — and with no default route, they went nowhere.

**Fix:** Immediate test with `sudo ip route add default via 192.168.4.1`
(WireGuard reachability confirmed instantly), then made permanent in the
TrueNAS UI: **Network → Interfaces → edit eno1 → IPv4 Default Gateway →
`192.168.4.1`**, plus the nameserver in **Network → Global Configuration →
Nameservers → `192.168.4.1`** (OPNsense runs the LAN's DNS resolver). The
CLI fix doesn't survive a reboot; the UI one does.

**Lesson:** "Same subnet works, cross-subnet doesn't" is a gateway problem
until proven otherwise. This one would also have silently broken
replication failure alerts later — alert emails need the gateway to leave
the box. Verify static network configs with `ip route`, not just the UI
form.

---

## 2026-10-07 — Primary server exhaust fan dying: SYS_FAN2 low-RPM warnings in IPMI

**Setup:** Primary TrueNAS box — Supermicro X13SAE-F, i7-12700K, TrueNAS
SCALE 25.04.2.6. Headless, in a home office.

**Symptom:** The IPMI event log showed SYS_FAN2 dipping to 280–420 RPM
several times on Sept 29 (Lower Critical threshold 420, Lower
Non-recoverable 280), recovering each time — plus a faint vibration from
the chassis.

**Root cause:** A dusty exhaust fan with a failing bearing. The IPMI events
had been warning about it for over a week. Cleaning the chassis properly
(stop apps, shut down from the UI, unplug the PSU, hold each fan still
while blowing) cleared the dust, but the fan started vibrating afterward —
a dying bearing doesn't get better with cleaning.

**Fix:** Ordered an ARCTIC P12 Pro PST LN 120mm PWM as the replacement.
Swap plan for when it arrives: stop apps and shut down from the UI, unplug,
replace the fan on the SYS_FAN2 header (airflow arrow pointing out the rear
— it's the exhaust), power on, verify pools ONLINE and apps started, then
watch the IPMI event log for a week for any new SYS_FAN2 low-RPM events.

**Lesson:** IPMI fan-threshold events are an early warning, not noise. A
fan that repeatedly dips and recovers is telling you its bearing is going —
clean first, but if the noise changes character after cleaning, order the
replacement instead of hoping.
