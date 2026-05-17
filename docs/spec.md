# c-plex — Project Specification

**Status:** Living document. Update in place as the project evolves.
**Last updated:** 2026-05-17

This document is the single source of truth for what c-plex is, what hardware
it runs on, what software it uses, and how it is operated. Implementation
plans and runbooks live elsewhere and reference this spec.

---

## 1. Overview

c-plex is a self-hosted media streaming platform for a single household. A
Plex Media Server hosts a movie and TV library that is built and maintained
automatically: household members request titles through a web UI, an
automation stack finds and downloads them via BitTorrent through a VPN, and
the resulting files are imported into Plex with correct naming and metadata.
Viewers stream from any Plex client (TV, phone, browser, console).

The platform runs on a single repurposed PC on the home LAN. Torrent traffic
is isolated inside a VPN tunnel with a kill-switch. Remote viewing uses
Plex's built-in remote-access mechanism.

## 2. Goals and non-goals

### Goals

- Serve a 1080p movie and TV library to 2-5 household members.
- Support up to ~3 concurrent streams (mix of direct play and transcoded).
- Let users request new content through a self-service UI.
- Automatically download, organise, and import requested content.
- Keep all torrent traffic inside a VPN tunnel with a verifiable kill-switch.
- Be reproducible: the entire stack is defined in a single git repository
  and can be redeployed from scratch in under an hour.

### Non-goals

- 4K, HDR, or Dolby Vision content.
- More than ~5 concurrent streams.
- Public access to admin UIs.
- Sharing with users outside the household.
- Backups of the media library (configs are backed up; media is treated as
  reconstructable).
- Storage redundancy (RAID, mergerfs, SnapRAID).
- Usenet, audiobooks, music, comics, live TV, or DVR.

## 3. Audience and scale

| Dimension | Target |
|---|---|
| Total users | 2-5 (household) |
| Concurrent streams | Up to 3 typical, 5 peak |
| Library size (year 1) | Up to ~4 TB (≈500-800 movies or equivalent TV) |
| Quality | 1080p (h.264 / h.265), no 4K |
| Access pattern | Mostly home LAN, occasional remote |

## 4. Hardware requirements

### 4.1 Host

Single physical machine, always-on, wired to the home router.

| Component | Minimum | Recommended | Notes |
|---|---|---|---|
| CPU | 4 cores / 8 threads, x86_64 | Same or better | All transcoding is offloaded to the GPU when available |
| RAM | 8 GB | 16 GB | 8 GB works; 16 GB gives real headroom for cache and future additions |
| GPU | None (CPU-only fallback) | Discrete Nvidia (NVENC-capable) | Pascal (GTX 10xx) or newer recommended for h.265 encode |
| OS drive | 50 GB free on internal SSD/HDD | Same | Holds Ubuntu, Docker, and `/opt/appdata` (configs + container DBs) |
| Media drive | 1× 4 TB external USB 3.0 HDD | Same | ext4. Mounted at `/mnt/media-01`, bind-mounted as `/data` in containers |
| Network | Wired Gigabit Ethernet | Same | Wi-Fi is not supported for the server |
| UPS | None | 600 VA basic UPS | Prevents corruption from brownouts during writes |

The currently available host is a repurposed laptop with Ubuntu 20.04.6 LTS
installed. It will be upgraded to Ubuntu 24.04 LTS before deployment (see
§5.1).

### 4.2 Network (WAN)

| Resource | Requirement |
|---|---|
| Download bandwidth | ≥ 50 Mbps (for timely library growth) |
| Upload bandwidth | ≥ 10 Mbps if remote streaming is used; ≥ 25 Mbps comfortable for 2 concurrent remote 1080p streams |
| Public IP | Not required; Plex relay works behind NAT |
| Router | UPnP enabled OR ability to manually port-forward TCP 32400 to the host |

### 4.3 Subscriptions

| Service | Required | Cost | Purpose |
|---|---|---|---|
| Plex Pass | Conditional — see §6.4 | $5/mo or $120 lifetime | Required to enable hardware transcoding (Nvidia NVENC). Also unlocks mobile sync, skip intro/credits, and other quality-of-life features. |
| Torrent VPN | Yes | ~$5/mo | Must allow P2P. Commonly used providers that do: Mullvad, ProtonVPN, AirVPN, IVPN. Provider choice is deferred to the operator. |

## 5. Software architecture

### 5.1 Host OS

- **Distribution:** Ubuntu Server 24.04 LTS (Noble).
- **Why:** current LTS, supported through April 2029 (April 2034 with ESM),
  excellent Docker and Nvidia driver support, lowest overhead on 8 GB RAM.
- **Required packages:**
  - `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin`
    (from Docker's official APT repo, not Ubuntu's older `docker.io`)
  - `nvidia-driver-535-server` or newer (proprietary)
  - `nvidia-container-toolkit`
  - `ufw`, `unattended-upgrades`, `openssh-server`
- **User model:** all containers run as a single non-root UID/GID
  (`PUID=1000`, `PGID=1000`) that owns `/data` and `/opt/appdata` on the
  host. Container file ownership matches host user ownership.

### 5.2 Container inventory

All services run as Docker containers orchestrated by a single
`docker-compose.yml`.

| Tier | Service | Image | Port | Purpose |
|---|---|---|---|---|
| Media | Plex | `lscr.io/linuxserver/plex:latest` | 32400 (host net) | Stream library to clients; NVENC transcode |
| Media | Tautulli | `lscr.io/linuxserver/tautulli:latest` | 8181 | Plex usage stats and history |
| Automation | Sonarr | `lscr.io/linuxserver/sonarr:latest` | 8989 | TV-series automation (v4) |
| Automation | Radarr | `lscr.io/linuxserver/radarr:latest` | 7878 | Movie automation (v5) |
| Automation | Prowlarr | `lscr.io/linuxserver/prowlarr:latest` | 9696 | Indexer aggregator; pushes indexer config to Sonarr/Radarr |
| Automation | FlareSolverr | `ghcr.io/flaresolverr/flaresolverr:latest` | 8191 (internal) | Solves Cloudflare challenges on public indexers |
| Automation | Seerr | `ghcr.io/seerr-team/seerr:latest` | 5055 | User-facing request UI (Plex-SSO) |
| Automation | Bazarr | `lscr.io/linuxserver/bazarr:latest` | 6767 | Subtitle automation |
| Automation | Recyclarr | `ghcr.io/recyclarr/recyclarr:latest` | n/a | Syncs TRaSH-Guide quality profiles into Sonarr/Radarr on schedule |
| Download | gluetun | `qmcgaw/gluetun:latest` | n/a | VPN tunnel + healthcheck; provides network namespace |
| Download | qBittorrent | `lscr.io/linuxserver/qbittorrent:latest` | 8080 (via gluetun) | Torrent client; shares gluetun's network namespace |
| Ops | Watchtower | `containrrr/watchtower:latest` | n/a | Weekly auto-update of container images (optional) |
| Ops | Homepage | `ghcr.io/gethomepage/homepage:latest` | 3000 | Single-page dashboard linking all services (optional) |

All ports in the table above are **published on the host** (reachable from
the LAN at the host's IP) **except FlareSolverr (8191) and Recyclarr,
Watchtower, gluetun, qBittorrent** — these have no host port mapping.
FlareSolverr is reached only by Prowlarr over the internal docker bridge;
qBittorrent's WebUI is reached via gluetun's published 8080.

LinuxServer.io images are preferred where available for consistent
`PUID`/`PGID`/`TZ` env-var handling and weekly base-image security updates.

Image tags default to `:latest` with Watchtower handling updates. Operators
who prefer manual control may pin to specific version tags; the spec
supports both.

### 5.3 Network topology

```
                           Home LAN (192.168.x.0/24)
                                    │
                              ┌─────┴──────┐
                              │  Ubuntu    │
                              │  Host      │
                              └─────┬──────┘
                                    │
                            docker0 bridge
                                    │
  ┌─────────────┬─────────────┬─────┴──────┬──────────────┬───────────┐
  │             │             │            │              │           │
 Plex         Sonarr/        Seerr     Prowlarr +      gluetun     Tautulli /
 :32400       Radarr         :5055     FlareSolverr   (VPN ns)     Bazarr /
 host-NW      :8989/:7878                              │            Homepage
              (bridge)                          ┌─────┴──────┐
                                                │ qBittorrent│
                                                │ :8080      │
                                                │ shares     │
                                                │ gluetun    │
                                                │ netns      │
                                                └────────────┘
                                                       │
                                                  VPN tunnel
                                                       │
                                                   Internet
```

- **Plex** uses `network_mode: host` so it can advertise itself on the LAN
  via Plex GDM auto-discovery.
- **qBittorrent** uses `network_mode: "service:gluetun"`. It has no other
  network interface, so when gluetun is unhealthy or down, qBittorrent has
  no network access. This *is* the VPN kill-switch.
- **All other containers** join one user-defined bridge network and
  communicate by container name (e.g., Sonarr → `http://gluetun:8080` for
  the qBittorrent WebUI, Seerr → `http://radarr:7878`).
- **No reverse proxy.** The only port deliberately reachable from the
  internet is Plex's 32400, which Plex itself negotiates via UPnP or relay.

### 5.4 Data flow: request to stream

1. User opens Seerr in a browser, searches a movie, clicks **Request**.
2. Seerr posts the request to Radarr.
3. Radarr queries Prowlarr's indexer aggregate. FlareSolverr handles
   Cloudflare-protected indexers transparently.
4. Radarr scores returned releases against its quality profile (synced from
   TRaSH guides by Recyclarr) and picks the best match.
5. Radarr sends the chosen torrent/magnet to qBittorrent via the
   gluetun-exposed API.
6. qBittorrent downloads through the VPN tunnel into `/data/torrents/movies/`.
7. On completion, qBittorrent notifies Radarr. Radarr **hardlinks** the file
   into `/data/media/movies/Movie Name (Year)/Movie.mkv` with correct naming.
8. Plex's library watcher detects the new file, fetches metadata from TMDB,
   and the title appears in the library.
9. Seerr marks the request as **Available** and notifies the requester.
10. User streams from any Plex client. Plex direct-plays if the client
    supports the codec/container; otherwise NVENC transcodes on the fly
    (Mode A, §6.4).

### 5.5 Storage layout

The library and download trees share a single filesystem under one parent.
This is the prerequisite for hardlinks to work; without it, "import" silently
falls back to copy and doubles disk usage.

```
/data/                              ← single mountpoint (4 TB external HDD)
├── torrents/                       ← qBittorrent writes here
│   ├── movies/
│   ├── tv/
│   └── incomplete/                 ← .!qB partial downloads
└── media/                          ← Plex reads from here
    ├── movies/
    │   └── Movie Name (2024)/
    │       └── Movie Name (2024).mkv     ← hardlink → torrents/movies/...
    └── tv/
        └── Show Name (2023)/
            └── Season 01/
                └── Show.Name.S01E01.mkv  ← hardlink → torrents/tv/...
```

The exact same path tree (`/data` → `/data`) is mounted into Plex, Sonarr,
Radarr, qBittorrent, and Bazarr. If any container sees the data under a
different path, hardlinks will not work between containers.

Container configuration and databases live at `/opt/appdata/<service>/` on
the host's internal drive — bind-mounted into each container at the path
that service expects (typically `/config`).

### 5.6 Quality profiles and content strategy

- **Recyclarr** runs on a schedule (e.g., daily) and syncs TRaSH-Guide
  quality profiles and custom formats into Sonarr and Radarr.
- **Default profile (Mode A — Plex Pass + NVENC):** WEB-1080p with
  preference for h.265 where available; falls back gracefully to h.264.
  Max episode/movie size capped by TRaSH defaults.
- **Default profile (Mode B — direct-play-only):** h.264 in MP4/MKV
  containers, AAC/AC3 audio, 1080p max, ~8 Mbps bitrate ceiling. x265 and
  HDR formats blocked. Chosen for broadest client compatibility without
  transcoding.
- Recyclarr's `config.yml` is committed to the repo. Profile changes go
  through git, not through the Sonarr/Radarr UI.

## 6. Operational requirements

### 6.1 Repository layout

```
/opt/c-plex/                        ← project root (this git repo, checked out on the host)
├── docker-compose.yml
├── .env                            ← gitignored; secrets only
├── .env.example                    ← committed; documents every required variable
├── docs/
│   └── spec.md                     ← this file
├── recyclarr/
│   └── config.yml
├── gluetun/                        ← VPN provider config (no secrets)
└── README.md
```

### 6.2 Secrets

- VPN credentials, Plex claim token, *arr API keys, Seerr API keys all live
  in `.env`. `.env` is in `.gitignore`. `.env.example` lists every variable
  name (with placeholder values) so the file is self-documenting.
- Operator keeps a copy of `.env` in a password manager.

### 6.3 Image updates

- **Watchtower** runs weekly (default: Sunday 04:00 local). Pulls latest
  images, restarts containers cleanly. Sends one notification per run
  (Discord webhook or ntfy).
- Operators who prefer manual updates may disable Watchtower and run
  `docker compose pull && docker compose up -d` on their own cadence.

### 6.4 Plex transcoding modes

The spec supports both modes. Operators choose at deployment time based on
whether they hold a Plex Pass.

**Mode A — Plex Pass present (recommended).**
The Plex container is granted GPU access via `runtime: nvidia` with env
vars `NVIDIA_VISIBLE_DEVICES=all` and
`NVIDIA_DRIVER_CAPABILITIES=compute,video,utility`. In Plex Settings →
Transcoder, "Use hardware acceleration when available" and
"Use hardware-accelerated video encoding" are enabled. The transcode
working directory is set to a tmpfs mount to spare the HDD. NVENC
comfortably handles 4-6 concurrent 1080p transcodes on the spec'd hardware.

For older consumer Nvidia GPUs, the proprietary driver limits NVENC to 3
concurrent sessions. The `keylase/nvidia-patch` community patch removes
this cap and is recommended only if the limit is hit in practice.

**Mode B — no Plex Pass (fallback).**
The Plex container has no GPU access. Hardware acceleration is disabled in
Plex settings. The spec mandates the direct-play-only quality profile
(§5.6) so that no transcoding is ever needed in normal operation. Operators
choosing this mode accept that any client unable to direct-play a given
file will fail to play it rather than fall back to slow CPU transcoding.

## 7. Security and access

### 7.1 Internet exposure

- **Plex (32400)** is the only deliberately internet-reachable service.
  Plex itself negotiates NAT traversal via UPnP or its relay service.
- **All admin UIs** (Sonarr, Radarr, Prowlarr, qBittorrent, Tautulli,
  Bazarr, Seerr admin) are LAN-only.
- **For remote admin**, the recommended path is to install Tailscale on
  the host and on the operator's phone/laptop, then access admin UIs over
  the Tailscale mesh. Tailscale is listed as an optional add-on; the
  baseline spec does not require it.
- **Seerr's user-facing UI** is LAN-only in this spec. Exposing it
  publicly (reverse proxy + TLS + auth) is out of scope until/unless the
  user base grows beyond the household.

### 7.2 VPN kill-switch

The kill-switch is structural, not configured: qBittorrent shares
gluetun's network namespace and has no other network interface, so traffic
cannot route around the VPN even on misconfiguration.

The compose file declares:

- `depends_on: { gluetun: { condition: service_healthy } }` on qBittorrent,
  so qBittorrent does not start until gluetun reports healthy.
- A `HEALTHCHECK` on gluetun that pings the VPN endpoint.

Two manual verifications are mandatory after initial deploy and after any
gluetun config change:

1. **Kill-switch test:** stop gluetun; confirm qBittorrent's WebUI loses
   network access; restart gluetun; confirm recovery.
2. **IP leak test:** from inside the qBittorrent container,
   `curl ifconfig.me` returns the VPN provider's IP, not the home ISP's.

### 7.3 Authentication

- **Plex Server** is claimed by the operator's Plex account at first start.
- **Household members** each have their own Plex account; library access
  is granted via Plex Home or Plex Friends.
- **Seerr** uses Plex SSO; no separate credentials.
- **All *arr UIs** use built-in form auth with operator-chosen passwords,
  enforced as "Forms (Login Page)" auth method.

### 7.4 Host hardening

Minimal, scoped to a household single-host deployment:

- `ufw` allows: SSH (22) from LAN, Plex (32400) from anywhere, all admin
  ports (8989, 7878, 9696, 8080, 5055, 6767, 8181, 3000) from LAN only
  (`192.168.x.0/24`).
- `unattended-upgrades` enabled for OS security patches.
- SSH key-based auth only; password auth disabled in `sshd_config`.
- No fail2ban, no IDS, no log shipping. Not justified at this scale.

## 8. Backups and disaster recovery

### 8.1 What is backed up

| Artefact | Location | Backup target | Cadence | Retention |
|---|---|---|---|---|
| `docker-compose.yml`, `.env.example`, `recyclarr/`, `gluetun/`, `docs/` | git repo | Remote git host (private) | On every change | Full git history |
| `.env` (secrets) | Host `/opt/c-plex/.env` | Operator's password manager | On every change | Latest only |
| Container configs and DBs | `/opt/appdata/` | Tarball to a second drive or cloud bucket | Nightly | 7 daily + 4 weekly |
| Media library | `/data/media/` | Not backed up | n/a | n/a (reconstructable) |

### 8.2 Recovery scenarios

| Failure | Recovery procedure | Estimated downtime |
|---|---|---|
| OS drive dies | Reinstall Ubuntu 24.04; clone the git repo to `/opt/c-plex`; restore `/opt/appdata` from latest backup; restore `.env` from password manager; `docker compose up -d`. | ~1 hour |
| Media drive dies | Replace drive; reformat ext4; mount as `/mnt/media-01`. Sonarr/Radarr request history is intact, so re-search and re-download from indexers. | Days to weeks for full library rebuild |
| Both drives die | Rebuild from git repo and indexers as above; lose all configs except whatever is in git. | Days to weeks |
| Compose config corruption | `git checkout` last known good revision; `docker compose up -d`. | Minutes |

## 9. Monitoring

Lightweight, sized for household scale:

- **Tautulli** — Plex usage and per-user watch history.
- **Container healthchecks** — every service defines a Docker `HEALTHCHECK`;
  Docker restarts on consecutive failures. gluetun's healthcheck is
  load-bearing for kill-switch behaviour.
- **Notifications** — single channel (Discord webhook or ntfy). Seerr
  notifies users of request status; Sonarr/Radarr notify the operator of
  errors and successful imports; Watchtower notifies on update runs.
- **No Prometheus, Grafana, Loki, or alerting stack.** Out of scope at
  this scale.

## 10. Runbook (referenced, not inlined)

The following runbook items live as separate documents under `docs/runbooks/`
and reference this spec. They are listed here so the spec's surface area is
explicit:

- Post-deploy verification checklist (kill-switch, NVENC, hardlink test,
  Plex claim, indexer config).
- Adding a new user to Plex and Seerr.
- Re-syncing TRaSH guides via Recyclarr after a profile drift.
- Rotating the VPN provider.
- Troubleshooting: "title stuck in download queue", "import failed",
  "no indexers responding", "Plex relay slow / disconnects".
- Upgrading Ubuntu, Docker, or the Nvidia driver.

## 11. Open questions and deferred decisions

| Question | Owner | Status |
|---|---|---|
| Plex Pass: purchase or defer? Drives transcoding mode (§6.4). | Operator | Open |
| Which VPN provider? (Mullvad / Proton / AirVPN / IVPN are the standard P2P-allowing options.) | Operator | Open |
| Notification channel: Discord webhook, ntfy, or email? | Operator | Open |
| RAM upgrade: 8 GB → 16 GB now or wait? | Operator | Open (deferred per §4.1) |
| Add Tailscale for remote admin? | Operator | Open (recommended optional add-on, §7.1) |

## 12. Maintenance of this document

This spec is a living document. Update it in place whenever the project
changes in a way that affects requirements, architecture, operations, or
scope. Specifically:

- **Adding a service** → update §5.2 inventory, §5.3 topology, and any
  affected port lists in §7.4.
- **Changing storage** → update §5.5 and §4.1.
- **Changing the access model** (e.g., exposing Seerr publicly) → update
  §7.1 and move the item out of §2's non-goals.
- **Resolving an open question in §11** → bake the decision into the
  relevant section and remove the entry from §11.
- **Each substantive update** → bump the "Last updated" date at the top.

Per-change history lives in git; this document captures the current state.
