# quadlet-beszel

Quadlet setup for [Beszel](https://github.com/henrygd/beszel) — a lightweight server monitoring hub with historical data, Docker stats, and alerts (`docker.io/henrygd/beszel`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `beszel.container` | Quadlet unit file |
| `beszel.env` | Default environment variables |
| `beszel.override.env.template` | Template for local overrides (APP_URL) |
| `beszel-backup.service` | Systemd service: rsync data directory to backup location |
| `beszel-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/beszel -s /usr/sbin/nologin beszel

REPO_URL=https://github.com/mkoester/quadlet-beszel.git
REPO=~beszel/quadlet-beszel
```

```sh
# 2. Enable linger
sudo loginctl enable-linger beszel

# 3. Clone this repo into the service user's home
sudo -u beszel git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u beszel mkdir -p ~beszel/.config/containers/systemd
sudo -u beszel mkdir -p ~beszel/data

# 5. Create .override.env from template and fill in required values
sudo -u beszel cp $REPO/beszel.override.env.template $REPO/beszel.override.env
sudo -u beszel nano $REPO/beszel.override.env

# 6. Symlink all quadlet files from the repo
sudo -u beszel ln -s $REPO/beszel.container ~beszel/.config/containers/systemd/beszel.container
sudo -u beszel ln -s $REPO/beszel.env ~beszel/.config/containers/systemd/beszel.env
sudo -u beszel ln -s $REPO/beszel.override.env ~beszel/.config/containers/systemd/beszel.override.env

# 7. Reload and start
sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user daemon-reload
sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user start beszel

# 8. Verify
sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user status beszel
```

## Configuration

### Environment variables

`beszel.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |

`beszel.override.env` (created from template) must set:

| Variable | Description |
|---|---|
| `APP_URL` | Full HTTPS URL where Beszel is accessible (e.g. `https://beszel.example.com`) |

To apply changes after editing the override file:

```sh
sudo -u beszel nano ~beszel/quadlet-beszel/beszel.override.env
sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user restart beszel
```

## Reverse proxy (Caddy)

Add a site block to your Caddyfile:

```
beszel.example.com {
    reverse_proxy localhost:8090
}
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## Backup

Beszel stores all data (PocketBase database and configuration) in `/beszel_data` inside the container (`~beszel/data/` on the host). See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup.

The backup service uses `rsync` to copy the entire data directory to the backup location.

```sh
# 1. Create backup staging directory (owned by beszel, readable by backup-readers group)
sudo mkdir -p /var/backups/beszel
sudo chown beszel:backup-readers /var/backups/beszel
sudo chmod 750 /var/backups/beszel

# 2. Symlink the backup service and timer from the repo
sudo -u beszel mkdir -p ~beszel/.config/systemd/user
sudo -u beszel ln -s $REPO/beszel-backup.service ~beszel/.config/systemd/user/beszel-backup.service
sudo -u beszel ln -s $REPO/beszel-backup.timer ~beszel/.config/systemd/user/beszel-backup.timer

# 3. Enable and start the timer
sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user daemon-reload
sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user enable --now beszel-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@beszel-host:/var/backups/beszel/ /path/to/local/backup/beszel/
```

## Notes

- Port `8090` is bound to `127.0.0.1` only — Caddy handles TLS termination.
- All persistent data is stored at `~beszel/data/` on the host.
- `APP_URL` must be set to the HTTPS URL before first start.
- After first start, open `APP_URL` in a browser to create the initial admin account.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning)). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u beszel XDG_RUNTIME_DIR=/run/user/$(id -u beszel) systemctl --user enable --now podman-image-prune@30.timer
  ```
