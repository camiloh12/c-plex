# Troubleshooting

## "Title stuck in download queue"

Likely causes:
- No seeders for the chosen release.
- Indexer returned bad metadata.

Steps:
1. In Sonarr/Radarr Activity → Queue, hover the title → see error.
2. Right-click → Remove → "Remove from download client" → blocklist.
3. The *arr will search again automatically. If multiple grabs fail,
   tighten the quality profile or add more indexers.

## "Import failed"

Likely cause: the imported path differs from the download path inside
the container (breaks hardlinks).

Steps:
1. `docker compose exec sonarr ls /data/torrents/tv/<dir>` — confirm file exists.
2. `docker compose exec sonarr ls /data/media/tv` — confirm root folder exists.
3. Check Sonarr → System → Logs for the exact error.
4. If "permission denied", verify ownership: `ls -la /data/torrents/tv/<dir>`
   should be UID/GID 1000.

## "No indexers responding"

Likely cause: indexer is down, behind Cloudflare, or rate-limited.

Steps:
1. In Prowlarr → Indexers → click each → Test. Note which fail.
2. If "Cloudflare challenge", tag the indexer with the FlareSolverr proxy.
3. If "401/403", credentials may have expired (private trackers) — re-auth.
4. Check FlareSolverr logs: `docker compose logs flaresolverr --tail 30`.

## "Plex relay slow / disconnects"

Cause: external user is on Plex relay (max 1 Mbps total).

Fix:
- Ensure Remote Access shows "fully accessible" (UPnP or manual forward).
- If it shows "outside network unavailable", forward TCP 32400 → host IP
  manually on your router.

## "qBittorrent shows no network"

Cause: gluetun is unhealthy or stopped.

Steps:
1. `docker compose ps gluetun` — check status.
2. `docker compose logs gluetun --tail 50` — check for VPN auth errors.
3. Restart: `docker compose restart gluetun` then `docker compose restart qbittorrent`.
4. If persistent, rotate to a different VPN server in `.env` `SERVER_COUNTRIES`.

## "Plex transcoding falls back to CPU"

Cause (Mode A): NVENC misconfigured.

Steps:
1. On host: `nvidia-smi` should work and show the GPU.
2. `docker compose exec plex nvidia-smi` should ALSO work (proves passthrough).
3. Plex Settings → Transcoder shows the GPU in the "Hardware encoding device"
   dropdown.
4. If GPU is not in dropdown: check `runtime: nvidia` and `NVIDIA_*` env vars
   in compose.
