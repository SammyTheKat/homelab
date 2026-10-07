# Media automation pipeline (*arr stack)

## Problem

Wanted a self-running media library: request → find → download → organize → serve, with minimal manual intervention.

## The pipeline

```mermaid
flowchart LR
    Seerr["Seerr<br/>(requests)"] --> Sonarr["Sonarr / Radarr"]
    Sonarr --> Jackett["Jackett + FlareSolverr<br/>(indexers)"]
    Jackett --> qbit["qBittorrent<br/>(via VPN tunnel)"]
    qbit --> Sonarr
    Sonarr --> Jellyfin["Jellyfin<br/>(serve)"]
    Jellyfin --> Jellystat["Jellystat<br/>(stats)"]
```

- **Seerr**: friends and family request movies/shows through a simple UI.
- **Sonarr/Radarr**: track wanted content, talk to indexers, hand off to the downloader, then rename and file everything into the library.
- **Jackett + FlareSolverr**: aggregate torrent indexers; FlareSolverr solves the Cloudflare challenges that would otherwise block indexer queries.
- **qBittorrent**: downloads, routed through a SOCKS5 proxy — if the proxy is unreachable, transfers fail closed rather than leaking.
- **Jellyfin**: serves the finished library; **Jellystat** tracks watch stats; **Tunarr** builds live-TV channels from the library.

Supporting cast: Jdownloader2 and PlexRipper for one-off grabs, MeTube for YouTube downloads, Audiobookshelf for audiobooks, RomM for retro games, Immich for photo backup.

## Storage layout

The pipeline is split across pools by speed tier:

| Pool | Holds | Why |
|---|---|---|
| AppsNVME (NVMe) | All app configs and working data (Handbrake, makemkv, Minecraft, spoolman datasets live here) | Containers do lots of small I/O; NVMe keeps them snappy |
| NVMEStorage (NVMe) | Reworks staging | Fast scratch space |
| Pool1 (HDD) | `Media` dataset — the finished library Jellyfin serves | Bulk sequential reads; HDDs are fine and cheap per TB |
| Pool2 (HDD) | `Storage` dataset — general storage | Bulk storage |

Downloads land on fast storage, the *arr apps process and rename, and the finished library lives on the HDD pools. Jellyfin only ever reads the finished library — a failed download never corrupts what's being watched.

## Design decisions

| Decision | Why |
|---|---|
| Split download vs. serve | The *arr apps manage files; Jellyfin only reads the finished library — a failed download never corrupts what's being watched |
| FlareSolverr alongside Jackett | Some indexers sit behind CAPTCHAs that silently break searches; FlareSolverr solves them, which fixed the mysteriously empty search results |
| SOCKS5-proxied download client | Download traffic segmented from the home network; configured to fail closed if the proxy drops |

## What broke / lessons learned

**RomM's bad update.** A RomM update shipped with bugs that crashed the instance. Rather than debugging someone else's broken release at midnight, I rolled back to the previous working version and waited for the next weekly update, which fixed it. Lesson: with apps that update weekly, the rollback button is a feature — pin what works, and let the upstream fix their regression.

**RomM 5.3.1: the breaking config change.** The next RomM incident was different — not a bug, a breaking change. 5.3.1 removed the `filesystem.roms_folder` config key and hard-failed on startup (`Invalid config.yml`), with the migration assuming you can hand-edit the file. For TrueNAS catalog installs the config lives in system-managed storage with no UI file editor, so the only options were unsupported shell surgery or rollback. I rolled back to 5.2.0 (rolling back the app snapshots too, since 5.3.1 had started database migrations before dying) and filed it upstream (rommapp/romm#5177) — turns out an Unraid user had hit the same wall and the team had already merged a partial fix. Lesson: know which kind of broken you're looking at. Buggy release → roll back and wait. Breaking change → roll back, then go participate: file the issue, because appliance-style users are a constituency worth representing.

**250 GB of stale Docker images.** The apps NVMe pool started running low on space. Investigation showed over 250 GB of old, unused Docker images had accumulated from months of app updates. Ran an image prune from the TrueNAS shell — with the critical precaution of making sure every app was *running* first, so the prune couldn't delete the image behind a live container. Reclaimed the space with zero downtime. Lesson: container updates don't clean up after themselves; image hygiene is a recurring ops task, not a one-time fix.

More stories will land here as they happen — FlareSolverr/Cloudflare fights, Jackett indexer outages, container-vs-dataset permission battles.

## Still to document

- [ ] Sanitized docker-compose / TrueNAS app configs → `configs/`
- [ ] Dataset layout (where downloads land vs. where the library lives)
- [ ] Note which services are exposed to friends vs. LAN-only
