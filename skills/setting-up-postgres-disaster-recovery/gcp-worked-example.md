# Worked example: GCE VM, Debian 12, PostgreSQL 18 + TimescaleDB, pgBackRest → GCS

This sequence was run end to end on a small production database (about 600 MB): one restart of about 4 seconds, and
a restore drill that went from nothing to a queryable copy in about 5 minutes. Adapt the names; on AWS or Azure, swap
the bucket and credential steps (`repo1-type=s3` / `azure`) and the snapshot commands.

```bash
PROJECT=my-project ZONE=us-central1-a REGION=us-central1 VM=db-1 DATA_DISK=db-1-data BOOT_DISK=db-1
SA=db-sa@my-project.iam.gserviceaccount.com BUCKET=my-project-pg-backups PGVER=18 STANZA=main
BUCKET_LOCATION=us-east1      # another region, or a dual-region
DATA=/pgdata/$PGVER/main CONF=/etc/postgresql/$PGVER/main
```

## 1. Inventory (read-only, on the VM)

```bash
dpkg-query -W -f='${Package}=${Version}\n' | grep -E '^(postgresql-[0-9]|timescaledb|pgbackrest)'
grep -rh '^deb ' /etc/apt/sources.list.d/            # which repos the versions came from
for db in $(sudo -u postgres psql -XAtc "select datname from pg_database where not datistemplate"); do
  echo "$db: $(sudo -u postgres psql -d "$db" -XAtc "select string_agg(extname||' '||extversion, ', ') from pg_extension")"
done
sudo -u postgres psql -XAtc "select datname, datlocprovider, datcollate from pg_database"
sudo -u postgres psql -XAtc "show shared_preload_libraries"; sudo cat $DATA/postgresql.auto.conf
df -h / /pgdata; locale -a; . /etc/os-release; echo "$PRETTY_NAME"; ldd --version | head -1
```
From your workstation:
```bash
gcloud compute disks describe $DATA_DISK --zone $ZONE --format="value(resourcePolicies)"
gcloud compute snapshots list --filter="sourceDisk~$VM" --format="table(name,sourceDisk.basename(),creationTimestamp)"
gcloud compute routers list && gcloud compute routers nats list --router <router> --region $REGION   # egress for a drill VM
gcloud compute instances describe $VM --zone $ZONE --format="value(serviceAccounts[].email,serviceAccounts[].scopes)"
```

## 2. Snapshots on both disks

```bash
# no backups at all? take one now; it's minutes away
gcloud compute disks snapshot $DATA_DISK $BOOT_DISK --zone $ZONE \
  --snapshot-names pre-dr-data-$(date +%Y%m%d),pre-dr-boot-$(date +%Y%m%d)
gcloud compute resource-policies create snapshot-schedule pg-daily --region $REGION \
  --daily-schedule --start-time 15:00 --max-retention-days 14
for d in $DATA_DISK $BOOT_DISK; do                  # one disk per call
  gcloud compute disks add-resource-policies $d --resource-policies pg-daily --zone $ZONE
done
# the next day: one snapshot per disk must be listed
```

## 3. Bucket and pgBackRest

```bash
gcloud storage buckets create gs://$BUCKET --location $BUCKET_LOCATION --uniform-bucket-level-access \
  --public-access-prevention --soft-delete-duration 14d
echo '{"rule":[{"action":{"type":"Delete"},"condition":{"age":30,"matchesPrefix":["logical/"]}}]}' > lc.json
gcloud storage buckets update gs://$BUCKET --lifecycle-file lc.json
gcloud storage buckets add-iam-policy-binding gs://$BUCKET --member serviceAccount:$SA --role roles/storage.objectAdmin
# the VM also needs a storage read-write (or cloud-platform) access scope; changing scopes requires stopping the VM
```
On the VM:
```bash
sudo apt-get install -y pgbackrest
sudo tee /etc/pgbackrest.conf >/dev/null <<EOF
[global]
repo1-type=gcs
repo1-gcs-bucket=$BUCKET
repo1-gcs-key-type=auto
repo1-path=/pgbackrest
repo1-retention-full-type=count
repo1-retention-full=4
compress-type=zst
compress-level=3
process-max=2
start-fast=y
archive-async=y
spool-path=/var/spool/pgbackrest
log-level-console=info
log-level-file=detail

[$STANZA]
pg1-path=$DATA
pg1-port=5432
EOF
sudo chown root:postgres /etc/pgbackrest.conf && sudo chmod 640 /etc/pgbackrest.conf
sudo install -d -o postgres -g postgres -m 750 /var/spool/pgbackrest /var/log/pgbackrest
sudo -u postgres pgbackrest --stanza=$STANZA stanza-create
```

## 4. WAL archiving: the one restart

```bash
sudo tee $CONF/conf.d/archiving.conf >/dev/null <<EOF
archive_mode = on
archive_command = 'pgbackrest --stanza=$STANZA archive-push %p'
archive_timeout = 60
EOF
# preflight: parse the whole config without touching the running server; never restart on a bad file
P="sudo -u postgres /usr/lib/postgresql/$PGVER/bin/postgres -c config_file=$CONF/postgresql.conf -C"
if [ "$($P archive_mode)" = on ] && [ "$($P wal_level)" != minimal ]; then
  sudo systemctl restart postgresql@$PGVER-main
else
  sudo rm $CONF/conf.d/archiving.conf; echo "preflight failed - NOT restarted"
fi
sudo -u postgres pgbackrest --stanza=$STANZA check          # forces a WAL switch and waits for it in the bucket
sudo -u postgres psql -XAtc "select archived_count, failed_count from pg_stat_archiver"   # re-read if 0
```
Rollback: remove `archiving.conf` and restart.

## 5. First full backup, timers, logical dumps

```bash
sudo -u postgres pgbackrest --stanza=$STANZA --type=full backup && sudo -u postgres pgbackrest info
```
`/etc/systemd/system/pgbackrest@.service`:
```ini
[Unit]
Description=pgBackRest %i backup
After=postgresql@18-main.service
OnFailure=pg-backup-alert@%N.service

[Service]
Type=oneshot
User=postgres
Nice=10
ExecStart=/usr/bin/pgbackrest --stanza=main --type=%i backup
```
`pgbackrest-full.timer` uses `OnCalendar=Sun *-*-* 20:30:00 UTC` and `Unit=pgbackrest@full.service`.
`pgbackrest-diff.timer` uses `OnCalendar=Mon..Sat *-*-* 20:30:00 UTC` and `Unit=pgbackrest@diff.service`.
Both have `Persistent=true` and `[Install] WantedBy=timers.target`.

`/usr/local/bin/pg-logical-dump`, run by `pg-logical-dump.service` (`User=postgres`, with the same `After=` and
`OnFailure=` lines) from `pg-logical-dump.timer` (`OnCalendar=*-*-* 21:30:00 UTC`, `Persistent=true`):
```bash
#!/bin/bash
# Restoring a TimescaleDB database: select timescaledb_pre_restore(); pg_restore ...; select timescaledb_post_restore();
set -euo pipefail
dest="gs://my-project-pg-backups/logical/$(date -u +%F)"   # EDIT: your bucket
for db in $(psql -XAtc "select datname from pg_database where not datistemplate and datname <> 'postgres'"); do
  pg_dump -Fc -Z 6 "$db" | gcloud storage cp --quiet - "$dest/$db.dump"
done
pg_dumpall --globals-only --no-role-passwords | gcloud storage cp --quiet - "$dest/globals.sql"
```
Prove it: `sudo systemctl start pg-logical-dump.service`, then `gcloud storage cat …/<db>.dump > x && pg_restore -l x | grep -c "TABLE DATA"`.
TimescaleDB prints a harmless "circular foreign-key constraints … continuous_agg" warning.

## 6. Restore drill on a separate VM

Make the stop point provable before you start. These writes go to production, so get the owner's approval:
```bash
sudo -u postgres psql -c "create table if not exists dr_drill(note text, at timestamptz default clock_timestamp())" \
  -c "insert into dr_drill values ('before')"
# wait a minute, note T (UTC), wait a minute
sudo -u postgres psql -c "insert into dr_drill values ('after')"
sudo -u postgres pgbackrest --stanza=$STANZA check      # archives the 'after' commit; without a commit past T, PG13+
                                                         # fails: "recovery ended before configured recovery target"
```
Pass: the restored copy has `before` and not `after`. Drop `dr_drill` afterwards.

```bash
gcloud compute instances create pg-restore-drill --zone $ZONE --machine-type e2-medium \
  --image-family debian-12 --image-project debian-cloud --boot-disk-size 20GB \
  --service-account $SA --scopes storage-ro --tags <your-restricted-ssh-tag> --labels purpose=restore-drill
# sshd answers ~30 s after RUNNING ("connection refused" until then)
```
On the drill VM, with the exact versions from the inventory:
```bash
sudo apt-get update && sudo apt-get install -y postgresql-common curl gnupg
sudo sed -i 's/^#\?\s*create_main_cluster.*/create_main_cluster = false/' /etc/postgresql-common/createcluster.conf
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y
echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt-archive.postgresql.org/pub/repos/apt bookworm-pgdg-archive main" \
  | sudo tee /etc/apt/sources.list.d/pgdg-archive.list
echo "deb https://packagecloud.io/timescale/timescaledb/debian/ bookworm main" | sudo tee /etc/apt/sources.list.d/timescaledb.list
curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/timescaledb.gpg
# copy each version string from the inventory; they look like 18.4-1.pgdg12+1 or 2.27.1~debian12-1804
sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  postgresql-18=<ver> postgresql-client-18=<ver> postgresql-18-jit=<ver> postgresql-18-pgvector=<ver> \
  timescaledb-2-postgresql-18=<ver> timescaledb-2-loader-postgresql-18=<ver> pgbackrest=<ver>

sudo tee /etc/pgbackrest.conf >/dev/null <<EOF
[global]
repo1-type=gcs
repo1-gcs-bucket=$BUCKET
repo1-gcs-key-type=auto
repo1-path=/pgbackrest
archive-async=n

[$STANZA]
pg1-path=/var/lib/postgresql/18/drill
EOF
D=/var/lib/postgresql/18/drill
sudo install -d -o postgres -g postgres -m 755 /var/lib/postgresql/18 && sudo install -d -o postgres -g postgres -m 700 $D
sudo -u postgres pgbackrest --stanza=$STANZA --type=time --target="<YYYY-MM-DD HH:MM:SS>+00" \
  --target-action=promote --archive-mode=off restore
sudo -u postgres bash -c "echo '' > $D/postgresql.conf; echo 'local all postgres peer' > $D/pg_hba.conf; : > $D/pg_ident.conf"
sudo -u postgres /usr/lib/postgresql/18/bin/pg_ctl -D $D -l /var/lib/postgresql/drill.log start -o \
  "-p 5432 -c listen_addresses=localhost -c archive_mode=off -c shared_preload_libraries=timescaledb -c timescaledb.max_background_workers=0"
for i in $(seq 1 300); do                       # up to 10 min; stop early if recovery died
  sudo -u postgres psql -XAtc "select not pg_is_in_recovery()" 2>/dev/null | grep -qx t && break
  sudo grep -qE "FATAL|PANIC" /var/lib/postgresql/drill.log && { sudo tail -20 /var/lib/postgresql/drill.log; break; }
  sleep 2
done
sudo grep -E "recovery stopping|archive recovery complete|ready to accept" /var/lib/postgresql/drill.log
```
Compare row counts with production as of the target (`count(*) where created_at <= '<target>'` there), then run
`gcloud compute instances delete pg-restore-drill --zone $ZONE --delete-disks=all`.

What a pass looks like: WAL replay takes seconds, the log reads `recovery stopping before commit of transaction
<n>, time <first commit after the target>`, and the counts match in every database.

## 7. Alerts

Install `pg-backup-check` (next to this file) to `/usr/local/bin/`. Then:
```bash
sudo tee /etc/default/pg-backup-check >/dev/null <<EOF
PG_BACKUP_PGDATA=$DATA
PG_BACKUP_DUMP_URL=gs://$BUCKET/logical
PG_BACKUP_MAIN_DB=<largest database>
PG_BACKUP_MAIN_DB_MIN_BYTES=<about a fifth of its dump size>
EOF
# webhook: root-only, sent over stdin so it never appears in a command line or a local file
printf 'SLACK_WEBHOOK_URL=%s\n' "$WEBHOOK" | gcloud compute ssh $VM --zone $ZONE --command \
  'sudo sh -c "umask 077; cat > /etc/pg-backup-alert.env"'
```
Units (all `Type=oneshot`, `User=postgres`,
`EnvironmentFile=/etc/default/pg-backup-check`, `EnvironmentFile=/etc/pg-backup-alert.env`):

| Unit | ExecStart | Extra |
|---|---|---|
| `pg-backup-check.service` + `.timer` (`OnCalendar=*-*-* *:15:00 UTC`) | `pg-backup-check check` | `OnFailure=pg-backup-alert@%N.service` |
| `pg-backup-daily.service` + `.timer` (`OnCalendar=*-*-* 22:30:00 UTC`, `Persistent=true`) | `pg-backup-check daily` | `OnFailure=pg-backup-alert@%N.service` |
| `pg-backup-alert@.service` | `pg-backup-check unit-failed %i` | `SupplementaryGroups=systemd-journal` (to read the failed unit's log); no `OnFailure` (no loops) |

Prove it:
```bash
sudo -u postgres -H env $(cat /etc/default/pg-backup-check) pg-backup-check daily --dry-run
sudo systemctl start pg-backup-daily.service                  # real summary post
sudo systemctl start pgbackrest@bogus.service                 # rejected before doing anything; alert must arrive
sudo systemctl reset-failed pgbackrest@bogus.service
sudo systemctl enable --now pg-backup-check.timer pg-backup-daily.timer
```

## Emergency: rewind the whole cluster to before a bad write

```bash
# 1. preserve first: snapshot both disks and pg_dump the current state (rows written after the bad statement are
#    real data and the rewind discards them, for every database on the cluster)
# 2. stop the app
sudo systemctl stop postgresql@18-main
sudo -u postgres pgbackrest --stanza=$STANZA --delta --type=time --target="<YYYY-MM-DD HH:MM:SS>+00" \
  --target-action=promote restore
sudo systemctl start postgresql@18-main
sudo -u postgres pgbackrest --stanza=$STANZA --type=full backup     # fresh baseline on the new timeline
```
If only one table or database was hit, restore a copy on a drill VM instead and copy the rows across.
