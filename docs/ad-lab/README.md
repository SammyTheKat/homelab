# Active Directory test lab

## Problem

The DFW sysadmin market scan flagged hands-on Active Directory as the
experience gap: AD concepts are familiar (GPOs, password resets), but there
is no lived experience running a domain. The goal is a working AD
environment at home — domain controller, real GPOs, user provisioning —
plus a portfolio write-up that proves it. The lab then doubles as a
practice range for the scenarios that come up in day-to-day admin work:
account lockouts, GPO troubleshooting, user provisioning and
deprovisioning.

## Design decisions

- **Hardware:** spare Dell OptiPlex 7000 SFF — 20GB RAM (16GB Samsung
  DDR4-3200 + 4GB DDR4-2400, runs at 2400), one 256GB M.2 SSD with an open
  second M.2 slot (spare 500GB and 256GB M.2 SSDs on hand). No RAM surgery
  on the OPNsense box needed.
- **Placement:** vertical 3D-printed stand next to the OPNsense box;
  monitor + keyboard attached for initial setup, then it runs headless.
- **Hypervisor:** Proxmox VE on the 7000.
- **Isolation:** the lab lives on its own virtual subnet, `10.20.30.0/24`,
  NATed behind the host. The production LAN (`192.168.4.0/24`) is untouched —
  a misconfigured GPO in the lab cannot touch the real network.
- **Management:** Proxmox web UI and VMs reachable over the existing
  WireGuard tunnel; no new firewall rules on OPNsense.
- **Domain controller:** Windows Server 2022 (ISO already in the library:
  `Storage/Programs/ISO_Tools`). Client VMs for join/drill testing.
- **Docs:** written live during the build — no reconstructed-from-memory
  steps.

See [network.md](network.md) for the lab topology.

## Pre-build downloads

- [x] **Proxmox VE 9.2-1 ISO** — downloaded Oct 7 (**x86_64**, 1.71 GB;
  SHA256 `4e88fe416df9b527624a175f24c9aa07c714d3332afb1ee3dbf3879573ef2c6c`
  verified). Flashed to USB with Rufus in DD mode (same lesson as the
  replica build); the installer stick verified good Oct 8.
- [x] **Windows Server 2022 eval ISO** — downloaded from the Microsoft Eval
  Center on Oct 7.
- [x] **Windows 11 Enterprise eval ISO** — downloaded from the Microsoft
  Eval Center.
- [x] **virtio-win stable ISO** (0.1.302) — Windows has no inbox drivers for
  Proxmox's VirtIO disk/network, so the Server 2022 installer won't see a
  disk without it. Same ISO covers the Windows 11 client (plus the
  qemu-guest-agent for clean shutdowns from Proxmox).

All three ISOs were uploaded to the Proxmox node's `local` storage through
the web UI on build day. The original plan was an SMB/CIFS share off the
TrueNAS ISO library; dropped on the day — simpler to push three files over
gigabit than to dig up SMB credentials. The flaky ISO-storage USB stick
stays retired.

## Build log

Friday, October 9, 2026 — bare metal to working domain in about four hours.

- **Proxmox VE 9.2 install** (graphical installer, monitor + keyboard
  attached): hostname `pve-adlab.home.arpa`, management IP
  `192.168.4.50/24`, gateway and DNS `192.168.4.1`. Verified `.50` was
  free first — ping from the Aquarium came back unreachable and it was
  absent from an Advanced IP Scanner sweep. Target disk confirmed as the
  256GB M.2 (not the USB stick) before hitting Install. Post-install:
  dismissed the no-subscription nag, disabled the `pve-enterprise` repo,
  enabled `pve-no-subscription`.
- **Lab bridge:** created `vmbr1` — `10.20.30.1/24`, autostart on,
  **bridge ports left empty** (no physical NIC; this is the isolation).
  NAT via iptables `MASQUERADE` of `10.20.30.0/24` out `vmbr0`,
  persisted with `post-up`/`post-down` lines in `/etc/network/interfaces`
  (they must be tab-indented under the `vmbr1` stanza or ifupdown ignores
  them) plus `net.ipv4.ip_forward=1`. Applied with `ifreload -a`;
  verified with `iptables -t nat -L POSTROUTING`.
- **dc01 VM:** 2 cores, 4096 MiB RAM, 80GB thin-provisioned disk
  (discard on, VirtIO SCSI single), OVMF/q35, NIC on **vmbr1** with the
  VirtIO model. Both ISOs attached via the wizard's "additional drive
  for VirtIO drivers" option. At the Windows disk screen: no drives
  visible (expected) → Load Driver → `viostor\2k22\amd64` from the
  virtio CD → the 80GB disk appeared. Installed **Server 2022 Standard
  (Desktop Experience)**.
- **DC prep:** installed the `netkvm` network driver
  (`netkvm\2k22\amd64`) and `qemu-ga-x86_64.msi` from the virtio CD.
  Static IP `10.20.30.10/24`, gateway `10.20.30.1`, DNS `127.0.0.1`.
  Renamed to `DC01`, rebooted.
- **Promotion:** added the Active Directory Domain Services role (pulled
  in DNS Server and the RSAT tools automatically). Promoted to DC of a
  **new forest, `ad.lab`** (NetBIOS `AD`, 2016 functional levels, DSRM
  password set; the DNS-delegation warning is expected with no parent
  zone). Added DNS forwarders `1.1.1.1` / `8.8.8.8` so the lab resolves
  internet names through the NAT. `nslookup ad.lab` → `10.20.30.10`.
  First `dcdiag /q` showed first-boot service noise (SystemLog,
  DFSREvent); re-ran after 15 minutes — SystemLog clean, only aging
  promotion-time DFSREvent entries left, which clear on their own.
- **client01 VM:** Windows 11 Enterprise, 4 cores, 8192 MiB RAM, 80GB
  thin disk, NIC on vmbr1, virtio ISO attached. Disk driver at install:
  `viostor\w11\amd64`. OOBE has no DHCP to talk to on vmbr1, so
  bypassed the network requirement (`oobe\bypassnro`), created a local
  `labadmin` account (kept as break-glass), declined all privacy
  toggles. Installed netkvm + guest agent, static IP `10.20.30.11/24`,
  gateway `10.20.30.1`, DNS **`10.20.30.10`** (the DC — domain join
  fails without this). Renamed to `CLIENT01`, rebooted, joined
  `ad.lab` ("Welcome to the ad.lab domain"), rebooted, logged in as
  `AD\Administrator` — first domain login proved the full chain
  (DNS → Kerberos → DC).
- **Move to final spot:** shut down both VMs, then the node; relocated
  the 7000 to the vertical 3D-printed stand next to the OPNsense box,
  disconnected monitor/keyboard, booted headless. Managed entirely via
  `192.168.4.50:8006` from the Aquarium from that point on.
- **OU design and test users:** created `Sales`, `IT`, and
  `Disabled Users` OUs under `ad.lab`; test users `sarah` and `mike`
  in Sales, `alex` in IT. Created in ADUC, verified with
  `Get-ADUser -Filter *` — the same cmdlet family the scripting drills
  use.

Saturday, October 10, 2026 — drills 1–6 completed (each result logged
in [interview-drills.md](interview-drills.md)).

- **RDP management path:** the noVNC clipboard never worked, so Remote
  Desktop was enabled on DC01 and a persistent route added on the
  workstation (`route add 10.20.30.0 mask 255.255.255.0 192.168.4.50 -p`,
  run elevated). `mstsc` to `10.20.30.10` now gives a native clipboard
  for PowerShell work. The route is scoped to the lab subnet on one
  machine — LAN and internet traffic are untouched. Standard pattern:
  one admin workstation with a route to the lab subnet.

## What broke

- **DC01 wouldn't shut down from Proxmox** — "vm quit/powerdown failed,
  got timeout." Shut down cleanly from inside Windows (Start → Power)
  instead. Lesson: when the Proxmox Shutdown button times out, go in
  through the console — don't fight the hypervisor.
- **noVNC clipboard button did nothing** — the slide-out toolbar's
  clipboard icon never opened its text box. Worked around by typing short
  paths by hand until RDP was set up (see the Oct 10 build log entry):
  Remote Desktop on DC01 plus a persistent static route on the
  workstation gives full clipboard via mstsc.

## Drills

After the build, the lab becomes the practice range — see
[interview-drills.md](interview-drills.md): locked-out user, GPO not
applying, new-hire provisioning, deprovisioning, drive mappings, PowerShell
AD queries.
