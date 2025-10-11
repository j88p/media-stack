# Media Stack Helm Chart

This Helm chart deploys a full media stack (Plex, Sonarr, Radarr, qBittorrent, Overseerr, Prowlarr) with Gluetun VPN sidecars (except Plex) on Kubernetes. It supports hostPath volumes and NGINX Ingress for all apps (no TLS by default). Plex is exposed via ClusterIP + Ingress by default (no hostNetwork/NodePort required).

## Prerequisites

### Create the PIA VPN secret (required for Gluetun)

```sh
kubectl create secret generic pia-credentials \
   --from-literal=username='<your_pia_username>' \
   --from-literal=password='<your_pia_password>' \
   -n <namespace>
```

Replace `<your_pia_username>`, `<your_pia_password>`, and `<namespace>` with your details.

### Required Host Paths & Directories

Create these directories on your Kubernetes nodes before deploying:

- `/opt/docker_volumes/plex/config`
- `/opt/docker_volumes/plex/transcode`
- `/opt/docker_volumes/overseerr`
- `/opt/docker_volumes/prowlarr`
- `/opt/docker_volumes/qbittorrent/config`
- `/mnt/plexpool/movies`
- `/mnt/plexpool/tvshows`
- `/mnt/plexpool/barnfilmer`
- `/mnt/plexpool/barnserier`
- `/mnt/plexpool/newDownloads`

Adjust paths in `values.yaml` if you use different locations. Ensure directories are writable by the expected UID/GID (typically 1000:1000), or run Plex as root as configured below.

### Using fsmerge for plexpool

To combine multiples disks into a single mount point for Plex, use fsmerge or similar drive pooling tool

1. Install fsmerge:

   ```sh
   pip install fsmerge
   ```

2. Create a config file (e.g., `/etc/fsmerge/plexpool.conf`) listing your source directories.
3. Run fsmerge:

   ```sh
   fsmerge /etc/fsmerge/plexpool.conf /mnt/plexpool
   ```

4. Add to systemd or your startup scripts for persistence.

### Mounting disks by UUID in /etc/fstab

List your disks and get their UUIDs:

```sh
lsblk -o NAME,UUID,MOUNTPOINT
blkid
```

Edit `/etc/fstab` to mount by UUID:

```
UUID=<your-uuid> /mnt/plexpool/movies ext4 defaults 0 2
UUID=<your-uuid> /mnt/plexpool/tvshows ext4 defaults 0 2
...etc...
```

Create mount points and mount:

```sh
sudo mkdir -p /mnt/plexpool/movies
sudo mount -a
```

Repeat for each disk. Ensure permissions allow your containers to read/write.

**Note:** Don't forget to format new disks with ext4 before mounting. Example:

```sh
sudo mkfs.ext4 /dev/sdX
```

Replace `/dev/sdX` with your disk device name.

## Quick Start

1. **Clone the repository:**

   ```sh
   git clone https://github.com/j88p/media-stack.git
   cd media-stack/helm/media-stack-v1.0
   ```

2. **Edit `values.yaml`:**
   - Set your media/config paths under `plex`, `sonarr`, `radarr`, etc.
   - Set VPN credentials and region for Gluetun sidecars.
   - Set Ingress hostnames for each app (e.g., `son.lan`, `rad.lan`, `qbit.lan`, `plex.lan`).
   - Optionally add entries in your `/etc/hosts` pointing those names to your Ingress controller IP (e.g., `192.168.1.128`).

3. **Install the chart:**

   ```sh
   helm install media-stack .
   ```

   Or upgrade:

   ```sh
   helm upgrade --install media-stack .
   ```

4. **Access your apps:**
   - Plex: via Ingress (`http://plex.lan`) routing to Service `plex-service` on port 32400
   - Other apps: via Ingress (`http://son.lan`, `http://rad.lan`, `http://qbit.lan`, `http://pro.lan`, `http://ovr.lan`)

## Customization

- **Volumes:** Edit `values.yaml` to match your hostPath/NFS setup.
- **Ingress:** Hostnames are set in `values.yaml`. TLS/cert-manager can be added later. Ingress class defaults to `nginx` and is configurable via `global.ingressClass`.
- **Services:** Service names use the `-service` suffix (e.g., `sonarr-service`). Each app exposes a ClusterIP Service by default. An optional `servicePort` may be set per app to expose a different Service port while the container keeps its native port.
- **Plex networking:** Plex is ClusterIP + Ingress by default. No `hostNetwork` or `NodePort` is required. Probes and Service target port 32400.
- **Environment Variables:** Managed via ConfigMaps for easy updates. Plex can be configured to run root (PUID/PGID 0) if needed for plugin initialization; otherwise make sure `/opt/docker_volumes/plex` is owned by `1000:1000`.

## Uninstall

```sh
helm uninstall media-stack
```

## Troubleshooting

- 404 from NGINX: ensure your client resolves the hostname to the Ingress controller IP (add to `/etc/hosts` if needed) and that the Ingress class matches your controller. `kubectl describe ingress -n <ns> <name>` to verify.
- 503 from NGINX: the Service likely has no ready endpoints. Check readiness and endpoints:
  - `kubectl get endpoints -n <ns>`
  - `kubectl get pods -n <ns> -o wide`
- Sonarr/Radarr can’t reach qBittorrent: confirm qBittorrent’s WebUI port in `/config/qBittorrent/qBittorrent.conf` (often 8080 or 8081). Set `.port` in values to the container’s real port and use the in-cluster Service DNS (e.g., `qbittorrent-svc:8080`) in Sonarr.
- Plex readiness: ensure Plex listens on 32400 and the container has write access to `/config`. If plugins fail to initialize when non-root, either chown `/opt/docker_volumes/plex` to `1000:1000` or run Plex as root (PUID/PGID 0 plus `fsGroup: 1000`).

## Notes

- Gluetun sidecars route traffic through VPN for privacy.
- Plex runs outside VPN for LAN streaming.
- TLS/cert-manager can be enabled later for secure Ingress.

## License

MIT
