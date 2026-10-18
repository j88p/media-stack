# media-stack

Helm chart for a self-hosted media server running on Kubernetes. Deploys **Plex**, **Sonarr**, **Radarr**, **qBittorrent**, **Prowlarr** and **Seerr** — all traffic-sensitive apps run with a dedicated **Gluetun VPN sidecar** (PIA / OpenVPN).

> **Note:** This chart is built for a single-node MicroK8s setup with Gateway API (nginx-gateway-fabric) and mergerfs storage. Some assembly required for other environments.

---

## Stack

| App | Purpose | VPN |
|-----|---------|-----|
| Plex | Media server | No (direct) |
| Sonarr | TV show management | Yes (gluetun sidecar) |
| Radarr | Movie management | Yes (gluetun sidecar) |
| qBittorrent | Torrent client | Yes (gluetun sidecar) |
| Prowlarr | Indexer aggregator | Yes (gluetun sidecar) |
| Seerr | Request management | No (direct) |

qBittorrent also runs a **port-updater sidecar** that automatically applies the PIA-forwarded port to qBittorrent via its API — no manual port configuration needed.

---

## Prerequisites

- Kubernetes cluster (tested on MicroK8s, Ubuntu 24.04)
- [Gateway API CRDs](https://gateway-api.sigs.k8s.io/) installed
- [nginx-gateway-fabric](https://github.com/nginx/nginx-gateway-fabric) as the Gateway controller
- A `Gateway` resource named `nginx-gateway` in namespace `nginx-gateway` with a listener section named `houseslab-local` for `*.local.houseslab.org` (or adapt `values.yaml` to your own gateway)
- mergerfs (or compatible) storage pool at `/mnt/plexpool`
- Private Internet Access (PIA) subscription

---

## Setup

### 1. Create the namespace

```sh
kubectl create namespace plex
```

Label it so the Gateway allows HTTPRoutes from it:

```sh
kubectl label namespace plex expose-via-gw=true
```

### 2. Create the PIA secret

```sh
kubectl create secret generic pia-credentials \
  --from-literal=username='<your_pia_username>' \
  --from-literal=password='<your_pia_password>' \
  -n plex
```

### 3. Create required directories

```sh
sudo mkdir -p \
  /opt/docker_volumes/plex/config \
  /opt/docker_volumes/plex/transcode \
  /opt/docker_volumes/sonarr \
  /opt/docker_volumes/radarr \
  /opt/docker_volumes/prowlarr \
  /opt/docker_volumes/qbittorrent/config \
  /opt/docker_volumes/seerr \
  /mnt/plexpool/movies \
  /mnt/plexpool/tvshows \
  /mnt/plexpool/barnfilmer \
  /mnt/plexpool/barnserier \
  /mnt/plexpool/newDownloads \
  /mnt/plexpool/incomplete \
  /mnt/plexpool/finished
```

Ensure UID/GID 1000 has read/write access:

```sh
sudo chown -R 1000:1000 /opt/docker_volumes /mnt/plexpool
```

### 4. Configure values.yaml

Key settings to review before installing:

```yaml
global:
  tz: Europe/Stockholm          # your timezone
  gateway:
    name: nginx-gateway         # your Gateway name
    namespace: nginx-gateway    # your Gateway namespace
    section: houseslab-local    # your listener section name
  vpn:
    region: "Netherlands"       # PIA region

apps:
  - name: plex
    env:
      ADVERTISE_IP: "https://plex.yourdomain.com,http://192.168.x.x:32400"
    volumes:
      config: /opt/docker_volumes/plex/config
      transcode: /opt/docker_volumes/plex/transcode
      movies: /mnt/plexpool/movies
      ...
```

All `ingressHost` values follow the pattern `<app>.local.houseslab.org` — change these to match your own domain.

---

## Install / Upgrade

```sh
# Install
microk8s helm install media-stack ~/repos/media-stack -n plex

# Upgrade (never use --reuse-values — it silently ignores values.yaml changes)
microk8s helm upgrade media-stack ~/repos/media-stack -n plex --force
```

After a configmap-only change, restart affected deployments manually:

```sh
microk8s kubectl rollout restart deployment/<name> -n plex
```

---

## Accessing apps

Apps are exposed via Gateway API HTTPRoutes on port 443 (HTTPS only). Add entries to your local `/etc/hosts` pointing to your Gateway's external IP:

```
192.168.1.241 plex.local.houseslab.org son.local.houseslab.org rad.local.houseslab.org qbit.local.houseslab.org pro.local.houseslab.org seerr.local.houseslab.org
```

Replace `192.168.1.241` with your Gateway controller's external IP.

| App | URL |
|-----|-----|
| Plex | https://plex.local.houseslab.org |
| Sonarr | https://son.local.houseslab.org |
| Radarr | https://rad.local.houseslab.org |
| qBittorrent | https://qbit.local.houseslab.org |
| Prowlarr | https://pro.local.houseslab.org |
| Seerr | https://seerr.local.houseslab.org |

---

## Post-install configuration

### qBittorrent first login

The first time qBittorrent starts, a temporary password is generated. Find it in the logs:

```sh
microk8s kubectl logs -n plex -l app=qbittorrent -c qbittorrent | grep -i password
```

Log in, then change the password under **Options → WebUI → Authentication**. The port-updater sidecar whitelists `10.0.0.0/8` for passwordless API access, so no credentials are needed for automation.

### Connect Prowlarr → Sonarr & Radarr

1. Open Prowlarr → **Settings → Apps**
2. Add Sonarr: `http://sonarr-service:8989`, API key from Sonarr → **Settings → General**
3. Add Radarr: `http://radarr-service:7878`, API key from Radarr → **Settings → General**

### Connect Sonarr & Radarr → qBittorrent

In Sonarr/Radarr → **Settings → Download Clients → Add → qBittorrent**:
- Host: `localhost`
- Port: `8080`
- No credentials needed (runs in same pod network namespace)

### Connect Seerr → Plex, Sonarr & Radarr

Open Seerr and follow the setup wizard. Use cluster-internal service names:
- Plex: `http://plex-service:32400`
- Sonarr: `http://sonarr-service:8989`
- Radarr: `http://radarr-service:7878`

---

## Storage (mergerfs)

The chart expects media at `/mnt/plexpool` — a mergerfs pool combining multiple drives. Example `/etc/fstab` entry:

```
/media/plex1:/media/plex3:/media/plex4 /mnt/plexpool fuse.mergerfs defaults,allow_other,use_ino,category.create=epmfs,nonempty 0 0
```

Mount each drive by UUID to avoid mismatches on reboot:

```sh
lsblk -o NAME,UUID,MOUNTPOINT   # find UUIDs
```

```
UUID=<disk-uuid> /media/plex1 ext4 defaults,nofail 0 2
```

---

## VPN details

- Provider: Private Internet Access (PIA), OpenVPN
- Port forwarding: enabled — gluetun writes the forwarded port to `/tmp/gluetun/forwarded_port`
- Port-updater sidecar on qBittorrent polls that file and applies the port via qBittorrent API
- All probes on VPN-sidecar apps use `exec wget localhost` (not `tcpSocket`) — gluetun's iptables blocks kubelet TCP probes from outside the pod

---

## Uninstall

```sh
microk8s helm uninstall media-stack -n plex
```

This removes all Deployments, Services, ConfigMaps and HTTPRoutes managed by the chart. Persistent data in `/opt/docker_volumes` and `/mnt/plexpool` is untouched.
