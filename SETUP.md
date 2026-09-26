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

Zulip's AI features and Hermes both also call a local **Ollama** instance. That
runs as a host systemd service, **not** as a container in this stack — see
[Local LLM Backend (Ollama)](#local-llm-backend-ollama).

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

### Local LLM Backend (Ollama)

Zulip's AI features (topic summarization) and the Hermes agent both call a local
Ollama instance. Ollama is **not** part of this compose stack — it runs as a host
systemd service and containers reach it via `host.docker.internal`.

| Item           | Value                                                 |
|----------------|-------------------------------------------------------|
| Service        | `ollama.service` (system user `ollama`, enabled)      |
| Listen address | `0.0.0.0:11434` (`OLLAMA_HOST`)                       |
| Unit override  | `/etc/systemd/system/ollama.service.d/override.conf`  |
| Model          | `qwen3.6:35b-a3b` (Q4_K_M, ~26 GB)                    |
| GPU            | RTX 4060 Ti, **8 GB VRAM**                            |

Who calls it:

| Consumer | Where it is configured                                                   |
|----------|--------------------------------------------------------------------------|
| Zulip    | `compose.override.yaml` → `TOPIC_SUMMARIZATION_MODEL` / `..._PARAMETERS` |
| Hermes   | `~/.hermes/config.yaml` → `custom_providers` / `ollama_num_ctx`          |

#### GPU configuration

The card has 8 GB of VRAM but the model is ~26 GB, so it runs as a **partial
offload**: as many layers as fit go on the GPU, the remainder stays on the CPU.
This only works when the NVIDIA kernel modules are loaded for the *running*
kernel — see [Ollama runs on CPU](#ollama-runs-on-cpu--nvidia-smi-fails) if they
are not.

Recommended `/etc/systemd/system/ollama.service.d/override.conf`:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
# Model is ~26 GB on an 8 GB card. One slot keeps the whole
# per-slot KV cache inside the VRAM budget.
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
# KV cache quantization (below) requires flash attention.
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
# 100k context needs ~9 GB of KV cache per slot at f16, which cannot fit.
Environment="OLLAMA_CONTEXT_LENGTH=32768"
```

Apply it with:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

> **Keep `ollama_num_ctx` in `~/.hermes/config.yaml` in sync** with
> `OLLAMA_CONTEXT_LENGTH`. It is currently `100000`, which overrides Ollama's
> default and would request a KV cache far larger than the card can hold.

#### Verifying the GPU is in use

```bash
# 1. The driver can see the card
nvidia-smi

# 2. Ollama detected a GPU - want library=CUDA,
#    NOT: library=cpu ... total_vram="0 B"
journalctl -u ollama --no-pager | grep "inference compute" | tail -1

# 3. The loaded model is resident on the GPU - want size_vram > 0
curl -s http://localhost:11434/api/ps | grep -o '"size_vram":[0-9]*'

# 4. Generation throughput
journalctl -u ollama -f | grep print_timing
```

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
# Back up first, outside the repo so a stray commit cannot pick it up
cp .env "/media/wde26t1/Archives/env-backup.$(date +%Y%m%d)"

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

Backups are stored on a secondary drive at `/media/wde26t1/Archives/`, written by
the `backup-zulip-hermes` script in this repo and run daily by a systemd timer.

A failed run leaves no directory behind, so the newest directory is always the last
**successful** backup. Check either:

```bash
systemctl list-timers backup-zulip-hermes.timer
ls -1d /media/wde26t1/Archives/*-zulip-hermes-backup | tail -3
```

### What Gets Backed Up

Sizes are from a real run.

| Source | What | Size | Why |
|--------|------|------|-----|
| Zulip PostgreSQL | `pg_dump -U zulip zulip` (SQL dump) | ~370 KB | Crash-consistent database restore |
| Zulip volumes | `zulip-docker_zulip` (uploads, avatars) | ~14 MB | Lost if volume is deleted |
| Zulip volumes | `zulip-docker_postgresql-14` (raw data dir) | ~17 MB | Redundant with SQL dump, raw restore option |
| Zulip volumes | `zulip-docker_rabbitmq` | ~170 KB | Message queue state |
| Zulip volumes | `zulip-docker_redis` | ~313 B | Cache (regenerates) |
| Hermes home | `~/.hermes/` — config, sessions, **skills** | ~28 MB | Bot identity, chat history, learned skills |
| Configs | `.env`, `compose.override.yaml`, `compose.hermes.yaml`, Hermes `.env` and `config.yaml` | ~18 KB | Environment snapshots |

About 58 MB per run.

`~/.hermes/skills/` is included deliberately. It is learned local state, not a git
checkout, so it cannot be re-fetched — an earlier version of this document excluded
it, which silently dropped it from every backup.

### Running a Backup

```bash
cd /media/nvm4t2/Projects/zulip-docker
./backup-zulip-hermes
```

Writes `/media/wde26t1/Archives/YYYY-MM-DD-zulip-hermes-backup/`. Safe to re-run:
running it again on the same day replaces that day's directory. The directory only
appears once every archive has been written and verified, so **a directory that
exists is always a complete backup**. Nothing requires `sudo`, because the Docker
daemon reads the root-owned volume data rather than the calling user.

| Variable | Default | Meaning |
|----------|---------|---------|
| `BACKUP_MOUNT` | `/media/wde26t1` | Mountpoint of the archive drive |
| `BACKUP_DEVICE` | `/dev/sdc1` | Device that mountpoint must resolve to |
| `BACKUP_ROOT` | `${BACKUP_MOUNT}/Archives` | Where dated directories are written |

The script refuses to run unless the archive drive is mounted **and** resolves to
`BACKUP_DEVICE`. Without that check, an unmounted drive would send all 58 MB into
the root filesystem and still look like a successful backup.

It also verifies every archive by decompressing it, requires the PostgreSQL dump to
end with `PostgreSQL database dump complete`, and writes `SHA256SUMS` so bit-rot can
be detected later:

```bash
cd /media/wde26t1/Archives/YYYY-MM-DD-zulip-hermes-backup && sha256sum -c SHA256SUMS
```

Everything is written mode `0600` inside a `0700` directory, because the archive
holds credentials and a full database dump.

### Automating the Backup (systemd timer)

The unit files live in `systemd/`. Install them once:

```bash
cd /media/nvm4t2/Projects/zulip-docker
sudo install -m 0644 systemd/backup-zulip-hermes.service \
                      systemd/backup-zulip-hermes.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now backup-zulip-hermes.timer
```

It runs daily at 03:30 as `jim` (deliberately not 00:00, which already has
dpkg-db-backup, logrotate and fstrim), with `Persistent=true` so a run missed while
the machine was off happens at the next boot.

To run it now and read the result:

```bash
sudo systemctl start backup-zulip-hermes.service
journalctl -u backup-zulip-hermes.service -n 40
```

The service is `Type=oneshot` and exits non-zero if any part of the backup fails, so
a bad run shows up in `systemctl --failed` rather than being silently skipped.

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

The examples below use `$BACKUP` for the backup directory. Set it to the one you are
restoring from, and check the contents before you start — a directory only exists
once the backup completed, but pick the right date deliberately rather than assuming
the newest is the one you want:

```bash
BACKUP=$(ls -1d /media/wde26t1/Archives/*-zulip-hermes-backup | tail -1)
echo "Restoring from: $BACKUP"
ls -l "$BACKUP"
sha256sum -c "$BACKUP/SHA256SUMS"   # verify the archives are intact first
```

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
zcat "$BACKUP/zulip-postgres.sql.gz" | \
  docker compose exec -T database psql -U zulip

# Restart Zulip
docker compose $COMPOSE_FILES start zulip
```

### Restore Zulip Uploaded Files (from volume tarball)

```bash
# Restore the zulip volume
docker run --rm -v zulip-docker_zulip:/target -v "$BACKUP":/backup alpine \
  sh -c "rm -rf /target/* && tar xzf /backup/zulip-volume-zulip.tar.gz -C /target"
```

### Full Disaster Recovery (from scratch on a new host)

```bash
# 1. Install Docker + Docker Compose
# 2. Clone the fork (the wiki3 branch exists only on the fork, not upstream)
git clone https://github.com/wiki3-ai/docker-zulip
cd docker-zulip
git checkout wiki3

# 3. Restore configs
cp "$BACKUP/zulip-env.txt" .env
cp "$BACKUP/zulip-compose-override.yaml" compose.override.yaml
cp "$BACKUP/zulip-compose-hermes.yaml" compose.hermes.yaml

# 4. Create volumes and restore data
docker volume create zulip-docker_zulip
docker volume create zulip-docker_postgresql-14
docker volume create zulip-docker_rabbitmq
docker volume create zulip-docker_redis

docker run --rm -v zulip-docker_zulip:/target -v "$BACKUP":/backup alpine \
  tar xzf /backup/zulip-volume-zulip.tar.gz -C /target
docker run --rm -v zulip-docker_postgresql-14:/target -v "$BACKUP":/backup alpine \
  tar xzf /backup/zulip-volume-postgresql-14.tar.gz -C /target
# Repeat for rabbitmq and redis

# 5. Start the stack
docker compose -f compose.yaml -f compose.override.yaml -f compose.hermes.yaml up -d

# 6. Restore Hermes data
tar xzf "$BACKUP/hermes-data.tar.gz" -C ~/

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

### Ollama runs on CPU / `nvidia-smi` fails

`nvidia-smi` failing with *"couldn't communicate with the NVIDIA driver"* means no
NVIDIA kernel module is loaded. Ollama then falls back to CPU **silently** — it still
works, just ~20x slower, and nothing in the Zulip UI reports a problem. Tell-tale
signs in `journalctl -u ollama`:

```
msg="inference compute" id=cpu library=cpu ... total_vram="0 B"
```

The cause is a **kernel upgrade whose driver modules were never installed**. Ubuntu
builds `linux-modules-nvidia-<driver>-<kernel>` per kernel version, and each one
pins `nvidia-kernel-common-<driver>` to an **exact** version:

```
Depends: nvidia-kernel-common-595 (>= 595.84), nvidia-kernel-common-595 (<= 595.84-1)
```

Because that pin is exact, a module package becomes uninstallable as soon as Ubuntu
supersedes the driver in the archive. On this host:

| Module package                                   | Pins driver | Archive offers           |
|--------------------------------------------------|-------------|--------------------------|
| `linux-modules-nvidia-595-open-7.0.0-28-generic` | `595.84-1`  | `595.71.05`, `595.91.07` |
| `linux-modules-nvidia-595-open-7.0.0-34-generic` | `595.91.07` | `595.91.07`              |

So installing modules for the kernel you are *already running* cannot work, and fails
like this:

```
linux-modules-nvidia-595-open-7.0.0-28-generic : Depends: nvidia-kernel-common-595
  (>= 595.84) but 595.71.05-0ubuntu0.24.04.1 is to be installed
```

**Diagnose:**

```bash
uname -r                                     # the kernel you are booted into
modinfo nvidia | head -3                     # "Module nvidia not found" == broken
ls /dev/nvidia*                              # should exist; "No such file" == broken
dpkg -l 'linux-modules-nvidia-*' | grep ^ii  # which kernels have modules built?
```

**Fix:** the only self-consistent version set is the **current kernel together with
the current driver**. Let apt move both at once, then reboot:

```bash
sudo apt update
sudo apt full-upgrade     # driver + new kernel + its matching signed modules
sudo reboot
```

`full-upgrade` (not `upgrade`) is required: moving to a new kernel means *adding*
`linux-image-*` packages, which plain `upgrade` will never do. Inspect the plan first:

```bash
apt-get -s full-upgrade | grep -E "^(Inst|Remv)" | grep -i nvidia
```

On this host that plan upgraded the driver stack to `595.91.07`, installed
`linux-modules-nvidia-595-open-7.0.0-34-generic`, and removed the stale
`linux-modules-nvidia-595-open-6.17.0-35-generic`.

**Expect SSH to drop mid-upgrade.** `tailscale` is itself part of the upgrade, and
restarting it tears down the connection. The `apt` transaction keeps running to
completion in the background, so verify it rather than re-running it:

```bash
grep -E "^(Start-Date|End-Date)" /var/log/apt/history.log | tail -2  # matched pair = finished
dpkg --audit                                                         # empty == clean state
```

If `sudo reboot` then reports *"Operation inhibited by APT"*, that inhibitor is
usually a leftover process rather than a live transaction. Confirm no `apt`/`dpkg`
process is running and no `APT` row appears in `systemd-inhibit --list`, then clear
it:

```bash
sudo kill <pid-from-the-message>
sudo reboot                      # or, if you have verified nothing is running: systemctl reboot -i
```

**After rebooting,** confirm the GPU is live before restarting Ollama:

```bash
uname -r                        # expect the new kernel
nvidia-smi
sudo systemctl restart ollama
journalctl -u ollama --no-pager | grep "inference compute" | tail -1
```

> **Secure Boot is enabled on this host.** Stay on Ubuntu's prebuilt module packages
> as above — they are signed by Canonical and load with no extra setup. Avoid DKMS
> (`nvidia-dkms-*`), whose self-built modules Secure Boot rejects until you enrol a
> Machine Owner Key via `mokutil --import` — an interactive process not worth the
> risk on a remote host. Verify a plan is DKMS-free with
> `apt-get -s full-upgrade | grep -i dkms`.

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
