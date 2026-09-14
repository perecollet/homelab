# perecollet-homelab

Personal homelab running on a mini PC. All services are containerised with Docker Compose, exposed via a Cloudflare Tunnel (no port forwarding needed), and routed by Caddy. Media streaming (Jellyfin) is served over LAN/WireGuard only — **not** through the tunnel — to comply with Cloudflare's terms on video streaming.

## Services

| Service | URL | Description |
|---|---|---|
| Portfolio | `perecollet.dev` | Personal website (React + Vite) |
| Photos | `photos.perecollet.dev` | Immich — self-hosted photo library |
| Files | `files.perecollet.dev` | FileBrowser — web file manager |
| Streaming | `stream.perecollet.dev` | Jellyfin — self-hosted media server (LAN/VPN only) |
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

Local network
   │
   ▼
Router DNS → mini PC IP (AdGuard filters all DNS queries)
   │
   ├── Caddy :443 (direct LAN HTTPS, real certs via DNS-01)
   │      │
   │      ├── stream.perecollet.dev → jellyfin:8096   (LAN/VPN only)
   │      └── ads.perecollet.dev    → adguard:80      (LAN/VPN only)
   │
   └── AdGuard DNS rewrite: *.perecollet.dev → mini PC IP
       (so LAN/VPN clients reach Caddy directly, never hairpinning out)
```

The Cloudflare Tunnel points to `http://caddy:2080` for the public subdomains — Caddy routes based on hostname. Caddy also listens on 443 for direct LAN HTTPS access, with real Let's Encrypt certs issued via the Cloudflare DNS-01 challenge (so even LAN-only hostnames get valid certificates without being publicly reachable). WireGuard gives clients full LAN access without going through the tunnel.

`stream.perecollet.dev` (Jellyfin) and `ads.perecollet.dev` (AdGuard) are deliberately **not** routed through the tunnel — they are reachable only on the LAN or over WireGuard, resolved by an AdGuard DNS rewrite that points `*.perecollet.dev` at the mini PC's LAN IP.

`autoheal` and `watchtower` run as sidecars alongside the stack: they watch the other containers over the Docker socket rather than serving traffic.

### Docker networks

| Network | Services | Notes |
|---|---|---|
| `proxy-nw` | caddy, cloudflared, adguard, filebrowser, immich-server, portfolio, n8n, n8n-db, jobspy-api, jellyfin | Main internal highway |
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
- An AMD/Intel iGPU with `/dev/dri/renderD128` available on the host (for Jellyfin hardware transcoding)

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
cp jellyfin/.env.example jellyfin/.env
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

**`jellyfin/.env`**

| Variable | Description |
|---|---|
| `JELLYFIN_DOMAIN` | Hostname for Jellyfin (e.g. `stream.perecollet.dev`) |
| `MEDIA_PATH` | Host path to the media library, mounted read-only (e.g. `/srv/media`) |
| `RENDER_GID` | GID of the host `render` group, needed for VA-API access to the iGPU. Find it with `getent group render` (e.g. `993`). **Host-specific — do not commit a value.** |

### 5. Create host data directories

```bash
sudo mkdir -p \
  /srv/adguard/work \
  /srv/adguard/conf \
  /srv/files \
  /srv/filebrowser/database \
  /srv/immich/library \
  /srv/immich/postgres \
  /srv/media/movies \
  /srv/media/shows \
  /srv/media/music \
  /srv/downloads

sudo chown -R $USER:$USER /srv/files /srv/filebrowser /srv/immich/library /srv/media /srv/downloads
```

n8n stores its data inside the repo directory (bind-mounted at `n8n/data`, `n8n/files`, and `n8n/db-data`) so no extra host directories are needed. Those paths are gitignored.

`/srv/media` holds the Jellyfin library (one sub-folder per library type). `/srv/downloads` is reserved for a future *arr* stack — keeping downloads and the library on the same filesystem lets imports use hardlinks. Both are exposed in FileBrowser so files can be uploaded through the web UI, and `/srv/media` is mounted read-only into Jellyfin.

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

**LAN DNS rewrite (required for LAN/VPN HTTPS access):** in **Filters → DNS rewrites**, add a wildcard rewrite so LAN and VPN clients reach Caddy directly instead of hairpinning out through Cloudflare:

| Domain | Answer |
|---|---|
| `*.perecollet.dev` | mini PC LAN IP (e.g. `192.168.1.62`) |
| `perecollet.dev` | mini PC LAN IP (e.g. `192.168.1.62`) |

This is what makes `stream.perecollet.dev` (Jellyfin) and `ads.perecollet.dev` resolve to Caddy on the LAN. It also covers every other subdomain, so it doubles as split-horizon DNS for the tunnelled services when you're at home. If this rewrite is missing (e.g. after an AdGuard config reset), LAN clients resolve the domains to Cloudflare's public IPs and everything hangs.

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
>
> `stream.perecollet.dev` (Jellyfin) and `ads.perecollet.dev` (AdGuard) are also **not** added to the tunnel. They are LAN/VPN-only, resolved by the AdGuard DNS rewrite above. Do not route Jellyfin through the tunnel — streaming video over Cloudflare's free tunnel violates its terms of service and can get the whole tunnel blocked.

### 9. Configure WireGuard peers

WireGuard generates peer configs automatically on first start. Retrieve a peer QR code with:

```bash
docker exec -it wireguard /app/show-peer iphone
docker exec -it wireguard /app/show-peer mac
```

Scan the QR code with the WireGuard app on each device. The VPN grants access to `192.168.1.0/24` using AdGuard (`192.168.1.62`) as DNS, so ad blocking and the LAN DNS rewrites apply over VPN too (which is how Jellyfin is reachable remotely).

### 10. Jellyfin setup

Jellyfin runs the official `jellyfin/jellyfin` image, transcoding on the host iGPU via VA-API, and is reached only on the LAN or over WireGuard through Caddy.

1. **Find the `render` GID** on the host and put it in `jellyfin/.env` as `RENDER_GID`:
   ```bash
   getent group render      # e.g. render:x:993:
   ```
2. **Set `MEDIA_PATH=/srv/media`** and **`JELLYFIN_DOMAIN=stream.perecollet.dev`** in `jellyfin/.env`.
3. **Start it:** `docker compose up -d jellyfin`
4. **Run the first-boot wizard** at `http://YOUR-MINI-PC-IP:8096` (direct, before Caddy): create the admin user and add libraries pointing at the in-container paths `/media/movies`, `/media/shows`, `/media/music`.
5. **Enable hardware transcoding:** Dashboard → Playback → set *Hardware acceleration* to **VA-API**, device `/dev/dri/renderD128`, and enable H264/HEVC decode + hardware encode. Leave HDR tone-mapping off unless needed (finicky on AMD).
6. **Caddy** serves it on the LAN (block already in the Caddyfile):
   ```
   stream.perecollet.dev {
       reverse_proxy jellyfin:8096
   }
   ```
7. The AdGuard wildcard rewrite (step 7) already resolves `stream.perecollet.dev` to the mini PC, so no extra DNS config is needed.

**Media layout:** Jellyfin identifies films best with one folder per movie including the year, e.g. `/srv/media/movies/The Hobbit - An Unexpected Journey (2012)/file.mkv`. The file name inside the folder doesn't matter; the folder name (with year) drives metadata matching. If a title is mis-identified, use the movie's ⋮ menu → **Identify** to correct it.

**Uploading media:** copy files straight to `/srv/media/...` over SSH (`rsync -avh --progress ...`) — best for large files, resumable, no browser. FileBrowser also exposes `/srv/media` for smaller uploads through the web UI. After adding files, run a library scan in Jellyfin.

**Transcoding note:** the host iGPU is an AMD Radeon 680M. VA-API handles H264/HEVC well; verify a hardware transcode is actually running with `docker exec jellyfin ps aux | grep ffmpeg` (look for `vaapi` in the ffmpeg command) while playing something forced to a lower quality.

### 11. Start everything

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

Keep infra and stateful services (adguard, caddy, cloudflared, wireguard, jellyfin, filebrowser, n8n, immich) on monitor-only and **pin their image tags** rather than tracking `:latest`, so an update is always a conscious, versioned change. Stateful services with their own database (FileBrowser, n8n, Immich) can break on a major version jump if `:latest` moves under them — FileBrowser is pinned for this reason.

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
│   ├── filebrowser.json     # FileBrowser config (root = /data)
│   └── compose.yaml         # Mounts /srv/files and /srv/media under /data
├── photos/
│   ├── compose.yaml         # Immich (server, ML, redis, postgres)
│   ├── .env                 # DB credentials, upload paths (gitignored)
│   └── .env.example
├── jellyfin/
│   ├── compose.yaml         # Jellyfin media server (VA-API transcoding)
│   ├── .env                 # JELLYFIN_DOMAIN, MEDIA_PATH, RENDER_GID (gitignored)
│   ├── .env.example
│   ├── config/              # Jellyfin config + database (gitignored)
│   └── cache/               # Jellyfin cache/transcodes (gitignored)
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

Media lives on the host under `/srv/media` (Jellyfin library) and `/srv/downloads`, outside the repo — never committed.

## FileBrowser default credentials

Username: `admin`
Password: `admin`

Change these immediately after first login. FileBrowser's browsable root is `/data`, under which `/srv/files` and `/srv/media` are mounted, so both your general files and the Jellyfin media library are reachable (and uploadable) from the web UI.

## License

MIT — see [`LICENSE`](./LICENSE).