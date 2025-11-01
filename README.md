# Traefik 3.5 — Reverse Proxy with TLS

This repository contains a server-agnostic Traefik setup (Docker + Docker Compose) to run Traefik as a reverse proxy for multiple Docker services with automated TLS via Let's Encrypt. The README below is concise and avoids duplication — it shows what to change for your server and how to run/validate the setup.

Project structure (example)

```
proxy/
├── docker-compose.yml          # main Traefik compose file (reads variables from .env)
├── config/
│   └── traefik.yml             # static Traefik configuration (uses /certs/acme.json)
├── certs/                      # host directory mounted into container at /certs (contains acme.json)
│   └── acme.json               # ACME storage (DO NOT commit)
├── .env                        # environment variables (LETSENCRYPT_EMAIL, CERTS_DIR, DASHBOARD_HOST, etc.)
├── systemd/
│   └── traefik.service         # example systemd unit file (edit WorkingDirectory)
```

Summary
- Quick Traefik setup for Linux servers running Docker
- How to add services (Docker Compose labels)
- Production tips (ACME, firewall, backups, dashboard security)

Files to check before deploying
- `docker-compose.yml` — main compose for Traefik (reads `.env`)
- `config/traefik.yml` — static configuration (entryPoints, providers, ACME resolver)
- `certs/` (or your configured `CERTS_DIR`) — directory to store ACME data (`acme.json`) — keep it out of git

## Requirements
- Docker Engine (>= 20.10) and Docker Compose v2 (or integrated `docker compose`)
- Ports 80 and 443 open on the server
- For production: valid DNS A/AAAA records for each domain

Note on SELinux/AppArmor: if enabled, you may need to relabel volumes or place them in approved paths.

## Variables (in `.env`)
- `LETSENCRYPT_EMAIL` — email for Let's Encrypt registration and expiry notices
- `CERTS_DIR` — host path mounted into the container at `/certs` (default: `./certs`)
- `DASHBOARD_HOST` — hostname for the Traefik dashboard router (if exposed)
- `BASICAUTH_USERS` — BasicAuth entries (htpasswd-style) for the dashboard (recommended)

Minimal `.env` example

```env
LETSENCRYPT_EMAIL=you@example.com
CERTS_DIR=./certs
DASHBOARD_HOST=dashboard.example.com
```

## Quick start (server-agnostic)
1. Clone/copy repository to the server.
2. Install `docker` and `docker compose`.
3. Edit `.env` (set `LETSENCRYPT_EMAIL`, `CERTS_DIR`, `DASHBOARD_HOST`, etc.).
4. If you use an external Docker network for Traefik and services, create it:

```bash
docker network create traefik
```

5. Validate and start Traefik:

```bash
docker compose config   # validate (reads .env)
docker compose up -d    # start
```

6. Check status and logs:

```bash
docker compose ps
docker compose logs -f traefik
```

## Creating and protecting the ACME file
Traefik stores ACME data in `acme.json`. Create and secure it on the host (uses `CERTS_DIR`):

```bash
mkdir -p "${CERTS_DIR:-./certs}"
touch "${CERTS_DIR:-./certs}/acme.json"
chmod 600 "${CERTS_DIR:-./certs}/acme.json"
```

Do not commit `acme.json` to git. Keep backups and secure file ownership.

## How to add a site
Attach the service to the Traefik network and add labels in its Compose file:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myservice.rule=Host(`example.com`)"
  - "traefik.http.routers.myservice.entrypoints=websecure"
  - "traefik.http.routers.myservice.tls=true"
  - "traefik.http.routers.myservice.tls.certresolver=letsencrypt"
  - "traefik.http.services.myservice.loadbalancer.server.port=80"
```

Replace `myservice` and `example.com` accordingly. For multiple domains use `Host(`a.com`) || Host(`b.com`)`.

## Dashboard security
- Prefer hostname-based access (set `DASHBOARD_HOST`) with HTTPS + BasicAuth and/or IP allowlist.
- Avoid publishing Traefik's port 8080 directly on the host unless behind extra controls.

## TLS / Let's Encrypt notes
- For testing use the staging ACME server to avoid rate limits. For production use the real CA.
- Traefik is configured to use `/certs/acme.json` inside the container; `CERTS_DIR` controls the host path.

## Common issues & troubleshooting
- ACME fails: verify DNS, port 80 accessibility, and that no upstream proxy blocks ACME challenges.
- Routing 404: ensure the service is attached to Traefik's network and labels are correct.
- Redirect loops: check app-level redirects and Traefik middlewares.

## Validation and runtime checks
After configuration changes, restart Traefik and follow logs:

```bash
docker compose up -d
docker compose logs -f traefik
```

## Systemd (optional)
An example unit exists at `systemd/traefik.service`. Edit `WorkingDirectory` and verify `docker`/`docker compose` availability before enabling.

## Local development vs Production setup

This repository includes a **conditional setup** that works both locally (with `.localhost` domains) and in production (with public domains and Let's Encrypt):

### Local Development (default)
- `DASHBOARD_HOST=dashboard.docker.localhost`
- Dashboard runs on **HTTP** (port 8080) — no ACME certificate requests
- `docker-compose.override.yml` automatically overrides the dashboard router to use the `web` entrypoint
- Access: `http://dashboard.docker.localhost:8080` (with BasicAuth)
- No SSL errors or ACME rejections

**To use locally:**
1. Keep `docker-compose.override.yml` in the repository
2. Edit `.env` to set `DASHBOARD_HOST=dashboard.docker.localhost` (or any `.localhost` domain)
3. Run: `docker compose up -d`

### Production Deployment (public domain)
- `DASHBOARD_HOST=dashboard.example.com` (replace with your real domain)
- Dashboard runs on **HTTPS** with Let's Encrypt certificates
- Delete or rename `docker-compose.override.yml` so the main compose file (with TLS + certresolver) is used
- Access: `https://dashboard.example.com` (with BasicAuth)
- Certificates auto-renew via ACME

**To switch to production:**
1. Edit `.env`: set `DASHBOARD_HOST=dashboard.example.com` and `LETSENCRYPT_EMAIL=you@example.com`
2. Delete `docker-compose.override.yml`: `rm docker-compose.override.yml`
3. Ensure ports 80/443 are open and DNS points to your server
4. Run: `docker compose up -d`

### How it works
- `docker-compose.yml` (main file) always defines the dashboard with HTTPS + Let's Encrypt
- `docker-compose.override.yml` (optional) overrides the dashboard labels to use HTTP instead
- Docker Compose automatically merges override files, so you can switch setups without editing compose files
- This is a standard Docker Compose pattern; see [Compose override docs](https://docs.docker.com/compose/compose-file/11-extension/)
