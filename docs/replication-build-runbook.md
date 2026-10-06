# Replication build runbook

Goal: second TrueNAS box replicating the primary's datasets on a schedule, with a tested restore.

**Target hardware:** Dell OptiPlex 9020 mini tower — chosen over the HP 580-023W (which turned up a bad DDR4 stick/slot during testing, and only has room for one HDD + boot drive).

**Drives:** boot SSD + 2× 2 TB HDDs in a stripe (~3.6 TB usable). No mirror on the replica — the primary is the redundant copy; if a replica drive dies, re-replicate.

**Capacity note:** primary holds ~2.9 TB today (Media 1.96 TiB + Storage ~0.94 TiB). It fits, with modest headroom. Priority order if space ever gets tight: `Pool2/Storage` first (irreplaceable: photos, documents, ROMs), `Pool1/Media` second (re-fetchable via the *arr pipeline).

## Phase 1 — Build the replica

- [ ] Check the ISO library first: `Pool2/Storage/Programs/ISO_Tools` — verify the TrueNAS ISO version matches the primary (25.04.2.6); re-flash if unsure
- [ ] Flash to USB with Rufus (also in the ISO_Tools folder)
- [ ] Install TrueNAS SCALE on the 9020 (match the primary's version, 25.04.2.6, or newer — ZFS replication wants the target at equal-or-newer feature flags)
- [ ] Set hostname (e.g. `truenas-replica`) and a static IP (e.g. `192.168.4.123`)
- [ ] Create a pool on the 2× 2 TB HDDs (stripe is fine for a replica target — it's a copy, not the primary)

## Phase 2 — Snapshots on the primary

- [ ] Create a periodic snapshot task for `Pool1/Media`
- [ ] Create a periodic snapshot task for `Pool2/Storage`
- [ ] Decide retention (suggestion: daily snapshots kept 2 weeks, weekly kept 2 months — tune to available space)
- [ ] Let one scheduled run complete and confirm snapshots exist

## Phase 3 — Replication task

- [ ] On the primary: Credentials → Backup Credentials → add SSH connection to the replica (semi-automatic setup with root key works LAN-side)
- [ ] Create replication task: **push**, source = `Pool1/Media` (+ `Pool2/Storage`), destination = replica pool
- [ ] Schedule it (suggestion: nightly, after the snapshot task)
- [ ] Run it once manually and watch it complete
- [ ] Confirm the datasets + snapshots exist on the replica

## Phase 4 — Prove it works

- [ ] **Restore test:** on the replica, clone a snapshot and copy a file out of it. A backup you haven't restored is a rumor.
- [ ] Record the date and result below

## Phase 5 — Ongoing

- [ ] Set up an alert (email/Discord) if a replication task fails
- [ ] Cold HDD copies: decide a refresh cadence

## Build log

| Date | What happened |
|---|---|
| | |

## Notes / things that broke

| Date | Issue | Fix |
|---|---|---|
| | | |
