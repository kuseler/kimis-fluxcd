# Media & Arr Stack (`media` Namespace)

This document describes the media automation, downloaders, and VPN killswitch architecture deployed on the **Homelab** cluster in the `media` namespace.

---

## 1. Architecture Overview

| Service | Purpose | Internal Service URL | Local Ingress Hostnames |
| :--- | :--- | :--- | :--- |
| **Gluetun** | ProtonVPN WireGuard gateway & killswitch (sidecar in `downloads` pod) | `http://sabnzbd.media.svc.cluster.local:8888` (HTTP proxy)<br>`socks5://sabnzbd.media.svc.cluster.local:8388` | — |
| **SABnzbd** | Usenet (NZB) download client (routed through Gluetun) | `http://sabnzbd.media.svc.cluster.local:8080` | `sabnzbd.homelab.home`, `sabnzbd.homelab.fritz.box` |
| **qBittorrent** | Torrent download client (routed through Gluetun) | `http://qbittorrent.media.svc.cluster.local:8081` | `qbittorrent.homelab.home`, `qbittorrent.homelab.fritz.box` |
| **Sonarr** | TV series automation | `http://sonarr.media.svc.cluster.local:8989` | `sonarr.homelab.home`, `sonarr.homelab.fritz.box` |
| **Radarr** | Movie automation | `http://radarr.media.svc.cluster.local:7878` | `radarr.homelab.home`, `radarr.homelab.fritz.box` |
| **Prowlarr** | Indexer manager & proxy | `http://prowlarr.media.svc.cluster.local:9696` | `prowlarr.homelab.home`, `prowlarr.homelab.fritz.box` |
| **Bazarr** | Subtitle automation (Sonarr/Radarr) | `http://bazarr.media.svc.cluster.local:6767` | `bazarr.homelab.home`, `bazarr.homelab.fritz.box` |
| **Lidarr** | Music library automation | `http://lidarr.media.svc.cluster.local:8686` | `lidarr.homelab.home`, `lidarr.homelab.fritz.box` |
| **Readarr** | Book & comic automation | `http://readarr.media.svc.cluster.local:8787` | `readarr.homelab.home`, `readarr.homelab.fritz.box` |
| **Jellyfin** | Media streaming server | `http://jellyfin.media.svc.cluster.local:8096` | `jellyfin.homelab.home`, `jellyfin.homelab.fritz.box` |

---

## 2. Gluetun + ProtonVPN Killswitch

Both **SABnzbd** and **qBittorrent** run inside the `downloads` Pod alongside the **Gluetun** sidecar container (`apps/homelab/media/downloads.yaml`).

* **Shared Network Namespace & Native Sidecar**: `gluetun` runs as a Kubernetes native sidecar (`initContainers` with `restartPolicy: Always` and a `startupProbe` running `/gluetun-entrypoint healthcheck`). `sabnzbd` and `qbittorrent` only start after the WireGuard tunnel and `iptables` killswitch are verified active, and all three containers share the Pod's network namespace (`tun0`).
* **Killswitch (`FIREWALL=on`)**: Gluetun enforces iptables rules that drop all non-VPN outbound traffic except the configured `FIREWALL_OUTBOUND_SUBNETS` (`10.42.0.0/16` Pod CIDR, `10.43.0.0/16` Service CIDR, `192.168.0.0/16` LAN).
* **Ingress Ports (`FIREWALL_INPUT_PORTS=8080,8081`)**: Allows Traefik and cluster workloads to reach SABnzbd (`8080`) and qBittorrent (`8081`).
* **Healthcheck**: Gluetun runs `/gluetun-entrypoint healthcheck` as a Kubernetes `livenessProbe` so the pod restarts automatically if the tunnel fails.

### Required Secret: `proton-vpn-secret`

Create the ProtonVPN WireGuard secret in the `media` namespace before or right after deploying:

```bash
kubectl create secret generic proton-vpn-secret \
  --namespace media \
  --from-literal=private-key="<YOUR_PROTON_WIREGUARD_PRIVATE_KEY>"
```

If using OpenVPN instead of WireGuard, switch `VPN_TYPE` to `openvpn` in `apps/homelab/media/downloads.yaml` and populate `username` and `password` keys in `proton-vpn-secret`.

---

## 3. Storage Layout

* **Config Storage**: Each application uses a dedicated `ReadWriteOnce` PVC (`2Gi`) defined in `infrastructure/homelab/media-storage.yaml`.
* **Shared Media & Downloads (`media-nfs-pvc`)**: Mounted at `/media` on Jellyfin, SABnzbd, qBittorrent, Sonarr, Radarr, Bazarr, Lidarr, and Readarr (`PUID=1000`, `PGID=1000`). Keeping downloads and media on the same NFS mount allows atomic moves/hardlinks:
  * `/media/downloads/incomplete`
  * `/media/downloads/complete`
  * `/media/movies`
  * `/media/tv`
  * `/media/music`
  * `/media/books`
