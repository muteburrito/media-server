# Jellyfin Media Server (Podman / Docker)

Self-hosted [Jellyfin](https://jellyfin.org/) media server running in a container on Fedora Server using Podman (or Docker). Jellyfin is a free and open-source alternative to Plex with no account or subscription required.

## Prerequisites

- Fedora Server (or any Linux distro with Podman/Docker)
- Podman installed (`sudo dnf install podman`)
- `podman-compose` (optional): `pip install podman-compose`

## Directory Structure

Create the required directories on the host:

```bash
mkdir -p ~/jellyfin/config ~/jellyfin/cache ~/jellyfin/media/movies ~/jellyfin/media/tv
```

```
~/jellyfin/
├── config/        # Jellyfin configuration and database
├── cache/         # Transcoding and image cache
└── media/
    ├── movies/    # Movie files
    └── tv/        # TV show files
```

## Configuration

The [docker-compose.yml](docker-compose.yml) uses the [official Jellyfin image](https://jellyfin.org/docs/general/installation/container/).

| Variable | Description |
|----------|-------------|
| `PUID` / `PGID` | User/group ID for file permissions (check with `id` command) |
| `TZ` | Timezone (e.g. `Asia/Kolkata`) |

Volumes use the `:Z` SELinux relabel flag, which is required on SELinux-enabled distros like Fedora.

## SELinux Setup (Fedora / RHEL)

On SELinux-enabled systems, containers cannot access host directories unless they have the correct SELinux context. **This must be done before starting the container.**

```bash
# 1. Set ownership to match PUID/PGID
sudo chown -R 1000:1000 /home/chinmay/jellyfin

# 2. Label the volumes so containers can access them
sudo chcon -Rt container_file_t /home/chinmay/jellyfin/config
sudo chcon -Rt container_file_t /home/chinmay/jellyfin/cache
sudo chcon -Rt container_file_t /home/chinmay/jellyfin/media

# 3. Verify the labels
ls -laZ /home/chinmay/jellyfin/
# Should show container_file_t on config/, cache/, and media/
```

The `:Z` flag on volume mounts in the compose file further relabels the directories to be private to this specific container. Both steps (manual `chcon` + `:Z` flag) are needed for reliable operation.

## Running with Podman

> **Important:** Use `sudo` (rootful Podman) to avoid UID remapping issues. Rootless Podman remaps UIDs, which causes permission denied errors even with correct SELinux labels.

### Option 1: Direct `podman run` (recommended)

```bash
sudo podman run -d \
  --name jellyfin \
  --network host \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Asia/Kolkata \
  -v /home/chinmay/jellyfin/config:/config:Z \
  -v /home/chinmay/jellyfin/cache:/cache:Z \
  -v /home/chinmay/jellyfin/media:/media:Z \
  --restart unless-stopped \
  jellyfin/jellyfin:latest
```

### Option 2: `podman-compose`

```bash
# Install system-wide (required for sudo access)
sudo pip install podman-compose

# Start the container
sudo podman-compose up -d
```

> **Note:** If `podman-compose` was installed with `pip install --user`, it will only be in `~/.local/bin/` and won't be available under `sudo`. Install system-wide with `sudo pip install podman-compose` or use the full path: `sudo $(which podman-compose) up -d`.

### Manage the container

```bash
# View logs
sudo podman logs -f jellyfin

# Stop
sudo podman stop jellyfin

# Remove
sudo podman rm jellyfin

# Check status
sudo podman ps
```

## Running with Docker

```bash
docker compose up -d
```

## Firewall (Fedora)

Port 8096 must be open for Jellyfin to be accessible on the network:

```bash
sudo firewall-cmd --add-port=8096/tcp --permanent
sudo firewall-cmd --reload
```

Verify:
```bash
sudo firewall-cmd --list-ports
# Should include: 8096/tcp
```

## Access

Once running, open Jellyfin in a browser:

```
http://<server-ip>:8096
```

The initial setup wizard will guide you through creating an admin account and adding media libraries.

## Troubleshooting

### Permission denied errors in container logs

**Symptoms:** `Permission denied` errors when the container tries to write to `/config` or `/cache`.

**Root cause:** Two issues compound on Fedora:
1. **SELinux** blocks container access to host directories that don't have `container_file_t` labels
2. **Rootless Podman** remaps UIDs (container UID 1000 maps to a different host UID)

**Fix:**
```bash
# Set correct SELinux labels
sudo chcon -Rt container_file_t /home/chinmay/jellyfin/config
sudo chcon -Rt container_file_t /home/chinmay/jellyfin/cache
sudo chcon -Rt container_file_t /home/chinmay/jellyfin/media

# Fix ownership
sudo chown -R 1000:1000 /home/chinmay/jellyfin

# Run rootful
sudo podman-compose up -d
```

### `podman-compose` not found under `sudo`

`pip install --user` installs to `~/.local/bin`, which isn't in root's PATH.

**Fix:** Either install system-wide (`sudo pip install podman-compose`) or use `sudo podman run` directly.

### Container running but port not reachable

Check that:
1. The firewall allows port 8096: `sudo firewall-cmd --list-ports`
2. Jellyfin is actually listening: `ss -tlnp | grep 8096`
3. Container logs show no errors: `sudo podman logs jellyfin`
