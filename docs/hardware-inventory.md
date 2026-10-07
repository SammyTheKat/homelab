# Hardware inventory

## Router

| Item | Detail |
|---|---|
| Device | Dell OptiPlex 7050 (small form factor) |
| OS | OPNsense |
| NICs | Onboard NIC + extra NIC added via a spare M.2 slot — with a **3D-printed housing** attached to the case to mount it |
| WAN | Fiber ISP via ONT |
| Role | Router, firewall, WireGuard server, port forwarding |

## Primary NAS

| Item | Detail |
|---|---|
| OS | TrueNAS SCALE 25.04.2.6 |
| Motherboard | Supermicro X13SAE-F, rev 1.02 |
| CPU | 12th Gen Intel Core i7-12700K |
| RAM | 63 GiB |
| GPU | Intel UHD 770 integrated graphics (Quick Sync) handle Jellyfin transcoding — efficient for multiple simultaneous streams at low power draw. NVIDIA GTX 1060 6GB installed for local AI experiments (Open WebUI) |
| LAN IP | 192.168.4.122 |

### Storage pools

| Pool | Media | Used / Available | Datasets |
|---|---|---|---|
| AppsNVME | NVMe | 77.4 GiB / 372.27 GiB | App data: Handbrake, makemkv, Minecraft, spoolman |
| NVMEStorage | NVMe | 800 KiB / 228.69 GiB | Reworks |
| Pool1 | HDD | 1.96 TiB / 3.36 TiB | Media (the main library) |
| Pool2 | HDD | 961.16 GiB / 837.09 GiB | Storage |

All datasets unencrypted. Fast NVMe pools hold app data and working sets; spinning disks hold the bulk media library.

## Spare / replication target (planned)

| Item | Detail |
|---|---|
| Device | Dell OptiPlex 9020 mini tower |
| Drives | 256GB boot SSD + 2× 2TB HDD in a stripe (~3.5TiB usable) |
| OS | TrueNAS SCALE (replication target, built Oct 2026) |

## Cold backup

| Item | Detail |
|---|---|
| Media | 4× 1TB + 7× 500GB HDDs on hand for cold copies |
| Contents | Main media collection |
| Cadence | Manual |

## Network

| Item | Detail |
|---|---|
| WAN | Fiber (ONT) → OPNsense; paid static IP |
| LAN subnet | 192.168.4.0/24 (TrueNAS is .122) |
| VLANs | None — flat network |
| Wi-Fi (main) | eero SO10001 in bridge mode (Wi-Fi 6E); separate guest SSID for Google Home / smart bulbs |
| Wi-Fi (legacy) | TP-Link Archer A6 in bridge mode, 2.4 GHz for older IoT devices; security camera wireless receiver hangs off it |
| Switch | Generic unmanaged switch |
| Wired runs | Cat6 drop through the attic to the living room (PS5) |
| VPN | WireGuard on OPNsense; peers: laptop, phone, handheld gaming systems |

### What's on the switch

- Daily-driver PC ("Aquarium", Hyte Y70 case): i9-14900KF, 32GB DDR5, RTX 4070 Ti SUPER 16GB, 1.86TB NVMe, Windows 11 Pro — accesses Pool1/Pool2 over SMB shares; runs Apollo for game streaming
- TrueNAS SCALE machine
- Cat6 run → living room (PS5)
- TP-Link Archer A6 (bridge mode)
- Previously: second OptiPlex 7050 running FieldStation42

## Power

Tripp Lite UPS protecting the OPNsense box, TrueNAS server, switch, and Wi-Fi routers.

## Also in the lab

- **3D printing**: OctoPrint + Manyfold + Spoolman (filament manager) in the app list; the M.2 NIC housing on the router was 3D printed
- **ISO & tools library**: curated collection on Pool2 (`Storage/Programs/ISO_Tools`) — TrueNAS, Bazzite and Ubuntu builds, OPNsense image, Windows Server 2022, plus Advanced IP Scanner, Rufus, and Hiren's BootCD PE
- **Previously experimented with**: FieldStation42 on a second OptiPlex 7050 (virtual TV station, since replaced by Tunarr)
