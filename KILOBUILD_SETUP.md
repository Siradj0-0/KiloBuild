# KiloBuild setup and operations guide

This guide explains the secrets, status links, logs, and KiloBuilder connection from first boot onward.

## 1. Private management and public status access

KiloBuild keeps the management API private and exposes a separate, status-only
listener for client dashboards. Configure the public listener on a different
port from the management API:

```bash
cd /opt/kilobuild
KILOBUILD_API_HOST=127.0.0.1 \
KILOBUILD_PUBLIC_STATUS_HOST=0.0.0.0 \
KILOBUILD_PUBLIC_STATUS_PORT=3002 \
KILOBUILD_PUBLIC_BASE_URL=http://<public-ip>:3002 \
bash scripts/install-vps.sh
```

Only the status page and `/api/status/<rotating-token>` are served on the
public listener. The public page is read-only: it does not expose project
administration, logs, secrets, or container controls. The management API
remains on `127.0.0.1`; put a reverse proxy in front of port 3002 when HTTPS is
available. Never expose port 3001 directly to clients.

## 2. The three secrets and what they do

### `apiSecret`

`apiSecret` protects the normal management API and CLI operations: listing projects, starting and stopping containers, reading logs, rebuilding, backups, and administration.

On the first daemon boot, KiloBuild generates a strong random value when one is not already configured. It prints the value once in the first-time setup block. It is saved in the daemon config file under the configured data directory, normally:

```text
.kilobuild/config.json
```

Keep it on the operator side. Put it in the CLI environment as `KILOBUILD_API_SECRET`; do not put it in a client project container.

### `KILOBUILD_INTERNAL_TOKEN`

This is the separate credential reserved for the KiloBuilder Discord bot. It only authorizes:

```text
GET /internal/status-url/{projectId}
```

KiloBuild generates it on first boot if it is missing, prints it once beside the API secret, and saves it in the same config file. If it is lost, read the `internalToken` value from the daemon's config file or replace it through the operator configuration flow and restart the bot with the new value.

Do not give this token to client bots. The KiloBuilder bot is the only container that should receive it.

### Per-project rotating status token

Each project has its own random 256-bit status token. It is stored in the embedded SQLite database in the `status_tokens` table. Tokens rotate every 15 minutes; the previous token remains valid for two minutes so an open page does not fail during the handoff.

Operators and KiloBuilder should not read the database directly. Ask the internal endpoint for the current URL. A status URL is a bearer credential: anyone who has it can view that project's public, read-only status page.

### Public status URL

The daemon intentionally listens on `127.0.0.1` by default so the management API is
not exposed to the internet. For a VPS status page, set the externally reachable
origin once in the global configuration:

```bash
kilo config set-global publicBaseUrl "https://status.example.com"
```

If the site is served directly by KiloBuild, use the public status listener
address and port instead, for example `http://169.58.126.151:3002`, and make
sure the firewall allows that port. A reverse proxy is preferred for HTTPS. The
VPS installer builds the status frontend and serves it from the status listener.

Do not use `http://127.0.0.1:3001` or `http://localhost:3001` as a public base URL:
those addresses only work on the VPS itself.

For direct access from the internet, reinstall the service with the public
status bind address and public URL. This keeps the setting repeatable after
reboots:

```bash
cd /opt/kilobuild
KILOBUILD_API_HOST=127.0.0.1 \
KILOBUILD_PUBLIC_STATUS_HOST=0.0.0.0 \
KILOBUILD_PUBLIC_STATUS_PORT=3002 \
KILOBUILD_PUBLIC_BASE_URL=http://169.58.126.151:3002 \
bash scripts/install-vps.sh
```

Replace the URL with the real HTTPS domain when a reverse proxy is in front of
the status listener.

## 3. What normal logs look like

### First daemon boot

Working:

```text
KiloBuild — first-time setup
API Secret (admin, general API):       <value>
Internal Token (KiloBuilder bot ONLY): <value>
Save these now — they will not be printed again.
[boot] OK Docker 27.5.1 (/var/run/docker.sock)
[boot] OK Embedded SQLite database is ready
[boot] Rotating status URLs: every 15 minutes with 2-minute grace
```

Needs attention:

```text
[boot] ERROR Docker is not reachable
```

The daemon can still start for database-only work, but container operations will fail until Docker is available.

### Normal project startup

Working:

```text
[kilobuild] Starting <project>...
[kilobuild] Node v22.x
[kilobuild] Launching: node index.js
[kilobuild] <project> running (PID 42)
```

Startup failures always begin with an explicit error line:

```text
[kilobuild] ERROR: required env var 'DATABASE_URL' is not set
[kilobuild] ERROR: dependency installation failed
[kilobuild] ERROR: build failed
[kilobuild] ERROR: migration failed
[kilobuild] ERROR: application exited during launch (code 1)
```

View these with `kilo logs <project>`.

### Project environment files

KiloBuild reads environment variables from the project directory when it
creates a container. The lookup order is:

1. `.env` in the project root
2. `config.env` in the project root
3. `.kilobuild/config.env`
4. Environment values saved through KiloBuild configuration

The first file that exists wins, and values from `.env` override saved
configuration values. The parser accepts normal `KEY=value` dotenv lines,
optional `export`, quoted values, blank lines, comments, and secrets
containing `#` or `=`. If the file changes, the next `kilo start` or
`kilo restart` recreates the container so Docker does not keep stale values.

### Crash and recovery

Working crash detection:

```text
[lifecycle] container crashed
```

The status page reports `Offline` and `Crashed` when the container exits with a non-zero code without an explicit operator stop. If the configured recovery policy allows it, recovery events also record the attempt and backoff in the events log.

An intentional stop reports `normal` instead, so it is not mislabeled as a crash.

### URL rotation

Working:

```text
[status] Rotated project status URLs
```

An old URL can continue working briefly during the two-minute grace period. An expired or invalid token returns the same generic `404 Status page not found` response.

## 4. Connect the KiloBuilder Discord bot

### Step 1: set only these two bot variables

In the KiloBuilder bot container, set:

```text
KILOBUILD_API_BASE_URL=http://<vps-host>:<daemon-port>
KILOBUILD_INTERNAL_TOKEN=<the Internal Token from first boot>
```

Do not set `KILOBUILD_INTERNAL_TOKEN` in any client bot.

If the bot uses the normal project-list endpoint, also provide the operator API bearer secret to that bot through its own protected secret store:

```text
KILOBUILD_API_SECRET=<the API Secret from first boot>
```

### Step 2: list projects

Use the API secret for the normal API:

```bash
curl "$KILOBUILD_API_BASE_URL/api/projects" \
  -H "Authorization: Bearer $KILOBUILD_API_SECRET"
```

Example response:

```json
[
  {
    "id": "project-uuid",
    "name": "billing-bot",
    "slug": "billing-bot",
    "status": "running"
  }
]
```

Save the matching `id`. Names and IDs are not interchangeable in the internal URL route.

### Step 3: fetch the current status link

```bash
curl "$KILOBUILD_API_BASE_URL/internal/status-url/project-uuid" \
  -H "Authorization: Bearer $KILOBUILD_INTERNAL_TOKEN"
```

Example response:

```json
{
  "projectId": "project-uuid",
  "url": "https://status.example.com/status/<rotating-token>",
  "expiresAt": "2026-08-05T20:15:00.000Z"
}
```

Post the returned `url` to Discord. Fetch it again whenever KiloBuilder needs a fresh link; never cache it permanently.

## 5. Useful operator commands

```bash
kilo list
kilo logs <project>
kilo logs <project> --follow
kilo events <project>
kilo fleet
  kilo disk
  kilo doctor
kilo version
kilo link <project>
kilo update <project> <archive.zip|.tar.gz>
kilo update --setup
kilo update
```

The management API is intended for operator access. The public status URL is deliberately narrower: it exposes project status only, not the admin API, logs, secrets, or lifecycle controls.

### `kilo link <project>` — get a status URL without the Discord bot

`kilo link <project>` fetches the same rotating public status link the Discord
bot uses internally (see [Step 3 above](#step-3-fetch-the-current-status-link)),
but over the normal operator `apiSecret` instead of the Discord-only
`KILOBUILD_INTERNAL_TOKEN`. Use it any time you need a project's current
status link and don't have the Discord bot handy:

```bash
kilo link my-bot
# https://status.example.com/status/8f2c...   (expires 2026-08-08 14:32:00)
```

Pass `--json` for a machine-readable `{ projectId, url, expiresAt }` payload.
The link rotates on the same schedule as the Discord bot's, so re-run the
command for a fresh one once the printed link expires.

### `kilo update <project> <archive>` — update a bot's code without touching its data

Redeploying a bot by hand — stopping it, replacing files while being careful
not to clobber `node_modules`, `dist`, or its SQLite database, then rebuilding
— is easy to get wrong. `kilo update <project> <archive>` does this safely:

```bash
kilo update my-bot ./my-bot-v2.zip
kilo update my-bot ./my-bot-v2.tar.gz --dry-run   # preview only, no changes
kilo update my-bot ./my-bot-v2.zip --yes          # skip the confirmation prompt
kilo update my-bot ./my-bot-v2.zip --no-rebuild   # sync files, rebuild later yourself
kilo update my-bot ./my-bot-v2.zip --prune        # also remove files not in the new archive
```

It accepts a `.zip` or `.tar.gz`/`.tgz` archive (a plain project folder, or one
wrapped in a single top-level directory — the common shape for a GitHub
source archive or `npm pack` output — either works). It extracts the archive,
hashes every file, and writes only what actually changed against what's
already on disk. The following are never touched, no matter what the archive
contains:

- `node_modules/`, `dist/`, `build/`, `.git/`, `.kilobuild/`
- `data/`, `.data/`, `db/`, `database/`, `storage/`, `.cache/`, `backups/`
- any `*.sqlite`, `*.sqlite3`, or `*.db` file (including `-journal`/`-wal`/`-shm` companions), anywhere in the tree
- any `.env` or `.env.*` file, anywhere in the tree

Add a `.kiloupdateignore` file to a project's root (one relative path per
line, `#` comments allowed) to protect additional project-specific paths.

By default the command deletes nothing that isn't in the new archive — it only
adds and updates files — and it triggers the same rebuild `kilo rebuild` does
once the sync finishes, so the running container picks up the change.

## 6. Install or update a VPS safely

From the latest KiloBuild checkout on the VPS, run the installer as `root`:

```bash
cd /opt/kilobuild
bash scripts/install-vps.sh
```

The installer installs dependencies, rebuilds the libraries, CLI, and API, installs
`kilo` into `/usr/local/bin`, and runs the daemon under the persistent
`kilobuild.service` system service. It does not overwrite `.kilobuild/config.json`.
It also builds the status frontend and serves it from the configured public
status listener.

### ZIP release updates

The public release source defaults to:

```text
https://github.com/Siradj0-0/KiloBuild
```

Each release is a repository-root file named exactly:

```text
KiloBuild(1.23.0).zip
```

The number in the filename must match the archive's root `VERSION` file. To
configure or change the source interactively:

```bash
kilo update --setup
```

The prompts ask for the GitHub owner, repository, release branch, and local
installation path. The settings contain public repository metadata only and
are stored in `.kilobuild/update.json`.

When `kilo update` runs, it lists the repository contents, selects the highest
semantic version from the `KiloBuild(x.y.z).zip` files, and compares it to the
installed `VERSION`. If newer, it validates the ZIP, rejects unsafe archive
paths, verifies the filename/version match, makes a timestamped `.kilobuild`
backup, installs the release, and runs the persistent service installer.
Project files, containers, logs, backups, tokens, and configuration remain in
the data directory and are not replaced by the release archive.

Use `kilo update --check` to check without installing:

```bash
kilo version
kilo update --check
kilo update
```

For every release, update `VERSION` and create a new root-level
`KiloBuild(x.y.z).zip` before uploading it to the public repository. Do not
replace an older release ZIP; the updater intentionally chooses the highest
version available.

Check the service with:

```bash
systemctl status kilobuild.service --no-pager
journalctl -u kilobuild.service -n 100 --no-pager
curl http://127.0.0.1:3001/api/healthz
```

Verify the website path from a machine that is not the VPS:

```bash
curl -I https://status.example.com/
curl -sS https://status.example.com/api/healthz
```

Do not launch the daemon with `nohup` for production use. The system service
starts it after Docker and brings it back after a crash or reboot.

## 7. Rotate credentials after exposure

If either credential was pasted into a chat, terminal transcript, issue, or
shared file, rotate both credentials:

```bash
cd /opt/kilobuild
bash scripts/rotate-vps-secrets.sh --yes
```

This invalidates the old `KILOBUILD_API_SECRET` and
`KILOBUILD_INTERNAL_TOKEN`. Update the operator CLI and KiloBuilder bot from the
new values in `.kilobuild/config.json`. The script saves a timestamped backup
and never prints the new secrets.