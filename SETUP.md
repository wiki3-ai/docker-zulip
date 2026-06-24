# Wiki3 Zulip Server — Setup & Operations Guide

> **Deployed:** 2026-06-17  
> **Zulip version:** 12.0 (Docker image `12.0-1`)  
> **Host:** haken (Ubuntu, Docker 29.5.3)  
> **URL:** https://chat.wiki3.ai

---

## Architecture

```
Internet
  │
  ▼
Cloudflare Edge (SSL termination)
  │
  ▼
cloudflared container ──► zulip container :80 (HTTP)
                              │
                  ┌───────────┼───────────┐───────────┐
                  ▼           ▼           ▼           ▼
              database    memcached   rabbitmq      redis
           (PostgreSQL 14)
```

### Containers (6 total)

| Service    | Image                                | Purpose                        |
|------------|--------------------------------------|--------------------------------|
| `zulip`    | `ghcr.io/zulip/zulip-server:12.0-1` | Zulip web app + workers        |
| `database` | `zulip/zulip-postgresql:14`         | PostgreSQL database            |
| `memcached`| `memcached:alpine`                   | Object cache                   |
| `rabbitmq` | `rabbitmq:4.2`                       | Message queue for workers      |
| `redis`    | `redis:alpine`                       | Caching, rate limiting         |
| `cloudflared` | `cloudflare/cloudflared:latest`   | Cloudflare Tunnel connector    |

### Docker Volumes

| Volume                       | Contents                  |
|------------------------------|---------------------------|
| `zulip-docker_zulip`        | Uploaded files, avatars   |
| `zulip-docker_postgresql-14`| Database data             |
| `zulip-docker_rabbitmq`     | RabbitMQ data             |
| `zulip-docker_redis`        | Redis data                |

---

## Project Files

```
/media/nvm4t2/Projects/zulip-docker/
├── .env                        # Secrets & tokens (gitignored)
├── compose.yaml                # Base compose (from upstream repo)
├── compose.override.yaml       # Our customizations (tracked in git)
├── manage.py                   # Wrapper for Zulip management commands
├── Dockerfile                  # For local builds (not used — we use prebuilt image)
└── SETUP.md                    # ← This file
```

---

## Configuration Details

### Networking & SSL

| Setting              | Value                 |
|----------------------|-----------------------|
| External hostname    | `chat.wiki3.ai`       |
| SSL termination      | Cloudflare (edge)     |
| Zulip listens on     | HTTP port 80          |
| Trusted proxy range  | `172.16.0.0/12` (Docker bridge) |
| Cloudflare Tunnel    | `cloudflared` container in same compose stack |

**Why `LOADBALANCER_IPS: "172.16.0.0/12"` instead of `TRUST_GATEWAY_IP`?**  
The `TRUST_GATEWAY_IP` option only trusts the Docker gateway (`172.18.0.1`), but requests from the `cloudflared` container arrive from `172.18.0.7`. Trusting the full Docker bridge subnet covers all containers.

### Outgoing Email (Google Workspace)

| Setting                    | Value                        |
|----------------------------|------------------------------|
| SMTP host                  | `smtp.gmail.com`             |
| SMTP port                  | 587 (STARTTLS)               |
| SMTP login (primary acct)  | `ai@fovi.com`                |
| From address (alias)       | `ai@wiki3.ai`                |
| App Password               | Stored in `.env` as `ZULIP__EMAIL_PASSWORD` |

**Important:** Google requires the **primary account email** for SMTP login, not an alias. `ai@wiki3.ai` is an alias for `ai@fovi.com`, so we authenticate with `ai@fovi.com` but send as `ai@wiki3.ai`.

To create a new App Password:
1. Sign in as `ai@fovi.com` at [myaccount.google.com](https://myaccount.google.com)
2. Security → 2-Step Verification → App passwords
3. Create one named "Zulip" and update `ZULIP__EMAIL_PASSWORD` in `.env`

### Admin Account

| Field    | Value            |
|----------|------------------|
| Name     | Jim White        |
| Email    | `jim@wiki3.ai`   |
| Realm    | Wiki3            |
| URL      | `https://chat.wiki3.ai` |

---

## Secrets

All secrets are defined in `.env` and loaded into Docker via `compose.override.yaml` using `environment:` references (Docker Compose secrets backed by env vars).

| Secret                        | Purpose                              |
|-------------------------------|--------------------------------------|
| `ZULIP__POSTGRES_PASSWORD`    | PostgreSQL `zulip` user password     |
| `ZULIP__MEMCACHED_PASSWORD`   | Memcached SASL auth                  |
| `ZULIP__RABBITMQ_PASSWORD`    | RabbitMQ `zulip` user password       |
| `ZULIP__REDIS_PASSWORD`       | Redis AUTH password                  |
| `ZULIP__SECRET_KEY`           | Django secret key                    |
| `ZULIP__EMAIL_PASSWORD`       | Google App Password for SMTP         |
| `CLOUDFLARE_TUNNEL_TOKEN`     | Cloudflare Tunnel authentication     |

To regenerate all service passwords:
```bash
cd /media/nvm4t2/Projects/zulip-docker
# Back up first!
cp .env .env.backup.$(date +%Y%m%d)

# Generate new passwords (update individual lines in .env)
openssl rand -hex 24   # for postgres, memcached, rabbitmq, redis
openssl rand -hex 32   # for secret_key
```

**Warning:** Changing `ZULIP__POSTGRES_PASSWORD` on a running deployment requires a manual `ALTER ROLE` query. See [Zulip docs on rotating PostgreSQL passwords](https://zulip.readthedocs.io/projects/docker/en/latest/how-to/compose-secrets.html).

---

## Day-to-Day Operations

### Start / Stop / Restart

```bash
cd /media/nvm4t2/Projects/zulip-docker

# Start everything
docker compose up -d

# Stop everything
docker compose down

# Restart only Zulip (after config changes)
docker compose restart zulip

# Restart Zulip picking up new .env values
docker compose up -d --force-recreate zulip

# Restart everything
docker compose down && docker compose up -d
```

> **Important:** `docker compose restart zulip` only restarts the process inside the
> existing container — it does **NOT** pick up environment variable changes from
> `compose.override.yaml`. Always use `docker compose up -d --force-recreate zulip`
> when you change environment variables, then run `zulip-puppet-apply` if the change
> affects `zulip.conf` settings (e.g. `CONFIG_*` variables).

### Health Checks

```bash
# Container status
docker compose ps

# Zulip health endpoint
curl -s http://localhost:80/health

# Quick status via Cloudflare tunnel
curl -sI https://chat.wiki3.ai | head -5

# Test SMTP credentials
python3 -c "
import smtplib
server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
server.login('ai@fovi.com', 'APP_PASSWORD_HERE')
print('SMTP OK')
server.quit()
"
```

### Logs

```bash
# Tail all logs
docker compose logs -f

# Tail only Zulip
docker compose logs -f zulip

# Tail only cloudflared
docker compose logs -f cloudflared

# Check Zulip error log inside container
docker compose exec zulip cat /var/log/zulip/errors.log

# Check email delivery log
docker compose exec zulip cat /var/log/zulip/send_email.log

# Check server request log
docker compose exec zulip tail -20 /var/log/zulip/server.log
```

### Zulip Management Commands

The `./manage.py` wrapper runs commands inside the Zulip container:

```bash
cd /media/nvm4t2/Projects/zulip-docker

# List realms
./manage.py list_realms

# Send password reset email
./manage.py send_password_reset_email -u user@example.com -r 2

# Change a user's email
./manage.py change_user_email -u old@example.com -e new@example.com -r 2

# Create a new realm
./manage.py create_realm 'OrgName' admin@example.com 'Admin Name'

# Run Django shell
./manage.py dbshell

# Full list of commands
./manage.py help
```

---

## Upgrades

### Upgrading the Zulip Docker Image

```bash
cd /media/nvm4t2/Projects/zulip-docker

# 1. Back up the database
docker compose exec -T database pg_dumpall -U zulip > backup_$(date +%Y%m%d).sql

# 2. Pull the new image (check https://github.com/zulip/docker-zulip/releases)
docker pull ghcr.io/zulip/zulip-server:NEW_VERSION

# 3. Update the image tag in compose.yaml (or override it in compose.override.yaml)
# Edit the image tag in the zulip service

# 4. Restart
docker compose up -d
# Zulip will run database migrations automatically on startup

# 5. Check logs for migration progress
docker compose logs -f zulip
```

### Upgrading Supporting Services

```bash
# Pull latest images for all services
docker compose pull

# Restart with new images
docker compose up -d
```

### Upgrading Docker Engine

```bash
sudo apt update && sudo apt upgrade docker-ce docker-ce-cli containerd.io
```

---

## Backups

### Database Backup

```bash
cd /media/nvm4t2/Projects/zulip-docker

# Full PostgreSQL dump
docker compose exec -T database pg_dumpall -U zulip > backup_$(date +%Y%m%d_%H%M%S).sql

# Compressed
docker compose exec -T database pg_dumpall -U zulip | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz
```

### Restore Database

```bash
# Stop Zulip first
docker compose stop zulip

# Restore
cat backup_YYYYMMDD_HHMMSS.sql | docker compose exec -T database psql -U zulip

# Restart
docker compose start zulip
```

### Backup Uploaded Files

```bash
# Back up the Zulip data volume
docker run --rm -v zulip-docker_zulip:/data -v $(pwd):/backup alpine \
  tar czf /backup/zulip-files-$(date +%Y%m%d).tar.gz -C /data .
```

### Full Backup Script

```bash
#!/bin/bash
# save as: /media/nvm4t2/Projects/zulip-docker/backup.sh
set -e
cd "$(dirname "$0")"
BACKUP_DIR="backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Backing up database..."
docker compose exec -T database pg_dumpall -U zulip | gzip > "$BACKUP_DIR/database.sql.gz"

echo "Backing up uploaded files..."
docker run --rm -v zulip-docker_zulip:/data -v "$(pwd)/$BACKUP_DIR":/backup alpine \
  tar czf /backup/files.tar.gz -C /data .

echo "Backing up .env..."
cp .env "$BACKUP_DIR/env.backup"

echo "Backup complete: $BACKUP_DIR"
ls -lh "$BACKUP_DIR"
```

Make it executable:
```bash
chmod +x /media/nvm4t2/Projects/zulip-docker/backup.sh
```

Run it:
```bash
cd /media/nvm4t2/Projects/zulip-docker && ./backup.sh
```

---

## Cloudflare Tunnel

### Setup (already done)

1. Created tunnel "haken" in Cloudflare Zero Trust dashboard
2. Added public hostname route: `chat.wiki3.ai` → `http://zulip:80`
3. Token stored in `.env` as `CLOUDFLARE_TUNNEL_TOKEN`

### Managing the Tunnel

```bash
# Check tunnel status
docker compose logs cloudflared | grep -i 'registered\|connected'

# Restart tunnel
docker compose restart cloudflared
```

### Adding More Hostnames

In the Cloudflare Zero Trust dashboard:
1. Networks → Tunnels → haken → Public Hostname
2. Add a public hostname
3. Set Service to `http://zulip:80` (or another container in the stack)

### Rotating the Tunnel Token

1. Cloudflare Zero Trust → Networks → Tunnels → haken → Configure
2. Get a new token
3. Update `CLOUDFLARE_TUNNEL_TOKEN` in `.env`
4. `docker compose up -d --force-recreate cloudflared`

---

## Troubleshooting

### Zulip returns 500 "Configuration error: reverse proxies"

The `cloudflared` container's IP isn't in `LOADBALANCER_IPS`. Fix:
```yaml
# In compose.override.yaml, under zulip environment:
LOADBALANCER_IPS: "172.16.0.0/12"  # Trust entire Docker bridge
```
Then: `docker compose up -d --force-recreate zulip`

### SMTP authentication fails (535 BadCredentials)

- Verify 2-Step Verification is enabled on the Google account
- Use the **primary account email** (`ai@fovi.com`) not an alias (`ai@wiki3.ai`)
- Generate a fresh App Password at [myaccount.google.com](https://myaccount.google.com)
- Test with the Python snippet in the Health Checks section above

### Topic Summarize / AI features fail ("Egress proxying is denied")

Zulip routes **all outgoing HTTP through Smokescreen**, an SSRF proxy that blocks
private IP addresses by default. If your LLM backend (Ollama, local OpenAI-compatible
server, etc.) runs on the Docker host or in another container, Smokescreen will block it:

```
Egress proxying is denied to host 'host.docker.internal:11434':
  no valid IP found among resolved addresses -
  172.17.0.1 denied by rule 'Deny: Private Range'.
```

**Fix:** Add `CONFIG_http_proxy__allow_ranges` to the Zulip container environment
in `compose.override.yaml`:

```yaml
services:
  zulip:
    environment:
      # Allow Smokescreen to reach services on the Docker host
      CONFIG_http_proxy__allow_ranges: "172.17.0.0/16"
```

Then **recreate** the container (not just restart) and apply the Smokescreen config:

```bash
# 1. Recreate the container so it picks up the new env var
#    "docker compose restart" does NOT pick up env var changes!
docker compose up -d --force-recreate zulip

# 2. Wait for the container to be ready
sleep 10

# 3. Apply the Smokescreen config change (answer 'y' when prompted)
docker compose exec zulip /home/zulip/deployments/current/scripts/zulip-puppet-apply

# 4. Verify the config took effect
docker compose exec zulip grep -A2 http_proxy /etc/zulip/zulip.conf
# Should show: allow_ranges = 172.17.0.0/16

# 5. Verify Ollama is reachable from the container
docker compose exec zulip curl -s http://host.docker.internal:11434/api/tags | head -1
```

**Docs:** [Customizing the outgoing HTTP proxy](https://zulip.readthedocs.io/en/latest/production/deployment.html#customizing-the-outgoing-http-proxy)
and [`[http_proxy]` system configuration](https://zulip.readthedocs.io/en/latest/production/system-configuration.html#http-proxy).

### Cloudflare tunnel not connecting

```bash
# Check cloudflared logs
docker compose logs cloudflared

# Verify the token is correct
grep CLOUDFLARE_TUNNEL_TOKEN .env

# Force recreate the container
docker compose up -d --force-recreate cloudflared
```

### Zulip stuck in "health: starting"

```bash
# Check what's happening
docker compose logs -f zulip

# On first boot, database migrations can take several minutes
# Look for "entered RUNNING state" messages
```

### Container won't start after .env change

```bash
# Validate compose config
docker compose config

# Check for syntax errors in .env (no spaces around =)
cat .env
```

### Database issues

```bash
# Connect to PostgreSQL directly
docker compose exec database psql -U zulip

# Check database size
docker compose exec database psql -U zulip -c "SELECT pg_database_size('zulip');"
```

---

## Security Notes

- `.env` is in `.gitignore` — never commit it
- Docker credentials stored in `~/.docker/config.json` (plain text) — consider installing `docker-credential-secretservice` to use GNOME Keyring
- All passwords are randomly generated 48-character hex strings (except the Google App Password)
- SSL is handled by Cloudflare at the edge; Zulip only sees HTTP
- The `LOADBALANCER_IPS` setting trusts the entire Docker bridge subnet (`172.16.0.0/12`)

---

## References

- [Zulip Docker documentation](https://zulip.readthedocs.io/projects/docker/en/latest/)
- [Docker-zulip GitHub repo](https://github.com/zulip/docker-zulip)
- [Zulip production settings](https://zulip.readthedocs.io/en/stable/production/settings.html)
- [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Google App Passwords](https://support.google.com/accounts/answer/185833)
