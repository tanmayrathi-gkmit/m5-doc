# GamerHouse — CI/CD & Docker Infrastructure

> **Stack:** Django · Gunicorn · Celery · PostgreSQL · Redis · Docker · GitHub Actions · EC2

---

## Table of Contents

1. [Overview](#1-overview)
2. [Pipeline Triggers](#2-pipeline-triggers)
3. [CI Test Job](#3-ci-test-job-run-tests)
4. [Build & Push Job](#4-build--push-job-build-and-deploy)
5. [Deploy to EC2](#5-deploy-to-ec2-deploysh)
6. [Container Entrypoint](#6-container-entrypoint-entrypointsh)
7. [Docker Compose Stack](#7-docker-compose-stack)
8. [GitHub Secrets Reference](#8-github-secrets-reference)
9. [Observability](#9-observability)
10. [Rollback & Troubleshooting](#10-rollback--troubleshooting)

---

## 1. Overview

GamerHouse uses a fully automated CI/CD pipeline built on GitHub Actions. Every push to the `dev` branch (or merged pull request) triggers a test run, a Docker image build, and a zero-downtime deployment to an AWS EC2 instance.

The pipeline is **idempotent** — it can be run multiple times safely and bootstraps all infrastructure dependencies on first run.

```
Developer pushes to dev
        │
        ▼
  GitHub Actions
  ┌─────────────────────────────────────────────────────────┐
  │  Job 1: run-tests                                       │
  │    Postgres + Redis service containers                  │
  │    uv install → pytest                                  │
  └───────────────────────┬─────────────────────────────────┘
                          │  (pass)
                          ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Job 2: build-and-deploy                                │
  │    Login GHCR → Build image → Push → SSH EC2            │
  │    deploy.sh → Compose up → Health check                │
  └─────────────────────────────────────────────────────────┘
                          │
                          ▼
              EC2: Docker Compose stack
         web · postgres · redis · celery-worker · celery-beat
                          │
                          ▼
              Nginx reverse proxy (host)
              HTTPS via Let's Encrypt
```

The production runtime is a Docker Compose stack with five services: a Django/Gunicorn web server, PostgreSQL, Redis, a Celery worker, and Celery Beat. Nginx on the EC2 host acts as a reverse proxy and serves static files directly.

---

## 2. Pipeline Triggers

Defined in `.github/workflows/deploy.yml`:

| Event | Condition |
|---|---|
| `push` | Any push to `dev` or `ci-test` branch |
| `pull_request` | PR closed against `dev` **and** `merged == true` |
| `workflow_dispatch` | Manual trigger from the GitHub Actions UI |

### Concurrency

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

If a new push arrives while a deploy is already running, the older run is **cancelled immediately** to prevent stale-code deployments.

---

## 3. CI Test Job (`run-tests`)

Runs on `ubuntu-latest`. Spins up two **service containers** alongside the test runner:

| Service | Image | Health check |
|---|---|---|
| PostgreSQL | `postgres:16` | `pg_isready` |
| Redis | `redis:7-alpine` | `redis-cli ping` |

### Steps

| # | Step | Detail |
|---|---|---|
| 1 | Checkout repository | `actions/checkout@v4` |
| 2 | Install uv | `astral-sh/setup-uv@v5` with layer caching |
| 3 | Set up Python 3.13 | `uv python install` (version from `pyproject.toml`) |
| 4 | Install dependencies | `uv sync --all-extras --dev` (locked versions) |
| 5 | Run Pytest | `uv run pytest` — full suite against live service containers |

### Test environment variables

All values are isolated test placeholders — no production secrets are used:

```
DJANGO_SETTINGS_MODULE=gamer_house.settings
SECRET_KEY=django-insecure-test-key
DEBUG=True
POSTGRES_HOST=localhost / POSTGRES_PORT=5432
CELERY_BROKER_URL=redis://localhost:6379/0
AUTH_THROTTLE_RATE=1000/minute
```

> **Gate:** The `build-and-deploy` job only runs if this job succeeds.

---

## 4. Build & Push Job (`build-and-deploy`)

Requires `packages: write` permission to push to GHCR.

### 4.1 Dockerfile

Base image: `python:3.13-slim`. Uses `uv` pinned to `0.10.11` for reproducibility.

```dockerfile
# Layer order (cache-optimised)
COPY pyproject.toml uv.lock ./   # deps layer
RUN  uv sync --frozen --no-dev   # only rebuilds on dep changes

COPY . .                         # source layer

ENTRYPOINT ["/entrypoint.sh"]
CMD ["uv", "run", "newrelic-admin", "run-program", "gunicorn",
     "gamer_house.wsgi:application",
     "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "120"]
```

Key decisions:

- **Only `curl`** is installed as a system dep — needed by the entrypoint health wait
- **No dev dependencies** in the image (`--no-dev`)
- **Static files directory** created at build time; populated at runtime by `collectstatic`

### 4.2 Build steps

| # | Step | Action |
|---|---|---|
| 1 | Login to GHCR | `docker/login-action@v3` using `GITHUB_TOKEN` |
| 2 | Extract metadata | `docker/metadata-action@v5` → produces `sha-{SHA}` and `latest` tags |
| 3 | Set up Docker Buildx | `docker/setup-buildx-action@v3` |
| 4 | Build & push | `docker/build-push-action@v6` with registry cache |

### 4.3 Build cache strategy

```yaml
cache-from: type=registry,ref=ghcr.io/.../gamerhouse:latest
cache-to:   type=inline
```

Cache layers are stored **inside the image itself** (`inline`). Subsequent builds pull the previous `latest` image and reuse unchanged layers, keeping build times fast without a separate cache backend.

---

## 5. Deploy to EC2 (`deploy.sh`)

After the image is pushed, the workflow SSHs into EC2 (`appleboy/ssh-action@v1`, 600s timeout) and runs `scripts/deploy.sh`. All GitHub Secrets are exported as environment variables before the script runs.

The script is **idempotent** — every section checks whether the resource already exists before acting.

### 5.1 System bootstrap *(first run only)*

```bash
# Installed if missing (uses command -v / dpkg -s checks)
nginx
certbot + python3-certbot-nginx
docker          # via get.docker.com convenience script
docker compose  # v2 plugin (docker-compose-plugin)
newrelic-infra  # via official New Relic APT repo
```

### 5.2 New Relic configuration *(every run)*

`/etc/newrelic-infra.yml` is **rewritten on every deploy** to keep the license key current:

```yaml
license_key: <NEW_RELIC_LICENSE_KEY>
log_format: json
```

Log forwarding (`/etc/newrelic-infra/logging.d/gamerhouse.yml`) is configured once, pointing at `/opt/gamerhouse/logs/application.log` with tags `service: gamerhouse-web` and `environment: production`. The agent is restarted after every deploy.

### 5.3 `.env` file

Written fresh on every deploy from the injected secrets, then immediately `chmod 600`:

| Variable group | Variables |
|---|---|
| Django core | `SECRET_KEY`, `DEBUG=False`, `ALLOWED_HOSTS` |
| Database | `POSTGRES_NAME/USER/PASSWORD/HOST/PORT` |
| Celery / Redis | `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND`, `REDIS_CACHE_URL`, task time limits |
| Email | `EMAIL_HOST/PORT/USER/PASSWORD`, `DEFAULT_FROM_EMAIL` |
| Payments | `RAZORPAY_KEY_ID/KEY_SECRET/WEBHOOK_SECRET` |
| New Relic | `NEW_RELIC_LICENSE_KEY`, `NEW_RELIC_APP_NAME`, log settings |
| Logging | `LOG_LEVEL=INFO`, `LOG_FILE_ENABLED=True`, `LOG_CONSOLE_ENABLED=False` |

### 5.4 Image pull and tagging

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
docker pull "$IMAGE_NAME"                        # ghcr.io/.../gamerhouse:latest
docker tag  "$IMAGE_NAME" gamerhouse_web:latest  # local alias for Compose
docker image prune -f --filter "until=24h"       # free disk space
```

### 5.5 Compose stack restart

```bash
docker compose -f /opt/gamerhouse/docker-compose.yml up -d --remove-orphans
```

Only recreates containers whose image or config has changed. `--remove-orphans` removes containers for services no longer in the Compose file.

### 5.6 Health check

```bash
until curl -sf "http://localhost:8000/health/"; do
  # retries up to 30× with 5s delay (150s total)
done
```

If the app does not return HTTP 200 within 150 seconds, the deploy **aborts with a non-zero exit code**, failing the GitHub Actions step and leaving the old container running.

### 5.7 Nginx configuration *(first run only)*

```bash
sudo cp "$APP_DIR/nginx/gamerhouse.conf" /etc/nginx/sites-available/gamerhouse
sudo sed -i "s/__DOMAIN__/$DOMAIN/g" /etc/nginx/sites-available/gamerhouse
sudo ln -sf /etc/nginx/sites-available/gamerhouse /etc/nginx/sites-enabled/gamerhouse
sudo nginx -t && sudo systemctl reload nginx
```

The Nginx config proxies all traffic to `127.0.0.1:8000` (Docker web container) with 60/120s timeouts. `/static/` is served **directly from the bind-mount** at `/opt/gamerhouse/staticfiles/` with 30-day cache headers, bypassing Gunicorn entirely.

### 5.8 SSL certificate *(first run only)*

```bash
sudo certbot --nginx -d "$DOMAIN" \
  --non-interactive --agree-tos \
  --email "${EMAIL_HOST_USER}" \
  --redirect
```

Certbot modifies the Nginx config to add HTTPS and redirect HTTP → HTTPS. A daily cron is added:

```cron
0 3 * * * certbot renew --quiet --post-hook 'systemctl reload nginx'
```

---

## 6. Container Entrypoint (`entrypoint.sh`)

Every container starts `/entrypoint.sh` before its main process. The script has two phases.

### 6.1 PostgreSQL readiness wait

Uses a real `psycopg2` connection attempt (not just a TCP ping) — up to **30 retries × 2s = 60 seconds**:

```bash
until uv run python -c "import psycopg2, os; psycopg2.connect(...)"; do
  sleep 2
done
```

### 6.2 Django setup *(web service only)*

Skipped when `SKIP_DJANGO_SETUP=true` (set by Celery services):

```bash
python manage.py migrate --noinput        # apply pending migrations
python manage.py collectstatic --noinput  # populate /app/staticfiles

# Seed only on a fresh installation
if [ "$USER_COUNT" -eq "0" ]; then
  python manage.py seed_db
fi
```

> **Ordering:** Celery containers `depends_on: web: condition: service_healthy`, so migrations always run before any worker starts.

### 6.3 Process handoff

```bash
exec "$@"
```

Hands off to whichever `CMD` was specified — Gunicorn for the web service, Celery commands for worker/beat.

---

## 7. Docker Compose Stack

```
┌─────────────────── EC2 host ───────────────────────────────┐
│                                                            │
│  Nginx (host process)  ←──── HTTPS :443 / HTTP :80        │
│       │                                                    │
│       │ proxy_pass :8000       /static/ → bind-mount       │
│       ▼                                                    │
│  ┌──────────┐   SQL    ┌───────────┐                       │
│  │   web    │─────────▶│ postgres  │  volume: postgres_data │
│  │ Gunicorn │          └───────────┘                       │
│  │  :8000   │   Redis  ┌───────────┐                       │
│  └──────────┘─────────▶│   redis   │  no persistence        │
│       ▲                └───────────┘                       │
│       │ service_healthy                                    │
│  ┌──────────────┐  ┌───────────────┐                       │
│  │celery-worker │  │  celery-beat  │  volume: celerybeat_data│
│  │ concurrency=2│  │ Persistent    │                       │
│  └──────────────┘  │ Scheduler     │                       │
│                    └───────────────┘                       │
└────────────────────────────────────────────────────────────┘
```

### Service summary

| Service | Image | Key config |
|---|---|---|
| `postgres` | `postgres:16` | Volume: `postgres_data`. Health: `pg_isready`. Always restarts. |
| `redis` | `redis:7-alpine` | No persistence (`--save '' --appendonly no`). Health: `redis-cli ping`. |
| `web` | `gamerhouse_web` (local) | Gunicorn on `:8000`. Bind-mount `staticfiles`. Health: `/health/` endpoint. |
| `celery-worker` | `gamerhouse_web` (reused) | `SKIP_DJANGO_SETUP=true`. `--concurrency=2`. Depends on `web` healthy. |
| `celery-beat` | `gamerhouse_web` (reused) | `PersistentScheduler`. Volume: `celerybeat_data` for schedule file. |

> The Celery services **reuse the same image** as `web` — built once, referenced three times. This ensures worker code is always in sync with the app.

### Volume mounts

| Volume / bind | Purpose |
|---|---|
| `postgres_data` (named) | Persists PostgreSQL data across container restarts |
| `celerybeat_data` (named) | Persists Celery Beat's schedule file |
| `/opt/gamerhouse/staticfiles` (bind) | Shared between `web` container and host Nginx |
| `./logs` (bind) | All container logs written to host filesystem for New Relic ingestion |

---

## 8. GitHub Secrets Reference

Configure all secrets under **Settings → Secrets and variables → Actions**:

| Secret | Purpose |
|---|---|
| `EC2_HOST` | Public IP or hostname of the EC2 instance |
| `EC2_USER` | SSH username (e.g. `ubuntu`) |
| `EC2_SSH_KEY` | Private SSH key for EC2 access (PEM format) |
| `GHCR_PAT` | GitHub PAT with `read:packages` scope (used by EC2 to pull from GHCR) |
| `DJANGO_SECRET_KEY` | Django `SECRET_KEY` for production |
| `POSTGRES_NAME` / `POSTGRES_USER` / `POSTGRES_PASSWORD` | Database credentials |
| `EMAIL_HOST` / `EMAIL_PORT` / `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` / `DEFAULT_FROM_EMAIL` | SMTP configuration |
| `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` / `RAZORPAY_WEBHOOK_SECRET` | Payment gateway credentials |
| `FRONTEND_URL` | Origin URL used for CORS and email links |
| `NEW_RELIC_LICENSE_KEY` | New Relic ingest license key |

> `GITHUB_TOKEN` is auto-provided by Actions — no manual setup needed for pushing to GHCR from the workflow itself.

---

## 9. Observability

### 9.1 New Relic APM

The Django app runs under `newrelic-admin run-program gunicorn ...`, instrumenting the process with the New Relic Python agent. Application-level log forwarding to New Relic is **disabled** (`NEW_RELIC_APPLICATION_LOGGING_FORWARDING_ENABLED=false`) to avoid double-ingestion — the Infrastructure Agent handles log forwarding separately.

### 9.2 New Relic Infrastructure Agent

Installed on the **EC2 host** (not inside Docker). Monitors host-level metrics (CPU, memory, disk, network) and forwards application logs:

```yaml
# /etc/newrelic-infra/logging.d/gamerhouse.yml
logs:
  - name: gamerhouse-app
    file: /opt/gamerhouse/logs/application.log
    attributes:
      service: gamerhouse-web
      environment: production
```

### 9.3 Log configuration

| Setting | Production value |
|---|---|
| `LOG_LEVEL` | `INFO` |
| `LOG_FILE_ENABLED` | `True` (writes to `/app/logs/application.log`) |
| `LOG_CONSOLE_ENABLED` | `False` |

The `./logs` directory is bind-mounted into **all containers** (web, celery-worker, celery-beat), so logs from all services appear in the same files on the host.

---

## 10. Rollback & Troubleshooting

### 10.1 Manual rollback

```bash
# SSH into EC2
ssh -i key.pem ubuntu@<EC2_HOST>

# Pull a specific previous build
docker pull ghcr.io/tanmayrathi-gkmit/gamerhouse:sha-<COMMIT_SHA>

# Retag and restart
docker tag ghcr.io/tanmayrathi-gkmit/gamerhouse:sha-<COMMIT_SHA> gamerhouse_web:latest
docker compose -f /opt/gamerhouse/docker-compose.yml up -d
```

### 10.2 Useful commands

```bash
# Check all service statuses
docker compose -f /opt/gamerhouse/docker-compose.yml ps

# Tail logs for specific services
docker compose -f /opt/gamerhouse/docker-compose.yml logs --tail=100 web
docker compose -f /opt/gamerhouse/docker-compose.yml logs --tail=100 celery-worker
docker compose -f /opt/gamerhouse/docker-compose.yml logs --tail=100 celery-beat

# Restart a single service
docker compose -f /opt/gamerhouse/docker-compose.yml restart web

# Run a management command inside the web container
docker compose -f /opt/gamerhouse/docker-compose.yml exec web uv run python manage.py shell

# Check Nginx config
sudo nginx -t
sudo systemctl status nginx

# Test SSL renewal
sudo certbot renew --dry-run
```

### 10.3 Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Health check times out on deploy | Migrations failed or Gunicorn crashed | `docker compose logs web` |
| Celery not processing tasks | Redis unreachable or wrong `CELERY_BROKER_URL` | Verify `.env` and `redis` service health |
| Static files return 404 | `collectstatic` didn't run or bind-mount mismatch | Confirm `/opt/gamerhouse/staticfiles/` is populated on host |
| SSL cert renewal fails | Certbot cron not configured or port 80 blocked | `sudo certbot renew --dry-run` |
| GHCR pull fails on EC2 | `GHCR_PAT` expired or missing `read:packages` scope | Regenerate PAT and update the GitHub Secret |
| New Relic not receiving data | License key wrong or agent not running | `sudo systemctl status newrelic-infra` |
| Container exits immediately | Missing env var or bad `.env` syntax | Check `docker compose logs <service>` for startup errors |
