# Rotate VPN Provider

When changing VPN provider (e.g., Mullvad → ProtonVPN) or rotating credentials:

1. Get new WireGuard credentials from the new provider (private key + addresses).
2. Edit `/opt/c-plex/.env`:
   ```
   VPN_SERVICE_PROVIDER=newprovider
   WIREGUARD_PRIVATE_KEY=<new-key>
   WIREGUARD_ADDRESSES=<new-addresses>
   SERVER_COUNTRIES=<countries>
   ```
3. Restart the affected containers:
   ```
   docker compose restart gluetun qbittorrent
   ```
4. Run the kill-switch and IP-leak tests from
   `docs/runbooks/post-deploy-verification.md` (§§2-3).
5. Update the password manager with new credentials.

Supported providers and their config requirements:
https://github.com/qdm12/gluetun-wiki
