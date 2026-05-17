# CLAUDE.md

This file provides guidance for Claude Code when working in this repository.

## Project overview

c-plex is a self-hosted Plex media streaming platform for a single household
(2-5 users). It runs as a Docker Compose stack on a single Ubuntu Server host
with a torrent automation pipeline (Sonarr/Radarr/Prowlarr + qBittorrent)
isolated inside a VPN tunnel, and a user-facing request UI (Seerr).

The **single source of truth** is `docs/spec.md` (living document).
The **current implementation plan** is in `docs/plans/`.
**Operational procedures** live in `docs/runbooks/` (created during execution).

## Two-host workflow

This is an infrastructure project with two distinct contexts. Every task in
the plan is prefixed `[DEV]` or `[HOST]` to indicate where it runs.

- **DEV** — the Windows machine where this git repo lives
  (`C:\Users\camil\projects\c-plex\`). All config-file authoring,
  spec/plan editing, and git commits happen here.
- **HOST** — the Ubuntu Server laptop where the stack actually runs
  (`/opt/c-plex/`). Deploy steps (`docker compose up -d`, health checks,
  VPN kill-switch tests, drive mounts, OS upgrades) run here over SSH or
  at the console.

Respect this distinction. Don't try to `docker compose up` from Windows —
the stack lives on the Ubuntu box.

## Document conventions

| Document type | Path | Naming | Lifecycle |
|---|---|---|---|
| Spec | `docs/spec.md` | No date prefix | LIVING — update in place; bump "Last updated:" header |
| Plan | `docs/plans/YYYY-MM-DD-*.md` | Date-prefixed | Point-in-time; not edited after execution starts |
| Runbook | `docs/runbooks/*.md` | No date prefix | Created during plan execution; updated as procedures evolve |

The "no date for the spec" rule is per the operator's explicit instruction —
the spec is the project's living contract, not an artifact frozen in time.

## Critical project invariants

These break the stack if violated:

1. **Hardlink path consistency.** `/data` must be bind-mounted at the SAME
   path (`/data` → `/data`) into every container that touches it
   (Plex, Sonarr, Radarr, qBittorrent, Bazarr). Mounting it under a
   different path in any container silently breaks hardlinks and doubles
   disk usage. See spec §5.5.
2. **VPN kill-switch is structural, not configured.** qBittorrent uses
   `network_mode: "service:gluetun"`. It has no other network interface.
   Never give it its own network. Run the kill-switch + IP-leak tests after
   any change to `gluetun` config (spec §7.2).
3. **Single shared UID/GID (PUID=1000, PGID=1000)** for all containers.
   Don't mix UIDs — file ownership consistency is the only thing keeping
   container ↔ host file access working cleanly.
4. **`.env` is NEVER committed.** It contains VPN private keys, *arr API
   keys, Plex claim tokens. Always reference vars via `.env.example`.

## Spec coverage and target scale

- **Audience:** 2-5 household members, ≤3 concurrent streams typical,
  5 peak.
- **Quality:** 1080p only — no 4K, no HDR, no Dolby Vision. Hardware can't
  support 4K transcoding (4 CPU cores + consumer GPU).
- **Library:** ~4 TB single external USB HDD. No RAID, no mergerfs, no
  SnapRAID. Media is treated as reconstructable; not backed up.
- **Access:** home LAN + occasional remote streaming via Plex's built-in
  remote access (UPnP or relay). No reverse proxy, no public admin UIs.
- **OS target:** Ubuntu Server 24.04 LTS. The current host runs 20.04.6
  and is being upgraded as part of deployment.

## Plex Pass / transcoding modes

Spec §6.4 defines two transcoding modes:

- **Mode A** (with Plex Pass): NVENC hardware transcoding via the discrete
  Nvidia GPU. Plex container gets `runtime: nvidia` and the `NVIDIA_*` env
  vars. Comfortable handling of 4-6 concurrent 1080p transcodes.
- **Mode B** (no Plex Pass): direct-play-only. Plex container has no GPU
  access. Quality profiles enforce h.264/MP4-MKV with broad client
  compatibility so transcoding is never needed.

The plan defaults to Mode A. Task 4.1 documents the Mode B variant inline.
The Plex Pass purchase decision is still open per spec §11.

## Don't

- Don't propose 4K, HDR, multi-server clustering, Usenet, audiobooks/music/
  comics, live TV — all explicit non-goals (spec §2).
- Don't add a reverse proxy or expose admin UIs to the internet without
  revising spec §7.1 first.
- Don't recommend RAID/mergerfs/SnapRAID or media backups — explicit
  operator choices (spec §2).
- Don't commit `.env` or any file containing real credentials.
- Don't use date prefixes on the spec filename.
- Don't expand non-goals incrementally without updating the spec first.

## When changing things

- Adding a service → update spec §5.2 inventory, §5.3 topology, and the ufw
  rules in §7.4. Add the service to `docker-compose.yml` as a separate
  commit per existing convention.
- Changing storage layout → update spec §5.5 and §4.1.
- Resolving an open question (spec §11) → bake the decision into the
  relevant spec section and remove the entry from §11. Bump "Last updated:".
- Each substantive spec change → bump "Last updated:" at the top.
