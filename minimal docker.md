# Minimal Docker Baseline for the Homelab

This note focuses on the leanest practical Docker-based starting point for the ThinkPad T450 homelab, with `n8n` as the main workload.

Use this document when the goal is to stay simple and resource-conscious.
Use [deploy n8n.md](deploy n8n.md) when you want the fuller deployment flow for `n8n`.
Use [set-up homelab.md](set-up homelab.md) when you need the Debian host setup and overall homelab baseline first.

## Recommended Topic Sequence

1. What is the leanest `n8n` Compose setup?
2. How should backups be handled on a small box?
3. What folder structure should the homelab use?
4. What is the recommended minimal container stack?
5. When should this move from SQLite to Postgres?
6. What should change later for reverse proxy and public access?

---

## Topic 1: What is the leanest `n8n` Compose setup?

### Short answer

Use a single-container `n8n` Compose file with a bind-mounted `data/` directory, timezone settings, execution pruning, and a basic health check.

### Why this works

- Fits the T450 well
- Keeps memory and operational overhead low
- Uses SQLite first, which is acceptable for a small single-user setup
- Preserves the `.n8n` state directory from the start

### Minimal `docker-compose.yml`

Create `~/homelab/services/n8n/docker-compose.yml`:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      TZ: Asia/Ho_Chi_Minh
      GENERIC_TIMEZONE: Asia/Ho_Chi_Minh

      # Local LAN access for now
      N8N_HOST: 192.168.1.16
      N8N_PORT: 5678
      N8N_PROTOCOL: http

      # Keep execution history under control on a small SSD
      EXECUTIONS_DATA_PRUNE: "true"
      EXECUTIONS_DATA_MAX_AGE: "168"
      EXECUTIONS_DATA_PRUNE_MAX_COUNT: "10000"

      # Helpful while staying on SQLite
      DB_SQLITE_VACUUM_ON_STARTUP: "true"

      # Optional but useful on a privacy-conscious homelab
      N8N_DIAGNOSTICS_ENABLED: "false"
      N8N_PERSONALIZATION_ENABLED: "false"
    volumes:
      - ./data:/home/node/.n8n
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:5678/healthz >/dev/null 2>&1 || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 20s
```

### Start and verify

```bash
cd ~/homelab/services/n8n
docker compose up -d
docker ps
docker logs -f n8n
```

Open:

```text
http://192.168.1.16:5678
```

### Notes

- `TZ` and `GENERIC_TIMEZONE` keep schedules aligned with the local timezone
- `./data` is the critical persistent state for this setup
- pruning matters early on when the host has a small SSD

For the broader deployment framing around `.env`, encryption keys, and exposure models, see [deploy n8n.md](deploy n8n.md).

---

## Topic 2: How should backups be handled on a small box?

### Short answer

Use a two-layer backup model:

1. file-level backup of the `data/` directory
2. logical exports of workflows and credentials

### What to back up

Primary state:

```text
~/homelab/services/n8n/data
```

Also preserve:

- the eventual `.env` file if you introduce one later
- the `N8N_ENCRYPTION_KEY` if you move to the deployment pattern in [deploy n8n.md](deploy n8n.md)

### Backup script

Create `~/homelab/scripts/backup-n8n.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE="$HOME/homelab/backups/n8n"
STAMP="$(date +%F_%H%M%S)"
DEST="$BASE/$STAMP"

mkdir -p "$DEST"

# 1. Raw state backup
tar -czf "$DEST/n8n-data.tgz" -C "$HOME/homelab/services/n8n" data

# 2. Logical export backup
mkdir -p "$DEST/export"

docker exec -u node n8n n8n export:workflow --all --output=/tmp/workflows
docker exec -u node n8n n8n export:credentials --all --output=/tmp/credentials

docker cp n8n:/tmp/workflows "$DEST/export/workflows"
docker cp n8n:/tmp/credentials "$DEST/export/credentials"

docker exec -u node n8n rm -rf /tmp/workflows /tmp/credentials

cat > "$DEST/README.txt" <<EOF
Backup timestamp: $STAMP
Host: $(hostname)
Includes:
- n8n data tarball
- exported workflows
- exported credentials
EOF

# Keep only last 14 backups
ls -1dt "$BASE"/* 2>/dev/null | tail -n +15 | xargs -r rm -rf
```

Make it executable:

```bash
chmod +x ~/homelab/scripts/backup-n8n.sh
```

Run it manually:

```bash
~/homelab/scripts/backup-n8n.sh
```

### Cron example

```cron
30 2 * * * /home/ngminhtrung/homelab/scripts/backup-n8n.sh >/home/ngminhtrung/homelab/logs/backup-n8n.log 2>&1
```

### Restore order

1. restore `data/`
2. import workflows and credentials if needed
3. restart the container

---

## Topic 3: What folder structure should the homelab use?

### Short answer

Keep the homelab layout explicit, service-oriented, and predictable.

### Recommended layout

```text
~/homelab
├── services
│   ├── n8n
│   │   ├── docker-compose.yml
│   │   ├── .env
│   │   └── data/
│   ├── caddy
│   │   ├── docker-compose.yml
│   │   ├── Caddyfile
│   │   └── data/
│   ├── dozzle
│   │   └── docker-compose.yml
│   ├── uptime-kuma
│   │   ├── docker-compose.yml
│   │   └── data/
│   └── filebrowser
│       ├── docker-compose.yml
│       └── data/
├── backups
│   ├── n8n/
│   └── compose/
├── scripts
│   ├── backup-n8n.sh
│   ├── log-energy.sh
│   ├── update-all.sh
│   ├── healthcheck.sh
│   └── latest-energy-log
├── logs
│   ├── backup-n8n.log
│   ├── maintenance.log
│   └── energy/
│       ├── current.csv
│       └── README.txt
├── systemd
│   ├── energy-log.service
│   └── energy-log.timer
└── docs
    ├── ports.md
    ├── services.md
    └── recovery-runbook.md
```

### Why this layout works

- each service is isolated
- data stays near the service that owns it
- scripts and backups stay outside container state
- recovery is easier because paths remain stable

### Recommended naming for future discoverability

If you choose the ultra-minimal energy logger, use the word `energy` everywhere, even if the data is really a mix of power, battery, and CPU package metrics.

Use these exact paths and names:

- Script: `~/homelab/scripts/log-energy.sh`
- Summary helper: `~/homelab/scripts/summarize-energy.sh`
- Log folder: `~/homelab/logs/energy/`
- Current log file: `~/homelab/logs/energy/current.csv`
- Human note: `~/homelab/logs/energy/README.txt`
- Easy shortcut: `~/homelab/scripts/latest-energy-log`
- Service name: `energy-log.service`
- Timer name: `energy-log.timer`

That gives you several recovery paths later:

- `find ~/homelab -iname '*energy*'`
- `rg -n "energy|power|battery|rapl" ~/homelab`
- `systemctl list-timers | grep energy`
- `ls ~/homelab/logs/energy`

### Ultra-minimal energy logger plan

Use a host script plus a `systemd` timer, not a container.

What it should do:

- Run every 5 minutes
- Append one CSV row to `~/homelab/logs/energy/current.csv`
- Record timestamp, AC state, battery percentages, battery energy, battery power draw, and any readable Intel RAPL counters
- Keep a `README.txt` in the same folder that explains what created the file and how to query it
- Maintain `~/homelab/scripts/latest-energy-log` as a symlink to `../logs/energy/current.csv`

Suggested CSV columns:

```text
timestamp,ac_online,bat0_pct,bat1_pct,bat0_wh,bat1_wh,bat0_w,bat1_w,rapl_package_uj,rapl_core_uj,rapl_uncore_uj,rapl_dram_uj
```

### Why this is easier to find later

- The path lives under both `scripts/` and `logs/`, which are the first places most people check.
- The timer and service both contain `energy`, so `systemctl` searches are obvious.
- The log folder contains its own `README.txt`, so opening the directory explains itself.
- `current.csv` is plain on purpose, which makes it easy to grep, inspect in File Browser, or import elsewhere later.

### Suggested `README.txt` contents

Store this in `~/homelab/logs/energy/README.txt`:

```text
Energy logging for this host lives here.

Main log:
- ~/homelab/logs/energy/current.csv

Collector:
- ~/homelab/scripts/log-energy.sh

Scheduler:
- energy-log.service
- energy-log.timer

Helpful commands:
- tail -n 20 ~/homelab/logs/energy/current.csv
- ~/homelab/scripts/summarize-energy.sh 24
- systemctl status energy-log.timer
- systemctl list-timers | grep energy
- find ~/homelab -iname '*energy*'
```

This structure builds on the Debian host baseline described in [set-up homelab.md](set-up homelab.md).

---

## Topic 4: What is the recommended minimal container stack?

### Short answer

Start with five conservative containers:

1. `n8n`
2. `caddy`
3. `dozzle`
4. `uptime-kuma`
5. `filebrowser`

### Why this stack

| Container | Purpose | Why it belongs |
| --- | --- | --- |
| `n8n` | workflow engine | primary workload |
| `caddy` | reverse proxy | simple path toward stable routing and HTTPS later |
| `dozzle` | log viewer | fast container log inspection |
| `uptime-kuma` | service monitoring | lightweight reachability checks |
| `filebrowser` | file access | easy browsing of backups and service folders |

### Why not Postgres first

For this machine, SQLite is the lower-friction choice early on. Postgres is useful later, but it adds memory use, operational overhead, and another backup path to manage.

### Practical rollout

Phase 1:

- run only `n8n`
- validate backups
- keep access on the LAN

Phase 2:

- add `caddy`, `dozzle`, and `uptime-kuma`
- introduce `filebrowser` if remote file access is actually helpful

Phase 3:

- add Postgres only when the operational benefits outweigh the overhead

For the deployment-first version of this progression, see [deploy n8n.md](deploy n8n.md).

---

## Topic 5: When should this move from SQLite to Postgres?

### Short answer

Move to Postgres when recovery discipline, workflow volume, or operational importance matters more than staying minimal.

### Good reasons to switch

- workflow count grows materially
- execution history becomes operationally important
- backups need more structure
- upgrades become more frequent
- automation load becomes more serious

### Recommendation

For the T450:

- Phase 1: stay on SQLite
- Phase 2: move to `n8n + postgres` only when there is a clear need

---

## Topic 6: What should change later for reverse proxy and public access?

### Short answer

Keep the first deployment LAN-only. When you later place `n8n` behind a reverse proxy, add the URL-related settings explicitly.

### Later adjustments

When introducing a reverse proxy and stable external URLs:

- set `WEBHOOK_URL`
- set `N8N_PROXY_HOPS=1`
- move from raw port access to hostname-based access
- prefer HTTPS if webhooks or remote access are involved

### Current recommendation

For an initial homelab deployment, skip the extra proxy-specific settings and keep the stack simple.

For the more production-minded version of this step, see [deploy n8n.md](deploy n8n.md).
