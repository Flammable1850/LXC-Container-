# Media Server — Proxmox LXC Container

Docker-based media stack running inside a single unprivileged LXC container on Proxmox.

## Overview

| Item | Value |
|---|---|
| Host | Proxmox VE |
| Container ID | `100` |
| Hostname | `docker-lxc` |
| Container type | Unprivileged LXC |
| Container runtime | Docker CE + Compose plugin |
| Shared storage mount | `/mnt/media` (bind-mounted into all relevant containers) |

## Architecture

```
Proxmox Host
└── LXC 200 "docker-lxc" (unprivileged, nesting=1, keyctl=1)
    └── Docker Engine
        ├── portainer      (container management UI)
        ├── jellyfin       (media server, HW transcoding via /dev/dri)
        ├── sonarr         (TV automation)
        ├── radarr         (movie automation)
        ├── prowlarr       (indexer manager — syncs to Sonarr/Radarr)
        ├── qbittorrent    (torrent client, routed through gluetun)
        └── gluetun        (VPN sidecar — kill-switch for qBittorrent)
```

Reverse proxy note: routes have been run through both Traefik and Nginx Proxy Manager at different points — confirm which one is currently deployed before editing this section.

## LXC Container Config (`/etc/pve/lxc/200.conf`)

```ini
arch: amd64
hostname: docker-lxc
cores: <n>
memory: <MB>
swap: <MB>

# Required for Docker-in-LXC
features: nesting=1,keyctl=1

# GPU passthrough for Jellyfin hardware transcoding
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

**Why both `nesting` and `keyctl` matter:** unprivileged LXC containers need `nesting=1` for Docker to run at all, and `keyctl=1` on top of that or several images throw permission errors during startup.

**GPU device numbers aren't universal** — confirm on the Proxmox host first:
```bash
ls -la /dev/dri
```
Match the major/minor numbers shown (typically `226,0` for `card0` and `226,128` for `renderD128`) before writing them into the config. After any host kernel/driver update, recheck — these can renumber.

**Applying config changes:**
```bash
pct stop 200
pct start 200
```

Then verify inside the container:
```bash
pct exec 200 -- ls -la /dev/dri
```

If the container is unprivileged and `/dev/dri` shows up but Jellyfin still can't transcode, it's a GID mapping issue — the render group inside the container needs to map to a UID/GID range Proxmox has allocated in `/etc/subgid`.

## Docker Compose

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9443:9443"
      - "8000:8000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer/data:/data

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
      - "8920:8920"
    volumes:
      - ./jellyfin/config:/config
      - ./jellyfin/cache:/cache
      - /mnt/media:/media
    devices:
      - /dev/dri:/dev/dri

  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=<provider>
      - VPN_TYPE=<wireguard|openvpn>
    volumes:
      - ./gluetun:/gluetun

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    depends_on:
      - gluetun
    restart: unless-stopped
    volumes:
      - ./qbittorrent/config:/config
      - /mnt/media/downloads:/downloads

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    ports:
      - "9696:9696"
    volumes:
      - ./prowlarr/config:/config

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    ports:
      - "8989:8989"
    volumes:
      - ./sonarr/config:/config
      - /mnt/media:/media

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    ports:
      - "7878:7878"
    volumes:
      - ./radarr/config:/config
      - /mnt/media:/media
```

Real values to fill in: `<provider>` and `VPN_TYPE` for gluetun, plus any environment variables your VPN provider config requires (WireGuard keys or OpenVPN credentials go in a `.env` file, not committed).

## Wiring The Stack Together

1. **Prowlarr first** — add indexers here; it syncs them to Sonarr and Radarr automatically.
2. **Sonarr / Radarr** — point the download client at qBittorrent (`gluetun`'s IP since qBittorrent shares its network namespace) on port `8080`, download path `/downloads`.
3. **Jellyfin** — add libraries pointing at `/media/tv` and `/media/movies` under the shared mount.
4. **Reverse proxy** — expose clean hostnames (e.g. `jellyfin.local`) instead of raw ports.

## Storage Layout

```
/mnt/media/
├── downloads/     ← qBittorrent lands files here
├── tv/            ← Sonarr-managed, Jellyfin library
└── movies/        ← Radarr-managed, Jellyfin library
```

All containers that touch media mount the same host path so Sonarr/Radarr renames are visible to Jellyfin without a copy step.

## VPN Kill Switch — Verify It Actually Works

Don't assume Gluetun's kill switch works just because it's configured. Test it:
```bash
docker compose stop gluetun
docker compose exec qbittorrent curl -s ifconfig.me
```
This should fail to connect (no fallback to your real IP), since qBittorrent has no network path once gluetun is down.

## Logging

Docker's default `json-file` driver can fill the container's disk unbounded. Set rotation in `/etc/docker/daemon.json` inside the LXC:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
Restart Docker after editing: `systemctl restart docker`.

## Backups

- **Container-level:** `vzdump` snapshot of LXC 200 from the Proxmox host, scheduled.
- **App-level:** back up each service's `./<service>/config` directory separately — faster to restore a single app's settings than a full container snapshot.
- **Compose + `.env`:** version-controlled (secrets excluded) so the stack can be redeployed from a clean container if needed.

## Update Strategy

Manual pulls with changelog review, rather than Watchtower auto-updates — consistent with a cautious update posture after the June 2026 AUR supply-chain incident.
```bash
docker compose pull
docker compose up -d
```

## Maintenance Checklist

- [ ] `/dev/dri` still passed through after any Proxmox host update
- [ ] Kill switch still verified after gluetun/provider changes
- [ ] Docker logs not filling container disk
- [ ] `vzdump` backup job completed successfully
- [ ] `pct exec 200 -- docker stats` — no single container starving the rest
- [ ] Reverse proxy routes still resolve after any container IP change

## Known Gotchas

- GPU device nodes can renumber after a Proxmox host kernel update — always recheck `ls -la /dev/dri` on the host before assuming passthrough is broken elsewhere.
- Unprivileged LXC + Docker requires both `nesting=1` and `keyctl=1` — missing either causes cryptic container startup failures, not an obvious error.
- YAML indentation in `docker-compose.yml` is easy to break with mixed editors — a single misaligned space under `services:` will fail the whole file. Rewriting via heredoc is a reliable fix if `docker compose` reports a parse error.
- qBittorrent has no network of its own (`network_mode: service:gluetun`) — if gluetun isn't healthy first, qBittorrent silently has no connectivity rather than erroring clearly.

---
*Replace all `<placeholder>` values and confirm CTID/resource numbers match your actual `/etc/pve/lxc/200.conf` before committing.*
