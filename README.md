Readme · MD
# perecollet-homelab
 
Personal homelab running on a mini PC. All services are containerised with Docker Compose, exposed via a Cloudflare Tunnel (no port forwarding needed), and routed by Caddy.
 
## Services
 
| Service | URL | Description |
|---|---|---|
| Portfolio | `perecollet.dev` | Personal website (React + Vite) |
| Photos | `photos.perecollet.dev` | Immich — self-hosted photo library |
| Files | `files.perecollet.dev` | FileBrowser — web file manager |
| Automation | `n8n.perecollet.dev` | n8n — self-hosted workflow automation |
| Ads | `ads.perecollet.dev` | AdGuard Home — network-wide ad blocking (LAN only) |
| VPN | `vpn.perecollet.dev:51820` | WireGuard — remote LAN access via VPN |
 
### Supporting services (no public URL)
 
| Service | Description |
|---|---|
| autoheal | `willfarrell/autoheal` — watches containers labelled `autoheal=true` and restarts them if their healthcheck turns unhealthy (AdGuard carries this label) |
| watchtower | `nickfedor/watchtower` — checks for newer images of running containers. Runs in **monitor-only** mode: sends a Telegram alert when an update is available but does **not** apply it |
| jobspy-api | `rainmanjam/jobspy-api` — job-board scraping API consumed by the n8n job-offer scoring workflow |
| cloudflare-ddns | `favonia/cloudflare-ddns` — keeps `vpn.perecollet.dev` pointed at the host's public IP for WireGuard |
 
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
   │      └── n8n.perecollet.dev    → n8n:5678
   │
   └── WireGuard UDP :51820 (vpn.perecollet.dev)
          │
          ▼
       LAN access
          │
          └── ads.perecollet.dev    → adguard:80  (VPN only)
 
Local network
   │
   ▼
Router DNS → mini PC IP (AdGuard filters all DNS queries)
```
 
The Cloudflare Tunnel points to `http://caddy:2080` for all subdomains — Caddy routes based on hostname. Caddy also listens on 443 for direct LAN HTTPS access. WireGuard gives clients full LAN access without going through the tunnel.
 
`autoheal` and `watchtower` run as sidecars alongside the stack: they watch the other containers over the Docker socket rather than serving traffic.
 
### Docker networks
 
| Network | Services | Notes |
|---|---|---|
| `proxy-nw` | caddy, cloudflared, adguard, filebrowser, immich-server, portfolio, n8n, n8n-db, jobspy-api | Main internal highway |
| `immich-nw` | immich-server, immich-machine-learning, redis, database | Immich internal — `internal: true` |
| `wireguard-nw` | wireguard | Isolated, UDP 51820 exposed |
| `ddns-nw` | cloudflare-ddns | Isolated, external DNS only |
| `default` | autoheal, watchtower | Sidecars — Docker-socket access only |
 
## Prerequisites
 
- Docker and Docker Compose v2
- A Cloudflare account with your domain managed there
- A Cloudflare API token with `Zone:DNS:Edit` permission (for Caddy ACME)
- A separate Cloudflare API token with `Zone:DNS:Edit` permission (for DDNS)
- A Cloudflare Tunnel token (created via Zero Trust dashboard)
- A Telegram bot token + chat ID (for Watchtower update alerts)
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
cp n8n/.env.example n8n/.env
cp watchtower/.env.example watchtower/.env
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
 
**`n8n/.env`**
 
| Variable | Description |
|---|---|
| `N8N_DOMAIN` | Public hostname for n8n (e.g. `n8n.perecollet.dev`) |
| `N8N_ENCRYPTION_KEY` | Key used to encrypt saved credentials — generate with `openssl rand -hex 32`. **Never change after first run.** |
| `N8N_DB_USER` | Postgres username for n8n |
| `N8N_DB_PASSWORD` | Postgres password for n8n |
| `N8N_DB_NAME` | Postgres database name for n8n |
 
**`watchtower/.env`**
 
| Variable | Description |
|---|---|
| `WT_TG_TOKEN` | Telegram bot token (from BotFather) for update alerts |
| `WT_TG_CHATID` | Telegram chat ID that receives the alerts |
 
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
 
n8n stores its data inside the repo directory (bind-mounted at `n8n/data`, `n8n/files`, and `n8n/db-data`) so no extra host directories are needed. Those paths are gitignored.
 
### 6. Create the Docker networks
 
```bash
docker network create proxy-nw
```
 
The remaining networks (`immich-nw`, `wireguard-nw`, `ddns-nw`, `default`) are created automatically by Docker Compose on first `up`.
 
### 7. AdGuard initial setup
 
AdGuard needs a one-time setup wizard before it can serve the admin UI on port 80.
 
1. Start only AdGuard: `docker compose up -d adguard`
2. Open `http://YOUR-MINI-PC-IP:3000` and complete the wizard — set DNS on port 53, admin UI on port 80
3. Remove the `3000:3000/tcp` port from `ads/compose.yaml`
4. Restart AdGuard: `docker compose up -d adguard`
> Give the mini PC a static local IP (DHCP reservation in your router) before this step. Then point your router's primary DNS to the mini PC's IP so AdGuard filters all network traffic.
 
**Bootstrap DNS:** set a plain-IP bootstrap (`9.9.9.9`, `1.1.1.1`) in AdGuard's upstream settings. AdGuard needs it to resolve the hostname of its own DoH upstreams at startup — without it, a cold boot can leave AdGuard unable to bring up DNS.
 
**Healthcheck + autoheal:** AdGuard runs a healthcheck against its admin API on the loopback interface, and is labelled `autoheal=true` so the `autoheal` sidecar restarts it if it turns unhealthy. The healthcheck must target AdGuard's configured admin port (the one Caddy proxies to for `ads.perecollet.dev`).
 
### 8. Configure Cloudflare Tunnel public hostnames
 
In the Cloudflare Zero Trust dashboard, set all hostnames to point to `http://caddy:2080`:
 
| Subdomain | Service |
|---|---|
| `perecollet.dev` | `http://caddy:2080` |
| `photos.perecollet.dev` | `http://caddy:2080` |
| `files.perecollet.dev` | `http://caddy:2080` |
| `n8n.perecollet.dev` | `http://caddy:2080` |
 
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
 
## Monitoring & auto-updates
 
Two sidecars keep an eye on the stack over the Docker socket:
 
- **autoheal** restarts any container that turns `unhealthy`, as long as it carries the `autoheal=true` label. It only helps with *runtime* health failures of a running container — it cannot recover a container that fails to start, so it is not a substitute for correct restart policies and boot ordering.
- **watchtower** checks daily for newer images of the running containers. It runs in **monitor-only** mode: when an update is available it sends a Telegram alert but leaves the container untouched. Updates are then applied manually so infra services stay stable.
The scheduled check runs at `0 0 8 * * *` (08:00 **UTC** → 10:00 Europe/Madrid). Set `TZ=Europe/Madrid` on the watchtower service to align it to local time.
 
**Force a check now (without waiting for the schedule):**
 
```bash
docker exec watchtower /watchtower --run-once --monitor-only
```
 
**Apply an update after an alert:**
 
```bash
git pull
docker compose pull <service>
docker compose up -d <service>
```
 
**Enable automatic updates for a specific low-risk service** (while the global default stays monitor-only), add a label to that service in its `compose.yaml`:
 
```yaml
    labels:
      - "com.centurylinklabs.watchtower.monitor-only=false"
```
 
Keep infra services (adguard, caddy, cloudflared, wireguard) on monitor-only and pin their image tags rather than tracking `:latest`, so an update is always a conscious, versioned change.
 
## Updating services
 
Watchtower notifies you when updates are available; apply them manually:
 
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
│   └── compose.yaml         # AdGuard Home + autoheal sidecar
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
├── portfolio/
│   ├── Dockerfile           # Multi-stage Vite build → nginx
│   ├── compose.yaml
│   └── perecolletsite/      # Cloned separately, gitignored
├── n8n/
│   ├── compose.yaml         # n8n + Postgres (n8n-db)
│   ├── .env                 # N8N_DOMAIN, N8N_ENCRYPTION_KEY, DB credentials (gitignored)
│   ├── .env.example
│   ├── data/                # n8n app data (gitignored)
│   ├── files/               # n8n user files (gitignored)
│   └── db-data/             # Postgres data (gitignored)
├── jobspy/
│   └── compose.yaml         # JobSpy scraping API (feeds the n8n job-scoring workflow)
└── watchtower/
    ├── compose.yaml         # Watchtower — image update monitor (notify-only)
    ├── .env                 # WT_TG_TOKEN, WT_TG_CHATID (gitignored)
    └── .env.example
```
 
## FileBrowser default credentials
 
Username: `admin`
Password: `admin`
 
Change these immediately after first login.
 
## License
 
MIT — see [`LICENSE`](./LICENSE).
 
