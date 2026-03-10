# Vibe Kanban — Self-Hosting on Unraid

## Overview

Vibe Kanban runs on `vibe-kanban.atoplix.com`, deployed via Docker on an Unraid NAS.
Traffic is handled by Nginx Proxy Manager (already running on Unraid) which forwards to the app on port `8090`.

## Architecture

```
Internet → vibe-kanban.atoplix.com → Nginx Proxy Manager → localhost:8090 → remote-server (8081)
                                                                              remote-db (postgres)
                                                                              electric (ElectricSQL)
```

## Key Files

| File | Purpose |
|------|---------|
| `.env.remote` | Secrets and environment variables (never commit this) |
| `crates/remote/docker-compose.prod.yml` | Docker Compose config for all services |

## Persistent Data on Unraid

All data is stored under `/mnt/user/appdata/vibe-kanban/` on the NAS:

| Path | Service |
|------|---------|
| `/mnt/user/appdata/vibe-kanban/postgres` | PostgreSQL database |
| `/mnt/user/appdata/vibe-kanban/electric` | ElectricSQL state |

## Deploy / Redeploy

Run from the repo root on your laptop:

```bash
DOCKER_HOST=ssh://unraid docker compose \
  --env-file .env.remote \
  -f crates/remote/docker-compose.prod.yml \
  up -d --build
```

- `--build` is only needed when you've pulled new changes. For a plain restart, omit it.
- Docker targets the Unraid NAS via SSH automatically.

## Update to a New Version

```bash
git pull origin main

DOCKER_HOST=ssh://unraid docker compose \
  --env-file .env.remote \
  -f crates/remote/docker-compose.prod.yml \
  up -d --build
```

## Check Status

```bash
DOCKER_HOST=ssh://unraid docker compose \
  --env-file .env.remote \
  -f crates/remote/docker-compose.prod.yml \
  ps
```

All 3 services should show `(healthy)`:
- `remote-remote-db-1` — PostgreSQL
- `remote-remote-server-1` — App server (port 8090)
- `remote-electric-1` — ElectricSQL sync

## View Logs

```bash
# All services
DOCKER_HOST=ssh://unraid docker compose \
  --env-file .env.remote \
  -f crates/remote/docker-compose.prod.yml \
  logs -f

# Specific service
DOCKER_HOST=ssh://unraid docker compose \
  --env-file .env.remote \
  -f crates/remote/docker-compose.prod.yml \
  logs -f remote-server
```

## Stop / Restart

```bash
# Stop all
DOCKER_HOST=ssh://unraid docker compose \
  --env-file .env.remote \
  -f crates/remote/docker-compose.prod.yml \
  down

# Restart a specific service
DOCKER_HOST=ssh://unraid docker compose \
  --env-file .env.remote \
  -f crates/remote/docker-compose.prod.yml \
  restart remote-server
```

## Nginx Proxy Manager Setup

| Field | Value |
|-------|-------|
| Domain | `vibe-kanban.atoplix.com` |
| Forward hostname | `192.168.0.10` (Unraid local IP) |
| Forward port | `8090` |
| SSL | Let's Encrypt |

## OAuth

GitHub OAuth app: [github.com/settings/developers](https://github.com/settings/developers)
- Callback URL: `https://vibe-kanban.atoplix.com/v1/oauth/github/callback`

## Backup Database

```bash
DOCKER_HOST=ssh://unraid docker compose \
  -f crates/remote/docker-compose.prod.yml \
  exec remote-db pg_dump -U remote remote > backup_$(date +%Y%m%d).sql
```

## Restore Database

```bash
DOCKER_HOST=ssh://unraid docker compose \
  -f crates/remote/docker-compose.prod.yml \
  exec -T remote-db psql -U remote remote < backup_20240101.sql
```
