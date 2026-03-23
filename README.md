# 🎬 Home Media Arr Stack Server

A self-hosted home media server built on an HP Z800 workstation running TrueNAS SCALE. This project uses Docker containers managed through Dockge to run a full "Arr stack" — a suite of applications that automate media management and streaming.

> **Note:** This project was built by following community tutorials and documentation. The goal was hands-on learning — understanding how Linux, Docker, networking, and self-hosted services work together in a real environment.

---

## 🖥️ Hardware

| Component | Details |
|-----------|---------|
| Machine | HP Z800 Workstation |
| CPU | Intel Xeon X5675 @ 3.07GHz (single active — second CPU not posting) |
| RAM | 47.0 GiB ECC (~96 GiB expected with both CPUs operational) |
| Boot Drive | 1x 298.09 GiB HDD (TrueNAS OS) |
| Storage Pool | 2x HDD Mirrored (465.76 GiB + 298.09 GiB) → 286.49 GiB usable (ZFS uses smaller drive size) |
| Tank Pool | 2x 465.76 GiB HDD Mirrored → 181.07 GiB usable |
| Unassigned | 1x 232.89 GiB HDD (available as spare) |
| Total Drives | 6 HDDs |
| OS | TrueNAS SCALE Community Edition v25.04.2.6 |

> ⚠️ **Note:** The HP Z800 supports dual Intel Xeon X5675 CPUs. The second CPU is currently not posting, so the system is running on a single CPU with half the expected memory capacity. Full specs will be restored once the second CPU issue is resolved.

---

## 💡 Project Philosophy — Budget Build

This entire server was built using **recycled and spare hardware** — an old HP Z800 workstation that hadn't been used in years, paired with spare hard drives that were sitting around unused. The goal was to prove that you don't need expensive, brand-new equipment to run a fully functional, reliable home media server.

> "You don't need a $1,000 server to do this. You need patience and willingness to learn."

### Budget Build Highlights
- HP Z800 workstation sourced as old/unused hardware
- All HDDs were spare drives, not purchased new
- Free and open-source software stack (TrueNAS, Jellyfin, Docker)
- Running 10+ services simultaneously on a single aging Xeon CPU

### On Disk Sizes & Trade-offs
The Storage pool intentionally uses **mismatched drive sizes** (465 GiB + 298 GiB). In a ZFS mirror, usable space is limited to the smaller drive — meaning some capacity is "lost." This is a conscious trade-off: **data safety over raw storage capacity**. Without mirroring, you'd have more space but zero protection against drive failure.

---

## 💾 Storage Layout (TrueNAS SCALE)

Two ZFS datasets are configured, both with **mirrored drives for redundancy**:

| Dataset | Size | Purpose | Sharing |
|---------|------|---------|---------|
| Storage | ~286 GiB | General home storage | SMB (whole house network share) |
| Tank | ~181 GiB | App configs & media | Docker volumes |

### Why ZFS Mirroring?
Both datasets use mirrored drives, meaning data is written to two drives simultaneously. If one drive fails, no data is lost. This is a key concept in IT: **redundancy protects against hardware failure**.

---

## 🐳 Stack Overview

All services run as Docker containers managed by **Dockge** — a modern Docker Compose manager with a web UI.

### Services & Ports

| Service | Port | Purpose |
|---------|------|---------|
| Jellyfin | 8096 | Media streaming server |
| Jellyseerr | 5055 | Media request management |
| Sonarr | 8989 | TV show automation |
| Radarr | 7878 | Movie automation |
| Prowlarr | 9696 | Indexer/tracker management |
| Bazarr | 6767 | Subtitle automation |
| Profilarr | 6868 | Quality profile management |
| qBittorrent | 8080 | Download client (VPN protected) |
| Unpackerr | — | Automated archive extraction |
| Qui | 7476 | Queue management UI |

### How They Work Together

```
User Request (Jellyseerr)
        ↓
Sonarr (TV) / Radarr (Movies)
        ↓
Prowlarr (finds download sources)
        ↓
qBittorrent (downloads via VPN)
        ↓
Unpackerr (extracts archives)
        ↓
Jellyfin (streams to your devices)
        ↓
Bazarr (adds subtitles automatically)
```

---

## 🔒 Security & VPN

All download traffic is routed through **AirVPN** via a **WireGuard tunnel** built directly into the qBittorrent container. This means no downloads happen without VPN protection — if the VPN drops, the container loses internet access entirely.

### How the VPN is implemented

The VPN is configured entirely through the `qbittorrent` service in `compose.yaml`:

```yaml
- VPN_ENABLED=true          # Activates VPN inside the container
- VPN_CONF=wg0              # Points to the AirVPN WireGuard config file (wg0.conf)
- VPN_PROVIDER=generic      # Uses generic WireGuard provider mode
- VPN_LAN_NETWORK=10.0.0.0/24  # Excludes local network from VPN tunnel
```

The actual AirVPN credentials are stored in a WireGuard config file (`wg0.conf`) located in the qBittorrent config folder on TrueNAS — they are never exposed in the compose file.

| Setting | Purpose |
|---------|---------|
| `NET_ADMIN` capability | Allows the container to manage its own network interfaces |
| `src_valid_mark=1` | Required for WireGuard routing rules to work correctly |
| `disable_ipv6=1` | Prevents IPv6 traffic from bypassing the VPN tunnel (IP leak prevention) |
| `VPN_LAN_NETWORK` | Keeps local network accessible while routing all internet traffic through AirVPN |

- API keys are stored as environment variables, not hardcoded into scripts

---

## 📄 What is a Docker Compose File?

The `compose.yaml` file is written in **YAML** (Yet Another Markup Language) — not Bash. It's a configuration file that tells Docker:

- Which container images to use
- What ports to expose
- Where to store data (volumes)
- Environment variables (like timezone and user permissions)
- When to restart containers

Example snippet:
```yaml
jellyfin:
  container_name: jellyfin
  image: lscr.io/linuxserver/jellyfin:latest
  ports:
    - 8096:8096   # host_port:container_port
```

The `8096:8096` port mapping means: *"expose the container's internal port 8096 to port 8096 on the host machine."* This is how you access Jellyfin in your browser at `http://your-server-ip:8096`.

---

## 📦 compose.yaml

```yaml
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=568
      - PGID=568
      - TZ=America/New_York
    volumes:
      - /mnt/Tank/configs/prowlarr:/config
      - /mnt/Tank/Media/:/media
    ports:
      - 9696:9696
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=568
      - PGID=568
      - TZ=America/New_York
    volumes:
      - /mnt/Tank/configs/sonarr:/config
      - /mnt/Tank/Media:/media
    ports:
      - 8989:8989
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=568
      - PGID=568
      - TZ=America/New_York
    volumes:
      - /mnt/Tank/configs/radarr:/config
      - /mnt/Tank/Media:/media
    ports:
      - 7878:7878
    restart: unless-stopped

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=568
      - PGID=568
      - TZ=America/New_York
    volumes:
      - /mnt/Tank/configs/bazarr:/config
      - /mnt/Tank/Media:/media
    ports:
      - 6767:6767
    restart: unless-stopped

  profilarr:
    image: santiagosayshey/profilarr:beta
    container_name: profilarr
    ports:
      - 6868:6868
    volumes:
      - /mnt/Tank/configs/profilarr:/config
    environment:
      - TZ=America/New_York
    restart: unless-stopped

  unpackerr:
    image: golift/unpackerr
    container_name: unpackerr
    volumes:
      - /mnt/Tank/Media:/media
    restart: unless-stopped
    user: 568:568
    environment:
      - TZ=America/New_York
      - UN_SONARR_0_URL=http://YOUR_SERVER_IP:8989
      - UN_SONARR_0_API_KEY=YOUR_SONARR_API_KEY
      - UN_SONARR_0_PATHS_0=/media/downloads
      - UN_SONARR_0_PROTOCOLS=torrent
      - UN_RADARR_0_URL=http://YOUR_SERVER_IP:7878
      - UN_RADARR_0_API_KEY=YOUR_RADARR_API_KEY
      - UN_RADARR_0_PATHS_0=/media/downloads
      - UN_RADARR_0_PROTOCOLS=torrent

  qbittorrent:
    container_name: qbittorrent
    image: ghcr.io/hotio/qbittorrent:release-5.1.2
    restart: unless-stopped
    ports:
      - 8080:8080
    environment:
      - PUID=568
      - PGID=568
      - TZ=America/New_York
      - VPN_ENABLED=true
      - VPN_CONF=wg0
      - VPN_PROVIDER=generic
      - VPN_LAN_NETWORK=10.0.0.0/24
    cap_add:
      - NET_ADMIN
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=1
    volumes:
      - /mnt/Tank/configs/qbittorrent:/config
      - /mnt/Tank/Media:/media

  qui:
    image: ghcr.io/autobrr/qui:latest
    container_name: qui
    user: 568:568
    restart: unless-stopped
    ports:
      - 7476:7476
    volumes:
      - /mnt/Tank/configs/qui:/config
      - /mnt/Tank/Media:/media

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - LOG_LEVEL=debug
      - TZ=America/New_York
    ports:
      - 5055:5055
    user: 568:568
    volumes:
      - /mnt/Tank/configs/jellyseerr:/app/config
    restart: unless-stopped

  jellyfin:
    container_name: jellyfin
    image: lscr.io/linuxserver/jellyfin:latest
    environment:
      - PUID=568
      - PGID=568
      - TZ=America/New_York
    ports:
      - 8096:8096
    restart: unless-stopped
    volumes:
      - /mnt/Tank/configs/jellyfin:/config
      - /mnt/Tank/Media:/media

networks: {}
```

---

## 🛡️ Data Protection

Even on a budget build, data protection was taken seriously. TrueNAS automates several layers of protection:

### ZFS Scrub Tasks
Scrubs verify data integrity across the entire pool — detecting and correcting silent corruption.

| Pool | Schedule |
|------|---------|
| Storage | Every Sunday at 11:00 PM |
| Tank | Every Sunday at 12:00 AM |

### Periodic Snapshots
Point-in-time snapshots of the Storage pool allow recovery from accidental deletion or corruption.

| Frequency | Retention |
|-----------|----------|
| Every hour | 3 days |
| Every day | 2 weeks |
| Every week | 2 months |

### S.M.A.R.T. Disk Health Tests
S.M.A.R.T. tests monitor physical drive health and catch early signs of failure before data is lost.

| Test Type | Disks | Schedule |
|-----------|-------|---------|
| LONG | sdb, sdc (Storage pool) | Monthly — 1st of month at 5:00 AM |
| SHORT | All Disks | Every Sunday at 12:00 AM |

---

## 📚 Resources & References

- [TrueNAS SCALE Documentation](https://www.truenas.com/docs/scale/)
- [Jellyfin Documentation](https://jellyfin.org/docs/)
- [LinuxServer.io Docker Images](https://docs.linuxserver.io/)
- [Dockge GitHub](https://github.com/louislam/dockge)
- [Trash Guides (Arr Stack Wiki)](https://trash-guides.info/) — community wiki for Arr stack quality profiles and setup
- [Servers at Home Wiki](https://wiki.serversatho.me) — the primary guide used to build this stack
- [YouTube Tutorial](https://www.youtube.com/watch?v=AzGq2lJSKpo) — the video walkthrough that inspired this build
- [AirVPN](https://airvpn.org) — VPN provider used with WireGuard for download privacy

---

## 🧠 What I Learned

- How to install and configure **TrueNAS SCALE** on server hardware
- Setting up **ZFS datasets** with mirroring for data redundancy
- Configuring **SMB file shares** for network-wide access
- Deploying and managing **Docker containers** using Dockge
- Understanding **Docker Compose YAML** syntax and structure
- How **port mapping** works in containerized applications
- Integrating a **WireGuard VPN** at the container level for privacy
- How the **Arr stack** applications communicate with each other via APIs

---

## 🚧 Future Plans

- [ ] Set up remote access (Tailscale or Cloudflare Tunnel)
- [ ] Add Heimdall or Homarr as a dashboard
- [ ] Learn to write Docker Compose files from scratch
- [ ] Add monitoring with Uptime Kuma
- [ ] Document the WireGuard VPN setup process

---

*Built as part of a home lab learning journey alongside studying for CompTIA A+*
