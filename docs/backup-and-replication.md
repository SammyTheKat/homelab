# Backup & replication strategy

## Current state

- Manual backups of the main media collection to external hard disks.
- Scrutiny monitors drive health on the primary TrueNAS box (early warning for failing disks).

Honest assessment: the manual HDD backups work, but they're not automated, not versioned, and not tested on a schedule. That's the gap this project closes.

## The plan: second TrueNAS replication target

A spare PC and extra drives are available. The build:

1. **Assemble the box** — install TrueNAS on the spare PC (TODO: specs, drive count/sizes).
2. **Snapshot schedule on primary** — e.g. daily snapshots of media datasets, kept 2 weeks; weekly snapshots kept 2 months. (TODO: finalize retention.)
3. **Replication task** — TrueNAS → TrueNAS replication over the LAN via SSH, pushing snapshots to the second box on a schedule.
4. **Restore test** — actually restore a file from a snapshot on the replica. An untested backup is a rumor.
5. **Keep one cold HDD copy** — the existing manual disks become the offline/off-site-ish third copy (3-2-1: 3 copies, 2 media types, 1 offline).

```mermaid
flowchart LR
    Primary["Primary TrueNAS<br/>(snapshots)"] -->|Replication task<br/>(scheduled, SSH)| Replica["Second TrueNAS<br/>(spare PC)"]
    Primary -.->|Manual, periodic| Cold["External HDDs<br/>(cold copy)"]
    Scrutiny["Scrutiny<br/>(drive health)"] -.-> Primary
```

## Design decisions (to finalize during the build)

| Decision | Rationale |
|---|---|
| TrueNAS replication vs. rsync | Block-level, incremental, preserves snapshots — and it's a marketable skill |
| Replica on LAN vs. off-site | LAN is what's available; honest about the trade-off (fire/flood risk remains) — cold HDDs mitigate |
| Snapshot retention policy | Balance drive space against how far back you might need to go |

## Build log

TODO: date-stamped entries as the build happens —
what hardware went in, the exact replication task config, first successful run, first restore test, and anything that broke.

## TODO

- [ ] Spare PC specs + drive inventory
- [ ] Snapshot schedule + retention policy
- [ ] Replication task configuration (screenshots or exported config, redacted)
- [ ] First restore test: date + result
- [ ] Decide cold-HDD refresh cadence
