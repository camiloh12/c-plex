# Re-sync TRaSH Guides via Recyclarr

If a quality profile in Sonarr/Radarr has drifted (manual edits in the UI
overrode the Recyclarr config), re-run the sync to restore baseline.

## Manual sync

```
cd /opt/c-plex
docker compose run --rm recyclarr sync
```

Expected: output lists profile and custom-format changes applied. No errors.

## After changing recyclarr/config.yml

1. Edit `recyclarr/config.yml` on the DEV machine.
2. Commit and push.
3. On the host:
   ```
   cd /opt/c-plex && git pull
   docker compose run --rm recyclarr sync
   ```

## Scheduled sync

Recyclarr runs daily at 03:00 via root cron (see Task 3.4 Step 6).
Verify the cron entry: `sudo crontab -l`.

Log file: `/var/log/recyclarr.log`.
