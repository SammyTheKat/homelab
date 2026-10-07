# Replication build runbook

Goal: second TrueNAS box replicating the primary's datasets on a schedule, with a tested restore.

**Target hardware:** Dell OptiPlex 9020 mini tower — chosen over the HP 580-023W (which turned up a bad DDR4 stick/slot during testing, and only has room for one HDD + boot drive).

**Drives:** boot SSD + 2× 2 TB HDDs in a stripe (~3.6 TB usable). No mirror on the replica — the primary is the redundant copy; if a replica drive dies, re-replicate.

**Capacity note:** primary holds ~2.9 TB today (Media 1.96 TiB + Storage ~0.94 TiB). It fits, with modest headroom. Priority order if space ever gets tight: `Pool2/Storage` first (irreplaceable: photos, documents, ROMs), `Pool1/Media` second (re-fetchable via the *arr pipeline).

## Phase 1 — Build the replica

- [x] Check the ISO library first: `Pool2/Storage/Programs/ISO_Tools` — verified TrueNAS 25.04.2.6
- [x] Flash to USB with Rufus (DD mode — ISO mode fails UEFI boot with GRUB "unknown filesystem")
- [x] Install TrueNAS SCALE 25.04.2.6 on the 9020 (switched Dell BIOS from Legacy to UEFI; disabled Secure Boot)
- [x] Set hostname and static IP `192.168.4.123` (via web UI — console menu lacked a static option; gateway popup: keep existing `192.168.4.1`, don't retype)
- [x] Create a pool on the 2× 2 TB HDDs — `replica`, stripe, 3.51 TiB usable

Build notes: the HP 580-023W failed to POST (bad DDR4 stick/slot) and only fits one HDD + boot drive, so the OptiPlex 9020 mini tower was chosen instead.

## Phase 2 — Snapshots on the primary

- [x] Periodic snapshot task for `Pool1/Media` — daily midnight, recursive, 2-week retention, schema `auto-%Y-%m-%d_%H-%M`
- [x] Periodic snapshot task for `Pool2/Storage` — daily 1 AM, recursive, 2-week retention, same schema
- [x] Manual seed snapshots `auto-2026-10-06_10-45` taken on both datasets to start replication immediately

Lesson: snapshot tasks must target the same datasets the replication sources use (`Pool1/Media`, not the `Pool1` root), with Recursive checked.

## Phase 3 — Replication task

- [x] SSH connection `truenas-replica` → 192.168.4.123 (semi-automatic setup)
- [x] Replication task `Pool1,Pool2 - replica`: PUSH over SSH+NETCAT, no transfer encryption (LAN), sources `Pool1/Media` + `Pool2/Storage` recursive, destination `replica` pool
- [ ] First run completed (in progress as of 2026-10-06 ~10:50 CDT — ~2.9 TB initial seed)
- [ ] Confirm the datasets + snapshots exist on the replica

Lesson: the snapshot naming schema on the replication task must match the periodic task's schema (`auto-%Y-%m-%d_%H-%M`). A leftover custom regex (`replica-seed`) caused the first run to match zero snapshots and "succeed" instantly with nothing transferred. Cleared the regex and linked both periodic tasks instead.

## Phase 4 — Prove it works ✅ (2026-10-06)

- [x] **Restore test:** on the replica, cloned `replica/Pool2/Storage@auto-2026-10-06_10-45` to `replica/restore-test-2026-10-06`, recovered `Work/job.txt`, and `sha256sum` matched the primary's copy exactly (`71087593…b9a5517` both ends). Backup proven real, not a rumor. Clone destroyed after the test.
- [x] Record the date and result below

## Phase 5 — Ongoing

- [x] Alerting: email alerts configured on the primary (`.122`) via Gmail SMTP — Alert Service (E-Mail, Warning level) + system Email Options. Test mail received 2026-10-07. Covers replication failures plus drive/pool/scrub alerts. (Setup notes: Gmail OAuth is available as an alternative to app passwords; the admin user needs an email set under Credentials → Local Users; port/security must match — 587/TLS or 465/SSL, not Plain.)
- [x] Cold HDD copies: **quarterly refresh** (decided 2026-10-07). 4× 1TB drives hold the full ~2.9TB set; first pass is a full copy, then rsync incrementals. Verify every refresh (checksums/file counts). Store disconnected, ideally off-site. Rationale: media barely churns, the replica covers hardware failure, and quarterly is a cadence that'll actually happen.

## Build log

| Date | What happened |
|---|---|
| 2026-10-06 | Pool1/Media (~1.97T) and Pool2/Storage (~960G) seeds completed to the 9020 replica. Restore test passed: cloned snapshot, recovered Work/job.txt, sha256sums matched primary. Phase 4 complete. |
| 2026-10-07 | First scheduled overnight run succeeded — task finished against the auto-2026-10-07 snapshots. Snapshot → replicate loop now proven end-to-end on the schedule. |

## Notes / things that broke

| Date | Issue | Fix |
|---|---|---|
| | | |
