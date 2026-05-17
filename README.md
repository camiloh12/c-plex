# c-plex

Self-hosted Plex media streaming platform for a single household.

## Documentation

- **Spec:** [`docs/spec.md`](docs/spec.md) — full project specification (living document).
- **Plans:** [`docs/plans/`](docs/plans/) — implementation plans (point-in-time).
- **Runbooks:** [`docs/runbooks/`](docs/runbooks/) — operational procedures (created during deployment).
- **For Claude:** [`CLAUDE.md`](CLAUDE.md) — project context and conventions.

## What it is

A monolithic Docker Compose stack on a single Ubuntu Server host:

- **Plex Media Server** for streaming, with optional Nvidia NVENC hardware transcoding.
- **Seerr** — user-facing request UI; household members request titles in a browser.
- **Sonarr / Radarr / Prowlarr / FlareSolverr / Recyclarr / Bazarr** — full *arr automation: indexer aggregation, quality profile management, subtitle handling, auto-import.
- **qBittorrent + gluetun** — torrent client isolated in a VPN namespace with a structural kill-switch.
- **Tautulli / Homepage / Watchtower** — usage stats, dashboard, weekly auto-updates.

See [`docs/spec.md`](docs/spec.md) for the full architecture and design decisions.

## Quick deploy (on the Ubuntu host)

Prerequisites: Ubuntu 24.04 LTS, Docker Engine + Compose plugin, NVIDIA Container Toolkit (Mode A only). See spec §5.1 for full prerequisites; see [`docs/plans/`](docs/plans/) for the step-by-step deployment plan.

```bash
sudo mkdir -p /opt/c-plex && sudo chown $USER:$USER /opt/c-plex
git clone git@github.com:camiloh12/c-plex.git /opt/c-plex
cd /opt/c-plex
cp .env.example .env       # then edit .env with real secrets
docker compose up -d
```

After deploy, work through [`docs/runbooks/post-deploy-verification.md`](docs/runbooks/post-deploy-verification.md).

## Common operations

| Task | Runbook |
|---|---|
| Add a household user | `docs/runbooks/add-user.md` |
| Troubleshoot a failure | `docs/runbooks/troubleshooting.md` |
| Rotate VPN provider | `docs/runbooks/rotate-vpn-provider.md` |
| Re-sync quality profiles | `docs/runbooks/recyclarr-resync.md` |
| Upgrade Ubuntu / Docker / NVIDIA | `docs/runbooks/upgrades.md` |

## Repository layout

```
.
├── README.md                       ← this file
├── CLAUDE.md                       ← Claude Code project context
├── .gitignore
├── .env.example                    ← documents every required secret (committed)
├── .env                            ← real secrets (gitignored)
├── docker-compose.yml              ← the stack
├── recyclarr/
│   └── config.yml                  ← TRaSH-Guide quality profiles
├── gluetun/                        ← VPN provider config (no secrets)
├── scripts/
│   └── backup-appdata.sh           ← nightly /opt/appdata tarball
└── docs/
    ├── spec.md                     ← living project specification
    ├── plans/                      ← implementation plans
    └── runbooks/                   ← operational procedures
```
