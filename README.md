# KiloBuild

A production-grade, self hosted platform for deploying, managing, monitoring, and maintaining unlimited Node.js applications and Discord bots inside isolated Docker containers. Think PM2 + Coolify + Docker Compose — focused on Node.js, built to scale from 1 project to 1,000+.

You own the VPS. You own the containers. You own the data. KiloBuild just runs on top of what's already yours.

- **Discord:** https://discord.gg/A5x7E7j8mm — support, feature requests, and announcements
- **License:** Elastic License 2.0 — free to self-host and use commercially, not for reselling as a hosted service (see [Rights & license](#rights--license) below)

---

## Table of contents

- [Why open source (well — source-available)](#why-open-source-well--source-available)
- [Rights & license](#rights--license)
- [Your rights as a user](#your-rights-as-a-user)
- [Features](#features)
- [How it works](#how-it-works)
- [Installation](#installation)
- [Using the `kilo` CLI](#using-the-kilo-cli)
- [Updating](#updating)
- [Security notes](#security-notes)
- [Roadmap / suggested features](#roadmap--suggested-features)
- [Community](#community)

---

## Why open source (well — source-available)

KiloBuild manages other people's Docker containers, secrets, and databases — that's a lot of trust to ask for from a closed binary. Publishing the source isn't a marketing decision here, it's the actual trust mechanism. One honest caveat up front: KiloBuild's license (ELv2) restricts reselling it as a hosted service, so it's technically "source-available" rather than OSI-approved "open source" — but every line of code is public and auditable either way, which is what actually matters for trust:

- **You can read every line that touches your server.** Nothing runs that isn't in this repository.
- **You can audit the Dockerfile, docker-compose.yml, and startup.sh** that KiloBuild generates for your projects before you ever run them — see [`lib/kilobuild-templates`](lib/kilobuild-templates).
- **No telemetry, no phone-home, no analytics.** KiloBuild only talks to Docker's local socket, your database, and (if you enable it) GitHub, to check for releases you point it at.
- **Secrets stay local.** `apiSecret`, `KILOBUILD_INTERNAL_TOKEN`, and per-project env vars are stored in `.kilobuild/config.json` and `.env` files on your own disk — never transmitted anywhere by KiloBuild itself. See [`KILOBUILD_SETUP.md`](KILOBUILD_SETUP.md) for the full breakdown of what each secret does.
- **Forkable by design.** If this project ever stops being maintained, or you disagree with a direction it takes, you can fork it and keep running your own version with zero migration cost — it's just files on your VPS.

## Rights & license

KiloBuild is released under the **Elastic License 2.0 (ELv2)** — see [`LICENSE`](LICENSE) for the full, unmodified text.

Copyright (c) 2026 Siradj0-0 (the KiloBuild project).

ELv2 is a "source-available" license (not OSI-approved "open source" in the strict sense, but the code is 100% public and readable — same trust signal, different label). In plain language:

- **Use it for anything, including commercially.** Run it on your own VPS, use it to operate your own paid Discord bots, set it up for a client as a contractor — all explicitly fine, with zero royalty owed to me.
- **You cannot resell KiloBuild itself as a hosted/managed service.** You may not take this software and offer it to third parties as "KiloBuild hosting" or an equivalent managed product that gives your users access to a substantial part of its functionality. If you want to build and sell a hosting product on top of KiloBuild, reach out first.
- **Credit stays on.** You may not remove, alter, or obscure the copyright and license notices, whatever you do with the code — that's a license term, not a suggestion.
- **You can still fork, modify, and self-host modified versions freely** — the restriction is on offering the software *as a service to others*, not on using or changing it yourself.

## Your rights as a user

Under ELv2, you are free to:

- **Use** KiloBuild for anything — personal projects, client work, running your own commercial Discord bots or SaaS on top of it, internal company tooling. Commercial use is explicitly fine.
- **Read** every file. There is no hidden or obfuscated code, no separate "enterprise" fork with withheld features.
- **Modify** it. Change templates, add commands, rip out features you don't want.
- **Redistribute** it, modified or not, as long as the copyright and license notices stay intact and you're not offering it to others as a hosted/managed service.
- **Fork it** for your own use, with no obligation to contribute changes back (though contributions are always welcome).
- **Leave at any time.** Nothing about KiloBuild locks your projects in — every project it manages is a normal Docker container with a normal `Dockerfile` and `docker-compose.yml` sitting in a normal folder. If you stop using KiloBuild tomorrow, `docker compose up` still works on every project it built for you.

The one thing you can't do is take this codebase and turn around and sell it — or a hosted version of it — as your own product without my permission. Everything else about how you use it day-to-day is unrestricted.

What KiloBuild does **not** do, and never will, without a very clear opt-in:

- Collect telemetry or usage analytics
- Require an internet connection or a hosted account to function
- Phone home to check licensing
- Inject tracking, ads, or upsells into anything it generates for your projects

## Features

- **One-command project import** — point KiloBuild at a Node.js project (or Discord bot) directory and it auto-detects the package manager, TypeScript setup, entry point, and dependencies.
- **Automatic Dockerfile / docker-compose.yml / startup.sh generation** — correctly picks a single-stage image for interpreted TypeScript (tsx/ts-node/bun) versus a two-stage build for compiled projects, based on what your `package.json` actually declares, not guesswork.
- **Full container lifecycle** — start, stop, restart, and rebuild any project as an isolated Docker container from one CLI.
- **Live monitoring** — CPU, memory, network, and HTTP health checks per project, plus a fleet-wide view across every managed project.
- **Structured logging** — per-project logs with rotation and retention, tailable and follow-able from the CLI.
- **Self-healing** — crashed containers restart automatically with configurable exponential backoff and jitter.
- **Backup & restore** — full tar.gz backups of a project including its volumes, with one-command restore.
- **Safe in-place bot updates** — `kilo update <bot> <archive>` diffs a new release archive against the running project and updates only what changed, without ever touching `node_modules`, `dist`, its database, or `.env` files.
- **Direct status link access** — `kilo link <bot>` fetches a project's current public status URL without needing the Discord bot in the loop.
- **Public, rotating status pages** — an optional lightweight public status page per project (up/down + basic health) that's separate from and far narrower than the admin API.
- **Plugin system** — hook into any lifecycle event (start, stop, crash, rebuild, etc.) with your own plugin, including a built-in Discord webhook plugin for deploy/crash notifications.
- **Diagnostics** — `kilo doctor` and `kilo disk` for at-a-glance health and disk usage across your whole fleet.
- **Self-updating** — `kilo update` pulls the highest versioned release from a GitHub repo you configure and updates KiloBuild itself in place, without touching any project data.

## How it works

```
Your VPS
├── Docker Engine                  ← KiloBuild talks to this via the local socket
├── kilobuild.service              ← the management daemon (Express API), binds to 127.0.0.1
├── kilo (CLI)                     ← talks only to the daemon's REST API, never touches Docker directly
├── .kilobuild/                    ← embedded SQLite DB, config, rotating status tokens
└── /your/projects/*/
    ├── Dockerfile                 ← generated, regenerated on `kilo rebuild` unless hand-edited
    ├── docker-compose.yml         ← generated, same rule
    ├── .kilobuild/startup.sh      ← generated, always regenerated
    ├── data/ (or db/, .env, etc.) ← yours, never touched by generation or updates
    └── your actual project code
```

Every subsystem (Docker integration, monitoring, logging, recovery, backups, plugins) communicates through a typed internal event bus rather than calling each other directly — see [`ARCHITECTURE.md`](ARCHITECTURE.md) for the full design, data models, and lifecycle diagrams.

## Installation

Requirements: a Linux VPS with Docker installed, Node.js, and `pnpm`.

```bash
git clone https://github.com/Siradj0-0/KiloBuild.git /opt/kilobuild
cd /opt/kilobuild
bash scripts/install-vps.sh
```

The installer installs dependencies, builds the libraries/CLI/API, installs `kilo` into `/usr/local/bin`, and runs the daemon under a persistent `kilobuild.service` system service. See [`KILOBUILD_SETUP.md`](KILOBUILD_SETUP.md) for the full setup guide, including what each of the three secrets (`apiSecret`, `KILOBUILD_INTERNAL_TOKEN`, and the per-project rotating status token) does and how to rotate them.

## Using the `kilo` CLI

```bash
# Bring a project under management
kilo discover                      # scan for Node.js projects in the current directory
kilo create <name>                 # scaffold a brand-new project
kilo import <path>                 # import an existing project directory

# Day-to-day
kilo list                          # list every managed project
kilo start <project>
kilo stop <project>
kilo restart <project>
kilo rebuild <project>             # regenerate Docker files + rebuild the image
kilo inspect <project>             # full detail on one project
kilo logs <project> --follow
kilo stats <project>               # live CPU/memory/network
kilo health <project>
kilo events <project>              # lifecycle event history
kilo shell <project> -- <cmd>      # run a command inside the container

# Fleet-wide
kilo fleet                         # status + resources across every project
kilo disk                          # backup/log disk usage across every project
kilo doctor                        # system diagnostics

# Safe updates without touching data
kilo link <project>                # get the current public status URL directly
kilo update <project> <archive>    # update a bot's code from a .zip/.tar.gz, data untouched
kilo update <project> <archive> --dry-run

# Backup / restore
kilo backup create <project>
kilo restore <project> <backup-id>

# Config & plugins
kilo config get <project>
kilo plugins list

kilo version
```

Run `kilo <command> --help` for the full option list on any command.

## Updating

**KiloBuild itself:**

```bash
kilo update --setup     # point at a GitHub repo hosting KiloBuild(x.y.z).zip releases (first time only)
kilo update --check     # see if a newer version is available
kilo update             # install it — project data is never touched
```

**A single bot/project**, without losing its database or config:

```bash
kilo update <project> ./release.zip --dry-run   # preview first
kilo update <project> ./release.zip             # apply, then auto-rebuilds
```

Full details on what's protected during a project update are in [`KILOBUILD_SETUP.md`](KILOBUILD_SETUP.md#5-useful-operator-commands).

## Security notes

- The management daemon binds to `127.0.0.1` by default, not `0.0.0.0`.
- `apiSecret` gates the full admin API; `KILOBUILD_INTERNAL_TOKEN` is a narrower, Discord-bot-only credential; the public status page uses a separate rotating per-project token with no admin access at all.
- Containers run as a non-root user (`appuser`, UID 1001) inside the image.
- `tini` runs as PID 1 for correct signal handling and to avoid zombie processes.
- No host networking by default — each container gets a bridge network; volume mounts are opt-in and explicit.
- See [`KILOBUILD_SETUP.md`](KILOBUILD_SETUP.md) for credential rotation steps if a secret is ever exposed.

If you find a security issue, please report it privately in the Discord server rather than opening a public issue, so it can be fixed before it's disclosed.

## Roadmap / suggested features

Already planned:

- **Web dashboard** — a browser UI over the same management API the CLI uses.
- **Multi-host / fleet support** — manage projects spread across more than one VPS from one control point.
- **Integration test suite** — automated tests over the daemon + CLI, beyond the current typecheck-only CI.

Other ideas worth considering, roughly in order of how much trust/operational value they'd add:

- **Reverse proxy + automatic TLS** (Traefik or Caddy integration) so projects get `https://project.yourdomain.com` with zero manual nginx/certbot work.
- **Prometheus metrics endpoint** so fleet-wide stats can feed into existing monitoring instead of only the built-in `kilo fleet`/`kilo stats`.
- **Audit log** of who ran what admin action and when — useful the moment more than one operator shares a VPS.
- **Role-based API tokens** (read-only vs. deploy vs. full-admin) instead of a single `apiSecret` for everyone with access.
- **GitHub webhook auto-deploy** — push to a branch, KiloBuild pulls and runs `kilo update` automatically (built on the same diff-safe update path as `kilo update <bot> <archive>`).
- **Blue-green / canary rebuilds** — bring the new container up alongside the old one and only cut over once its health check passes, instead of stop-then-start.
- **One-command rollback** to the last known-good backup or image, not just restore-from-backup.
- **Resource quotas + alerting** — per-project memory/CPU caps with a Discord/webhook alert when a project approaches its limit.
- **Secrets manager integration** (e.g. Vault, or even just encrypted-at-rest `.env` files) as an alternative to plaintext env files on disk.
- **Project templates** — `kilo create <name> --template discord-bot-ts` style starter kits for common project shapes.
- **A plugin registry/marketplace** so community plugins (beyond the built-in Discord webhook one) are discoverable instead of hand-rolled every time.

Have a feature idea? Bring it to the Discord — that's the fastest way to get it prioritized.

## Community

- **Discord:** https://discord.gg/A5x7E7j8mm
- Bug reports, feature requests, and "how do I..." questions are all welcome there.
- Security issues: please report privately in Discord rather than in a public issue.
