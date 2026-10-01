---
name: recovering-lost-postgres-data
description: Use when rows or tables in a production PostgreSQL database were deleted, truncated, dropped or overwritten by mistake (a bad script, migration, test harness or agent), especially when there is no point-in-time recovery and the newest backup is old or missing.
---

# Recovering lost Postgres data

## Overview

Preserve first. Restore into a copy. Rebuild whatever the backup lacks from external systems of record, **using the
original ids**. Switch over deliberately. Every write to production goes through a reviewed script that does a dry run
first and that a human approves: the incident itself was an unreviewed write.

**With point-in-time recovery** this is short: freeze, find the moment, restore a copy, then copy rows back or rewind.
For the restore steps, **REQUIRED SUB-SKILL:** use setting-up-postgres-disaster-recovery. Everything below is for
when PITR doesn't exist.

## Order of work

1. **Freeze and preserve.** Minutes matter.
   - Stop whatever did it, and revoke its credentials. Before rotating a role, check whether the app uses it too.
   - If you hold a privileged credential yourself, use it **read-only** from now on.
   - Propose maintenance mode. An app serving empty tables creates duplicate accounts and double payments, and lets
     cron jobs downgrade "unpaid" users.
   - Payment-provider webhooks retry for a limited time (often about 24 h) and are then disabled. Note the deadline,
     or log the raw payloads.
   - Snapshot **every** disk before anything else, including the one holding `pg_wal`.
   - Copy `pg_wal` and the server logs off the box. Postgres recycles WAL segments, and they can hold **full-page
     images of the pages first modified after each checkpoint before the incident**. Coverage is partial, and
     TRUNCATE/DROP themselves log no row data. On PG16+, `pg_waldump --save-fullpage` plus `pg_filedump` can recover
     some real rows.
   - `pg_dump` the current state: rows written after the incident are real data.
   - Pin down the exact time from the logs and `pg_waldump`.
   - Check `pg_sequences`. If the statement used `RESTART IDENTITY`, new signups are reusing the ids of real users,
     and those post-incident rows will need new ids.
2. **Look for a newer copy.**
   - **List** the snapshots. An attached schedule may never have produced one.
   - Check replicas, staging clones, analytics exports, developer laptops and provider backups.
   - Any newer copy changes the plan.
3. **Restore the newest copy into a scratch database, never into prod.** Migrate it forward, then diff its schema
   against prod.
4. **Reconcile catalog tables before anything else.** Plans, prices, products and coupons also changed after the dump,
   and external systems hold the ids of catalog rows created since. Rebuild them from the provider (for example the
   payment provider's plan list) and from changelogs. **Remap ids** so restored subscriptions and payments point at
   the right rows.
5. **Pull every external source, raw and read-only,** and save it with checksums.
   - Rebuild in dependency order: users, then subscriptions, then payments.
   - Use the **original ids**. They hide in order and subscription notes, in receipt or purchase ids
     that embed the user id, and in notification payloads.
   - Check each source's blind spots (table below).
6. **Ask the owner; don't guess.** Write down every ambiguous mapping and get a decision on each:
   - payments without app ids (manual payment links);
   - users whose source doesn't say which product or tenant they belong to;
   - custom deals;
   - conflicting emails;
   - people who signed up again during the outage (same email, new id): merge them, and decide which id wins;
   - columns no source holds, such as password hashes and consent flags. A typical answer is NULL plus a forced
     reset.

   The decisions file becomes an input to the scripts.
7. **Write with scripts that follow the contract below.** Dry run, owner review, then execute.
8. **Switch over (or apply), then verify against the sources.**
   - Active subscriptions match the provider's active list.
   - Payment totals per month match.
   - 10 random users match across every source.
   - Sequences are reset **where the post-incident rows live**:
     `SELECT setval('t_id_seq', GREATEST((SELECT max(id) FROM t), (SELECT last_value FROM t_id_seq)));`. A plain
     `max(id)` can move a sequence backwards when rows were created after the incident.
9. **Afterwards:**
   - Tell the owner early and plainly what can't come back.
   - User communications and any legal or breach assessment, with approval.
   - Real backups: **REQUIRED SUB-SKILL:** setting-up-postgres-disaster-recovery.
   - Guards against a repeat (below).
   - Delete the forensic and PII copies, but only after the data is verified and the postmortem is done.
   - A postmortem.

## Source blind spots

| Source | What it holds | Trap |
|---|---|---|
| Payment provider (Razorpay, Stripe, ...) | subscriptions, payments, customers; app user ids in notes / metadata | **The payment email isn't the account email.** The customer record holds the account email. Manual payment links have no notes. Amounts are in minor units, times are UTC epochs |
| Email / CRM (Loops, Mailchimp, ...) | contacts and event properties | Often holds only the users who triggered a tracked event (such as starting a checkout), so it isn't a user list. Usually has no app user id |
| Signup notifications (a Slack channel) | user id, email, name, time | Can lack fields (which product or tenant). Look for gaps in the id sequence to find missing messages |
| Analytics, request logs | activity | Good for timelines; rarely enough to rebuild rows |

## Contract for scripts that write

- **Dry run by default**, printing or writing a CSV of the planned changes.
- `--execute` plus proof of **which server is actually connected**, checked against the live connection rather than
  config:
  - Compare `SELECT system_identifier FROM pg_control_system()` with the value you
    recorded for prod or for the scratch copy, or check a marker row that only that database has.
  - Treat `--yes-host <host>` as a second check against the engine's URL. `inet_server_addr()` returns the
    server-side address, which differs behind NAT or a proxy, and is NULL on a Unix socket.
  - Env precedence is how test scripts reach production: `DB_URL` silently winning over `DB_HOST`, or a `.env` found
    by walking up the directory tree. Pass an explicit DSN.
- **Idempotent:** `ON CONFLICT DO NOTHING` and existence checks. **Plain SQL**, not app code paths, so no emails,
  webhooks or Slack posts fire.
- **Single instance** (a file lock), with the **work list frozen before the run**. If a script computes its scope from
  the tables it is changing, a second overlapping run silently does something different.
- **Tags every row it writes**, for example with `source = 'incident_restore_<date>'` in an existing metadata
  column, or in a side table `(table, pk, source)` when there is no such column.
- **The owner runs it, or approves each run.** If a permission prompt, hook or classifier blocks a production write,
  hand the exact command to the human. Don't route around it with another tool, another credential, a script file or
  a split-up command.

## Guards against a repeat

- No privileged production credentials in local `.env` files.
- Development and agent roles without `TRUNCATE`, `DROP` or DDL; read-only roles for agent and MCP database tools.
- Code that refuses public or production hosts unless an explicit opt-in is set.
- A pre-execution hook that blocks destructive SQL aimed at production hosts.
