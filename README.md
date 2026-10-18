# media-stack

Helm chart for a self-hosted media server running on Kubernetes. Deploys **Plex**, **Sonarr**, **Radarr**, **qBittorrent**, **Prowlarr** and **Seerr** — all traffic-sensitive apps run with a dedicated **Gluetun VPN sidecar** (supports any [gluetun-compatible](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) VPN provider).

---

## Stack

| App | Purpose | VPN sidecar |
|-----|---------|-------------|
| Plex | Media server | No |
| Sonarr | TV show management | Yes |
| Radarr | Movie management | Yes |
| qBittorrent | Torrent client | Yes |
| Prowlarr | Indexer aggregator | Yes |
| Seerr | Request management | No |

Each VPN app runs gluetun as a sidecar in the same pod — no shared VPN proxy, no network policies needed. qBittorrent also runs a **port-updater sidecar** that watches the gluetun-forwarded port and applies it to qBittorrent automatically via API.

---

## Prerequisites

- Kubernetes cluster (single-node works fine — tested on MicroK8s, Ubuntu 24.04)
- Helm 3
- A VPN subscription supported by [gluetun](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
- Storage at the paths you configure in `values.yaml` (mergerfs pool recommended for multi-disk setups)
- One of:
  - **Gateway API** with [nginx-gateway-fabric](https://github.com/nginx/nginx-gateway-fabric) — set `global.routing.type: gateway`
  - **Standard Ingress** with any nginx ingress controller — set `global.routing.type: ingress`

---

## Quick start

### 1. Create the namespace

```sh
kubectl create namespace plex
```

If using Gateway API, label the namespace so the Gateway accepts routes from it:

```sh
kubectl label namespace plex expose-via-gw=true
```

### 2. Create the VPN secret

```sh
kubectl create secret generic pia-credentials \
  --from-literal=username='YOUR_VPN_USERNAME' \
  --from-literal=password='YOUR_VPN_PASSWORD' \
  -n plex
```

Change the secret name in `values.yaml` if your provider or secret name differs (`global.vpn.secret`).

### 3. Create required directories

At minimum you need config dirs and wherever you want media stored:

```sh
sudo mkdir -p \
  /opt/docker_volumes/plex/config \
  /opt/docker_volumes/plex/transcode \
  /opt/docker_volumes/sonarr \
  /opt/docker_volumes/radarr \
  /opt/docker_volumes/prowlarr \
  /opt/docker_volumes/qbittorrent/config \
  /opt/docker_volumes/seerr

sudo chown -R 1000:1000 /opt/docker_volumes
```

Media paths are configured per-app in `values.yaml` — point them wherever your files live.

### 4. Edit values.yaml

Key things to set before installing:

```yaml
global:
  tz: Europe/London                   # your timezone

  routing:
    type: gateway                     # "gateway" or "ingress"
    gateway:
      name: nginx-gateway             # your Gateway resource name
      namespace: nginx-gateway        # namespace it lives in
      section: houseslab-local        # listener section name
    ingressClass: nginx               # used when type is "ingress"

  vpn:
    provider: private internet access # your VPN provider (see gluetun docs)
    type: openvpn                     # openvpn or wireguard
    region: "Netherlands"             # server region
    secret: pia-credentials           # k8s secret with username/password

apps:
  - name: plex
    env:
      ADVERTISE_IP: "https://plex.yourdomain.com,http://192.168.x.x:32400"
    volumes:
      config: /your/plex/config
      transcode: /your/plex/transcode
      movies: /your/movies

  - name: sonarr
    ingressHost: son.yourdomain.com
    # add any custom media paths:
    extraVolumeMounts:
      - name: tvshows
        mountPath: /tvshows
        hostPath: /your/tvshows
```

### 5. Install

```sh
helm install media-stack . -n plex
```

Upgrade:

```sh
helm upgrade media-stack . -n plex
```

> **MicroK8s users:** use `microk8s helm` and `~/repos/media-stack` as the chart path. Never use `--reuse-values` — it silently ignores `values.yaml` changes.

---

## Accessing apps

### Gateway API (type: gateway)

Apps are exposed as HTTPRoutes on HTTPS (port 443). Add entries to your local `/etc/hosts` pointing to your Gateway's external IP:

```
192.168.1.241 plex.yourdomain.com son.yourdomain.com rad.yourdomain.com qbit.yourdomain.com pro.yourdomain.com seerr.yourdomain.com
```

Then open e.g. `https://son.yourdomain.com`.

### Standard Ingress (type: ingress)

Apps are exposed as Ingress objects. Add entries to `/etc/hosts` pointing to your ingress controller IP:

```
192.168.1.128 plex.yourdomain.com son.yourdomain.com rad.yourdomain.com qbit.yourdomain.com pro.yourdomain.com seerr.yourdomain.com
```

Then open e.g. `http://son.yourdomain.com`.

---

## Custom media volumes

Any extra mount — a second movie library, a kids category, a music folder — goes in `extraVolumeMounts` on the app that needs it:

```yaml
- name: radarr
  extraVolumeMounts:
    - name: kids-movies
      mountPath: /kids
      hostPath: /mnt/pool/kids-movies
    - name: documentaries
      mountPath: /docs
      hostPath: /mnt/pool/documentaries
```

No template changes needed — just add to `values.yaml`.

---

## Storage (mergerfs)

For multi-disk setups, [mergerfs](https://github.com/trapexit/mergerfs) combines drives into a single mount point. Example `/etc/fstab`:

```
/media/disk1:/media/disk2:/media/disk3 /mnt/pool fuse.mergerfs defaults,allow_other,use_ino,category.create=epmfs,nonempty 0 0
```

Mount drives by UUID to avoid reorder on reboot:

```sh
# find UUIDs
lsblk -o NAME,UUID,MOUNTPOINT

# in /etc/fstab:
UUID=<disk-uuid> /media/disk1 ext4 defaults,nofail 0 2
```

---

## VPN details

Any [gluetun-supported provider](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) works. The secret fields differ per provider — check the gluetun wiki for the exact keys required. Common examples:

**Private Internet Access (OpenVPN)**
```yaml
vpn:
  provider: private internet access
  type: openvpn
  secret: pia-credentials   # keys: username, password
```

**Mullvad (WireGuard)**
```yaml
vpn:
  provider: mullvad
  type: wireguard
  secret: mullvad-credentials  # keys: private_key, addresses (+ optional city/country)
```

**NordVPN (WireGuard)**
```yaml
vpn:
  provider: nordvpn
  type: wireguard
  secret: nordvpn-credentials  # keys: private_key
```

**ExpressVPN (OpenVPN)**
```yaml
vpn:
  provider: expressvpn
  type: openvpn
  secret: expressvpn-credentials  # keys: activation_code
```

Full provider list and secret fields: [gluetun provider docs](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)

**Other notes:**
- Port forwarding is enabled by default (PIA-specific) — the port-updater sidecar applies it to qBittorrent automatically. Disable with `portForwardUpdater: false` for other providers
- All probes on VPN apps use `exec wget localhost` instead of `tcpSocket` — gluetun's iptables blocks external TCP probes from the kubelet, so exec probes running inside the container are required
- Startup probe allows up to 10 minutes for slow app starts (failureThreshold 60 × periodSeconds 10)

---

## Post-install configuration

### qBittorrent first login

A temporary password is printed in the logs on first start:

```sh
kubectl logs -n plex -l app=qbittorrent -c qbittorrent | grep -i password
```

Log in, change the password under **Options → WebUI → Authentication**.

### Wire up the apps

1. **Prowlarr → Sonarr & Radarr**: Prowlarr → Settings → Apps → Add each, using cluster-internal URLs:
   - Sonarr: `http://sonarr-service:8989`
   - Radarr: `http://radarr-service:7878`
   - API keys from each app under Settings → General

2. **Sonarr & Radarr → qBittorrent**: Settings → Download Clients → qBittorrent
   - Host: `localhost`, Port: `8080`
   - No credentials needed (same pod network namespace)

3. **Seerr → Plex, Sonarr & Radarr**: follow the setup wizard, use:
   - Plex: `http://plex-service:32400`
   - Sonarr/Radarr: as above

---

## Uninstall

```sh
helm uninstall media-stack -n plex
```

Persistent data in your config and media directories is untouched.
