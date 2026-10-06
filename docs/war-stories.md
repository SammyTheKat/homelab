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
