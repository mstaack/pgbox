# pgbox operations

## Daily operation

```bash
sudo pgbox info
sudo pgbox doctor
sudo pgbox backups
```

## Backup schedule

Recommended simple schedule:

```cron
0 2 * * 0 /usr/local/bin/pgbox backup full
0 2 * * 1-6 /usr/local/bin/pgbox backup diff
0 * * * * /usr/local/bin/pgbox backup incr
```

## Restore latest

```bash
sudo pgbox restore latest
sudo pgbox doctor
```

## Restore point in time

```bash
sudo pgbox restore time "2026-06-17 13:00:00"
sudo pgbox doctor
```

## Update PostgreSQL image

```bash
sudo pgbox update
```

The command attempts a safety differential backup first.

## Update pgbox itself

Configure `PGBOX_UPDATE_URL` in `/opt/pgbox/.env`, then:

```bash
sudo pgbox self-update
```
