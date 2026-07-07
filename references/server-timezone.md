# Server Timezone & Database Date Dependencies

## The Problem

The `test_ip_usage` table tracks daily usage by `date('Y-m-d')` in PHP. This means:

- Daily quota resets are **timezone-dependent**
- If the server is on Asia/Tehran (+0330), a user in China (UTC+8) sees quota reset 4.5 hours off
- The `api_keys.expires_at` and `created_at` also use `datetime('now', '+8 hours')` in the schema

## What Was Fixed

Server timezone changed from **Asia/Tehran** to **Asia/Shanghai (UTC+8)**:

```
sudo timedatectl set-timezone Asia/Shanghai
```

## Check Current Timezone

```bash
timedatectl | grep "Time zone"
```

## Database Implications

- `test_ip_usage.date` — daily quota counter, uses PHP `date('Y-m-d')`
- `api_keys.expires_at` — stored as text, manually set
- `api_keys.created_at` — uses `datetime('now', '+8 hours')` in SQLite schema

All dates stored in the DB are in Asia/Shanghai time now.
