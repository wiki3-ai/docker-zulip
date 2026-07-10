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

Hermes agent (on same Docker network)
  │
  └──► connects to zulip:80 internally
       sends Host: chat.wiki3.ai via patched zulip client
```

### Containers (7 total)

| Service       | Image                                | Purpose                             |
|---------------|--------------------------------------|-------------------------------------|
| `zulip`       | `ghcr.io/zulip/zulip-server:12.0-1` | Zulip web app + workers             |
| `database`    | `zulip/zulip-postgresql:14`         | PostgreSQL database                 |
| `memcached`   | `memcached:alpine`                   | Object cache                        |
| `rabbitmq`    | `rabbitmq:4.2`                       | Message queue for workers           |
| `redis`       | `redis:alpine`                       | Caching, rate limiting              |
| `cloudflared` | `cloudflare/cloudflared:latest`      | Cloudflare Tunnel connector         |
| `hermes`      | `hermes-agent:zulip-test` (local)    | Hermes AI agent (Zulip gateway bot) |

### Docker Volumes

| Volume                       | Contents                  |
|------------------------------|---------------------------|
| `zulip-docker_zulip`        | Uploaded files, avatars   |
| `zulip-docker_postgresql-14`| Database data             |
| `zulip-docker_rabbitmq`     | RabbitMQ data             |
| `zulip-docker_redis`        | Redis data                |

### Hermes Runtime Data

| Path                    | Contents                     |
|-------------------------|------------------------------|
| `~/.hermes/config.yaml` | Hermes configuration         |
| `~/.hermes/.env`        | API keys & Zulip credentials |
| `~/.hermes/sessions/`   | Chat session DB (SQLite)     |
| `~/.hermes/skills/`     | Loaded skills                |
| `~/.hermes/logs/`       | Agent logs                   |

---

## Project Files

```
/media/nvm4t2/Projects/zulip-docker/
├── .env                        # Secrets & tokens (gitignored)
├── compose.yaml                # Base compose (from upstream repo)
├── compose.override.yaml       # Zulip customizations (tracked in git)
├── compose.hermes.yaml         # Hermes agent service (tracked in git)
├── manage.py                   # Wrapper for Zulip management commands
├── Dockerfile                  # For local builds (not used — we use prebuilt image)
└── SETUP.md                    # ← This file

/media/nvm4t2/Projects/hermes-agent/
├── gateway/platforms/zulip.py  # Zulip gateway adapter (Host-header patched)
└── Dockerfile                  # Used to build hermes-agent:zulip-test

/home/jim/Projects/ownersclub-gateway/
├── docker-compose.yml          # OwnersClub portal + Squid cache proxy
├── squid/squid.conf            # Squid config (SSL bump + StoreID)
├── squid/hf-store-id.py        # StoreID helper for HF/Xet cache dedup
└── portal/                     # Caddy web portal files
```

### Running the Full Stack

```bash
cd /media/nvm4t2/Projects/zulip-docker
docker compose -f compose.yaml -f compose.override.yaml -f compose.hermes.yaml up -d
```

> The `compose.hermes.yaml` file adds the Hermes agent to the zulip-docker
> stack. Both projects share the same Docker network so Hermes can reach
> Zulip via internal DNS (`zulip:80`) with `Host: chat.wiki3.ai` injected
> by the patched zulip client library.

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

Because Hermes is in a separate compose file, always specify all three files:

```bash
cd /media/nvm4t2/Projects/zulip-docker
COMPOSE_FILES="-f compose.yaml -f compose.override.yaml -f compose.hermes.yaml"

# Start everything
docker compose $COMPOSE_FILES up -d

# Stop everything
docker compose $COMPOSE_FILES down

# Restart only Zulip (after config changes)
docker compose $COMPOSE_FILES restart zulip

# Restart Zulip picking up new .env values
docker compose $COMPOSE_FILES up -d --force-recreate zulip

# Restart only Hermes (after code/config changes)
docker compose $COMPOSE_FILES up -d --force-recreate hermes

# Restart everything
docker compose $COMPOSE_FILES down && docker compose $COMPOSE_FILES up -d

# Quick restart shortcut (most common: rebuild Hermes + restart)
docker compose $COMPOSE_FILES up -d --force-recreate hermes
```

> **Important:** `docker compose restart zulip` only restarts the process inside the
> existing container — it does **NOT** pick up environment variable changes from
> `compose.override.yaml`. Always use `docker compose up -d --force-recreate zulip`
> when you change environment variables, then run `zulip-puppet-apply` if the change
> affects `zulip.conf` settings (e.g. `CONFIG_*` variables).

### Rebuilding Hermes

When the Hermes gateway adapter or any source code changes:

```bash
cd /media/nvm4t2/Projects/hermes-agent

# Build a new image
docker build -t hermes-agent:zulip-test -f Dockerfile .

# Restart Hermes with the new image
cd /media/nvm4t2/Projects/zulip-docker
docker compose $COMPOSE_FILES up -d --force-recreate hermes

# Verify Zulip connection
docker logs hermes 2>&1 | grep -i zulip
# Should see: ✓ zulip connected
```

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

---

## Backups

Backups are stored on a secondary drive at `/media/wde26t1/Archives/`.

### What Gets Backed Up

| Source | What | Size (approx) | Why |
|--------|------|---------------|-----|
| Zulip PostgreSQL | `pg_dump -U zulip zulip` (SQL dump) | ~200 KB | Crash-consistent database restore |
| Zulip volumes | `zulip-docker_zulip` (uploads, avatars) | ~12 MB | Lost if volume is deleted |
| Zulip volumes | `zulip-docker_postgresql-14` (raw data dir) | ~16 MB | Redundant with SQL dump, raw restore option |
| Zulip volumes | `zulip-docker_rabbitmq` | ~100 KB | Message queue state |
| Zulip volumes | `zulip-docker_redis` | ~4 KB | Cache (regenerates) |
| Hermes home | `~/.hermes/` (config, sessions, skills) | ~10 MB | Bot identity, chat history, learned skills |
| Configs | `.env`, `compose.override.yaml`, `compose.hermes.yaml`, Hermes `config.yaml` | — | Environment snapshots |

### Quick Backup (One-Liner)

Run this any time after significant changes:

```bash
# Creates dated archive in /media/wde26t1/Archives/YYYY-MM-DD-zulip-hermes-backup/
sudo -E bash -c '
  D=/media/wde26t1/Archives/$(date +%F)-zulip-hermes-backup
  mkdir -p "$D"
  echo "=== Hermes data ==="
  tar czf "$D/hermes-data.tar.gz" \
    --exclude=skills --exclude=cache --exclude=audio_cache --exclude=.cache \
    -C ~ .hermes/
  echo "=== PostgreSQL dump ==="
  docker exec zulip-docker-database-1 pg_dump -U zulip zulip | gzip > "$D/zulip-postgres.sql.gz"
  echo "=== Volumes ==="
  for vol in zulip-docker_zulip zulip-docker_postgresql-14 zulip-docker_rabbitmq zulip-docker_redis; do
    name=${vol#zulip-docker_}
    tar czf "$D/zulip-volume-${name}.tar.gz" -C "/var/lib/docker/volumes/${vol}/_data" .
  done
  echo "=== Configs ==="
  cp /media/nvm4t2/Projects/zulip-docker/.env "$D/zulip-env.txt" 2>/dev/null
  cp /media/nvm4t2/Projects/zulip-docker/compose.override.yaml "$D/"
  cp /media/nvm4t2/Projects/zulip-docker/compose.hermes.yaml "$D/"
  cp ~/.hermes/.env "$D/hermes-env.txt" 2>/dev/null
  cp ~/.hermes/config.yaml "$D/hermes-config.yaml"
  echo "=== Done: $(du -sh "$D" | cut -f1) ==="
  ls -lh "$D"
'
```

### Source Code Backups

All four repos are backed up to `/media/wde26t1/` via `rsync`:

```bash
# Quick sync (run after committing changes)
rsync -a --delete --exclude=.venv --exclude=.git --exclude=__pycache__ \
  --exclude='*.pyc' --exclude=node_modules --exclude=.playwright \
  /media/nvm4t2/Projects/hermes-agent/ /media/wde26t1/Projects/hermes-agent/

rsync -a --delete --exclude=.git \
  /media/nvm4t2/Projects/zulip-docker/ /media/wde26t1/Projects/zulip-docker/

rsync -a --delete --exclude=.git \
  /media/nvm4t2/Projects/hermes-docker/ /media/wde26t1/Projects/hermes-docker/

rsync -a --delete --exclude=.git --exclude=squid/cache --exclude=squid/logs \
  --exclude=squid/ssl_db \
  /home/jim/Projects/ownersclub-gateway/ /media/wde26t1/Workspace/ownersclub-gateway/
```

---

## Restore from Backup

### Restore Database (from SQL dump)

```bash
cd /media/nvm4t2/Projects/zulip-docker
COMPOSE_FILES="-f compose.yaml -f compose.override.yaml -f compose.hermes.yaml"

# Stop Zulip
docker compose $COMPOSE_FILES stop zulip

# Drop and recreate the database
docker compose exec -T database psql -U zulip -c "DROP DATABASE IF EXISTS zulip;"
docker compose exec -T database psql -U zulip -c "CREATE DATABASE zulip OWNER zulip;"

# Restore from compressed dump
zcat /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup/zulip-postgres.sql.gz | \
  docker compose exec -T database psql -U zulip

# Restart Zulip
docker compose $COMPOSE_FILES start zulip
```

### Restore Zulip Uploaded Files (from volume tarball)

```bash
# Restore the zulip volume
docker run --rm -v zulip-docker_zulip:/target -v /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup:/backup alpine \
  sh -c "rm -rf /target/* && tar xzf /backup/zulip-volume-zulip.tar.gz -C /target"
```

### Full Disaster Recovery (from scratch on a new host)

```bash
# 1. Install Docker + Docker Compose
# 2. Clone repos (or restore from /media/wde26t1 backup)
git clone https://github.com/zulip/docker-zulip
cd docker-zulip
git checkout wiki3

# 3. Restore configs
cp /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup/zulip-env.txt .env
cp /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup/zulip-compose-override.yaml compose.override.yaml
cp /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup/zulip-compose-hermes.yaml compose.hermes.yaml

# 4. Create volumes and restore data
docker volume create zulip-docker_zulip
docker volume create zulip-docker_postgresql-14
docker volume create zulip-docker_rabbitmq
docker volume create zulip-docker_redis

docker run --rm -v zulip-docker_zulip:/target -v /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup:/backup alpine \
  tar xzf /backup/zulip-volume-zulip.tar.gz -C /target
docker run --rm -v zulip-docker_postgresql-14:/target -v /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup:/backup alpine \
  tar xzf /backup/zulip-volume-postgresql-14.tar.gz -C /target
# Repeat for rabbitmq and redis

# 5. Start the stack
docker compose -f compose.yaml -f compose.override.yaml -f compose.hermes.yaml up -d

# 6. Restore Hermes data
tar xzf /media/wde26t1/Archives/2026-07-09-zulip-hermes-backup/hermes-data.tar.gz -C ~/

# 7. Rebuild Hermes image (if source is also restored)
cd /media/nvm4t2/Projects/hermes-agent
docker build -t hermes-agent:zulip-test -f Dockerfile .

# 8. Recreate Hermes with new image
cd /media/nvm4t2/Projects/zulip-docker
docker compose -f compose.yaml -f compose.override.yaml -f compose.hermes.yaml up -d --force-recreate hermes
```

> **Prefer the SQL dump** over the raw PostgreSQL volume tarball. The SQL dump
> is version-independent and will work across Zulip upgrades. The raw volume
> tarball is a last resort if the SQL dump is unavailable.

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

## System Recovery

When Docker Compose itself isn't responding or the stack is broken:

### 1. Check Docker daemon

```bash
sudo systemctl status docker
sudo journalctl -u docker --no-pager -n 50

# If Docker is dead:
sudo systemctl restart docker
```

### 2. Force-kill stuck containers

```bash
# List all containers
docker ps -a

# Force remove stuck containers
docker rm -f zulip-docker-zulip-1 zulip-docker-database-1 \
  zulip-docker-memcached-1 zulip-docker-rabbitmq-1 \
  zulip-docker-redis-1 zulip-docker-cloudflared-1 hermes

# Prune dead resources
docker container prune -f
```

### 3. Wipe and restart the compose stack

```bash
cd /media/nvm4t2/Projects/zulip-docker
COMPOSE_FILES="-f compose.yaml -f compose.override.yaml -f compose.hermes.yaml"

# Full teardown
docker compose $COMPOSE_FILES down -v --remove-orphans 2>/dev/null

# Recreate everything from scratch
docker compose $COMPOSE_FILES up -d

# Check Zulip initialisation (can take 5-10 min on first boot)
docker compose logs -f zulip
# Look for: "done getpeercred" + "zulip-puppet-apply completed successfully"
```

### 4. Database recovery (if PostgreSQL won't start)

```bash
# Check PostgreSQL logs
docker compose logs database

# If the data directory is corrupt, restore from backup:
docker compose stop zulip
docker compose rm -f database

# Restore the volume from tarball (see Restore section above)
# Then restart:
docker compose $COMPOSE_FILES up -d
```

### 5. Hermes stuck / won't start

```bash
# Check Hermes logs
docker logs hermes 2>&1 | tail -40

# Common fix: wrong Zulip credentials or connection
# Verify environment:
docker exec hermes env | grep -i zulip
# Expected:
#   ZULIP_SITE_URL=http://zulip:80
#   ZULIP_EXTERNAL_HOST=chat.wiki3.ai
#   ZULIP_BOT_EMAIL=hermes-bot@chat.wiki3.ai
#   ZULIP_API_KEY=<redacted>

# Test connection manually:
docker exec hermes python3 -c "
import zulip, requests
from requests.adapters import HTTPAdapter

class _HostHeaderAdapter(HTTPAdapter):
    def send(self, request, **kwargs):
        request.headers['Host'] = 'chat.wiki3.ai'
        return super().send(request, **kwargs)

class _PatchedZulipClient(zulip.Client):
    def ensure_session(self):
        super().ensure_session()
        if self.session:
            adapter = _HostHeaderAdapter()
            self.session.mount('http://', adapter)
            self.session.mount('https://', adapter)

c = _PatchedZulipClient(site='http://zulip:80',
    email='hermes-bot@chat.wiki3.ai',
    api_key='$(docker exec hermes env | grep ZULIP_API_KEY | cut -d= -f2)')
r = c.get_profile()
print('OK' if r.get('result') == 'success' else r)
"

# If that fails, check if Zulip is reachable at all:
docker exec hermes curl -s -H 'Host: chat.wiki3.ai' http://zulip:80/api/v1/server_settings | head -5
```

### 6. OwnersClub Gateway / Squid recovery

```bash
# The Squid proxy runs independently from the zulip-docker stack.
cd /home/jim/Projects/ownersclub-gateway

# Restart
docker compose restart squid

# Full rebuild + restart
docker compose up -d --force-recreate squid

# Check logs
docker logs ownersclub-squid --tail 30

# Test proxy
curl -x http://localhost:3128 -sk -o /dev/null -w "%{http_code}" --max-time 10 https://huggingface.co/
```

### 7. Emergency access without Docker

If Docker itself is broken and you need to access the PostgreSQL data directly:

```bash
# PostgreSQL data is at:
sudo ls /var/lib/docker/volumes/zulip-docker_postgresql-14/_data/

# You can install PostgreSQL directly and point it at this directory:
sudo apt install postgresql-14
sudo pg_ctlcluster 14 main start
# But this is almost never needed — fixing Docker is easier.
```
````
