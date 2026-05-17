# Post-Deploy Verification Checklist

Run this after every initial deploy and after any change to `gluetun`,
`qbittorrent`, or `plex` configuration. Every box must pass before the deploy
is considered complete.

## 1. Container health

- [ ] `docker compose ps` shows all expected services as `running`, with
      healthchecked services (`gluetun`) as `running (healthy)`.

## 2. VPN kill-switch (mandatory)

- [ ] `docker compose stop gluetun` followed by `curl http://localhost:8080`
      returns a connection error (qBittorrent has no network).
- [ ] `docker compose start gluetun` (wait 30s) and the same curl returns 200.

## 3. VPN IP-leak (mandatory)

- [ ] `docker compose exec qbittorrent curl -s https://am.i.mullvad.net/json`
      reports the VPN provider's IP, NOT the home ISP's IP.

## 4. Hardlink integrity

- [ ] After importing one test movie, `ls -li` on the file in
      `/data/torrents/movies/<dir>/` and `/data/media/movies/<dir>/`
      shows the SAME inode number.

## 5. NVENC transcoding (Mode A only)

- [ ] Force a transcode in a Plex client (lower quality below source).
      During playback, `nvidia-smi` on the host shows the Plex transcoder
      using the GPU.

## 6. End-to-end request flow

- [ ] In Seerr, request a test movie. Within 15 minutes it should appear in
      Plex library and be marked "Available" in Seerr.

## 7. Backups

- [ ] `sudo /opt/c-plex/scripts/backup-appdata.sh` runs without error and
      produces a tarball in the backup directory.

## 8. Remote access

- [ ] From a phone on cellular (off home Wi-Fi), the Plex app connects to
      the server and plays a file.

## 9. Notifications wired up

Spec §9 requires a single notification channel. For each service, confirm
the operator has configured at least one Connect/Notification target
(Discord webhook or ntfy URL — use values from `.env`'s `NOTIFY_*` vars):

- [ ] Sonarr → Settings → Connect → at least one notification connection
      added (Discord or ntfy), with "On Grab" and "On Import" enabled.
- [ ] Radarr → Settings → Connect → same as above.
- [ ] Seerr → Settings → Notifications → at least one channel enabled
      for request status updates.
- [ ] Watchtower → already configured in compose (Task 5.1) — confirm a
      test message arrived in the chosen channel.
