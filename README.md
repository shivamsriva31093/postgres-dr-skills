# postgres-dr: Claude Code skills for Postgres data recovery and disaster recovery

Two skills for [Claude Code](https://claude.com/claude-code) and other agents that read `SKILL.md` files. They come
from hands-on production work rather than from documentation: the traps they list are the ones that only show up when
you actually run a restore.

| Skill | Triggers when |
|---|---|
| `recovering-lost-postgres-data` | production rows or tables were deleted, truncated, dropped or overwritten by mistake, especially without PITR |
| `setting-up-postgres-disaster-recovery` | a self-hosted Postgres has no backups, only snapshots, no PITR, or backups that were never restored elsewhere; restore drills; rebuilding a lost VM |

`setting-up-postgres-disaster-recovery` also ships:
- `gcp-worked-example.md`: the full command sequence for GCE, Debian 12, PostgreSQL 18 + TimescaleDB, pgBackRest and GCS.
- `pg-backup-check`: a Python alert tool (pgBackRest status and age, WAL-archive backlog, logical-dump freshness and
  completeness, failed systemd units, disk usage). It posts to Slack, has an hourly mode that only speaks up on
  problems, a daily heartbeat, and an `OnFailure=` hook mode. It is configured through environment variables listed in
  its header.

## Install

As a Claude Code plugin:

```
/plugin marketplace add shivamsriva31093/postgres-dr-skills
/plugin install postgres-dr@postgres-dr
```

Or clone the repo and copy `skills/*` into `~/.claude/skills/`.

## How the skills were tested

Following the RED → GREEN → REFACTOR approach for skills:

1. **RED.** Agents planned both scenarios **without** the skills. The baselines were already strong on the textbook
   parts (pgBackRest, archiving, `--archive-mode=off`, scratch restores, maintenance mode), so the skills cover what
   they missed:
   - **Backup/DR:** exact-version packages disappearing from apt.postgresql.org; drill VMs without egress; testing
     alerts without breaking production; snapshot schedules that never produced a snapshot; `OnFailure` with `%N`.
   - **Recovery:** reconciling catalog tables and remapping their ids first; each source's blind spots; asking the
     owner instead of guessing; the contract for scripts that write; handing blocked production writes to a human.
2. **GREEN.** The same scenarios were re-run **with** the skills, and every one of those gaps was closed. The agents'
   feedback found a sequence reset that could move backwards, a `gcloud` command that takes only one disk, and Debian's
   umbrella `postgresql.service` used for ordering. All are fixed.
3. **REFACTOR.** A fresh agent answered targeted retrieval questions from the final text (9/9 found), then reviewed it
   for errors. It found, and these are now fixed:
   - `--target` without a UTC offset;
   - an emergency rewind that didn't preserve post-incident rows first;
   - `inet_server_addr()` used as a server-identity check (the skill now uses `system_identifier`);
   - the cumulative `failed_count`;
   - same-OS/glibc collation;
   - an unbounded wait loop.

`pg-backup-check` was run against a real server (`--dry-run`, healthy and forced-failure paths), and alerted
end-to-end through systemd `OnFailure=`.

## Scope and caveats

- Self-hosted Postgres on a VM. For managed Postgres, use the provider's PITR, but the drill section still applies.
- The worked example is GCP- and Debian-specific; the traps and the "done means" checklist are not.
- `pg-backup-check` lists dumps with `gcloud storage`. For S3 or Azure, adapt `list_dump_folder()` / `stat_object()`.

## Use at your own risk

These skills guide an agent through work on production databases: restarts, restores, and writes. They describe
approval gates for every such step, but you are responsible for what runs against your systems. Read the commands
before you approve them, and try a restore drill on a throwaway machine first.

## Contributing

Issues and pull requests are welcome, especially worked examples for other platforms (AWS, Azure, WAL-G, barman) and
traps you hit in a real restore. Keep additions to what an agent would otherwise get wrong.

## License

[MIT](LICENSE)
