# 🏠 Homelab — Proxmox + Docker Compose

![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Proxmox](https://img.shields.io/badge/Proxmox-VE-E57000?logo=proxmox&logoColor=white)
![Debian](https://img.shields.io/badge/OS-Debian-A81D33?logo=debian&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-mesh%20VPN-000000?logo=tailscale&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-Tunnel-F38020?logo=cloudflare&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-informational)

A self-hosted home server running on a **Lenovo M625q** mini PC, virtualized with **Proxmox**, with all services deployed via **Docker Compose** on Debian. The LAN is a flat network on an unmanaged **TP-Link** switch, with remote access provided by **Tailscale** and **Cloudflare Tunnel**.

# Home Lab – Proxmox + Docker Compose

A self-hosted home lab running on a single mini PC: Proxmox VE hosts one LXC container that runs the whole Docker Compose stack. It covers photo management (Immich), smart home (Home Assistant), network-wide ad blocking and local DNS (Pi-hole), reverse proxy (Nginx Proxy Manager), a public website behind a Cloudflare Tunnel, torrenting through a VPN, and monitoring.

![Home Lab architecture](docs/images/architecture.svg)

> Hostnames, Tailscale addresses and secrets in this repository are placeholders (`example.com`, `100.64.x.x`). Real values live in a local, git-ignored `.env`.

## Contents

1. [Hardware](#1-hardware)
2. [Virtualization](#2-virtualization)
3. [Networking and access](#3-networking-and-access)
4. [Services](#4-services)
5. [VPN isolation](#5-vpn-isolation)
6. [Storage and backup](#6-storage-and-backup)
7. [Configuration and secrets](#7-configuration-and-secrets)
8. [Getting started](#8-getting-started)
9. [Roadmap](#9-roadmap)

## 1. Hardware

| Component | Details |
|---|---|
| Compute | Lenovo ThinkCentre M625q (mini PC / thin client), 8 GB DDR4 RAM |
| System disk | 1× 512 GB SSD (Proxmox, LXC root filesystem, service configs and databases, downloads) |
| Data disks | 2× USB HDD: `/mnt/red` (photos) and `/mnt/black` (copy of the photos) |
| Network | Huawei DN8245V-70 router → TP-Link TL-PA7017P KIT powerline adapters → TP-Link LS1005G switch → mini PC (static IP) and main PC |

### 📸 The Rack

<!--
Add your own photos here — drop image files into ./docs/images/ and update the paths below.
Recommended: 1-4 photos, resized to ~1200px wide so the README stays fast to load.
-->

<p align="center">
  <img src="./docs/images/rack-front.jpg" alt="Server rack — front view" width="45%">
  <img src="./docs/images/rack-cabling.jpg" alt="Server rack — cabling detail" width="45%">
</p>

<p align="center">
  <em>Lenovo M625q + switch + network gear</em>
</p>

## 2. Virtualization

- **Hypervisor:** Proxmox VE, with 0.5 GB of RAM reserved for the host.
- **Docker host:** a single LXC container running Ubuntu with Docker. It gets the remaining 7.5 GB of RAM, all CPU cores and the rest of the SSD.
- **Storage access:** `/mnt/red` and `/mnt/black` are mounted on the Proxmox host and passed into the LXC as mount points.
- **Zigbee dongle:** passed through to the LXC and then to the Home Assistant container (`HA_USB_DEVICE`).
- **Source of truth:** everything inside the LXC is defined by [`docker-compose.yml`](docker-compose.yml).

## 3. Networking and access

| Layer | Solution |
|---|---|
| Physical | Huawei DN8245V-70 router → TP-Link TL-PA7017P KIT powerline adapters (link over the home mains wiring) → TP-Link LS1005G switch → mini PC (static IP) and main PC |
| LAN | `192.168.100.0/24`; the Docker LXC has a static address (`192.168.100.110`) |
| DNS | Pi-hole serves the LAN and resolves local names such as `immich.example.com` and `pve.example.com` |
| Reverse proxy | Nginx Proxy Manager terminates HTTPS for internal services |
| Public exposure | Cloudflare Tunnel (`cloudflared`) publishes **only** the static `website`; no ports are opened on the router |
| Remote access | Tailscale, installed on the Proxmox host, inside the Docker LXC, on the main PC and on a phone |

### Docker networks

| Network | Services |
|---|---|
| `public_web` | `cloudflared`, `website`, `uptime-kuma` |
| `proxy_net` | `npm`, `pihole`, `homarr`, `homarr-iframes`, `glances`, `filebrowser`, `samba`, `immich-server`, `gluetun`, `uptime-kuma` |
| `backend` | `immich-server`, `immich-postgres`, `immich-redis`, `immich-machine-learning`, `uptime-kuma` |
| `host` | `homeassistant` (needs mDNS and direct access to the Zigbee dongle) |
| `service:gluetun` | `qbittorrent` (shares Gluetun's network stack) |

## 4. Services

| Service | Image | Host ports | Purpose | Memory limit |
|---|---|---|---|---|
| `website` | `nginx:alpine` | 8080 | Static public site (via Cloudflare Tunnel) | 64 MB |
| `cloudflared` | `cloudflare/cloudflared` | – | Outbound tunnel to Cloudflare | 64 MB |
| `immich-server` | `immich-server:release` | 2283 | Photo management | – |
| `immich-postgres` | `pgvector/pgvector:pg14` | – | Immich database | – |
| `immich-redis` | `redis:6.2-alpine` | – | Immich cache and queues | – |
| `immich-machine-learning` | `immich-machine-learning:release` | – | Face recognition, smart search | – |
| `pihole` | `pihole/pihole` | 53 TCP/UDP, 8081 | LAN DNS and ad blocking | 150 MB |
| `homeassistant` | `home-assistant:stable` | host (8123) | Smart home, Zigbee | 500 MB |
| `npm` | `jc21/nginx-proxy-manager` | 80, 81, 443 | Reverse proxy | – |
| `filebrowser` | `filebrowser/filebrowser` | 8082 | Web file manager | 80 MB |
| `homarr` | `homarr-labs/homarr-test:v2` | 7575 | Dashboard (v2 beta) | 300 MB |
| `homarr-iframes` | `diogovalentte/homarr-iframes` | 8085 | iframe widgets for Homarr | 64 MB |
| `glances` | `nicolargo/glances` | 61208 | Host resource monitoring | 100 MB |
| `uptime-kuma` | `louislam/uptime-kuma` | 3001 | Service availability monitoring | 150 MB |
| `gluetun` | `qmcgaw/gluetun` | 8090, 6881 | VPN client (ProtonVPN, WireGuard, Switzerland) | 1024 MB |
| `qbittorrent` | `linuxserver/qbittorrent` | via gluetun | Torrent client | 2048 MB |
| `samba` | `dperson/samba` | 139, 445 | SMB share of the downloads directory | 150 MB |

Container logs use a shared `x-logging` YAML anchor (`json-file`, 10 MB × 3 files).

## 5. VPN isolation

- All internet traffic of `qbittorrent` goes through the `gluetun` container (`network_mode: "service:gluetun"`). If the ProtonVPN tunnel drops, qBittorrent loses connectivity instead of leaking traffic.
- `gluetun` only serves qBittorrent. Immich and Home Assistant do not use the VPN.
- This is isolation of **outbound internet traffic**, not network segmentation: `gluetun` also sits in `proxy_net`, and `FIREWALL_OUTBOUND_SUBNETS` allows the LAN and Tailscale nodes so that the qBittorrent WebUI stays reachable.

## 6. Storage and backup

| Location | Contents |
|---|---|
| SSD (directory next to `docker-compose.yml`) | Service configs, databases (Pi-hole, NPM, Uptime Kuma, Immich Postgres), qBittorrent downloads |
| `/mnt/red/immich` | Immich photo library (uploads) |
| `/mnt/black` | Copy of the photos from `/mnt/red` |

- **Container and LXC backups:** Proxmox's built-in backup (vzdump).
- **Photos:** replicated from `/mnt/red` to `/mnt/black` (both USB drives on the same host).

## 7. Configuration and secrets

Secrets and host-specific paths are kept in `.env`, which is git-ignored. Copy [`.env.example`](.env.example) and fill it in.

| Variable | Used by |
|---|---|
| `TZ`, `PUID`, `PGID` | Timezone and container user/group |
| `CLOUDFLARE_TUNNEL_TOKEN` | `cloudflared` |
| `DB_USERNAME`, `DB_PASSWORD`, `DB_DATABASE_NAME`, `DB_DATA_LOCATION`, `DB_HOSTNAME`, `REDIS_HOSTNAME` | Immich |
| `PIHOLE_PASSWORD` | Pi-hole web UI |
| `HA_USB_DEVICE` | Zigbee dongle device path |
| `HOMARR_KEY` | Homarr encryption key |
| `FILEBROWSER_MOUNT_DIR`, `GLANCES_MOUNT_DIR` | Directories exposed to File Browser and Glances |
| `PROTONVPN_PRIVATE_KEY`, `FIREWALL_OUTBOUND_SUBNETS` | Gluetun |
| `SAMBA_USER` | Samba (`user;password`) |

## 8. Getting started

```bash
git clone <this-repo-url>
cd <repo>
cp .env.example .env     # fill in real values
docker compose pull
docker compose up -d
docker compose ps
```

Then create proxy hosts in Nginx Proxy Manager (`:81`) and local DNS records in Pi-hole (`:8081`) for your own domain.

## 9. Roadmap

- [ ] Document the Proxmox backup target, schedule and retention
- [ ] Add an off-site copy of the photo library
- [ ] Pin image versions instead of `latest` / `release`
