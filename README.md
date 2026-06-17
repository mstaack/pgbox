# pgbox

A small KISS operator tool for running a secure PostgreSQL Docker host on Debian or Ubuntu with pgBackRest backups.

Target use case: one professional PostgreSQL server on a Hetzner/Debian/Ubuntu VM, simple enough to understand, but with real backup/restore mechanics.

## One-line install

```bash
curl -fsSL https://raw.githubusercontent.com/mstaack/pgbox/main/bin/pgbox | sudo tee /usr/local/bin/pgbox >/dev/null && sudo chmod +x /usr/local/bin/pgbox && sudo pgbox install
```

To update the `pgbox` script itself later (defaults to pulling from this repo):

```bash
sudo pgbox self-update
```

## What it creates

```text
/opt/pgbox
├── data/                         PostgreSQL data directory
├── backups/                      optional scratch/local dump area
├── pgbackrest-repo/              pgBackRest repository
├── pgbackrest-conf/              pgBackRest config
├── pgbackrest-log/               pgBackRest logs
├── postgres-conf/                PostgreSQL config
├── secrets/postgres_password     database password
├── docker-compose.yml
├── Dockerfile.postgres-pgbackrest
├── .env
├── state/state.json
└── PGBOX_INFO.txt
```

## Quick start

```bash
sudo pgbox install
sudo pgbox start
sudo pgbox doctor
sudo pgbox backup full
sudo pgbox backups
```

## Commands

### Install/setup

```bash
sudo pgbox install
sudo pgbox start
sudo pgbox stop
sudo pgbox restart
```

### Status/docs

```bash
sudo pgbox info
sudo pgbox doctor
sudo pgbox logs
sudo pgbox help
```

### Databases, users, permissions

```bash
sudo pgbox shell                      # open psql
sudo pgbox url [db]                   # connection URLs for TablePlus/DBeaver/psql

sudo pgbox db list
sudo pgbox db create shop [owner]
sudo pgbox db drop shop               # destructive, confirms

sudo pgbox user list
sudo pgbox user create shopuser       # prompts for password
sudo pgbox user create admin --super  # superuser role
sudo pgbox user password shopuser     # change password
sudo pgbox user drop shopuser

sudo pgbox grant shopuser shop rw     # ro | rw | all (default rw)
```

`rw` grants connect/usage plus data read-write **and** `CREATE` on schema `public`
(so app migrations work); `ro` is read-only; `all` is full privileges on the database.

### Backups

```bash
sudo pgbox backup full
sudo pgbox backup diff
sudo pgbox backup incr
sudo pgbox backups
```

### Restore

```bash
sudo pgbox restore latest
sudo pgbox restore time "2026-06-17 13:00:00"
sudo pgbox restore latest --repo 2          # restore from the offsite (S3) repo
```

Restore is destructive. The current data directory is moved into `/opt/pgbox/restore-old/` before restore.

### Provision a new server from existing backups

To stand up a replacement box from your offsite backups, install with the **same**
S3 credentials, path, and encryption passphrase as the original (so the stanza in the
bucket is recognized), then restore instead of starting fresh:

```bash
sudo pgbox install              # enter the original S3 + encryption settings
sudo pgbox restore latest --repo 2
sudo pgbox doctor
```

### Updates

```bash
sudo pgbox update       # safety backup + rebuild/pull + restart
sudo pgbox self-update  # update the pgbox script itself from PGBOX_UPDATE_URL
```

### Config/secrets

```bash
sudo pgbox config show
sudo pgbox config edit
sudo pgbox secrets
```

## Interactive install asks for

- Postgres image (default `postgres:17`; use any tag or custom registry image)
- database name
- database user
- password or generated password
- timezone
- network mode
- allowed IPs/CIDRs if public/private restricted mode is selected
- pgBackRest full backup retention count
- pgBackRest diff backup retention count
- Docker shared memory size
- pgBackRest process-max
- self-update raw script URL
- backup encryption at rest (default yes; AES-256 with a passphrase)
- offsite S3 backup (optional; defaults tuned for Cloudflare R2 EU)

## Encryption at rest

By default `pgbox install` encrypts every repository with `aes-256-cbc`. Backups and
archived WAL are encrypted by pgBackRest **before** they touch disk or leave the box,
so both the local repo and the offsite S3 bucket hold only ciphertext.

The passphrase is set at install (entered or generated) and stored in `/opt/pgbox/.env`.
**Keep a copy** — backups are unrecoverable without it, and the cipher cannot be changed
after the stanza is created without discarding existing backups.

## Offsite backups (S3 / Cloudflare R2)

Enable offsite backup during `pgbox install` to add a second pgBackRest repository
on any S3-compatible store. Every backup and all archived WAL are written to both the
local repo and the bucket, so the local repo stays fast and the bucket is your offsite copy.

Defaults are tuned for Cloudflare R2 EU (`region=auto`, `uri-style=path`); for R2 EU the
endpoint is `<ACCOUNT_ID>.eu.r2.cloudflarestorage.com`. The same prompts work for AWS S3, MinIO, Backblaze
B2, Hetzner Object Storage, etc.

Restore specifically from the offsite copy:

```bash
docker exec pgbox-postgres pgbackrest --stanza=main --repo=2 restore
```

To add or change offsite backup on an existing install, edit
`/opt/pgbox/pgbackrest-conf/pgbackrest.conf` directly (it is only generated at install time).

## Security defaults

- PostgreSQL binds to `127.0.0.1:5432` unless explicitly opened.
- UFW is enabled.
- If opened, port `5432` is allowed only from configured IPs/CIDRs.
- Password is stored in `/opt/pgbox/secrets/postgres_password`.
- The Docker container uses `no-new-privileges`.
- Logs are size-limited.

## Backup model

pgbox uses pgBackRest as the real backup engine:

- full backups
- differential backups
- incremental backups
- continuous WAL archiving
- point-in-time restore

Default retention during install should normally be something like:

```text
full backup retention: 8
 diff backup retention: 30
```

### Backup schedule

`pgbox install` sets the schedule automatically as a managed cron drop-in at
`/etc/cron.d/pgbox` (and ensures `cron` is installed and enabled):

```cron
0 2 * * 0   root /usr/local/bin/pgbox backup full   # weekly full
0 2 * * 1-6 root /usr/local/bin/pgbox backup diff   # daily differential
30 * * * *  root /usr/local/bin/pgbox backup incr   # hourly incremental
15 3 * * *  root /usr/local/bin/pgbox doctor >> /var/log/pgbox-doctor.log 2>&1
```

Continuous WAL archiving already covers point-in-time recovery between backups; the
hourly incremental gives faster standalone restore points on top of that. Edit
`/etc/cron.d/pgbox` to change the cadence; `pgbox reconfigure` regenerates it.
`pgbox info` shows the active retention and whether the schedule is installed.

WAL archiving is configured in PostgreSQL via:

```text
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

## Example install session

```text
PGBOX interactive install
Postgres image [postgres:17]: postgres:17
Database name [app]: eventbackend
Database user [app]: eventbackend
Database password (leave empty to generate):
Timezone [Europe/Berlin]:
Network mode: 1) localhost only  2) expose 5432 to selected IPs/CIDRs
Choose network mode [1]: 2
Allowed IPs/CIDRs, comma-separated: 10.10.10.0/24,203.0.113.10
pgBackRest full backup retention count [8]: 12
pgBackRest diff backup retention count [30]: 60
Docker shm_size [1g]: 1g
pgBackRest process-max [2]: 4
Self-update raw script URL [https://raw.githubusercontent.com/mstaack/pgbox/main/bin/pgbox]:
Encrypt backups? yes/no [yes]: yes
Encryption passphrase (leave empty to generate):
Enable offsite S3 backup? yes/no [no]: yes
S3 bucket: pgbox-backups
S3 endpoint, host only (R2 EU: <ACCOUNT_ID>.eu.r2.cloudflarestorage.com): abc123.eu.r2.cloudflarestorage.com
S3 region [auto]:
S3 access key id: ...
S3 secret access key:
S3 path/prefix [/pgbox]:
S3 URI style: path (R2/S3-compatible) or host (AWS) [path]:
S3 full backup retention count [12]:
==> Installed. Start with: pgbox start
```

## Example info output

```text
PGBOX INFO
Version:        0.1.0
App dir:        /opt/pgbox
Container:      pgbox-postgres
Image:          postgres:17
Database:       eventbackend
User:           eventbackend
Expose mode:    2
Allowed IPs:    10.10.10.0/24,203.0.113.10
Backup mode:    pgbackrest
Full retention: 12
Diff retention: 60

Postgres:       running
Health:         healthy
DB size:        48 GB
Connections:    12
Version SQL:    PostgreSQL 17.x

pgBackRest latest info:
...
```

## Important limitations

This is intentionally simple. It is not Kubernetes, not Patroni, and not HA.

Current v0.1 scope:

- single PostgreSQL primary
- Docker Compose
- pgBackRest local repository + optional encrypted offsite S3 repository
- interactive install with automatic backup schedule (`/etc/cron.d/pgbox`)
- PITR-capable local/offsite restore
- multiple databases, users, and grants via `pgbox db|user|grant`
- basic doctor/info/update/self-update commands

For stronger production durability, enable the offsite S3 repository at install time (see above), or add a remote pgBackRest repository later by editing `pgbackrest.conf`.

## License

MIT — see [LICENSE](LICENSE).
