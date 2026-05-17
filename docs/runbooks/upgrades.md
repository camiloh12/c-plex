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
