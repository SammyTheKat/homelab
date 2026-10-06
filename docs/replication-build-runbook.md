# Replication build runbook

Goal: second TrueNAS box replicating the primary's datasets on a schedule, with a tested restore.

**Target hardware:** HP Pavilion 580-023W — 256 GB SSD (OS), 1 TB HDD + one free SATA port.

## Phase 1 — Build the replica

- [ ] Check the ISO library first: `Pool2/Storage/Programs/ISO_Tools` — verify the TrueNAS ISO version matches the primary (25.04.2.6); re-flash if unsure
- [ ] Flash to USB with Rufus (also in the ISO_Tools folder)
- [ ] Install TrueNAS SCALE on the HP (match the primary's version, 25.04.2.6, or newer — ZFS replication wants the target at equal-or-newer feature flags)
- [ ] Set hostname (e.g. `truenas-replica`) and a static IP (e.g. `192.168.4.123`)
- [ ] Create a pool on the 1 TB HDD (single-disk stripe is fine for a replica target — it's a copy, not the primary)
- [ ] If adding the extra HDD later: note the date it went in here

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
