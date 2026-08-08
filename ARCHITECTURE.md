# KiloBuild Architecture

KiloBuild is a production-grade, self-hosted platform for deploying and managing unlimited Node.js applications (including Discord bots) in isolated Docker containers. It is designed to scale from a handful of projects to thousands, with no hardcoded project names, no arbitrary limits, and full observability at every layer.

---

## 1. Core Goals

| Goal | How |
|------|-----|
| Unlimited projects | No hardcoded lists; all state in the embedded SQLite database |
| Docker-native isolation | Each project has its own image, container, network, volumes |
| Self-healing | Configurable restart policies, exponential backoff, Docker event watcher |
| Auto-detection | File-based project scanner finds Node.js projects without registration |
| Extensible | Plugin hooks fire on every major lifecycle event |
| Production-grade | Pino structured logging, graceful shutdown, tini PID 1, startup validation |

---

## 2. Folder Structure

```
kilobuild/
├── lib/
│   ├── kilobuild-types/        # Shared TypeScript interfaces (no runtime deps)
│   ├── kilobuild-events/       # Type-safe event bus (EventEmitter)
│   ├── kilobuild-cache/        # Memory + disk-backed TTL cache
│   ├── kilobuild-config/       # Global + per-project configuration (Zod-validated)
│   ├── kilobuild-docker/       # Dockerode wrapper: containers, images, stats, events
│   ├── kilobuild-discovery/    # File-system project scanner
│   ├── kilobuild-templates/    # Dockerfile / Compose / startup script generator
│   ├── kilobuild-monitor/      # Container stats stream + HTTP health checks
│   ├── kilobuild-logging/      # Per-project structured log files + rotation
│   ├── kilobuild-recovery/     # Self-healing restart policies + backoff
│   ├── kilobuild-backup/       # tar.gz backup + restore
│   ├── kilobuild-plugins/      # Dynamic plugin loader + hook bridge
│   └── kilobuild-cli/          # `kilo` CLI (commander + chalk + ora + cli-table3)
├── artifacts/
│   └── api-server/             # Express 5 management daemon (REST API)
└── lib/db/                     # Drizzle ORM + embedded SQLite schema
```

---

## 3. Data Models

### `projects`
Canonical record of every managed project. Status is denormalised here to avoid a join on every list call.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| name | text | Unique human name |
| slug | text | kebab-case, used as Docker container/image prefix |
| path | text | Absolute host path to project root |
| status | enum | created / building / running / stopped / error / unknown |
| node_version | text | Detected or user-set, e.g. "22" |
| entry_point | text | Resolved start command |
| package_manager | enum | npm / pnpm / yarn / bun |
| project_type | enum | node / typescript / discord-bot / express-api / generic |
| tags | JSON text | String array |
| metadata | JSON text | Extensible key-value store |

### `containers`
One row per Docker container lifecycle. A project may accumulate many container rows over time (rebuilds, crashes).

### `project_configs`
Per-project config overrides. If absent, the global defaults apply. Stored as a single merged row per project.

### `backups`
Metadata about tar.gz archives stored on the host filesystem (path points outside the DB).

### `events_log`
Append-only event history. Written by the event bus listener. Used for audit trail, dashboards, and webhook triggers.

### `stats_snapshots`
Periodic container resource snapshots. Pruned automatically based on `statsRetentionDays`.

### `plugins_registry`
Installed plugin registry. The runtime PluginManager cross-references this.

---

## 4. Runtime Design

```
┌─────────────────────────────────────────────────────────┐
│                     Express Daemon                      │
│  /api/projects   /api/stats   /api/discover   /api/…   │
└───────────┬──────────────┬──────────────────────────────┘
            │              │
     ┌──────▼──────┐  ┌────▼──────────┐
     │  Container  │  │  ContainerMon │
     │  Manager    │  │  itor + Health│
     │ (Dockerode) │  │  Checker      │
     └──────┬──────┘  └────┬──────────┘
            │              │
     ┌──────▼──────────────▼──────┐
     │       Event Bus            │
     │  (KiloBuildEventBus)       │
     └──────┬───────────┬─────────┘
            │           │
   ┌────────▼──┐  ┌─────▼──────────┐
   │ Recovery  │  │ Plugin Manager │
   │ Manager   │  │ (hook bridge)  │
   └───────────┘  └────────────────┘
```

**Key flows:**

1. **Create project** → insert DB row → scaffold Dockerfile/Compose/startup → emit `project:created`
2. **Start** → find or create container → `docker start` → update DB → emit `container:started` → begin monitoring
3. **Container dies** → Docker event watcher fires → RecoveryManager evaluates policy → schedules backoff restart or emits `recovery:exhausted`
4. **Rebuild** → `docker build` (background) → on success: stop old container, recreate, start → emit `build:completed`
5. **Backup** → tar.gz project directory → write metadata to DB → emit `backup:created`

---

## 5. Docker Strategy

Every project owns:

| Resource | Name pattern |
|----------|-------------|
| Image | `kilobuild/<slug>:latest` |
| Container | `kilobuild-<slug>` |
| Label | `kilobuild.managed=true`, `kilobuild.project.id=<uuid>` |
| Log driver | `json-file` with `max-size` + `max-file` |
| Healthcheck | Configurable HTTP path via Docker native healthcheck |

The Docker event watcher monitors the event stream filtered to `label=kilobuild.managed=true` so KiloBuild only sees its own containers without iterating all containers on the host.

**Startup layer:** every container runs `/kilobuild/startup.sh` as PID 1 via `tini`. The startup script:
- Loads project environment values from `.env` first, then `config.env` or
  `.kilobuild/config.env`, with saved KiloBuild env settings as a fallback
- Validates required env vars against `.env.example`
- Optionally installs dependencies (autoInstall)
- Optionally runs build (autoBuild)
- Optionally runs migrations (autoMigrate)
- Traps SIGTERM/SIGINT for graceful shutdown
- Reports exit code

---

## 6. CLI Design

The `kilo` CLI communicates exclusively via the daemon REST API. It never calls Docker or the database directly.

```
kilo list                         List all projects
kilo create <name> [path]         Create + scaffold a new project
kilo import <path>                Import an existing project
kilo delete <name>                Delete project and container
kilo start <name>                 Start container
kilo stop <name>                  Stop container (graceful)
kilo restart <name>               Restart container
kilo rebuild <name>               Rebuild Docker image + restart
kilo logs <name>                  View recent logs
kilo inspect <name>               Full project detail
kilo stats [name]                 Resource usage (all or one)
kilo health <name>                Health check status
kilo backup create <name>         Create backup
kilo backup list <name>           List backups
kilo restore <name> <backupId>    Restore from backup
kilo config get [name]            Show config
kilo config set <name> <k> <v>    Update project config
kilo config set-global <k> <v>    Update global config
kilo discover [dirs...]           Scan and discover projects
kilo doctor                       System diagnostics
kilo plugins list                 List plugins
kilo plugins enable <name>        Enable plugin
kilo plugins disable <name>       Disable plugin
```

---

## 7. Event Flow

```
Event Bus (KiloBuildEventBus)

Emitters:                        Listeners:
  ContainerManager                 RecoveryManager     → schedule restart
  DockerEventWatcher               PluginManager       → call plugin hooks
  HealthChecker                    EventLog writer     → persist to DB
  BackupManager                    SSE broadcaster     → push to dashboard clients
  ConfigManager
  PluginManager
```

All events carry `{ id, type, projectId, payload, timestamp }`.

---

## 8. Lifecycle Diagrams

### Project Lifecycle

```
[CREATED] ──build──▶ [BUILDING] ──success──▶ [STOPPED]
                         │                       │
                         └──failure──▶ [ERROR]   │ start
                                                 ▼
                                            [RUNNING]
                                                 │
                                         crash (non-zero exit)
                                                 │
                                    recovery policy check
                                    ┌────────────┴──────────────┐
                               restart allowed              exhausted
                                    │                           │
                                [RUNNING]                  [ERROR]
```

### Container Restart Backoff

```
Attempt 1:  delay = base × 1.0  (+ ±20% jitter)
Attempt 2:  delay = base × 2.0
Attempt 3:  delay = base × 4.0
...
Attempt N:  delay = min(base × multiplier^(N-1), 5min)

If restartCount >= maxRestarts within restartWindowSeconds → EXHAUSTED
Window reset if last crash was > restartWindowSeconds ago.
```

---

## 9. Configuration Format

### Global (`/var/lib/kilobuild/config.json`)

```json
{
  "global": {
    "dataDir": "/var/lib/kilobuild",
    "projectsDir": "/var/lib/kilobuild/projects",
    "logDir": "/var/log/kilobuild",
    "backupDir": "/var/lib/kilobuild/backups",
    "dockerSocket": "/var/run/docker.sock",
    "defaultNodeVersion": "22",
    "defaultPackageManager": "npm",
    "monitoringIntervalMs": 10000,
    "healthCheckIntervalMs": 30000,
    "statsRetentionDays": 7,
    "eventsRetentionDays": 30,
    "pluginsDir": "/var/lib/kilobuild/plugins",
    "apiPort": 3001,
    "apiHost": "127.0.0.1",
    "apiSecret": null,
    "defaults": {
      "restartPolicy": "unless-stopped",
      "maxRestarts": 10,
      "memoryLimit": null,
      "cpuLimit": null
    }
  },
  "projects": {
    "<project-uuid>": {
      "restartPolicy": "on-failure",
      "maxRestarts": 5,
      "memoryLimit": "512m",
      "healthCheckEnabled": true,
      "healthCheckPath": "/health"
    }
  }
}
```

### Per-project (`kilobuild.json` in project root)

```json
{
  "$schema": "https://kilobuild.dev/schema/v1/project.json",
  "version": "1",
  "project": { "name": "my-bot", "slug": "my-bot", "type": "discord-bot" },
  "runtime": {
    "node": "22",
    "packageManager": "pnpm",
    "entryPoint": "src/index.ts",
    "autoInstall": true,
    "autoBuild": true,
    "autoMigrate": false
  },
  "docker": {
    "restart": "unless-stopped",
    "healthCheck": { "enabled": false, "path": "/health", "intervalSeconds": 30 }
  },
  "resources": { "memory": "512m", "cpu": "0.5" },
  "logging": { "rotation": true, "maxSizeMb": 100, "retentionDays": 30 },
  "plugins": []
}
```

---

## 10. Plugin System

Plugins live in `pluginsDir/<name>/index.js` and export a `PluginDefinition`:

```js
// pluginsDir/my-plugin/index.js
export default {
  name: "my-plugin",
  version: "1.0.0",
  description: "Sends a webhook on container crashes",
  hooks: {
    async onContainerCrashed(container) {
      await fetch(process.env.WEBHOOK_URL, {
        method: "POST",
        body: JSON.stringify({ event: "crash", containerId: container.containerId }),
      });
    }
  },
  async initialize(config) {
    console.log("Plugin ready", config);
  }
};
```

Plugins are isolated from KiloBuild internals. They receive events through the hook bridge and can call the public API via HTTP if they need to trigger actions.

---

## 11. Implementation Roadmap

| Phase | Deliverable | Status |
|-------|------------|--------|
| 1 | Core types, event bus, cache, config | ✅ Complete |
| 2 | Docker integration (container, image, stats, events) | ✅ Complete |
| 3 | Project discovery + template generator | ✅ Complete |
| 4 | Monitoring + health checks | ✅ Complete |
| 5 | Logging + log rotation | ✅ Complete |
| 6 | Self-healing recovery + backoff | ✅ Complete |
| 7 | Backup + restore | ✅ Complete |
| 8 | Plugin system | ✅ Complete |
| 9 | Embedded SQLite schema (Drizzle) | ✅ Complete |
| 10 | Management REST API (Express 5) | ✅ Complete |
| 11 | CLI (`kilo`) | ✅ Complete |
| 12 | DB push + integration tests | 🔜 Next |
| 13 | Web dashboard (future) | 🔜 Future |
| 14 | Multi-host / swarm support | 🔜 Future |

---

## Security Considerations

- The daemon API binds to `127.0.0.1` by default (not `0.0.0.0`)
- `apiSecret` can be set for bearer token authentication
- Containers run as non-root user (`appuser`, UID 1001)
- tini as PID 1 prevents zombie processes and handles signal forwarding correctly
- No host networking by default; bridge network per container
- Volume mounts are opt-in and explicitly configured
