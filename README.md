# perecollet-homelab

Personal homelab running on a mini PC. All services are containerised with Docker Compose, exposed via a Cloudflare Tunnel (no port forwarding needed), and routed by Caddy.

## Services

| Service | URL | Description |
|---|---|---|
| Portfolio | `perecollet.dev` | Personal website (React + Vite) |
| Photos | `photos.perecollet.dev` | Immich — self-hosted photo library |
| Files | `files.perecollet.dev` | FileBrowser — web file manager |
| Ads | `ads.perecollet.dev` | AdGuard Home — network-wide ad blocking (LAN only) |
| VPN | `vpn.perecollet.dev:51820` | WireGuard — remote LAN access via VPN |

## Architecture

```
Internet
   │
   ├── Cloudflare Tunnel (cloudflared)
   │      │
   │      ▼
   │   Caddy :2080 (internal HTTP, no redirect)
   │      │
   │      ├── perecollet.dev        → portfolio_site:80
   │      ├── photos.perecollet.dev → immich_server:2283
   │      ├── files.perecollet.dev  → filebrowser:80
   │      └── ads.perecollet.dev    → adguard:80
   │
   └── WireGuard UDP :51820 (vpn.perecollet.dev)
          │
          ▼
       LAN access (192.168.1.0/24)

Local network
   │
   ▼
Router DNS → mini PC IP (AdGuard filters all DNS queries)
```

### Docker networks

| Network | Services | Notes |
|---|---|---|
| `proxy-nw` | caddy, cloudflared, adguard, filebrowser, immich-server, portfolio | Main internal highway |
| `immich-nw` | immich-server, immich-machine-learning, redis, database | Immich internal — `internal: true` |
| `wireguard-nw` | wireguard | Isolated, UDP 51820 exposed |
| `ddns-nw` | cloudflare-ddns | Isolated, external DNS only |

The Cloudflare Tunnel points to `http://caddy:2080` for all subdomains — Caddy routes based on hostname. Caddy also listens on 443 for direct LAN HTTPS access. WireGuard gives clients full LAN access without going through the tunnel.

## Prerequisites

- Docker and Docker Compose v2
- A Cloudflare account with your domain managed there
- A Cloudflare API token with `Zone:DNS:Edit` permission (for Caddy ACME)
- A separate Cloudflare API token with `Zone:DNS:Edit` permission (for DDNS)
- A Cloudflare Tunnel token (created via Zero Trust dashboard)
- Kernel headers installed on the host (required by WireGuard)
- Tailscale installed on all devices for remote SSH access

## First-time setup

### 1. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Clone the repo

```bash
cd ~
git clone https://github.com/you/homelab
cd homelab
```

### 3. Clone the portfolio source

```bash
cd portfolio
git clone https://github.com/you/perecolletsite
cd ..
```

### 4. Configure secrets

Each service has its own `.env` file. Copy and fill in each one:

```bash
cp caddy/.env.example caddy/.env
cp cloudflared/.env.example cloudflared/.env
cp photos/.env.example photos/.env
cp wireguard/.env.example wireguard/.env
```

**`caddy/.env`**

| Variable | Description |
|---|---|
| `ACME_EMAIL` | Email for Let's Encrypt certificate notifications |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with `Zone:DNS:Edit` (for ACME DNS challenge) |

**`cloudflared/.env`**

| Variable | Description |
|---|---|
| `CF_TUNNEL_TOKEN` | Cloudflare Tunnel token (from Zero Trust dashboard) |

**`photos/.env`**

| Variable | Description |
|---|---|
| `DB_PASSWORD` | Postgres password for Immich |
| `DB_USERNAME` | Postgres username for Immich |
| `DB_DATABASE_NAME` | Postgres database name for Immich |
| `UPLOAD_LOCATION` | Host path for Immich photo uploads (e.g. `/srv/immich/library`) |
| `DB_DATA_LOCATION` | Host path for Immich Postgres data (e.g. `/srv/immich/postgres`) |

**`wireguard/.env`**

| Variable | Description |
|---|---|
| `CLOUDFLARE_DDNS_TOKEN` | Cloudflare API token with `Zone:DNS:Edit` (for DDNS updates) |

### 5. Create host data directories

```bash
sudo mkdir -p \
  /srv/adguard/work \
  /srv/adguard/conf \
  /srv/files \
  /srv/filebrowser/database \
  /srv/immich/library \
  /srv/immich/postgres

sudo chown -R $USER:$USER /srv/files /srv/filebrowser /srv/immich/library
```

### 6. Create the Docker networks

```bash
docker network create proxy-nw
```

The remaining networks (`immich-nw`, `wireguard-nw`, `ddns-nw`) are created automatically by Docker Compose on first `up`.

### 7. AdGuard initial setup

AdGuard needs a one-time setup wizard before it can serve the admin UI on port 80.

1. Start only AdGuard: `docker compose up -d adguard`
2. Open `http://YOUR-MINI-PC-IP:3000` and complete the wizard — set DNS on port 53, admin UI on port 80
3. Remove the `3000:3000/tcp` port from `ads/compose.yaml`
4. Restart AdGuard: `docker compose up -d adguard`

> Give the mini PC a static local IP (DHCP reservation in your router) before this step. Then point your router's primary DNS to the mini PC's IP so AdGuard filters all network traffic.

### 8. Configure Cloudflare Tunnel public hostnames

In the Cloudflare Zero Trust dashboard, set all hostnames to point to `http://caddy:2080`:

| Subdomain | Service |
|---|---|
| `perecollet.dev` | `http://caddy:2080` |
| `photos.perecollet.dev` | `http://caddy:2080` |
| `files.perecollet.dev` | `http://caddy:2080` |
| `ads.perecollet.dev` | `http://caddy:2080` |

> `vpn.perecollet.dev` is **not** routed through the tunnel — it resolves to the host's public IP via DDNS and WireGuard connects directly over UDP 51820. Make sure your router forwards UDP 51820 to the mini PC.

### 9. Configure WireGuard peers

WireGuard generates peer configs automatically on first start. Retrieve a peer QR code with:

```bash
docker exec -it wireguard /app/show-peer iphone
docker exec -it wireguard /app/show-peer mac
```

Scan the QR code with the WireGuard app on each device. The VPN grants access to `192.168.1.0/24` using AdGuard (`192.168.1.62`) as DNS, so ad blocking applies over VPN too.

### 10. Start everything

```bash
docker compose up -d
```

## Remote access

[Tailscale](https://tailscale.com) is used for remote SSH access to the mini PC. Install it on both the mini PC and your client devices, sign in with the same account, and SSH using the Tailscale IP:

```bash
ssh youruser@100.x.x.x
```

## Updating services

```bash
git pull
docker compose pull
docker compose up -d
```

To rebuild the portfolio after a code change:

```bash
git pull
docker compose up -d --build portfolio
```

## Repository structure

```
homelab/
├── docker-compose.yml       # Root compose — includes all sub-projects
├── caddy/
│   ├── Caddyfile            # Reverse proxy config (port 2080 internal + 443 LAN)
│   ├── Dockerfile           # Custom Caddy build with Cloudflare DNS plugin
│   ├── compose.yaml
│   ├── .env                 # ACME_EMAIL, CLOUDFLARE_API_TOKEN (gitignored)
│   └── .env.example
├── cloudflared/
│   ├── compose.yaml         # Cloudflare Tunnel
│   ├── .env                 # CF_TUNNEL_TOKEN (gitignored)
│   └── .env.example
├── ads/
│   └── compose.yaml         # AdGuard Home
├── files/
│   ├── filebrowser.json     # FileBrowser config
│   └── compose.yaml
├── photos/
│   ├── compose.yaml         # Immich (server, ML, redis, postgres)
│   ├── .env                 # DB credentials, upload paths (gitignored)
│   └── .env.example
├── wireguard/
│   ├── compose.yaml         # WireGuard VPN + Cloudflare DDNS
│   ├── config/              # Generated peer configs (gitignored)
│   ├── .env                 # CLOUDFLARE_DDNS_TOKEN (gitignored)
│   └── .env.example
└── portfolio/
    ├── Dockerfile            # Multi-stage Vite build → nginx
    ├── compose.yaml
    └── perecolletsite/       # Cloned separately, gitignored
```

## FileBrowser default credentials

Username: `admin`
Password: `admin`

Change these immediately after first login.
