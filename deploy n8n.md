# Deploy n8n on the Homelab

This document is a second-pass rewrite focused on actual `n8n` deployment for the Debian-based ThinkPad T450 homelab described in `set-up homelab.md`.

It is organized as a sequence of practical questions and answers rather than a raw chat transcript.

## Recommended Topic Sequence

1. What is the recommended deployment model for n8n?
2. What should be installed on the Debian host first?
3. What folder structure should be used?
4. How should the Docker Compose stack be defined?
5. How should environment variables and secrets be handled?
6. How should n8n be exposed on the network?
7. How should persistence, updates, and backups be handled?
8. What is the minimum production-minded setup for this homelab?

---

## Topic 1: What is the recommended deployment model for n8n?

### Short answer

Deploy `n8n` as a Docker container on the Debian host, with persistent storage and a reverse proxy in front of it.

### Why this is the best fit

- Matches the Debian + Docker homelab baseline
- Easy to update and roll back
- Keeps application files isolated from the host
- Works well on modest T450 hardware
- Fits future expansion with Postgres, reverse proxy, and monitoring

### Recommended target state

```text
ThinkPad T450
  Debian 12 minimal
    Docker
      reverse proxy
      n8n
      optional Postgres
```

### Deployment recommendation

Start with:

- Docker Compose
- persistent volume for `n8n`
- `.env` file for secrets and URLs
- reverse proxy for HTTPS and cleaner access

---

## Topic 2: What should be installed on the Debian host first?

### Short answer

Install Docker, Docker Compose support, and a few basic admin tools.

### Baseline packages

```bash
sudo apt update
sudo apt install \
  docker.io \
  docker-compose \
  curl \
  git \
  ca-certificates \
  htop
```

### Ensure Docker is running

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### Optional convenience

If you want to run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Log out and back in after that change.

---

## Topic 3: What folder structure should be used?

### Short answer

Keep the stack under a dedicated directory such as `/opt/homelab/n8n`.

### Recommended layout

```text
/opt/homelab/n8n
  docker-compose.yml
  .env
  data/
```

### Why this layout works

- Easy to back up
- Easy to manage over SSH
- Keeps service configuration in one place
- Works cleanly with Git later if you want infra-as-code

### Example setup

```bash
sudo mkdir -p /opt/homelab/n8n/data
sudo chown -R $USER:$USER /opt/homelab/n8n
cd /opt/homelab/n8n
```

---

## Topic 4: How should the Docker Compose stack be defined?

### Short answer

Use a simple Compose file with one `n8n` service, persistent storage, restart policy, and environment variables loaded from `.env`.

### Minimal Compose example

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    env_file:
      - .env
    volumes:
      - ./data:/home/node/.n8n
```

### Notes

- `./data` stores workflows, credentials, and application state
- `5678` is the default n8n port
- `restart: unless-stopped` helps recovery after reboot

### Start the stack

```bash
docker compose up -d
```

If `docker compose` is unavailable on your system, use:

```bash
docker-compose up -d
```

### Verify the container

```bash
docker ps
docker logs --tail 100 n8n
```

---

## Topic 5: How should environment variables and secrets be handled?

### Short answer

Put configuration in a local `.env` file and set a strong encryption key before first use.

### Minimum recommended `.env`

```dotenv
N8N_HOST=n8n.local
N8N_PORT=5678
N8N_PROTOCOL=http
NODE_ENV=production
TZ=Asia/Ho_Chi_Minh
N8N_ENCRYPTION_KEY=replace-with-a-long-random-secret
WEBHOOK_URL=http://n8n.local/
```

### Important points

- `N8N_ENCRYPTION_KEY` protects stored credentials
- Keep the same encryption key across restarts and upgrades
- `WEBHOOK_URL` should match the final externally reachable URL
- `TZ` should match your preferred timezone

### If using a real domain and HTTPS

Example:

```dotenv
N8N_HOST=n8n.example.com
N8N_PORT=5678
N8N_PROTOCOL=https
NODE_ENV=production
TZ=Asia/Ho_Chi_Minh
N8N_ENCRYPTION_KEY=replace-with-a-long-random-secret
WEBHOOK_URL=https://n8n.example.com/
```

### Secret handling guidance

- Do not commit `.env` into a public repository
- Back up the encryption key with the data directory
- Treat the `.env` file as sensitive

---

## Topic 6: How should n8n be exposed on the network?

### Short answer

For initial homelab use, expose `5678` only on the local network. For longer-term use, put n8n behind a reverse proxy with HTTPS.

### Option 1: Local-only access

Access pattern:

```text
http://<server-ip>:5678
```

This is the fastest way to validate the deployment.

### Option 2: Reverse proxy in front of n8n

Recommended when:

- You want a stable hostname
- You plan to use webhooks
- You want HTTPS
- You want cleaner exposure than a raw port

### Typical architecture

```text
browser
  reverse proxy
    n8n:5678
```

### Reverse proxy options

- Nginx Proxy Manager
- Caddy
- Traefik

### Recommendation

For a homelab, a reverse proxy becomes important as soon as you want external webhooks, TLS, or cleaner routing with other services.

---

## Topic 7: How should persistence, updates, and backups be handled?

### Short answer

Treat the `data` directory and the encryption key as critical state.

### What must be preserved

- `/opt/homelab/n8n/data`
- `.env`
- especially `N8N_ENCRYPTION_KEY`

### Backup recommendation

At minimum, back up:

- the entire `/opt/homelab/n8n` directory

### Update workflow

```bash
cd /opt/homelab/n8n
docker compose pull
docker compose up -d
```

Or:

```bash
cd /opt/homelab/n8n
docker-compose pull
docker-compose up -d
```

### Before updating

- Confirm you have a backup of `data/`
- Keep the same encryption key
- Check container logs after upgrade

### Useful verification commands

```bash
docker ps
docker logs --tail 100 n8n
du -sh /opt/homelab/n8n/data
```

---

## Topic 8: What is the minimum production-minded setup for this homelab?

### Short answer

The minimum sensible setup is:

- Debian 12 minimal
- Docker-based `n8n`
- persistent local storage
- strong encryption key
- reverse proxy if using webhooks or external access
- basic backup routine

### Recommended progression

#### Phase 1: Fast local deployment

- Run `n8n` with Docker Compose
- Store data locally
- Access it on the LAN

#### Phase 2: Safer operational baseline

- Put it behind a reverse proxy
- Use a stable hostname
- Enable HTTPS
- Add backups

#### Phase 3: Better durability

- Move from SQLite-only defaults toward Postgres if workflows grow
- Add monitoring and log visibility
- Keep the Compose stack under version control

### Practical recommendation for your homelab

For a ThinkPad T450, start simple:

```text
Debian
  Docker
    reverse proxy
    n8n
```

Then add Postgres only when you actually need it.

---

## Final Consolidated Recommendation

If the goal is to deploy `n8n` on the Debian homelab:

1. Prepare a dedicated directory such as `/opt/homelab/n8n`
2. Run `n8n` with Docker Compose and a persistent `data` directory
3. Store config in `.env` and set a strong `N8N_ENCRYPTION_KEY`
4. Start with LAN access on port `5678`
5. Add a reverse proxy and HTTPS when you need stable URLs or webhooks
6. Back up both the `data` directory and the `.env` file before upgrades

This is the simplest deployment model that is still operationally sound for a homelab.
