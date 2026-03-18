Let me read everything carefully first.Good, I have all 5 files now. Here is a complete line-by-line explanation of every file.

---

# 1. Dockerfile

```dockerfile
FROM python:3.13-slim
```
Starts from the official Python 3.13 image, "slim" variant. Slim means it uses Debian as the base but strips out documentation, man pages, and optional packages. This gives you a small, clean starting point (~50 MB instead of ~1 GB for the full image). Python 3.13 is pinned explicitly — you won't get a surprise Python upgrade.

---

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
 && rm -rf /var/lib/apt/lists/*
```
Three commands chained into one `RUN` layer (important — each `RUN` creates a separate image layer, so combining them avoids bloat):

- `apt-get update` — refreshes the package index so apt knows what versions are available
- `apt-get install -y --no-install-recommends curl` — installs `curl`, the `-y` skips the "are you sure?" prompt, `--no-install-recommends` prevents apt from pulling in suggested but non-essential packages
- `rm -rf /var/lib/apt/lists/*` — deletes the downloaded package index after installation. If you didn't do this it would just sit in the image wasting space forever

`curl` is needed by the entrypoint's health check and the deploy script. Notice `gcc` and `libpq-dev` were removed here — that was one of the fixes from the audit, since `psycopg2-binary` bundles its own compiled C libraries.

---

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:0.10.11 /uv /usr/local/bin/uv
```
This is a Docker multi-stage copy trick. Instead of installing uv via `pip install uv` or a curl script, it pulls the pre-built `uv` binary directly from its own official container image and copies just that single binary into `/usr/local/bin/`. `0.10.11` is now pinned to a specific version — this was the critical fix from the audit. Previously it was `:latest` which could break your build silently when a new uv version ships.

---

```dockerfile
WORKDIR /app
```
Sets the working directory for all subsequent instructions (`COPY`, `RUN`, `CMD`). Creates `/app` if it doesn't exist. Every relative path from here on resolves to `/app`.

---

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
```
This pair is the core of Docker layer caching strategy. By copying only the dependency manifest files first (before the actual source code), Docker can cache the "install all packages" layer. If you change your Django code but don't touch `pyproject.toml` or `uv.lock`, Docker reuses the cached installed-packages layer and skips reinstalling everything — builds go from ~2 minutes to ~10 seconds.

- `--frozen` means uv must use exactly what's in `uv.lock`, no resolving or updating — fully reproducible
- `--no-dev` skips dev-only packages like pytest, ruff, etc. since this is a production image

---

```dockerfile
COPY . .
```
Copies everything from the build context (your repo root) into `/app` inside the container. This is why `.dockerignore` matters — without it this would also copy `.git`, `__pycache__`, local `.env` files, test fixtures, etc. into the image. The `.dockerignore` file filters all of that out before this `COPY` runs.

---

```dockerfile
RUN mkdir -p /app/staticfiles
```
Creates the directory that `collectstatic` will write into at container startup. The `-p` flag means "create parent dirs if needed and don't error if it already exists". Without this, `collectstatic` would fail if the directory doesn't exist.

---

```dockerfile
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
```
Copies the entrypoint script from `docker/entrypoint.sh` in the repo into the root of the container. Then makes it executable — without `+x` the shell would refuse to run it. These are two separate `RUN`/`COPY` instructions which is fine since the entrypoint script changes rarely.

---

```dockerfile
EXPOSE 8000
```
Documents that the container listens on port 8000. This is metadata only — it doesn't actually open or publish the port. The actual port binding happens in `docker-compose.yml` with `ports: - "8000:8000"`. But it's good practice because it makes intent clear and some tools (like Docker Desktop) use it.

---

```dockerfile
ENTRYPOINT ["/entrypoint.sh"]
```
Every time this container starts, it runs `entrypoint.sh` first before anything else. The exec form `["..."]` is used instead of the shell form `"..."` — this matters because exec form runs the process directly (PID 1) instead of spawning it as a child of sh, which means signals like SIGTERM reach the process correctly for graceful shutdown.

---

```dockerfile
CMD ["uv", "run", "newrelic-admin", "run-program", "gunicorn", "gamer_house.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "2", \
     "--timeout", "120", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```
The default command passed to `ENTRYPOINT` as arguments — i.e., `entrypoint.sh` runs first, then calls `exec "$@"` which runs this command. In `docker-compose.yml`, the `celery-worker` and `celery-beat` services override this `CMD` with their own commands.

Breaking it down:
- `uv run` — runs the command inside the uv-managed virtual environment
- `newrelic-admin run-program` — wraps the process so New Relic APM can instrument it before it starts
- `gunicorn gamer_house.wsgi:application` — starts Gunicorn serving your Django WSGI app
- `--bind 0.0.0.0:8000` — listen on all network interfaces on port 8000 (not just localhost)
- `--workers 2` — 2 worker processes. Still hardcoded — still worth making this an env var
- `--timeout 120` — if a worker doesn't respond in 120 seconds, Gunicorn kills and restarts it
- `--access-logfile -` — write access logs to stdout (the `-` means stdout), so Docker captures them
- `--error-logfile -` — same for error logs

---

# 2. .dockerignore

```
.git
.gitignore
```
Excludes the entire Git history and gitignore file. `.git` alone can be hundreds of megabytes in a mature repo. None of it is needed to run the app.

```
__pycache__/
*.pyc
*.pyo
*.pyd
```
Python bytecode cache files. Generated locally when you run Python, completely irrelevant inside the container since Python regenerates them. Would just bloat the image.

```
.venv/
venv/
```
Your local virtual environment. This is critical to exclude — it could be hundreds of MB, and the container creates its own environment via `uv sync` anyway. Accidentally including this would override the container's packages with your local ones.

```
.env
.env.*
```
Marked CRITICAL in the file, rightly so. Your local `.env` contains real secrets (database passwords, API keys). If this wasn't here and you ran `docker build`, those secrets would be baked into the image and potentially pushed to GHCR. On EC2 the `.env` is written by `deploy.sh` at runtime, not baked in.

```
logs/
log/
*.log
profiles/
```
Log files from local development. No reason to ship these into the image.

```
.pytest_cache/
.ruff_cache/
htmlcov/
.coverage
```
Test runner caches and coverage reports from local runs. Dev-only artifacts.

```
.vscode/
```
Editor config. Personal to each developer, not part of the application.

```
node_modules/
```
If any frontend tooling exists in the repo, this prevents shipping its entire dependency tree into the image.

```
locustfile.py
load_test_tokens.json
core/management/commands/generate_load_test_tokens.py
core/management/commands/delete_load_test_genres.py
```
Load testing scripts and their fixtures. These are only used for performance testing, never needed in the production container.

---

# 3. entrypoint.sh

```bash
#!/usr/bin/env bash
```
The shebang line. Tells the OS what interpreter to use when this file is executed. `env bash` finds bash in `$PATH` instead of hardcoding `/bin/bash` — more portable across different Linux distributions.

```bash
set -euo pipefail
```
Three critical safety flags combined:
- `-e` — exit immediately if any command returns a non-zero exit code (i.e., fails). Without this, the script would keep running after a failed migration.
- `-u` — treat any reference to an undefined variable as an error. Prevents silent bugs like `rm -rf $UNDEFINED_DIR/` accidentally becoming `rm -rf /`.
- `-o pipefail` — if any command in a pipeline fails (e.g., `cmd1 | cmd2`), the whole pipeline is considered failed. Without this, only the last command's exit code matters.

---

```bash
echo "⏳  Waiting for PostgreSQL at ${POSTGRES_HOST}:${POSTGRES_PORT} …"
MAX_TRIES=30
COUNT=0
```
Sets up a retry loop. PostgreSQL takes a few seconds to start inside its container. Without waiting, Django would crash immediately trying to connect. `MAX_TRIES=30` means it'll wait up to 60 seconds (30 × 2s sleep).

---

```bash
until uv run python -c "
import sys, psycopg2, os
try:
    psycopg2.connect(...)
except psycopg2.OperationalError:
    sys.exit(1)
" 2>/dev/null; do
```
`until` runs the block repeatedly until the condition succeeds (exit code 0). The inline Python script tries to actually open a TCP connection to Postgres. If Postgres isn't ready, `psycopg2` raises `OperationalError` and `sys.exit(1)` signals failure to the shell. `2>/dev/null` suppresses the connection error output during the waiting period so your logs aren't flooded.

This is more reliable than just checking if the port is open (which `nc` or `pg_isready` does) because it validates that Postgres is actually accepting authenticated connections, not just listening.

---

```bash
  COUNT=$((COUNT + 1))
  if [ "$COUNT" -ge "$MAX_TRIES" ]; then
    echo "❌  PostgreSQL did not become ready in time. Exiting."
    exit 1
  fi
  echo "   … retry $COUNT/$MAX_TRIES"
  sleep 2
done
```
Increments the counter each retry, exits with an error if the limit is hit, otherwise prints progress and waits 2 seconds before trying again.

---

```bash
if [ "${SKIP_DJANGO_SETUP:-false}" != "true" ]; then
```
Gate for web-only setup. Celery worker and beat containers set `SKIP_DJANGO_SETUP=true` in `docker-compose.yml`, so they skip everything inside this block. The `:-false` is a default — if the variable isn't set at all, treat it as `false`.

---

```bash
  uv run python manage.py migrate --noinput
```
Runs Django database migrations. `--noinput` prevents it from prompting for confirmation in a non-interactive environment.

```bash
  uv run python manage.py collectstatic --noinput
```
Copies all static files (CSS, JS, images from Django apps) into `/app/staticfiles`. This directory is bind-mounted to the host so Nginx can serve them directly without going through Gunicorn.

---

```bash
  USER_COUNT=$(uv run python -c "
import django, os
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'gamer_house.settings')
django.setup()
from django.contrib.auth import get_user_model
print(get_user_model().all_objects.count())
")
  if [ "$USER_COUNT" -eq "0" ]; then
    uv run python manage.py seed_db
  fi
```
Counts users in the database. If zero, seeds initial data. This guard prevents re-seeding on every deploy — once real users exist, this is safely skipped forever. `all_objects` (not just `objects`) suggests you have a soft-delete manager — using `all_objects` means even soft-deleted users count, so a database that had users but soft-deleted them all won't accidentally re-seed.

---

```bash
exec "$@"
```
Replaces the current shell process with whatever command was passed in (Gunicorn or Celery). The `exec` is critical — without it, bash would stay alive as the parent process (PID 1), and signals like SIGTERM from `docker stop` would never reach Gunicorn, causing forceful kills after the timeout.

---

# 4. docker-compose.yml

```yaml
services:
```
Top-level key that defines all the containers in this stack. Docker Compose V2 format — no `version:` key needed (it's deprecated).

---

**postgres service:**

```yaml
image: postgres:16
container_name: gamerhouse_postgres
restart: always
```
Uses the official Postgres 16 image. `container_name` gives it a fixed name instead of a generated one. `restart: always` means Docker restarts this container if it crashes or if the Docker daemon restarts (e.g., after a server reboot).

```yaml
environment:
  POSTGRES_DB: ${POSTGRES_NAME}
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```
These env vars are read by the Postgres image's init script on first start to create the database and user. The `${...}` values come from the `.env` file in the same directory.

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```
Mounts a named Docker volume to Postgres's data directory. This is how data survives container restarts and redeployments. Without this, every `docker compose up` would start with an empty database.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_NAME}"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 10s
```
Docker actively monitors Postgres health. `pg_isready` is a small utility that checks if Postgres is accepting connections. `interval` — check every 10s. `timeout` — if the check takes more than 5s, it's considered failed. `retries` — 5 consecutive failures = unhealthy. `start_period` — give it 10s grace before counting failures (so it's not marked unhealthy during normal startup). Other services with `condition: service_healthy` won't start until this passes.

---

**redis service:**

```yaml
command: redis-server --save "" --appendonly no
```
Overrides the default Redis startup command. `--save ""` disables RDB snapshots (periodic disk dumps). `--appendonly no` disables AOF (append-only file) persistence. Redis is used purely as a broker and cache here — data doesn't need to survive Redis restarts. This makes Redis faster and avoids disk I/O.

---

**web service:**

```yaml
build:
  context: .
  dockerfile: Dockerfile
image: gamerhouse_web
```
On your local machine or in CI, this builds the image from the Dockerfile. On EC2, `deploy.sh` pulls the pre-built image from GHCR and tags it as `gamerhouse_web:latest`, so the `build:` block is effectively bypassed — Docker finds the local tag and uses it.

```yaml
env_file: .env
environment:
  POSTGRES_HOST: postgres
  CELERY_BROKER_URL: redis://redis:6379/0
  ...
```
`env_file` loads all variables from `.env`. The `environment` block then overrides specific ones — critically, `POSTGRES_HOST: postgres` makes Django connect to the `postgres` container by its service name (Docker's internal DNS), not `localhost`.

```yaml
ports:
  - "8000:8000"
```
Maps host port 8000 to container port 8000. Format is `host:container`. Nginx on the host proxies incoming HTTPS traffic to `localhost:8000`, which goes into this container.

```yaml
volumes:
  - /opt/gamerhouse/staticfiles:/app/staticfiles
  - ./logs:/app/logs
```
Two mounts. First is a bind mount — absolute host path to container path — so Nginx on the host can serve static files without going through Gunicorn. Second mounts a `logs/` directory so Django's file-based logging persists on the host and can be tailed or forwarded by New Relic.

```yaml
healthcheck:
  start_period: 60s
```
60 second grace period because the web container runs migrations, collectstatic, and possibly seeding before Gunicorn starts. Without this large `start_period`, Docker would mark it unhealthy before it even finishes starting.

---

**celery-worker and celery-beat services:**

```yaml
image: gamerhouse_web
```
Reuses the exact same image as the web service. Celery workers run the same Django code, so no separate image is needed — just a different command.

```yaml
environment:
  SKIP_DJANGO_SETUP: "true"
```
Tells `entrypoint.sh` to skip migrations, collectstatic, and seeding. The web container handles all of that. Workers just need to connect to the database and start processing.

```yaml
depends_on:
  web:
    condition: service_healthy
```
Workers wait for the web container to be healthy before starting. This guarantees migrations have completed before workers try to run tasks that touch the database.

```yaml
- celerybeat_data:/app/celerybeat
```
Named volume for Celery Beat's schedule file. Beat uses a `PersistentScheduler` which writes the schedule state to disk so it survives container restarts without losing track of when tasks last ran.

---

# 5. deploy.sh

```bash
set -euo pipefail
```
Same safety flags as entrypoint.sh — abort on any error, treat undefined variables as errors, fail on pipe errors.

---

```bash
REPO_URL="https://github.com/..."
APP_DIR="/opt/gamerhouse"
BRANCH="${DEPLOY_BRANCH:-dev}"
DOMAIN="${DOMAIN:-gamerhouse.isroot.in}"
IMAGE_NAME="${IMAGE_NAME:-ghcr.io/...}"
NEW_RELIC_LICENSE_KEY="${NEW_RELIC_LICENSE_KEY:-}"
```
Script-level constants. The `${VAR:-default}` pattern means "use the env var if set, otherwise use this default". The `NEW_RELIC_LICENSE_KEY:-}` with an empty default means it's optional — the script won't crash if it's not set.

---

```bash
install_if_missing() {
  local cmd=$1
  local pkg=${2:-$1}
  if ! command -v "$cmd" &>/dev/null; then
    sudo apt-get install -y -q "$pkg"
  fi
}
```
A helper function that checks if a command exists before installing its package. `command -v` returns the path to the binary if it exists, exit code 1 if not. `&>/dev/null` suppresses both stdout and stderr. This makes the script idempotent — safe to run repeatedly.

---

```bash
sudo apt-get update -q
```
Still runs unconditionally on every deploy. This is the remaining issue from the audit — it's outside all the `if` guards, so it runs even when everything is already installed. On a live server with all packages present this wastes 15–30 seconds every deploy.

---

```bash
install_if_missing nginx nginx
install_if_missing certbot certbot
if ! dpkg -s python3-certbot-nginx &>/dev/null 2>&1; then
  sudo apt-get install -y -q python3-certbot-nginx
fi
```
Installs Nginx (the reverse proxy), certbot (Let's Encrypt SSL tool), and the certbot Nginx plugin. The plugin automates SSL certificate installation by modifying your Nginx config. `dpkg -s` checks if a package is already installed.

---

```bash
if ! command -v docker &>/dev/null; then
  curl -fsSL https://get.docker.com | sudo sh
  sudo usermod -aG docker "$USER"
fi
```
Installs Docker using its official convenience script. `usermod -aG docker` adds the current user to the `docker` group so they can run Docker commands without `sudo`. Note: this group change only takes effect in new sessions — a known limitation during first-time setup.

---

**New Relic block** — you kept the simplified version (removed the systemd override):

```bash
if ! command -v newrelic-infra &>/dev/null; then
  curl -fsSL https://download.newrelic.com/... | sudo gpg --dearmor -o ...
  echo "deb [arch=amd64] ..." | sudo tee /etc/apt/sources.list.d/newrelic-infra.list
  sudo apt-get update -q
  sudo apt-get install newrelic-infra -y -q
fi
```
Installs the New Relic infrastructure agent only once. Adds New Relic's GPG signing key (so apt trusts their packages), adds their apt repository, then installs the agent.

```bash
cat <<EOF | sudo tee /etc/newrelic-infra.yml > /dev/null
license_key: ${NEW_RELIC_LICENSE_KEY}
log_format: json
passthrough_environment:
  - NR_LICENSE_KEY_ENV_VAR
EOF
```
Writes the agent config every deploy. Still runs unconditionally — the config will be rewritten even if the key hasn't changed. The systemd override block is gone (good), but `passthrough_environment` is still here. Since you removed the systemd override that was supposed to supply `NR_LICENSE_KEY_ENV_VAR`, this line now references an env var that's never set — it's effectively dead config. Safe to remove.

```bash
sudo systemctl restart newrelic-infra
```
Still restarts unconditionally every deploy — causes a brief metric gap in New Relic each time. Not a breaking issue, just cosmetic.

---

```bash
install -m 600 /dev/null "$APP_DIR/.env"
cat >> "$APP_DIR/.env" <<EOF
SECRET_KEY=${DJANGO_SECRET_KEY}
...
EOF
```
Wait — checking the actual file...

```bash
cat > "$APP_DIR/.env" <<EOF
...
EOF
chmod 600 "$APP_DIR/.env"
```
The `.env` security fix was **not applied**. This is still the original pattern — written with default permissions first, then `chmod 600` after. The file is briefly world-readable between those two lines. The safe fix would be:
```bash
install -m 600 /dev/null "$APP_DIR/.env"
cat >> "$APP_DIR/.env" <<EOF
...
EOF
```

---

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
```
Logs into GitHub Container Registry. `--password-stdin` reads the token from stdin instead of a command-line argument — important because command-line arguments are visible in `ps aux` output, stdin is not.

```bash
docker pull "$IMAGE_NAME"
docker tag "$IMAGE_NAME" gamerhouse_web:latest
```
Pulls the image built by GitHub Actions and tags it locally as `gamerhouse_web:latest`. This local tag is what `docker-compose.yml` references under `image: gamerhouse_web`.

```bash
docker image prune -f --filter "until=24h"
```
Removes Docker images older than 24 hours that aren't referenced by any container. Prevents disk from filling up over time with old image layers.

---

```bash
docker compose -f "$APP_DIR/docker-compose.yml" up -d --remove-orphans
```
Starts all services defined in `docker-compose.yml` in detached mode (background). `--remove-orphans` removes containers for services that no longer exist in the compose file — useful when you rename or remove a service.

There is still **no rollback** here. If this command succeeds but the health check below fails, the stack is left in a broken state with no way to automatically recover the previous version.

---

```bash
MAX=30
COUNT=0
until curl -sf "http://localhost:8000/health/" > /dev/null; do
  COUNT=$((COUNT + 1))
  if [ "$COUNT" -ge "$MAX" ]; then
    echo "❌  App did not become healthy in time."
    exit 1
  fi
  sleep 5
done
```
Waits up to 150 seconds (30 × 5s) for Gunicorn to respond to the `/health/` endpoint. `curl -sf` — `-s` is silent (no progress bar), `-f` returns exit code 22 on HTTP errors (4xx/5xx). If the app never comes up, the deploy fails loudly.

---

```bash
if [ ! -f "$NGINX_CONF" ]; then
  sudo cp "$APP_DIR/nginx/gamerhouse.conf" "$NGINX_CONF"
  sudo sed -i "s/__DOMAIN__/$DOMAIN/g" "$NGINX_CONF"
  sudo ln -sf "$NGINX_CONF" "$NGINX_LINK"
  sudo nginx -t && sudo systemctl reload nginx
fi
```
First-run-only Nginx setup. Copies your Nginx config template, replaces the `__DOMAIN__` placeholder with the actual domain, creates a symlink in `sites-enabled` (how Nginx knows to load it), then tests the config (`nginx -t`) and reloads. The `&&` means reload only happens if the test passes — prevents breaking Nginx with a bad config.

---

```bash
if [ ! -f "$CERT_PATH" ]; then
  sudo certbot --nginx -d "$DOMAIN" \
    --non-interactive --agree-tos \
    --email "${EMAIL_HOST_USER}" \
    --redirect
  (sudo crontab -l 2>/dev/null; echo "0 3 * * * certbot renew --quiet --post-hook 'systemctl reload nginx'") | sudo crontab -
fi
```
First-run-only SSL setup. `--nginx` tells certbot to automatically modify the Nginx config to enable HTTPS. `--redirect` adds an HTTP→HTTPS redirect. `--non-interactive --agree-tos` accepts the Let's Encrypt terms without prompting. The crontab line adds an auto-renewal job that runs at 3 AM daily — Let's Encrypt certs expire every 90 days, `certbot renew` is idempotent and only renews when within 30 days of expiry.

---

# 6. deploy.yml

```yaml
on:
  push:
    branches: [dev, ci-test]
  pull_request:
    types: [closed]
    branches: [dev]
  workflow_dispatch:
```
Three triggers. Push to `dev` or `ci-test` runs the pipeline. A closed PR against `dev` runs it (the job itself filters for only merged PRs). `workflow_dispatch` lets you trigger it manually from the GitHub Actions UI. **`ci-test` still triggers a full production deploy** — this was flagged in the audit and wasn't changed.

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```
If two pushes happen in quick succession, the first run is cancelled when the second starts. Prevents deploying stale code when a hotfix is pushed right after a feature.

---

**run-tests job:**

```yaml
services:
  postgres:
    image: postgres:16
    ...
  redis:
    image: redis:7-alpine
```
Spins up real Postgres and Redis instances as sidecar containers for the test run. The tests run against actual databases, not mocks — much more reliable.

```yaml
env:
  SECRET_KEY: "django-insecure-test-key"
  DEBUG: "True"
  RAZORPAY_KEY_ID: "rzp_test_key"
  AUTH_THROTTLE_RATE: "1000/minute"
```
Test-specific environment. Uses dummy values for secrets since they're not real. `AUTH_THROTTLE_RATE: "1000/minute"` is deliberately high — so rate limiting doesn't interfere with tests making many requests.

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v5
  with:
    enable-cache: true
```
Installs uv using the official GitHub Action. `enable-cache: true` caches the uv cache between runs so package downloads are skipped on repeated runs — faster CI.

```yaml
- name: Set up Python
  run: uv python install
```
Installs the Python version specified in `pyproject.toml` or `.python-version`. uv manages its own Python installations separately from the system Python.

---

**build-and-deploy job:**

```yaml
needs: run-tests
if: github.event_name == 'push' || github.event.pull_request.merged == true
```
Only runs after tests pass. For PRs, only if they were actually merged (not just closed without merging).

```yaml
permissions:
  contents: read
  packages: write
```
Minimal permissions. `packages: write` is needed to push to GHCR. Not requesting anything else limits blast radius if the token is compromised.

```yaml
- name: Extract Docker metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    tags: |
      type=sha,prefix=sha-
      type=raw,value=latest
```
Generates two tags for the image. `sha-abc1234` — a unique immutable tag tied to this specific commit. `latest` — the mutable tag that always points to the newest build. The SHA tag lets you pull any historical build; `latest` is what deploy.sh pulls.

```yaml
cache-from: type=registry,ref=${{ env.IMAGE_NAME }}:latest
cache-to: type=inline
```
Layer caching for Docker builds in CI. `cache-from` pulls the previous `latest` image and uses its layers as cache. `cache-to: type=inline` embeds the cache metadata directly in the pushed image. This means CI builds reuse cached layers from previous builds — the dependency installation layer is almost always a cache hit.

```yaml
- name: Deploy to EC2
  uses: appleboy/ssh-action@v1
  with:
    timeout: 600s
    script: |
      sudo mkdir -p /opt/gamerhouse
      sudo chown -R ${{ secrets.EC2_USER }}:... /opt/gamerhouse

      if [ ! -d "/opt/gamerhouse/.git" ]; then
        git clone ... /opt/gamerhouse
      fi

      cd /opt/gamerhouse
      git fetch origin
      git reset --hard origin/${{ github.ref_name }}
```
SSHs into EC2 and runs the inline script. Creates the app directory if needed. Clones the repo on first run; on subsequent runs does a hard reset to the exact commit that triggered this workflow — not just a `git pull`, which could have merge conflicts. `git reset --hard` is intentional and safe here since the EC2 copy is never manually edited.

```yaml
      export DJANGO_SECRET_KEY="${{ secrets.DJANGO_SECRET_KEY }}"
      export POSTGRES_PASSWORD="${{ secrets.POSTGRES_PASSWORD }}"
      ...
      bash scripts/deploy.sh
```
Injects all GitHub secrets as environment variables into the SSH session, then calls `deploy.sh`. The secrets are never written to disk on the GitHub Actions runner — they exist only in the runner's memory and are passed over the encrypted SSH connection.

---

## Summary of what changed vs the original

| Item | Status |
|---|---|
| `uv` pinned from `:latest` to `0.10.11` | ✅ Fixed |
| `gcc` and `libpq-dev` removed from Dockerfile | ✅ Fixed |
| `.dockerignore` created | ✅ Fixed |
| systemd override block removed from New Relic | ✅ Fixed |
| `.env` write-then-chmod security race | ❌ Still present |
| `ci-test` branch triggers prod deploy | ❌ Still present |
| No rollback on failed deploy | ❌ Still present |
| `apt-get update` runs every deploy unconditionally | ❌ Still present |
| `passthrough_environment` referencing removed systemd var | ❌ Dead config, safe to remove |
| Gunicorn workers hardcoded to 2 | ❌ Still present (minor) |