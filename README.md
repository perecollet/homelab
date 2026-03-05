# perecollet-homelab

Personal homelab running on a mini PC. All services are containerised with Docker Compose, exposed via a Cloudflare Tunnel (no port forwarding needed), and served over HTTPS by Caddy.

## Services

| Service | URL | Description |
|---|---|---|
| Portfolio | `portfolio.perecollet.dev` | Personal website (React + Vite) |
| Photos | `photos.perecollet.dev` | Immich — self-hosted photo library |
| Files | `files.perecollet.dev` | FileBrowser — web file manager |
| Ads | `ads.perecollet.dev` | AdGuard Home — network-wide ad blocking |

## Architecture

```
Internet
   │
   ▼
Cloudflare Tunnel (cloudflared)
   │
   ▼
Caddy  ──  HTTPS + automatic certificates via Cloudflare DNS challenge
   │
   ├── portfolio_site  (nginx serving Vite build)
   ├── immich_server
   ├── filebrowser
   └── adguard         (LAN only — DNS server for the local network)
```

All containers share a single internal Docker network (`proxy-nw`). Only Caddy and AdGuard expose ports to the host.

## Prerequisites

- Docker and Docker Compose v2
- A Cloudflare account with your domain managed there
- A Cloudflare API token with `Zone:DNS:Edit` permission
- A Cloudflare Tunnel token (created via Zero Trust dashboard)

## First-time setup

### 1. Clone the repo

```bash
git clone https://github.com/you/homelab
cd homelab
```

### 2. Clone the portfolio source

```bash
cd portfolio
git clone https://github.com/you/perecolletsite
cd ..
```

### 3. Configure secrets

Copy the example and fill in your values:

```bash
cp .env.example .env
```

| Variable | Description |
|---|---|
| `DOMAIN` | Your root domain (e.g. `perecollet.dev`) |
| `CF_TOKEN` | Cloudflare API token (for Caddy DNS challenge) |
| `CF_TUNNEL_TOKEN` | Cloudflare Tunnel token |
| `DB_PASSWORD` | Postgres password for Immich |
| `DB_USERNAME` | Postgres username for Immich |
| `DB_DATABASE_NAME` | Postgres database name for Immich |
| `UPLOAD_LOCATION` | Host path for Immich photo uploads (e.g. `/srv/immich/library`) |
| `DB_DATA_LOCATION` | Host path for Immich Postgres data (e.g. `/srv/immich/postgres`) |

### 4. Create host data directories

```bash
sudo mkdir -p \
  /srv/adguard/work \
  /srv/adguard/conf \
  /srv/files \
  /srv/filebrowser/database \
  /srv/immich/library \
  /srv/immich/postgres

sudo chown -R $USER:$USER /srv/files /srv/immich/library
```

### 5. AdGuard initial setup

AdGuard needs a one-time setup wizard before it can serve the admin UI on port 80.

1. Temporarily expose port 3000 in `ads/compose.yaml`
2. Start only AdGuard: `docker compose up -d adguard`
3. Open `http://YOUR-MINI-PC-IP:3000` and complete the wizard
4. Remove the `3000:3000` port from `ads/compose.yaml`
5. Point your router's DNS to the mini PC's local IP

> Give the mini PC a static local IP (or a DHCP reservation in your router) before doing this, otherwise DNS for your whole network will break on reboot.

### 6. Start everything

```bash
docker compose up -d
```

## Repository structure

```
homelab/
├── docker-compose.yml       # Root compose — includes all sub-projects
├── .env                     # Secrets (gitignored)
├── caddy/
│   ├── Caddyfile            # Reverse proxy + HTTPS config
│   ├── Dockerfile           # Custom Caddy build with Cloudflare DNS plugin
│   └── compose.yaml
├── cloudflared/
│   └── compose.yaml         # Cloudflare Tunnel
├── ads/
│   └── compose.yaml         # AdGuard Home
├── files/
│   ├── filebrowser.json     # FileBrowser config
│   └── compose.yaml
├── photos/
│   └── compose.yaml         # Immich (server, microservices, ML, redis, postgres)
└── portfolio/
    ├── Dockerfile            # Multi-stage Vite build → nginx
    ├── compose.yaml
    └── perecolletsite/       # Cloned separately, gitignored
```

## Updating services

```bash
docker compose pull
docker compose up -d
```

## FileBrowser default credentials

Username: `admin`
Password: `admin`

Change these immediately after first login.
