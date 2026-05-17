# c-plex Initial Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy the c-plex media streaming platform end-to-end from an empty git repo to a working stack on the Ubuntu host, per `docs/spec.md`.

**Architecture:** Single Ubuntu 24.04 LTS host, all services as Docker containers in one Compose file. Torrent traffic isolated in a VPN namespace (gluetun + qBittorrent). Nvidia NVENC transcoding optional (Mode A) with a CPU-direct-play-only fallback (Mode B). Configuration-as-code lives in this git repo, deployed by cloning to `/opt/c-plex/` on the host.

**Tech Stack:** Ubuntu Server 24.04, Docker Engine + Compose plugin, NVIDIA Container Toolkit, Plex Media Server, Sonarr/Radarr/Prowlarr/Bazarr, Seerr, qBittorrent, gluetun, FlareSolverr, Recyclarr, Tautulli, Watchtower, Homepage.

**Two hosts referenced in this plan:**
- **DEV** = the Windows machine where this git repo lives (`C:\Users\camil\projects\c-plex\`). Used to author/commit/push config files.
- **HOST** = the Ubuntu Server box where the stack runs (`/opt/c-plex/` after clone).

Every command is prefixed with `[DEV]` or `[HOST]`. When ambiguous, default is `[HOST]`.

**Prerequisites (operator-provided, not plan tasks):**
- Plex account exists.
- VPN provider chosen and credentials available (per §11 of spec).
- GitHub repo `git@github.com:camiloh12/c-plex.git` already pushed-to (done).
- Plex Pass decision made (Mode A or Mode B per §6.4). Mode A is the default in this plan; Task 4.1 includes the Mode B variant.

---

## Phase 0 — Repo scaffolding

### Task 0.1: Add .gitignore and README skeleton

**Files:**
- Create: `C:\Users\camil\projects\c-plex\.gitignore`
- Create: `C:\Users\camil\projects\c-plex\README.md`

- [ ] **Step 1: Verify clean repo state**

`[DEV]` Run:
```
cd C:\Users\camil\projects\c-plex
git status
```

Expected: `nothing to commit, working tree clean`. Single commit in log (the spec).

- [ ] **Step 2: Create .gitignore**

Create `C:\Users\camil\projects\c-plex\.gitignore` with this exact content:

```gitignore
# Secrets
.env
*.env.local
*.env.production

# Runtime container data (kept in /opt/appdata on host, never in repo)
appdata/
data/

# Editor/OS noise
.vscode/
.idea/
*.swp
.DS_Store
Thumbs.db

# Backup tarballs
*.tar.gz
*.tar
```

- [ ] **Step 3: Create README.md**

Create `C:\Users\camil\projects\c-plex\README.md`:

```markdown
# c-plex

Self-hosted Plex media streaming platform for a single household.

See `docs/spec.md` for the full project specification.
See `docs/plans/` for implementation plans.
See `docs/runbooks/` for operational runbooks (created during deployment).

## Quick deploy (on the Ubuntu host)

```bash
# Prerequisites: Ubuntu 24.04 LTS, Docker, NVIDIA Container Toolkit (Mode A)
# See docs/spec.md §5.1 for full prerequisites.

sudo mkdir -p /opt/c-plex && sudo chown $USER:$USER /opt/c-plex
git clone git@github.com:camiloh12/c-plex.git /opt/c-plex
cd /opt/c-plex
cp .env.example .env       # then edit .env with real secrets
docker compose up -d
```

After deploy, run through `docs/runbooks/post-deploy-verification.md`.
```

- [ ] **Step 4: Verify files exist**

`[DEV]` Run:
```
git status
```

Expected: two new untracked files (`.gitignore`, `README.md`).

- [ ] **Step 5: Commit**

`[DEV]`:
```
git add .gitignore README.md
git commit -m "Add gitignore and README skeleton"
git push
```

---

### Task 0.2: Create directory skeleton

**Files:**
- Create: `C:\Users\camil\projects\c-plex\recyclarr\.gitkeep`
- Create: `C:\Users\camil\projects\c-plex\gluetun\.gitkeep`
- Create: `C:\Users\camil\projects\c-plex\scripts\.gitkeep`
- Create: `C:\Users\camil\projects\c-plex\docs\runbooks\.gitkeep`

- [ ] **Step 1: Verify directories don't exist yet**

`[DEV]`:
```
Test-Path C:\Users\camil\projects\c-plex\recyclarr
Test-Path C:\Users\camil\projects\c-plex\gluetun
Test-Path C:\Users\camil\projects\c-plex\scripts
Test-Path C:\Users\camil\projects\c-plex\docs\runbooks
```

Expected: all return `False`.

- [ ] **Step 2: Create each directory with a .gitkeep**

`[DEV]` create each directory and place an empty `.gitkeep` inside. Content of each `.gitkeep`: empty file.

- [ ] **Step 3: Verify**

`[DEV]`:
```
git status
```

Expected: four new untracked `.gitkeep` files.

- [ ] **Step 4: Commit**

`[DEV]`:
```
git add recyclarr/.gitkeep gluetun/.gitkeep scripts/.gitkeep docs/runbooks/.gitkeep
git commit -m "Add directory skeleton for stack config, scripts, runbooks"
git push
```

---

### Task 0.3: Create .env.example with all required variables

**Files:**
- Create: `C:\Users\camil\projects\c-plex\.env.example`

- [ ] **Step 1: Create .env.example**

Create `C:\Users\camil\projects\c-plex\.env.example` with the following exact content:

```env
# c-plex environment variables
# Copy this file to .env on the host, then fill in real values.
# .env is gitignored.

# ──────────────────────────────────────────────
# Host identity (used by LinuxServer.io images)
# ──────────────────────────────────────────────
# `id $USER` on the host to get these values. 1000:1000 is the default for the first user.
PUID=1000
PGID=1000
TZ=America/New_York

# ──────────────────────────────────────────────
# Filesystem paths (on the host)
# ──────────────────────────────────────────────
# Config root for all container app-data.
APPDATA=/opt/appdata
# Data root containing /data/torrents and /data/media on the external HDD.
DATA=/data

# ──────────────────────────────────────────────
# VPN (gluetun)
# ──────────────────────────────────────────────
# Provider name as accepted by gluetun. See https://github.com/qdm12/gluetun-wiki
VPN_SERVICE_PROVIDER=mullvad
VPN_TYPE=wireguard
# WireGuard private key from your VPN provider.
WIREGUARD_PRIVATE_KEY=REPLACE_ME
# WireGuard "addresses" string from your provider's config (e.g., 10.64.0.1/32).
WIREGUARD_ADDRESSES=REPLACE_ME
# Comma-separated list of countries or cities, provider-specific. Leave blank for auto.
SERVER_COUNTRIES=

# ──────────────────────────────────────────────
# Plex
# ──────────────────────────────────────────────
# Get a fresh claim token from https://plex.tv/claim (valid for 4 minutes).
# Only needed on first boot to bind the server to your account.
PLEX_CLAIM=

# ──────────────────────────────────────────────
# Notifications (used by *arr, Seerr, Watchtower)
# ──────────────────────────────────────────────
# Discord webhook URL OR ntfy topic URL. Pick one channel, configure it per service.
NOTIFY_DISCORD_WEBHOOK=
NOTIFY_NTFY_URL=

# ──────────────────────────────────────────────
# Optional: Watchtower update schedule (cron expression, default Sunday 04:00)
# ──────────────────────────────────────────────
WATCHTOWER_SCHEDULE=0 0 4 * * 0
```

- [ ] **Step 2: Verify**

`[DEV]`:
```
git status
```

Expected: one new untracked file `.env.example`.

- [ ] **Step 3: Commit**

`[DEV]`:
```
git add .env.example
git commit -m "Add .env.example documenting all required secrets and config"
git push
```

---

## Phase 1 — Ubuntu host preparation

> All tasks in Phase 1 run on **[HOST]** over SSH. Verify SSH access from `[DEV]` first (`ssh user@host-ip`).

### Task 1.1: Upgrade Ubuntu 20.04 → 24.04 LTS

This is a two-step upgrade (20.04 → 22.04 → 24.04). Do this on the console with physical access available; SSH may drop during the upgrade.

- [ ] **Step 1: Verify current version**

`[HOST]`:
```
lsb_release -a
```

Expected: `Ubuntu 20.04.6 LTS` (or close).

- [ ] **Step 2: Update current release fully**

`[HOST]`:
```
sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y
sudo reboot
```

- [ ] **Step 3: Upgrade to 22.04**

`[HOST]` after reboot:
```
sudo do-release-upgrade
```

Answer prompts conservatively (keep existing config files when asked). Reboot when done.

- [ ] **Step 4: Verify 22.04 reached**

`[HOST]`:
```
lsb_release -a
```

Expected: `Ubuntu 22.04.X LTS`.

- [ ] **Step 5: Upgrade to 24.04**

`[HOST]`:
```
sudo do-release-upgrade
```

Reboot when done.

- [ ] **Step 6: Verify 24.04 reached**

`[HOST]`:
```
lsb_release -a
```

Expected: `Ubuntu 24.04.X LTS`.

- [ ] **Step 7: No commit**

Host-only change; nothing to commit in the repo.

---

### Task 1.2: Install Docker Engine + Compose plugin

- [ ] **Step 1: Verify Docker is not installed (or is the old Ubuntu version)**

`[HOST]`:
```
docker --version
```

Expected: command not found, OR a version from Ubuntu's repo (e.g., 24.x). Either way, proceed.

- [ ] **Step 2: Remove old Docker packages if present**

`[HOST]`:
```
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

- [ ] **Step 3: Add Docker's official APT repository**

`[HOST]`:
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

- [ ] **Step 4: Install Docker Engine + Compose plugin**

`[HOST]`:
```
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- [ ] **Step 5: Verify**

`[HOST]`:
```
sudo docker run --rm hello-world
docker compose version
```

Expected: `Hello from Docker!` message; Compose version `v2.x` printed.

- [ ] **Step 6: Add current user to docker group**

`[HOST]`:
```
sudo usermod -aG docker $USER
```

Log out and back in (or reboot) for the group change to take effect.

- [ ] **Step 7: Verify rootless docker access**

`[HOST]` after re-login:
```
docker ps
```

Expected: empty container list, no permission error.

---

### Task 1.3: Install NVIDIA driver + Container Toolkit (Mode A only)

> Skip this task if running Mode B (no Plex Pass / no GPU transcoding). See `docs/spec.md` §6.4.

- [ ] **Step 1: Identify the Nvidia GPU**

`[HOST]`:
```
lspci | grep -i nvidia
```

Expected: a line identifying the GPU model. Record it.

- [ ] **Step 2: Install Nvidia proprietary driver**

`[HOST]`:
```
sudo ubuntu-drivers install
```

This auto-selects the recommended driver for the detected GPU. Reboot when done.

- [ ] **Step 3: Verify driver loaded**

`[HOST]` after reboot:
```
nvidia-smi
```

Expected: table showing GPU model, driver version (≥535), CUDA version. If `nvidia-smi` is not found or shows errors, troubleshoot before proceeding.

- [ ] **Step 4: Install NVIDIA Container Toolkit**

`[HOST]`:
```
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

- [ ] **Step 5: Verify Docker can see the GPU**

`[HOST]`:
```
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Expected: same GPU table as Step 3, run from inside a container. If this fails, do NOT proceed to Plex setup.

---

### Task 1.4: Format and mount the 4 TB external HDD

- [ ] **Step 1: Identify the device**

Plug in the external HDD. `[HOST]`:
```
lsblk -f
```

Expected: a new device (commonly `/dev/sda` or `/dev/sdb`) appears with no filesystem or with a pre-existing one. Record the device path. **VERIFY THIS IS THE EXTERNAL DRIVE — formatting the wrong device destroys data.**

- [ ] **Step 2: Partition (optional, full-disk single partition)**

`[HOST]` (replace `/dev/sdX` with the device from Step 1):
```
sudo parted /dev/sdX --script mklabel gpt
sudo parted /dev/sdX --script mkpart primary ext4 0% 100%
```

- [ ] **Step 3: Format ext4**

`[HOST]`:
```
sudo mkfs.ext4 -L media-01 /dev/sdX1
```

- [ ] **Step 4: Create mountpoint and mount**

`[HOST]`:
```
sudo mkdir -p /mnt/media-01
sudo mount /dev/sdX1 /mnt/media-01
```

- [ ] **Step 5: Get the UUID and add to /etc/fstab**

`[HOST]`:
```
sudo blkid /dev/sdX1
```

Copy the `UUID="..."` value. Edit `/etc/fstab`:
```
sudo nano /etc/fstab
```

Append (replace UUID with the value from blkid):
```
UUID=XXXX-XXXX-XXXX-XXXX /mnt/media-01 ext4 defaults,nofail 0 2
```

- [ ] **Step 6: Verify fstab is valid**

`[HOST]`:
```
sudo umount /mnt/media-01
sudo mount -a
df -h /mnt/media-01
```

Expected: drive is mounted at `/mnt/media-01` with ~4 TB available. If `mount -a` errors, fix `/etc/fstab` before continuing (a bad fstab can prevent the system from booting).

- [ ] **Step 7: Create /data symlink**

`[HOST]`:
```
sudo ln -s /mnt/media-01 /data
ls -ld /data
```

Expected: `/data -> /mnt/media-01`.

- [ ] **Step 8: Create the data tree with correct ownership**

`[HOST]`:
```
sudo mkdir -p /mnt/media-01/torrents/{movies,tv,incomplete}
sudo mkdir -p /mnt/media-01/media/{movies,tv}
sudo chown -R 1000:1000 /mnt/media-01
ls -la /data/
```

Expected: `torrents/` and `media/` directories owned by UID/GID 1000.

---

### Task 1.5: Create host directories and project root

- [ ] **Step 1: Create /opt/c-plex and /opt/appdata**

`[HOST]`:
```
sudo mkdir -p /opt/c-plex /opt/appdata
sudo chown -R $USER:$USER /opt/c-plex
sudo chown -R 1000:1000 /opt/appdata
ls -ld /opt/c-plex /opt/appdata
```

Expected: `/opt/c-plex` owned by the operator user; `/opt/appdata` owned by UID/GID 1000.

- [ ] **Step 2: Clone the repo into /opt/c-plex**

`[HOST]`:
```
git clone git@github.com:camiloh12/c-plex.git /opt/c-plex
cd /opt/c-plex
ls
```

Expected: `.gitignore`, `README.md`, `.env.example`, `docs/`, `recyclarr/`, `gluetun/`, `scripts/` present.

- [ ] **Step 3: Copy .env.example to .env and fill in real values**

`[HOST]`:
```
cp .env.example .env
nano .env
```

Fill in real values for: `PUID/PGID/TZ`, `VPN_*`, `WIREGUARD_*`, `PLEX_CLAIM` (get fresh from plex.tv/claim — only needed for Plex first boot), notification URLs. Save and store a copy in your password manager.

- [ ] **Step 4: Verify .env is NOT tracked by git**

`[HOST]`:
```
git status
```

Expected: `.env` not listed (gitignored). Working tree clean otherwise.

---

### Task 1.6: Host hardening (ufw, sshd, unattended-upgrades)

- [ ] **Step 1: Configure unattended-upgrades**

`[HOST]`:
```
sudo apt-get install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Answer "Yes" to the prompt to enable automatic security updates.

Verify:
```
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Expected: contains `APT::Periodic::Unattended-Upgrade "1";`.

- [ ] **Step 2: Configure SSH key-only auth**

`[HOST]` (assuming your SSH key is already authorized on this host — if not, add it to `~/.ssh/authorized_keys` BEFORE this step or you'll lock yourself out):
```
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload ssh
```

Verify (do NOT close your current SSH session yet — open a new one):
```
ssh user@host-ip
```

Expected: new session connects via key. If you can't get in, restore from backup via console: `sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config && sudo systemctl reload ssh`.

- [ ] **Step 3: Configure ufw firewall**

`[HOST]`:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.0/16 to any port 22 proto tcp comment 'SSH from LAN'
sudo ufw allow 32400/tcp comment 'Plex (any)'
# Admin UIs (LAN-only). Adjust 192.168.x.0/24 to your actual LAN subnet.
for port in 8989 7878 9696 8080 5055 6767 8181 3000; do
  sudo ufw allow from 192.168.0.0/16 to any port $port proto tcp comment "Admin UI port $port (LAN)"
done
sudo ufw --force enable
sudo ufw status verbose
```

Expected: status table lists each rule.

> **Note:** `192.168.0.0/16` covers most home subnets (192.168.x.x). If your network uses `10.x.x.x` or `172.16.x.x`, adjust accordingly.

---

## Phase 2 — VPN-isolated download tier

### Task 2.1: Add gluetun service and verify VPN tunnel

**Files:**
- Create: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Confirm `docker-compose.yml` does not exist yet**

`[DEV]`:
```
Test-Path C:\Users\camil\projects\c-plex\docker-compose.yml
```

Expected: `False`.

- [ ] **Step 2: Create docker-compose.yml with gluetun only**

Create `C:\Users\camil\projects\c-plex\docker-compose.yml`:

```yaml
name: c-plex

networks:
  default:
    name: c-plex

services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER}
      - VPN_TYPE=${VPN_TYPE}
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
      - TZ=${TZ}
      # qBittorrent WebUI port published by gluetun (Task 2.2 will use this)
      - FIREWALL_VPN_INPUT_PORTS=
    ports:
      - "8080:8080"   # qBittorrent WebUI (reserved for Task 2.2)
    volumes:
      - ${APPDATA}/gluetun:/gluetun
    healthcheck:
      test: ["CMD", "wget", "-qO-", "https://am.i.mullvad.net/connected"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

- [ ] **Step 3: Commit and push**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add gluetun VPN container as first service in stack"
git push
```

- [ ] **Step 4: Pull and start on host**

`[HOST]`:
```
cd /opt/c-plex
git pull
docker compose up -d gluetun
sleep 30
docker compose ps gluetun
```

Expected: `gluetun` shows `running (healthy)`. If `unhealthy`, check logs: `docker compose logs gluetun --tail 50`.

- [ ] **Step 5: Verify the VPN tunnel is active**

`[HOST]`:
```
docker compose exec gluetun wget -qO- https://am.i.mullvad.net/json
```

Expected: JSON showing `"mullvad_exit_ip": true` (or the equivalent for your provider). The reported `ip` should NOT match your home ISP's public IP.

Compare to your real IP:
```
curl -s https://am.i.mullvad.net/json
```

The two IPs MUST differ.

---

### Task 2.2: Add qBittorrent inside gluetun's network namespace

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append qBittorrent service**

Append to `C:\Users\camil\projects\c-plex\docker-compose.yml` (after the `gluetun` block, indented as a sibling service):

```yaml
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
    volumes:
      - ${APPDATA}/qbittorrent:/config
      - ${DATA}:/data
```

- [ ] **Step 2: Commit and push**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add qBittorrent sharing gluetun network namespace (VPN kill-switch)"
git push
```

- [ ] **Step 3: Pull and start on host**

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d qbittorrent
sleep 15
docker compose ps
```

Expected: both `gluetun` and `qbittorrent` running.

- [ ] **Step 4: Reach qBittorrent WebUI**

`[HOST]`:
```
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

Expected: `200`. From another machine on the LAN, browse `http://HOST-IP:8080`. First-boot username is `admin`; the temp password is printed in container logs:
```
docker compose logs qbittorrent | grep -i password
```

Log in, set a real password via WebUI → Tools → Options → Web UI → Authentication.

- [ ] **Step 5: VERIFY KILL-SWITCH (mandatory)**

`[HOST]`:
```
docker compose stop gluetun
sleep 5
docker compose ps qbittorrent
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

Expected: `qbittorrent` may show as running but its WebUI is unreachable (`000` or connection refused). It has no network because gluetun's namespace is gone.

Restart and verify recovery:
```
docker compose start gluetun
sleep 30
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

Expected: `200`.

- [ ] **Step 6: VERIFY IP-LEAK (mandatory)**

`[HOST]`:
```
docker compose exec qbittorrent curl -s https://am.i.mullvad.net/json
```

Expected: VPN provider's IP, NOT the home ISP's IP. If this returns the home IP, STOP and debug before adding *arr services.

- [ ] **Step 7: Configure qBittorrent paths**

In qBittorrent WebUI:
- Tools → Options → Downloads → Default Save Path: `/data/torrents/`
- Keep incomplete torrents in: `/data/torrents/incomplete`
- Append `.!qB` extension to incomplete files: enabled
- Tools → Options → Web UI → "Bypass authentication for clients on localhost": enabled (so *arr services can use the local API without auth)
- Tools → Options → Web UI → "Bypass authentication for clients in whitelisted IP subnets": enabled, with `172.16.0.0/12` (Docker bridge networks)

- [ ] **Step 8: Add categories for *arr integration**

In qBittorrent WebUI → Categories pane (left), add:
- `tv` with save path `/data/torrents/tv`
- `movies` with save path `/data/torrents/movies`

These category names must match what Sonarr/Radarr send (configured in Tasks 3.2 and 3.3).

---

## Phase 3 — Automation tier

### Task 3.1: Add Prowlarr + FlareSolverr

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append services**

Append to `docker-compose.yml`:

```yaml
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    restart: unless-stopped
    environment:
      - LOG_LEVEL=info
      - TZ=${TZ}
    # No host port published; only Prowlarr inside the bridge reaches this.

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    ports:
      - "9696:9696"
    volumes:
      - ${APPDATA}/prowlarr:/config
    depends_on:
      - flaresolverr
```

- [ ] **Step 2: Commit, push, deploy**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Prowlarr (indexer aggregator) and FlareSolverr"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d flaresolverr prowlarr
sleep 15
docker compose ps flaresolverr prowlarr
```

Expected: both running.

- [ ] **Step 3: Verify Prowlarr UI reachable**

`[HOST]`:
```
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:9696
```

Expected: `200`. Browse from LAN: `http://HOST-IP:9696`. Set authentication on first boot (Forms login).

- [ ] **Step 4: Configure FlareSolverr in Prowlarr**

In Prowlarr UI → Settings → Indexers → Add Indexer Proxy:
- Type: FlareSolverr
- Name: `flaresolverr`
- Host: `http://flaresolverr:8191/`
- Tags: leave empty initially

Save. Verify with the Test button.

- [ ] **Step 5: Add at least one indexer**

In Prowlarr UI → Indexers → Add Indexer:
- Add 2-3 public indexers (e.g., 1337x, RARBG-mirrors, TorrentGalaxy) or your private trackers.
- For any indexer that requires Cloudflare bypass, tag it with the FlareSolverr proxy from Step 4.

Test each indexer. They should respond without errors.

---

### Task 3.2: Add Sonarr and integrate with Prowlarr + qBittorrent

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Sonarr service**

Append to `docker-compose.yml`:

```yaml
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    ports:
      - "8989:8989"
    volumes:
      - ${APPDATA}/sonarr:/config
      - ${DATA}:/data
```

- [ ] **Step 2: Commit, push, deploy**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Sonarr for TV series automation"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d sonarr
sleep 15
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8989
```

Expected: `200`. Browse `http://HOST-IP:8989` and set Forms auth.

- [ ] **Step 3: Configure root folder in Sonarr**

Sonarr UI → Settings → Media Management → Root Folders → Add Root Folder:
- Path: `/data/media/tv`

Save.

- [ ] **Step 4: Add qBittorrent as download client**

Sonarr UI → Settings → Download Clients → Add → qBittorrent:
- Name: `qbittorrent`
- Host: `gluetun` (yes — qBittorrent shares gluetun's network namespace, so it's reachable at the gluetun container name)
- Port: `8080`
- Username/Password: leave blank if you enabled "Bypass authentication for clients in whitelisted IP subnets" with the Docker bridge range (Task 2.2 Step 7)
- Category: `tv` (matches Task 2.2 Step 8)

Test → Save.

- [ ] **Step 5: Connect Sonarr to Prowlarr**

In Prowlarr UI → Settings → Apps → Add → Sonarr:
- Sync Level: `Full Sync`
- Prowlarr Server: `http://prowlarr:9696`
- Sonarr Server: `http://sonarr:8989`
- API Key: copy from Sonarr → Settings → General → API Key

Save. Prowlarr will push all configured indexers to Sonarr. Verify in Sonarr → Settings → Indexers that they appear.

- [ ] **Step 6: Verify end-to-end with a test search**

In Sonarr UI → Add New, search for a TV show with broad torrent availability (e.g., a popular older series). Add it with a quality profile. Watch the Activity → Queue to confirm a release is grabbed and sent to qBittorrent. Then verify in qBittorrent that it's downloading via the VPN.

> Do NOT proceed to Task 3.3 if Sonarr can't grab and hand off to qBittorrent — debug first.

---

### Task 3.3: Add Radarr and integrate

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Radarr service**

Append to `docker-compose.yml`:

```yaml
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    ports:
      - "7878:7878"
    volumes:
      - ${APPDATA}/radarr:/config
      - ${DATA}:/data
```

- [ ] **Step 2: Commit, push, deploy**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Radarr for movie automation"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d radarr
sleep 15
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7878
```

Expected: `200`. Browse `http://HOST-IP:7878` and set Forms auth.

- [ ] **Step 3: Configure root folder**

Radarr UI → Settings → Media Management → Root Folders → Add:
- Path: `/data/media/movies`

- [ ] **Step 4: Add qBittorrent as download client**

Same as Sonarr Step 4, but:
- Name: `qbittorrent`
- Host: `gluetun`
- Port: `8080`
- Category: `movies`

- [ ] **Step 5: Connect Radarr to Prowlarr**

In Prowlarr UI → Settings → Apps → Add → Radarr:
- Prowlarr Server: `http://prowlarr:9696`
- Radarr Server: `http://radarr:7878`
- API Key: copy from Radarr → Settings → General

Save. Verify indexers appear in Radarr → Settings → Indexers.

- [ ] **Step 6: Verify end-to-end with a test movie**

Add a test movie in Radarr. Confirm it's grabbed, downloaded via VPN, imported (hardlinked) to `/data/media/movies/<Name>/`.

- [ ] **Step 7: Verify hardlink (mandatory)**

`[HOST]`:
```
ls -li /data/torrents/movies/<grabbed_dir>/*.mkv /data/media/movies/<imported_dir>/*.mkv
```

Expected: BOTH files show the SAME inode number in the first column. If inode numbers differ, the import copied instead of hardlinked — STOP and debug paths before continuing.

---

### Task 3.4: Add Recyclarr and apply TRaSH-Guide quality profiles

**Files:**
- Create: `C:\Users\camil\projects\c-plex\recyclarr\config.yml`
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Create recyclarr config**

Create `C:\Users\camil\projects\c-plex\recyclarr\config.yml`:

```yaml
# Recyclarr config — syncs TRaSH-Guide quality profiles into Sonarr and Radarr.
# Mode A (with NVENC) tolerates h.265; Mode B (no Plex Pass) must use h.264-only profiles.
# This config defaults to Mode A. For Mode B, swap the radarr/sonarr templates per the
# comments below.

sonarr:
  sonarr-main:
    base_url: http://sonarr:8989
    api_key: !env_var SONARR_API_KEY
    include:
      # WEB-1080p — broad-compat, h.264-preferred profile (works for both Modes)
      - template: sonarr-v4-quality-profile-web-1080p
      - template: sonarr-v4-custom-formats-web-1080p
    quality_profiles:
      - name: WEB-1080p
        upgrade:
          allowed: true
          until_quality: WEB-1080p
          until_score: 10000
        min_format_score: 0
        quality_sort: top
        score_set: default

radarr:
  radarr-main:
    base_url: http://radarr:7878
    api_key: !env_var RADARR_API_KEY
    include:
      # Mode A: HD Bluray + WEB (allows h.265 in WEB).
      # Mode B: replace these two lines with: hd-bluray-web (h.264-only equivalent).
      - template: radarr-quality-profile-hd-bluray-web
      - template: radarr-custom-formats-hd-bluray-web
    quality_profiles:
      - name: HD Bluray + WEB
        upgrade:
          allowed: true
          until_quality: Bluray-1080p
          until_score: 10000
        min_format_score: 0
        quality_sort: top
        score_set: default
```

- [ ] **Step 2: Append Recyclarr to docker-compose.yml**

Append to `docker-compose.yml`:

```yaml
  recyclarr:
    image: ghcr.io/recyclarr/recyclarr:latest
    container_name: recyclarr
    user: "${PUID}:${PGID}"
    environment:
      - TZ=${TZ}
      # API keys are loaded from .env so we don't put them in config.yml.
      - SONARR_API_KEY=${SONARR_API_KEY}
      - RADARR_API_KEY=${RADARR_API_KEY}
    volumes:
      - ${APPDATA}/recyclarr:/config
      - ./recyclarr/config.yml:/config/recyclarr.yml:ro
    restart: "no"
    # Run on-demand: `docker compose run --rm recyclarr sync`
    # Or set a cron via the host (see Step 5 below).
    entrypoint: ["recyclarr", "sync"]
    depends_on:
      - sonarr
      - radarr
```

- [ ] **Step 3: Add API keys to .env**

`[HOST]`:
```
# Get Sonarr API key:
docker compose exec sonarr cat /config/config.xml | grep -i apikey
# Get Radarr API key:
docker compose exec radarr cat /config/config.xml | grep -i apikey
```

Append to `/opt/c-plex/.env`:
```
SONARR_API_KEY=<value-from-sonarr>
RADARR_API_KEY=<value-from-radarr>
```

Also add the variable names to `.env.example` (no real values):

`[DEV]` edit `.env.example`, append:
```
# *arr API keys (obtained after first boot of each service)
SONARR_API_KEY=REPLACE_ME
RADARR_API_KEY=REPLACE_ME
```

- [ ] **Step 4: Commit and push**

`[DEV]`:
```
git add recyclarr/config.yml docker-compose.yml .env.example
git commit -m "Add Recyclarr for TRaSH-Guide profile sync"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
```

- [ ] **Step 5: Run Recyclarr once manually**

`[HOST]`:
```
docker compose run --rm recyclarr sync
```

Expected: output shows quality profiles and custom formats being added/updated in Sonarr and Radarr. No errors.

Verify in Sonarr → Settings → Profiles that `WEB-1080p` exists with TRaSH scores populated. Same for Radarr → Profiles `HD Bluray + WEB`.

- [ ] **Step 6: Schedule daily Recyclarr sync via host cron**

`[HOST]`:
```
sudo crontab -e
```

Append:
```
0 3 * * * cd /opt/c-plex && /usr/bin/docker compose run --rm recyclarr sync >> /var/log/recyclarr.log 2>&1
```

Verify cron entry: `sudo crontab -l`.

---

### Task 3.5: Add Bazarr

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Bazarr service**

Append to `docker-compose.yml`:

```yaml
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    ports:
      - "6767:6767"
    volumes:
      - ${APPDATA}/bazarr:/config
      - ${DATA}:/data
```

- [ ] **Step 2: Commit, push, deploy**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Bazarr for subtitle automation"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d bazarr
sleep 15
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:6767
```

Expected: `200`. Browse `http://HOST-IP:6767`.

- [ ] **Step 3: Connect Bazarr to Sonarr and Radarr**

Bazarr UI → Settings → Sonarr:
- Host: `sonarr`, Port: `8989`, API Key from Sonarr.
- Test → Save.

Same for Radarr.

- [ ] **Step 4: Configure subtitle providers**

Settings → Providers → enable at least two (e.g., OpenSubtitles.com — requires free account, Subscene, etc.). Add credentials where required.

Settings → Languages → add desired subtitle languages and the profile each library should use.

---

## Phase 4 — Media serving tier

### Task 4.1: Add Plex (Mode A — NVENC) and claim server

> If running Mode B (no Plex Pass), use the alternate service block at the bottom of this task.

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Plex service (Mode A — with Nvidia GPU)**

Append to `docker-compose.yml`:

```yaml
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    restart: unless-stopped
    network_mode: host
    runtime: nvidia
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - VERSION=docker
      - PLEX_CLAIM=${PLEX_CLAIM}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
    volumes:
      - ${APPDATA}/plex:/config
      - ${DATA}/media:/data/media:ro
      - /tmp/plex-transcode:/transcode
```

> **Mode B alternative (no Plex Pass, no GPU passthrough):** remove `runtime: nvidia` and the two `NVIDIA_*` env vars from the block above. Skip Step 7 of this task.

- [ ] **Step 2: Set up the tmpfs transcode directory**

`[HOST]`:
```
sudo mkdir -p /tmp/plex-transcode
sudo chown 1000:1000 /tmp/plex-transcode
# Make it a tmpfs (in-memory) to spare the HDD. Add to /etc/fstab for persistence:
echo "tmpfs /tmp/plex-transcode tmpfs defaults,size=4G,uid=1000,gid=1000 0 0" | sudo tee -a /etc/fstab
sudo mount /tmp/plex-transcode
df -h /tmp/plex-transcode
```

Expected: `/tmp/plex-transcode` mounted as tmpfs, 4G size.

- [ ] **Step 3: Get fresh Plex claim token**

Open https://plex.tv/claim in a browser (you must be logged in to your Plex account). Copy the token (valid 4 minutes).

`[HOST]` update `.env`:
```
nano /opt/c-plex/.env
# Set PLEX_CLAIM=claim-XXXXXXX
```

- [ ] **Step 4: Commit and push**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Plex with Nvidia NVENC hardware transcoding (Mode A)"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d plex
sleep 30
docker compose logs plex --tail 30
```

Expected: logs show Plex booting and binding to the claim token. Service running.

- [ ] **Step 5: Verify Plex UI reachable**

From a LAN browser: `http://HOST-IP:32400/web`.

Expected: Plex setup wizard appears (or jumps straight to library view if claim already linked). Complete setup:
- Server name: `c-plex` (or whatever you prefer)
- Add libraries: Movies → `/data/media/movies`, TV Shows → `/data/media/tv`

- [ ] **Step 6: Enable remote access**

Plex Settings → Remote Access → enable. Plex will attempt UPnP first; if your router doesn't support it, manually port-forward TCP 32400 to the host. Status should turn green ("fully accessible").

- [ ] **Step 7: Enable HW transcoding (Mode A only)**

Plex Settings → Transcoder:
- Hardware-accelerated video encoding: enabled
- Use hardware acceleration when available: enabled
- Transcoder temporary directory: `/transcode`

Force a test transcode: in a Plex client, play a high-bitrate file and lower the quality below the source's native bitrate. While playing, on the HOST:
```
nvidia-smi
```

Expected: processes table shows the Plex Transcoder using the GPU. If only CPU is being used, debug `runtime: nvidia` and the env vars.

---

### Task 4.2: Add Tautulli

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Tautulli service**

Append to `docker-compose.yml`:

```yaml
  tautulli:
    image: lscr.io/linuxserver/tautulli:latest
    container_name: tautulli
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    ports:
      - "8181:8181"
    volumes:
      - ${APPDATA}/tautulli:/config
```

- [ ] **Step 2: Commit, push, deploy, verify**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Tautulli for Plex stats"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d tautulli
sleep 15
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8181
```

Expected: `200`. Browse `http://HOST-IP:8181`.

- [ ] **Step 3: Connect Tautulli to Plex**

First-boot wizard asks for Plex token. Get one from Plex Web → click any item → three-dot menu → Get Info → View XML → in URL, `X-Plex-Token=...`.

Set Plex server: `http://HOST-IP:32400`, token from above. Save.

---

### Task 4.3: Add Seerr

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Seerr service**

Append to `docker-compose.yml`:

```yaml
  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    container_name: seerr
    restart: unless-stopped
    environment:
      - LOG_LEVEL=info
      - TZ=${TZ}
      - PORT=5055
    ports:
      - "5055:5055"
    volumes:
      - ${APPDATA}/seerr:/app/config
    depends_on:
      - plex
      - sonarr
      - radarr
```

- [ ] **Step 2: Commit, push, deploy, verify**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Seerr for user-facing media requests"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d seerr
sleep 20
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5055
```

Expected: `200`. Browse `http://HOST-IP:5055`.

- [ ] **Step 3: Run Seerr setup wizard**

In Seerr UI:
- Sign in with Plex
- Add Plex server: `http://HOST-IP:32400`
- Add Sonarr: `http://sonarr:8989`, API key, default quality profile `WEB-1080p`, root folder `/data/media/tv`
- Add Radarr: `http://radarr:7878`, API key, default quality profile `HD Bluray + WEB`, root folder `/data/media/movies`

Sync libraries.

- [ ] **Step 4: Verify end-to-end request flow**

In Seerr, request a small test movie or TV episode. Confirm:
1. Seerr request marked "Pending" → "Approved" (auto-approve by default for the admin).
2. Radarr/Sonarr picks it up; release grabbed in queue.
3. qBittorrent downloads.
4. Imported (hardlinked) to `/data/media/...`.
5. Plex library scan picks it up.
6. Seerr marks request "Available".

If any step fails, debug before moving to Phase 5.

---

## Phase 5 — Operations

### Task 5.1: Add Watchtower for weekly auto-updates

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Watchtower service**

Append to `docker-compose.yml`:

```yaml
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    environment:
      - TZ=${TZ}
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_SCHEDULE=${WATCHTOWER_SCHEDULE}
      - WATCHTOWER_NOTIFICATIONS=shoutrrr
      - WATCHTOWER_NOTIFICATION_URL=${NOTIFY_DISCORD_WEBHOOK}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

> If using ntfy instead of Discord, replace the `WATCHTOWER_NOTIFICATION_URL` reference with a shoutrrr-format ntfy URL. See https://containrrr.dev/shoutrrr/.

- [ ] **Step 2: Commit, push, deploy**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Watchtower for weekly image auto-updates"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d watchtower
docker compose logs watchtower --tail 20
```

Expected: logs show "Scheduling first run at..." with the next Sunday 04:00.

---

### Task 5.2: Add Homepage dashboard (optional)

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\docker-compose.yml`

- [ ] **Step 1: Append Homepage service**

Append to `docker-compose.yml`:

```yaml
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    ports:
      - "3000:3000"
    volumes:
      - ${APPDATA}/homepage:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

- [ ] **Step 2: Commit, push, deploy**

`[DEV]`:
```
git add docker-compose.yml
git commit -m "Add Homepage dashboard"
git push
```

`[HOST]`:
```
cd /opt/c-plex && git pull
docker compose up -d homepage
sleep 10
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000
```

Expected: `200`. Browse `http://HOST-IP:3000`.

- [ ] **Step 3: Configure services in Homepage**

Edit `/opt/appdata/homepage/services.yaml` to add bookmarks for each UI:

```yaml
- Media:
    - Plex:
        href: http://HOST-IP:32400/web
        icon: plex.png
    - Seerr:
        href: http://HOST-IP:5055
        icon: seerr.png
- Admin:
    - Sonarr:
        href: http://HOST-IP:8989
        icon: sonarr.png
    - Radarr:
        href: http://HOST-IP:7878
        icon: radarr.png
    - Prowlarr:
        href: http://HOST-IP:9696
        icon: prowlarr.png
    - qBittorrent:
        href: http://HOST-IP:8080
        icon: qbittorrent.png
    - Bazarr:
        href: http://HOST-IP:6767
        icon: bazarr.png
    - Tautulli:
        href: http://HOST-IP:8181
        icon: tautulli.png
```

Reload Homepage UI to verify.

---

### Task 5.3: Nightly /opt/appdata backup

**Files:**
- Create: `C:\Users\camil\projects\c-plex\scripts\backup-appdata.sh`

- [ ] **Step 1: Create backup script**

Create `C:\Users\camil\projects\c-plex\scripts\backup-appdata.sh`:

```bash
#!/usr/bin/env bash
# Nightly backup of /opt/appdata to a tarball on a separate location.
# Retention: 7 daily + 4 weekly.
#
# Usage: BACKUP_DIR=/path/to/backups ./backup-appdata.sh
# Schedule via root cron (see Task 5.3 Step 5).
#
# BACKUP_DIR MUST point to storage separate from the media drive.
# Per spec §8.1, putting backups on the same drive as the library defeats
# the purpose. Use a second internal drive, a USB stick, or a cloud-mounted
# rclone target.

set -euo pipefail

if [ -z "${BACKUP_DIR:-}" ]; then
  echo "ERROR: BACKUP_DIR env var is required (no safe default — must NOT be the media drive)." >&2
  exit 1
fi

SOURCE="/opt/appdata"
DATE="$(date +%Y-%m-%d)"
WEEK="$(date +%Y-W%V)"
DAILY="$BACKUP_DIR/daily-$DATE.tar.gz"
WEEKLY="$BACKUP_DIR/weekly-$WEEK.tar.gz"

mkdir -p "$BACKUP_DIR"

# Stop containers so DB files are consistent during tar
cd /opt/c-plex
docker compose stop

tar -czf "$DAILY" -C "$(dirname "$SOURCE")" "$(basename "$SOURCE")"

# On Sundays, also create the weekly snapshot (hardlink, no extra space)
if [ "$(date +%u)" = "7" ]; then
  cp -l "$DAILY" "$WEEKLY"
fi

docker compose start

# Retention: keep last 7 daily and last 4 weekly
find "$BACKUP_DIR" -maxdepth 1 -name 'daily-*.tar.gz' -type f | sort | head -n -7 | xargs -r rm
find "$BACKUP_DIR" -maxdepth 1 -name 'weekly-*.tar.gz' -type f | sort | head -n -4 | xargs -r rm

echo "Backup complete: $DAILY"
```

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add scripts/backup-appdata.sh
git commit -m "Add nightly /opt/appdata backup script"
git push
```

- [ ] **Step 3: Pull and make executable**

`[HOST]`:
```
cd /opt/c-plex && git pull
chmod +x scripts/backup-appdata.sh
```

- [ ] **Step 4: Choose a backup target separate from the media drive**

Per spec §8.1, the backup must NOT live on the same physical drive as the
media library. Options:
- A second internal HDD/SSD (e.g., `/mnt/backup-01`).
- A small USB stick dedicated to backups (configs are <1 GB tarballs).
- A cloud bucket mounted via rclone (e.g., Backblaze B2, Storj, S3).

Create the chosen directory and confirm ownership:
```
sudo mkdir -p /mnt/backup-01           # adjust path to your chosen target
sudo chown $USER:$USER /mnt/backup-01
```

- [ ] **Step 5: Test the script manually**

`[HOST]`:
```
sudo BACKUP_DIR=/mnt/backup-01 ./scripts/backup-appdata.sh
ls -lh /mnt/backup-01/
```

Expected: `daily-YYYY-MM-DD.tar.gz` exists. Containers were stopped and restarted during the run.

If you skipped Step 4 and try to run with no `BACKUP_DIR`, the script
intentionally fails with an error rather than picking an unsafe default.

- [ ] **Step 6: Schedule via root cron**

`[HOST]`:
```
sudo crontab -e
```

Append (substitute your `BACKUP_DIR`):
```
30 3 * * * BACKUP_DIR=/mnt/backup-01 /opt/c-plex/scripts/backup-appdata.sh >> /var/log/c-plex-backup.log 2>&1
```

(3:30 AM — after the Recyclarr sync at 3:00 AM.) Verify: `sudo crontab -l`.

---

## Phase 6 — Documentation

### Task 6.1: Write post-deploy verification runbook

**Files:**
- Create: `C:\Users\camil\projects\c-plex\docs\runbooks\post-deploy-verification.md`

- [ ] **Step 1: Create runbook**

Create `C:\Users\camil\projects\c-plex\docs\runbooks\post-deploy-verification.md`:

```markdown
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
```

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add docs/runbooks/post-deploy-verification.md
git commit -m "Add post-deploy verification runbook"
git push
```

---

### Task 6.2: Write add-user runbook

**Files:**
- Create: `C:\Users\camil\projects\c-plex\docs\runbooks\add-user.md`

- [ ] **Step 1: Create runbook**

Create `C:\Users\camil\projects\c-plex\docs\runbooks\add-user.md`:

```markdown
# Add a User to Plex and Seerr

## 1. Plex (library access)

1. In Plex Web → Settings → Users & Sharing → Library Access → Invite Friend.
2. Enter the new user's email or Plex username.
3. Select which libraries to share (typically both Movies and TV).
4. (Optional) Restrict ratings / labels for younger users.
5. Save. The user accepts the invite from their own Plex account.

## 2. Seerr (request access)

Seerr auto-imports Plex Home / Friends as users after they first log in.
No manual step is required for read/request access.

To grant additional permissions (auto-approve, admin):
1. Have the user log into Seerr once via Plex SSO.
2. Seerr admin → Users → click the new user → Edit Permissions.
3. Grant desired permissions; Save.
```

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add docs/runbooks/add-user.md
git commit -m "Add user-onboarding runbook"
git push
```

---

### Task 6.3: Write troubleshooting runbook

**Files:**
- Create: `C:\Users\camil\projects\c-plex\docs\runbooks\troubleshooting.md`

- [ ] **Step 1: Create runbook**

Create `C:\Users\camil\projects\c-plex\docs\runbooks\troubleshooting.md`:

````markdown
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
````

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add docs/runbooks/troubleshooting.md
git commit -m "Add troubleshooting runbook"
git push
```

---

### Task 6.4: Write VPN rotation runbook

**Files:**
- Create: `C:\Users\camil\projects\c-plex\docs\runbooks\rotate-vpn-provider.md`

- [ ] **Step 1: Create runbook**

Create `C:\Users\camil\projects\c-plex\docs\runbooks\rotate-vpn-provider.md`:

```markdown
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
```

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add docs/runbooks/rotate-vpn-provider.md
git commit -m "Add VPN provider rotation runbook"
git push
```

---

### Task 6.5: Write Recyclarr resync runbook

**Files:**
- Create: `C:\Users\camil\projects\c-plex\docs\runbooks\recyclarr-resync.md`

- [ ] **Step 1: Create runbook**

Create `C:\Users\camil\projects\c-plex\docs\runbooks\recyclarr-resync.md`:

```markdown
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
```

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add docs/runbooks/recyclarr-resync.md
git commit -m "Add Recyclarr resync runbook"
git push
```

---

### Task 6.6: Write upgrade runbook

**Files:**
- Create: `C:\Users\camil\projects\c-plex\docs\runbooks\upgrades.md`

- [ ] **Step 1: Create runbook**

Create `C:\Users\camil\projects\c-plex\docs\runbooks\upgrades.md`:

````markdown
# Upgrades

## Container images

Automated by Watchtower (Sunday 04:00 local). No manual action required.

To upgrade manually before the scheduled run:
```
cd /opt/c-plex
docker compose pull
docker compose up -d
```

To pin a service to a specific version (e.g., to roll back):
1. Edit `docker-compose.yml`, change `image: lscr.io/linuxserver/plex:latest`
   to `image: lscr.io/linuxserver/plex:1.40.x`.
2. Commit, push, pull on host.
3. `docker compose up -d <service>`.

## Ubuntu OS

Security patches: automatic via `unattended-upgrades`.
Major version (24.04 → 26.04): manual when next LTS releases.

```
sudo apt update && sudo apt full-upgrade -y
# Wait for next LTS, then:
sudo do-release-upgrade
```

Run from console (or `screen`/`tmux` over SSH) — connection may drop.

## NVIDIA driver

Only upgrade when a Plex update requires it.

```
sudo apt update && sudo apt install -y nvidia-driver-XXX-server
sudo reboot
nvidia-smi   # verify new version
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

If the in-container `nvidia-smi` fails after driver upgrade, reinstall the
NVIDIA Container Toolkit and restart Docker:
```
sudo apt install --reinstall -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker compose up -d plex
```

## Docker

Automatic security patches via APT.

To upgrade manually:
```
sudo apt update && sudo apt install --only-upgrade -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl restart docker
docker compose up -d
```
````

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add docs/runbooks/upgrades.md
git commit -m "Add upgrade runbook"
git push
```

---

### Task 6.7: Update README with link to runbooks

**Files:**
- Modify: `C:\Users\camil\projects\c-plex\README.md`

- [ ] **Step 1: Edit README to reference runbooks**

Replace the existing README.md content with:

```markdown
# c-plex

Self-hosted Plex media streaming platform for a single household.

## Documentation

- **Spec:** `docs/spec.md` — full project specification (living document).
- **Plans:** `docs/plans/` — implementation plans (point-in-time).
- **Runbooks:** `docs/runbooks/` — operational procedures.

## Quick deploy (on the Ubuntu host)

```bash
# Prerequisites: Ubuntu 24.04 LTS, Docker, NVIDIA Container Toolkit (Mode A only)
# See docs/spec.md §5.1 for full prerequisites.

sudo mkdir -p /opt/c-plex && sudo chown $USER:$USER /opt/c-plex
git clone git@github.com:camiloh12/c-plex.git /opt/c-plex
cd /opt/c-plex
cp .env.example .env       # then edit .env with real secrets
docker compose up -d
```

After deploy, work through `docs/runbooks/post-deploy-verification.md`.

## Common operations

- Add a user → `docs/runbooks/add-user.md`
- Troubleshoot a failure → `docs/runbooks/troubleshooting.md`
- Rotate VPN provider → `docs/runbooks/rotate-vpn-provider.md`
- Re-sync quality profiles → `docs/runbooks/recyclarr-resync.md`
- Upgrade Ubuntu / Docker / NVIDIA → `docs/runbooks/upgrades.md`
```

- [ ] **Step 2: Commit, push**

`[DEV]`:
```
git add README.md
git commit -m "Update README to reference runbooks and link spec/plans"
git push
```

---

## Final verification

- [ ] **Run the full post-deploy verification checklist**

Open `docs/runbooks/post-deploy-verification.md` and tick every box.
Every check must pass. If any fails, debug before declaring the deploy done.

- [ ] **Resolve the open questions in spec §11 that have been decided**

Edit `docs/spec.md` to remove entries from §11 that have been resolved during
this deployment, and bake the decisions into the relevant sections.
Bump "Last updated" at the top. Commit and push.

---

## Notes for the implementer

- **Order matters within phases.** Don't skip ahead — later tasks depend on
  earlier ones (e.g., Sonarr's qBittorrent integration depends on the
  kill-switch being verified).
- **Don't combine compose-file edits across tasks** in one commit. Each
  service addition is its own commit so a bad change is easy to revert.
- **Mode A vs Mode B** is decided once, in Task 4.1. The rest of the plan
  works identically.
- **All `[HOST]` commands assume you're in `/opt/c-plex/`** unless stated.
- **`docker compose` (no hyphen) is the modern syntax.** If you see
  `docker-compose: command not found`, the Compose plugin isn't installed
  (Task 1.2 Step 4).
