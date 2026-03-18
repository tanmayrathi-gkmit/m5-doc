# CI/CD Documentation Breakdown

Let me read the Dockerfile and .dockerignore first since they weren't in the document context. Good, now I have everything. Here's the full line-by-line breakdown of every file.


---

# Dockerfile

```dockerfile
FROM python:3.13-slim
```
The base image. `python:3.13-slim` is the official Python image built on Debian but with non-essential packages stripped out — about 50 MB vs 350 MB for the full image. You get Python pre-installed, nothing more.

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
 && rm -rf /var/lib/apt/lists/*
```
Three things chained in one `RUN` layer deliberately:
- `apt-get update` — refreshes the package index so apt knows what's available
- `apt-get install -y --no-install-recommends curl` — installs `curl` and nothing extra (`--no-install-recommends` skips suggested/recommended packages). `curl` is needed in the entrypoint healthcheck and by the deploy script
- `rm -rf /var/lib/apt/lists/*` — deletes the package index that was just downloaded. It's no longer needed and would just bloat the image layer. This is why all three commands are in **one** `RUN` — if you split them, the `rm` would be in a different layer and wouldn't actually reduce image size

**What changed from v1:** `gcc` and `libpq-dev` were removed. These were only needed to compile `psycopg2` from source. Since you use `psycopg2-binary`, it ships its own bundled libpq — no compilation needed. Removing them saves ~60 MB.


```dockerfile
COPY --from=ghcr.io/astral-sh/uv:0.10.11 /uv /usr/local/bin/uv
```
This is a multi-stage copy trick — it pulls the `uv` binary out of a published image without running that image. `--from=` can reference any image, not just a prior build stage. The binary lands at `/usr/local/bin/uv` so it's on `$PATH`.

**What changed from v1:** `latest` → `0.10.11`. A pinned version means your builds are reproducible — a new uv release can't silently break your pipeline.

```dockerfile
WORKDIR /app
```
Sets the working directory for all subsequent instructions (`COPY`, `RUN`, `CMD`). Also means when someone `docker exec`s into the container they land in `/app`. Creates the directory if it doesn't exist.

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
```
This is the **cache layer trick** — the most important optimization in this Dockerfile. Docker builds layers sequentially and caches each one. By copying only the dependency manifests *before* copying your source code, this layer only rebuilds when your dependencies change. If you only changed a Python file, Docker uses the cached layer here and skips reinstalling all packages. `--frozen` means fail if `uv.lock` is out of sync with `pyproject.toml` (no silent updates). `--no-dev` skips dev/test dependencies — test tools have no place in a production image.

```dockerfile
COPY . .
```
Copies your entire project source into `/app`. This comes *after* the dependency install layer deliberately — source code changes constantly, but dependencies change rarely. The `.dockerignore` file (explained below) controls what gets excluded.

```dockerfile
RUN mkdir -p /app/staticfiles
```
Pre-creates the directory that Django's `collectstatic` will write into. Without this, `collectstatic` would create it at runtime, but since the container's filesystem may have permission issues depending on the user, it's safer to guarantee it exists with correct ownership during build time.

```dockerfile
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
```
Copies the entrypoint script from your repo's `docker/` folder into the container root and makes it executable. It's placed at `/entrypoint.sh` (not inside `/app`) so it's clearly not application code.

```dockerfile
EXPOSE 8000
```
Documents that the container listens on port 8000. This doesn't actually open any port — it's metadata for humans and for tools like `docker-compose` to understand intent. The actual port binding happens in `docker-compose.yml` with `ports: "8000:8000"`.

```dockerfile
ENTRYPOINT ["/entrypoint.sh"]
```
The entrypoint script runs first, before anything else. Array syntax (`["..."]`) is used instead of string syntax so it runs directly without a shell wrapper — signals like SIGTERM are passed directly to the process, which matters for graceful shutdown.

```dockerfile
CMD ["uv", "run", "newrelic-admin", "run-program", "gunicorn", "gamer_house.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "2", \
     "--timeout", "120", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```
The default command passed to the entrypoint as `$@`. Each part:
- `uv run` — runs the command inside the virtual environment uv manages
- `newrelic-admin run-program` — wraps the process so New Relic APM can instrument it before Python starts
- `gunicorn gamer_house.wsgi:application` — your actual WSGI server
- `--bind 0.0.0.0:8000` — listen on all interfaces inside the container
- `--workers 2` — 2 worker processes (still low — see open issue below)
- `--timeout 120` — kill a worker if it doesn't respond in 120s (prevents hung workers)
- `--access-logfile -` and `--error-logfile -` — write logs to stdout/stderr so Docker captures them via `docker logs`

`CMD` is overridden per-service in `docker-compose.yml` — celery-worker and celery-beat replace it with their own commands.

---

# .dockerignore

```
.git
.gitignore
```
Excludes the entire git history and git config. `.git` can be hundreds of MB in older repos. None of this belongs in an image.

```
__pycache__/
*.pyc
*.pyo
*.pyd
```
Python's compiled bytecode cache. Rebuilds automatically inside the container for the correct Python version — including your local machine's cache could cause version mismatches.

```
.venv/
venv/
```
Your local virtual environment. Could be gigabytes. The container builds its own venv from scratch via `uv sync`.

```
.env
.env.*
```
**Critical.** Prevents your local secrets file from ever being baked into the image. Without this, a `COPY . .` would silently include your database passwords and API keys in the image layer — visible to anyone who pulls it.

```
logs/
*.log
```
Log files have no place in a build image and can be large.

```
.pytest_cache/
.ruff_cache/
htmlcov/
.coverage
```
Test and linting artifacts — only relevant locally.

```
.vscode/
node_modules/
```
Editor config and any JS dependencies — irrelevant to a Python container.

```
locustfile.py
load_test_tokens.json
core/management/commands/generate_load_test_tokens.py
core/management/commands/delete_load_test_genres.py
```
Load-testing tools excluded from production image. Smart — these expose load-test utilities that have no business being on the production server.

---

# entrypoint.sh

```bash
set -euo pipefail
```
Three safety flags: `e` = exit immediately on any error, `u` = treat unset variables as errors, `o pipefail` = if any command in a pipe fails, the whole pipe fails (without this, `false | true` would succeed).

```bash
until uv run python -c "
import sys, psycopg2, os
try:
    psycopg2.connect(...)
except psycopg2.OperationalError:
    sys.exit(1)
" 2>/dev/null; do
```
Polls PostgreSQL by actually attempting a connection using `psycopg2`. This is more reliable than just checking if the port is open (TCP open ≠ database accepting queries). `2>/dev/null` suppresses error output during retries.

```bash
  COUNT=$((COUNT + 1))
  if [ "$COUNT" -ge "$MAX_TRIES" ]; then
    echo "❌  PostgreSQL did not become ready in time. Exiting."
    exit 1
  fi
  sleep 2
```
Retry loop with a hard cap of 30 tries × 2 seconds = 60 seconds maximum wait before giving up and failing loudly.

```bash
if [ "${SKIP_DJANGO_SETUP:-false}" != "true" ]; then
```
The gate that separates the web container from celery-worker and celery-beat. Those services set `SKIP_DJANGO_SETUP: "true"` in `docker-compose.yml` so they skip migrations, collectstatic, and seeding — which would be wrong to run from multiple containers simultaneously.

```bash
uv run python manage.py migrate --noinput
```
Applies any pending database migrations. `--noinput` prevents Django from asking interactive questions — mandatory in automated deploys.

```bash
uv run python manage.py collectstatic --noinput
```
Copies all static files (CSS, JS, images from all apps) into `/app/staticfiles`, which is bind-mounted to the host so Nginx can serve them directly without going through Gunicorn.

```bash
USER_COUNT=$(uv run python -c "
...
print(get_user_model().all_objects.count())
")
if [ "$USER_COUNT" -eq "0" ]; then
    uv run python manage.py seed_db
fi
```
Checks if any users exist before seeding. Without this guard, every deploy would wipe and re-seed your database. Uses `all_objects` (presumably a custom manager that includes soft-deleted users) rather than `objects` to avoid re-seeding if only soft-deleted users exist.

```bash
exec "$@"
```
Replaces the shell process with whatever command was passed in (`$@` = all arguments). `exec` is critical — without it, the shell becomes PID 1 and wraps Gunicorn as a child process. With `exec`, Gunicorn becomes PID 1 directly and receives OS signals (SIGTERM for graceful shutdown) correctly.

---

# docker-compose.yml

```yaml
image: postgres:16
```
Pinned to major version 16 — won't accidentally upgrade to postgres 17 which could require a data migration.

```yaml
restart: always
```
On every service. Tells Docker to restart the container automatically if it crashes or if Docker itself restarts (e.g. after an EC2 reboot). Without this, a crashed container just stays dead.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_NAME}"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 10s
```
`start_period` is the grace period — Docker won't count failed healthchecks against the retry limit during this window, giving postgres time to initialize its data directory on first run.

```yaml
command: redis-server --save "" --appendonly no
```
Disables both RDB snapshots (`--save ""`) and AOF persistence (`--appendonly no`). Redis is only used as a broker and cache here — data doesn't need to survive restarts. Persistence would just waste disk I/O.

```yaml
image: gamerhouse_web
```
On celery-worker and celery-beat — reuses the exact same image built by the `web` service instead of building a second image. They're the same codebase so this makes sense. The image must already exist locally (pulled by `deploy.sh`) before compose runs.

```yaml
depends_on:
  web:
    condition: service_healthy
```
Workers only start after the web container passes its healthcheck — guaranteeing migrations have run before any worker tries to process a task.

```yaml
- celerybeat_data:/app/celerybeat
```
Persists the beat schedule file across container restarts. Without this volume, the scheduler loses its state every time the container restarts and could double-fire scheduled tasks.

---

# deploy.sh

```bash
set -euo pipefail
```
Same as entrypoint — any error aborts the deploy immediately rather than silently continuing in a broken state.

```bash
BRANCH="${DEPLOY_BRANCH:-dev}"
```
`:-dev` is bash default syntax — use `dev` if `DEPLOY_BRANCH` is unset or empty. Every variable at the top follows this pattern so the script can be run manually without passing all env vars.

```bash
sudo apt-get update -q
```
Still runs unconditionally on every deploy. This is the remaining issue from the original review — it runs even when nothing needs to be installed. The `-q` flag reduces output but doesn't fix the unnecessary run.

```bash
install_if_missing() {
  if ! command -v "$cmd" &>/dev/null; then
```
`command -v` checks if a binary exists on `$PATH` — returns non-zero if not found. `&>/dev/null` silences both stdout and stderr. This is how the script stays idempotent.

```bash
curl -fsSL https://get.docker.com | sudo sh
```
Docker's official one-liner install script. `-f` = fail silently on HTTP errors, `-s` = silent, `-S` = show errors, `-L` = follow redirects. Piping to `sudo sh` runs it as root.

```bash
sudo usermod -aG docker "$USER"
```
Adds the current user to the `docker` group so they can run docker commands without `sudo`. **This only takes effect on the next login** — for the current SSH session it's already too late. The script works because subsequent docker commands use `sudo` implicitly through the compose plugin.

```bash
curl -fsSL https://download.newrelic.com/infrastructure_agent/gpg/newrelic-infra.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/newrelic-infra.gpg
```
Downloads New Relic's GPG signing key and converts it from ASCII armored format to binary format (`--dearmor`) that apt's trusted keyring directory expects. This lets apt verify that packages from New Relic's repo are authentic.

```bash
echo "deb [arch=amd64] ..." | sudo tee /etc/apt/sources.list.d/newrelic-infra.list
```
Adds New Relic's apt repository. `tee` writes to a file that requires root permission — you can't just use `>` because the redirection happens before `sudo` gets involved.

```bash
cat <<EOF | sudo tee /etc/newrelic-infra.yml > /dev/null
license_key: ${NEW_RELIC_LICENSE_KEY}
log_format: json
passthrough_environment:
  - NR_LICENSE_KEY_ENV_VAR
EOF
```
Heredoc (`<<EOF`) writes a multi-line string. `> /dev/null` suppresses `tee`'s stdout echo — you don't want the license key printed in the deploy log. **What changed from v1:** the redundant systemd override block was removed.

```bash
install -m 600 /dev/null "$APP_DIR/.env"
```
Wait — looking at the file again, this fix was **not applied**. The `.env` is still written with `cat > "$APP_DIR/.env"` then `chmod 600` after. The race condition from the original review is still present.

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
```
`--password-stdin` reads the password from stdin instead of a command argument — arguments are visible in `ps aux`, stdin is not. The pipe from `echo` keeps the token out of the process list.

```bash
docker tag "$IMAGE_NAME" gamerhouse_web:latest
```
Creates a local alias. `docker-compose.yml` references `gamerhouse_web` as the image name for celery services — this tag is what makes that work without compose trying to build or pull `gamerhouse_web` from a registry.

```bash
docker image prune -f --filter "until=24h"
```
Removes dangling images older than 24 hours. On a small EC2 instance, old image layers accumulate and fill the disk. `-f` skips the confirmation prompt.

```bash
docker compose -f "$APP_DIR/docker-compose.yml" up -d --remove-orphans
```
`-d` = detached (background). `--remove-orphans` removes containers for services that are no longer defined in the compose file — important when you rename or remove a service, otherwise old containers linger.

```bash
until curl -sf "http://localhost:8000/health/" > /dev/null; do
```
Polls the health endpoint from the host (not inside the container). `-s` = silent, `-f` = fail on HTTP error codes. `> /dev/null` discards the response body. The script only proceeds to configure Nginx once the app is actually serving traffic.

```bash
sudo cp "$APP_DIR/nginx/gamerhouse.conf" "$NGINX_CONF"
sudo sed -i "s/__DOMAIN__/$DOMAIN/g" "$NGINX_CONF"
sudo ln -sf "$NGINX_CONF" "$NGINX_LINK"
sudo nginx -t && sudo systemctl reload nginx
```
Copies your Nginx template, replaces the `__DOMAIN__` placeholder with the actual domain, creates a symlink in `sites-enabled` (the standard Nginx pattern), tests the config syntax (`nginx -t`), and only reloads if the test passes. `&&` ensures reload only happens if the test succeeds — a bad config won't take down the running server.

```bash
sudo certbot --nginx -d "$DOMAIN" \
  --non-interactive --agree-tos \
  --email "${EMAIL_HOST_USER}" \
  --redirect
```
Obtains a Let's Encrypt certificate, automatically modifies the Nginx config to add HTTPS, and sets up HTTP→HTTPS redirect. `--non-interactive` + `--agree-tos` skips all prompts. This only runs once because of the `if [ ! -f "$CERT_PATH" ]` guard.

```bash
(sudo crontab -l 2>/dev/null; echo "0 3 * * * certbot renew ...") | sudo crontab -
```
Appends a cron entry to root's crontab. `crontab -l` lists existing entries, `2>/dev/null` suppresses the error if there's no existing crontab. The whole thing is piped back to `crontab -` which replaces the crontab. The `grep -qF` guard above prevents duplicates.

---

# deploy.yml

```yaml
on:
  push:
    branches: [dev, ci-test]
```
**Still not fixed** — `ci-test` still triggers a full production deploy. This was flagged in the previous review.

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```
If two pushes happen quickly, the older workflow run is cancelled. Prevents deploying stale code over a newer deploy.

```yaml
if: github.event_name == 'push' || github.event.pull_request.merged == true
```
The PR trigger fires on both `closed` and `merged` events. This condition filters it to only proceed if the PR was actually merged, not just closed/abandoned.

```yaml
permissions:
  contents: read
  packages: write
```
Principle of least privilege — the workflow token only gets the permissions it needs. `packages: write` is required to push to GHCR. Without explicit permissions, GitHub gives workflows broad access.

```yaml
cache-from: type=registry,ref=${{ env.IMAGE_NAME }}:latest
cache-to: type=inline
```
Uses the previously pushed image as a build cache source. `inline` embeds cache metadata into the image itself so it can be used next time. This is what makes subsequent builds fast — only changed layers rebuild.

```yaml
timeout: 600s
```
The SSH deploy step has a 10-minute timeout. Without this, a hung deploy (e.g. waiting for a package that never arrives) would block the runner forever.

---

| Issue | Status |
| :--- | :--- |
| `uv` pinned to specific version | ✅ Fixed — `0.10.11` |
| `gcc`/`libpq-dev` removed | ✅ Fixed |
| `.dockerignore` added | ✅ Fixed — comprehensive |
| Systemd NR override removed | ✅ Fixed |
| `.env` world-readable race condition | ❌ Still present |
| `ci-test` branch deploys to prod | ❌ Still present |
| `apt-get update` runs every deploy | ❌ Still present |
| No rollback on failed deploy | ❌ Still present |
| Gunicorn workers hardcoded at 2 | ❌ Still present |
| No non-root user in Dockerfile | ❌ Still present |