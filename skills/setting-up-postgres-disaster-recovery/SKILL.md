---
name: setting-up-postgres-disaster-recovery
description: Use when a self-hosted PostgreSQL server has no backups, only disk snapshots, no point-in-time recovery, or backups that have never been restored on another machine; also when running a restore drill or rebuilding a lost Postgres VM from backups.
---

# Setting up Postgres disaster recovery

## Overview

A backup counts only after it has been **restored on a different machine and checked against production**. An alert
counts only after a **deliberate failure reached a human**. The configuration (pgBackRest, WAL archiving, timers) is
well documented and usually gets done right. This skill covers what usually gets missed: the drill, the alert proof,
and the traps that only show up during a real restore.

**Not for** managed Postgres (RDS, Cloud SQL, Neon, ...): use the provider's PITR, but still run the drill against it.

## Target state

| Layer | Protects against | Recovery point |
|---|---|---|
| Disk snapshots of **every** disk (data and boot) | VM or disk loss | ≤ 24 h |
| pgBackRest (or WAL-G) to object storage: WAL archiving with `archive_timeout = 60`, weekly full, daily diff | any bad write: TRUNCATE, DROP, a bad migration | ≈ 1 min, any moment |
| Nightly `pg_dump` per database plus `pg_dumpall --globals-only` | one database or table back; portability | 24 h |
| Alerts: job-failure hook, hourly check, daily heartbeat | any of the above silently stopping | — |

Bucket: another region or dual-region, soft delete, public access prevention, credentials from the instance identity
(no key files).

## Work in gated phases

Ask the owner before each phase and verify it before moving on. Only enabling WAL archiving needs a restart, unless the
VM's access scopes must change: on GCE that needs a VM stop, so fold it into the same window.

1. **Inventory (read-only).** Record:
   - Package versions (`dpkg -l`) and extension versions in every database.
   - OS release and glibc version (`/etc/os-release`, `ldd --version`).
   - `datcollate` / locale provider.
   - Data and config directories, and the disks.
   - Snapshot policies **and whether they have ever produced a snapshot**.
   - Network egress (NAT?) and service-account scopes.
2. **Snapshots.** If nothing exists yet, take an **on-demand snapshot of every disk now**: it's the first backup, and
   it takes minutes. Then attach a schedule to every disk. On Debian the boot disk holds `/etc/postgresql`.
3. **Bucket**, then **pgBackRest** and `stanza-create`.
4. **Archiving restart.**
   - Before restarting, parse the full config without touching the running server:
     `postgres -C archive_mode -c config_file=…` must print `on`, and `-C wal_level` must not be `minimal`. If either
     check fails, remove your file and do not restart.
   - Confirm the app reconnects on its own.
5. **First full backup, timers, logical dumps.**
6. **Restore drill on a separate VM** (pass criteria below).
7. **Alerts**, then a deliberate failure to prove them.
8. **Commit** verbatim copies of every installed file, plus a runbook with the package versions, to a repo. Secrets stay out.

Commands for all of this on GCP/Debian are in `gcp-worked-example.md`. The alert tool is `pg-backup-check`; its
settings are documented in the file header.

## Done means (each one observed, not assumed)

- `pgbackrest check` passes, and archiving isn't failing: `last_failed_time` is null or older than
  `last_archived_time`. `failed_count` is cumulative, so either use this test or run
  `pg_stat_reset_shared('archiver')` once setup is done. The view lags `check` by a moment, so re-read it.
- Snapshots are **listed** for every disk; "policy attached" doesn't count.
- `pg_restore -l` of tonight's dump lists the tables you expect.
- **Drill:**
  - Restored on a separate VM to a chosen time T.
  - Some commit after T is already archived. If not, PG13+ fails with `recovery ended before configured recovery
    target was reached`; `pgbackrest check` forces a WAL switch.
  - The log says `recovery stopping before commit … time …`.
  - The stop point is proven by marker rows: one inserted before T and one after (a tiny owner-approved table), or
    failing that by row counts as of T. Counts alone are weak: deletes and retention jobs after T break them.
  - Time from VM creation to a queryable copy recorded, and the VM deleted.
  - Repeated quarterly and after any change to the backup setup.
- A deliberate failure posted an alert, and the daily heartbeat arrived.
- Runbook and installed files are committed.

## Traps

| Trap | What happens | Do instead |
|---|---|---|
| Old package builds vanish | `Version '18.4-1.pgdg12+1' … was not found`: apt.postgresql.org keeps only the newest build. A physical restore needs the same major version, and TimescaleDB's loader opens `timescaledb-<extversion>.so`, so that version must match exactly | Add `deb … https://apt-archive.postgresql.org/pub/repos/apt <codename>-pgdg-archive main`; Timescale's packagecloud repo keeps old versions; record the versions in the runbook |
| Different OS release on the restore host | A different glibc or ICU changes collation order, so text indexes are silently corrupt after a physical restore | Restore onto the same OS release (glibc / ICU) as prod; otherwise `REINDEX` the text indexes, or use the logical dumps |
| Drill VM has no egress | A VM without an external IP and no Cloud NAT can't reach apt; without Private Google Access it can't reach the bucket | Check for NAT first (`gcloud compute routers nats list`), or give the drill VM an ephemeral IP plus a restrictive SSH tag |
| Target time read in the wrong timezone | `--target` without an offset is read in the server's `timezone` setting, giving the wrong stop point | Always write the offset: `--target="2026-10-01 15:50:00+00"` |
| Restored copy writes into the prod repo | A promoted copy archives its new timeline into the same stanza | Use `--archive-mode=off` **and** read-only bucket credentials (GCE `--scopes storage-ro`) |
| Restored data dir won't start | Debian keeps `postgresql.conf` and `pg_hba.conf` in `/etc`, not in the data dir | Write minimal ones (`local all postgres peer`), or restore `/etc` too |
| "Started" ≠ "recovered" | `pg_ctl -w` / `pg_ctlcluster start` return before recovery ends; with `hot_standby=off`, immediately | Wait for `pg_isready` **and** `pg_is_in_recovery() = f`, then read the log |
| A snapshot schedule that never ran | The policy was attached recently, or only to the data disk | List snapshots for each disk; add the boot disk |
| Archiving failure fills the disk | `pg_wal` grows until Postgres stops | Alert on the age of the oldest `pg_wal/archive_status/*.ready` file ("nothing archived in N min" false-alarms when the database is idle), plus disk %. Let the owner choose `archive-push-queue-max`: it drops WAL (a gap in PITR, prod stays up) instead of filling the disk (an outage) |
| Testing alerts by breaking prod config | You risk the real backup "for one run" | `systemctl start pgbackrest@bogus.service`: pgBackRest rejects `--type=bogus` before touching anything, so the unit fails and `OnFailure=` fires. Afterwards run `systemctl reset-failed pgbackrest@bogus.service` |
| Failure hook on template units | `%n` passes `pgbackrest@full.service` into the handler's instance name | `OnFailure=pg-backup-alert@%N.service` passes `pgbackrest@full` |
| Silence looks healthy | A dead checker or a dead VM sends nothing | Daily all-green heartbeat. Better still, an off-host scheduled job (e.g. Cloud Scheduler with Cloud Run) that alerts when the newest backup or archive object is stale. An idle database archives nothing, so either use a threshold longer than the backup interval, or have a timer write a heartbeat row every few minutes so WAL always flows |
| Ordering against the wrong unit | On Debian, `postgresql.service` is an umbrella unit that doesn't wait for the cluster | `After=postgresql@<ver>-<cluster>.service` |
| Webhook leaks | Committed, passed in command arguments, or in a world-readable env file | Root-only (600) `EnvironmentFile`, which systemd reads as root; send it over SSH stdin |
