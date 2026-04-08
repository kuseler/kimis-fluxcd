# kimis-fluxcd

GitOps repository for two k3s clusters managed with [Flux CD](https://fluxcd.io/).  
One cluster runs in the **homelab** (Raspberry Pi / local network), the other on a **VPS** (public internet).

---

## Table of Contents

- [Architecture overview](#architecture-overview)
- [Repository layout](#repository-layout)
- [Cluster: homelab](#cluster-homelab)
  - [Networking (CoreDNS + Traefik)](#networking-coredns--traefik)
  - [Tunnel client (rathole)](#tunnel-client-rathole)
  - [Applications](#applications-homelab)
- [Cluster: VPS](#cluster-vps)
  - [Networking (Traefik TCP passthrough)](#networking-traefik-tcp-passthrough)
  - [Tunnel server (rathole)](#tunnel-server-rathole)
  - [Applications](#applications-vps)
- [Infrastructure](#infrastructure)
- [Secrets](#secrets)
- [How Flux CD is wired](#how-flux-cd-is-wired)
- [Adding a new application](#adding-a-new-application)
- [Access URLs](#access-urls)

---

## Architecture overview

```
Internet
    │
    ▼
┌──────────────────────────────────────────────┐
│  VPS  (kimimueller.de)                       │
│  k3s cluster                                 │
│                                              │
│  Traefik (TCP passthrough, ports 80 / 443)  │
│      │                                       │
│  rathole server  ←── control :2333          │
│      │                                       │
│  Homepage dashboard  :3000 (LoadBalancer)    │
└──────────┬───────────────────────────────────┘
           │  encrypted TCP tunnel (rathole)
           │  ports 80, 443, 51820 (WireGuard UDP)
┌──────────▼───────────────────────────────────┐
│  Homelab  (192.168.1.x / fritz.box)          │
│  k3s cluster                                 │
│                                              │
│  Traefik (HTTP/HTTPS ingress controller)     │
│  ├── homepage.kimimueller.de                 │
│  ├── homepage.homelab.*                      │
│  ├── jellyfin.homelab.*                      │
│  └── working.homelab.*                       │
│                                              │
│  rathole client  ──► VPS:2333               │
│  CoreDNS  (resolves *.homelab.* → Traefik)  │
└──────────────────────────────────────────────┘
```

**Key idea:** All public traffic arrives at the VPS.  
Traefik on the VPS does TCP-level passthrough on ports 80 and 443 straight into the rathole server.  
The rathole server forwards those connections through the persistent tunnel to the rathole client running in the homelab, which hands them off to the homelab Traefik ingress controller.  
TLS is terminated by homelab Traefik, so the VPS never sees plaintext.

---

## Repository layout

```
kimis-fluxcd/
├── apps/
│   ├── homelab/          # Apps deployed to the homelab cluster
│   │   ├── coredns/
│   │   ├── homepage/
│   │   ├── media/
│   │   ├── rathole-client/
│   │   └── working-page/
│   └── vps/              # Apps deployed to the VPS cluster
│       ├── homepage/
│       ├── rathole/
│       └── traefik-ingress/
├── clusters/
│   ├── homelab/          # Flux Kustomization entry-points for homelab
│   └── vps/              # Flux Kustomization entry-points for VPS
└── infrastructure/
    ├── homelab/          # PVs, PVCs, namespaces for homelab
    └── vps/              # (currently empty, reserved for future use)
```

Each leaf directory contains a `kustomization.yaml` that lists the manifests it owns.  
Parent `kustomization.yaml` files aggregate child directories.

---

## Cluster: homelab

### Networking (CoreDNS + Traefik)

| Component | Role |
|-----------|------|
| **Traefik** | Standard k3s ingress controller; handles HTTP/HTTPS routing based on `Host:` header. Runs as a `LoadBalancer` service bound to `192.168.1.80`. |
| **CoreDNS** | Custom DNS server (`namespace: coredns`, `LoadBalancer` on UDP/TCP 53). Rewrites `*.homelab.fritz.box` and `*.homelab.local` to `traefik-ingress.local` (→ `192.168.1.80`), then forwards everything else to the Fritz!Box (`192.168.1.1`) and Google DNS (`8.8.8.8`). |

> **Router DNS tip:** Point your Fritz!Box's custom DNS server at the CoreDNS `LoadBalancer` IP so that all homelab devices resolve `*.homelab.*` to Traefik automatically.

### Tunnel client (rathole)

- **Namespace:** `rathole-client`
- **Image:** `rapiz1/rathole:v0.5.0`
- Connects to `kimimueller.de:2333` (VPS rathole server control port).
- Forwards three services through the tunnel:

| Service | Local address | Protocol |
|---------|--------------|----------|
| `http`  | `traefik.traefik.svc.cluster.local:80`  | TCP |
| `https` | `traefik.traefik.svc.cluster.local:443` | TCP |
| `fritzbox-wg` | `192.168.1.1:51444` | UDP |

- Auth token read from the `rathole-auth` Secret (key: `token`) — **created manually, not in Git**.

### Applications (homelab)

#### Homepage (`apps/homelab/homepage/`)

- **Image:** `ghcr.io/gethomepage/homepage:latest`
- **Namespace:** `default`
- **Port:** 3000 (ClusterIP Service, exposed via Traefik Ingress)
- **Config volume:** `hostPath: /var/lib/homepage/config` — place YAML config files here on the node.
- **Kubernetes widget:** enabled via `mode: cluster` ConfigMap mounted at `/app/config/kubernetes.yaml`. The pod uses a dedicated `ServiceAccount` with a read-only `ClusterRole` covering nodes, namespaces, pods, services, ingresses, deployments, replicasets, and metrics.
- **Allowed hosts:** `homepage.kimimueller.de`, `homepage.homelab.fritz.box`, `homepage.homelab.local`
- **Auto-discovery:** other apps annotate their Ingress with `gethomepage.dev/*` labels so Homepage picks them up automatically.

#### Jellyfin (`apps/homelab/media/`)

- Deployed via **HelmRelease** (`helm.toolkit.fluxcd.io/v2`) from the `jellyfin-helm` HelmRepository.
- **Namespace:** `media`
- **Ingress hosts:** `jellyfin.homelab.fritz.box`, `jellyfin.homelab.local`
- **Persistence:**
  - Config: `jellyfin-config-pvc` (1 Gi, `ReadWriteOnce`, provisioned in `infrastructure/homelab/jellyfin-storage.yaml`)
  - Media: `media-nfs-pvc` (3 Ti, NFS from `nas.fritz.box:/volume2/Gemeinsame_Dateien`, static binding)
- **Homepage widget annotations:** `gethomepage.dev/enabled: "true"`, group `Media`

#### Working page (`apps/homelab/working-page/`)

- Minimal nginx pod serving a static "i am working" HTML page from a ConfigMap.
- **Namespace:** `default`
- **Ingress hosts:** `homelab.local`, `homelab.fritz.box`, `working.homelab.fritz.box`, `working.homelab.local`
- Useful as a quick "is Traefik alive?" health check.

#### rathole client (`apps/homelab/rathole-client/`)

See [Tunnel client](#tunnel-client-rathole) above.

#### CoreDNS (`apps/homelab/coredns/`)

See [Networking](#networking-coredns--traefik) above.

---

## Cluster: VPS

### Networking (Traefik TCP passthrough)

The VPS Traefik is configured with two `IngressRouteTCP` rules that match **all** hostnames (`HostSNI('*')`) at the TCP layer:

| Rule | Entrypoint | Target | TLS |
|------|-----------|--------|-----|
| `rathole-http`  | `web` (80)      | `rathole:80`  | none |
| `rathole-https` | `websecure` (443) | `rathole:443` | passthrough |

Because these rules capture every connection at the TCP level before any HTTP routing, **it is not possible to add a standard Kubernetes `Ingress` object on ports 80 or 443 on the VPS.** Any new VPS application that needs its own port must use a `LoadBalancer` service.

### Tunnel server (rathole)

- **Namespace:** `rathole`
- **Image:** `rapiz1/rathole:v0.5.0`
- Listens on four ports:

| Port | Protocol | Purpose |
|------|----------|---------|
| 2333 | TCP | Control channel (clients connect here) |
| 80   | TCP | HTTP tunnel ingress |
| 443  | TCP | HTTPS tunnel ingress |
| 51820 | UDP | Fritz!Box WireGuard tunnel |

- **Services:**
  - `rathole` (ClusterIP) — internal ClusterIP used by Traefik `IngressRouteTCP` to forward port 80/443 traffic.
  - `rathole-external` (LoadBalancer) — exposes control `:2333` and WireGuard `:51820` directly on the node IP. Kept separate from the ClusterIP service to avoid port conflicts with Traefik's own LoadBalancer.
- Auth token read from the `rathole-auth` Secret (key: `token`) — **created manually, not in Git**.

### Applications (VPS)

#### Homepage (`apps/vps/homepage/`)

- **Image:** `ghcr.io/gethomepage/homepage:latest`
- **Namespace:** `default`
- **Port:** 3000 exposed as a **`LoadBalancer`** service (klipper-lb binds it to the VPS node IP).
  - Reason: cannot use a standard Ingress because Traefik TCP rules capture all port 80/443 traffic.
- **Config volume:** `hostPath: /var/lib/homepage/config`
- **Kubernetes widget:** same `mode: cluster` ConfigMap + read-only ClusterRole pattern as homelab.
- **Allowed hosts:** `homepage.vps.kimimueller.de`
- To access via a hostname, point an A record `homepage.vps.kimimueller.de` → VPS public IP, then open `http://homepage.vps.kimimueller.de:3000`.

---

## Infrastructure

### homelab (`infrastructure/homelab/`)

| File | Creates |
|------|---------|
| `namespace.yaml` | `media` namespace |
| `storage.yaml` | `media-nfs-pv` PersistentVolume + `media-nfs-pvc` PersistentVolumeClaim (NFS, 3 Ti) |
| `jellyfin-storage.yaml` | `jellyfin-config-pvc` PersistentVolumeClaim (1 Gi, `ReadWriteOnce`) |

### vps (`infrastructure/vps/`)

Currently empty (`resources: []`). Reserved for future VPS-specific infrastructure (e.g. cert-manager, external-dns).

---

## Secrets

Secrets are **not stored in Git**. They must be created manually on each cluster before Flux applies the dependent workloads.

| Secret name | Namespace | Key | Used by |
|-------------|-----------|-----|---------|
| `rathole-auth` | `rathole` (VPS) | `token` | rathole server deployment |
| `rathole-auth` | `rathole-client` (homelab) | `token` | rathole client deployment |
| `flux-system` | `flux-system` (both) | SSH key / token | Flux GitRepository authentication |

Create a rathole secret:
```bash
kubectl create secret generic rathole-auth \
  --namespace rathole \
  --from-literal=token='<your-shared-token>'
```

---

## How Flux CD is wired

Both clusters follow the same pattern:

```
clusters/<cluster>/flux-system/
  gotk-components.yaml   # Flux controllers (generated, do not edit)
  gotk-sync.yaml         # GitRepository + root Kustomization pointing at clusters/<cluster>/
  kustomization.yaml     # Aggregates the two files above

clusters/<cluster>/
  infrastructure.yaml    # Flux Kustomization → infrastructure/<cluster>/
  apps.yaml              # Flux Kustomization → apps/<cluster>/  (dependsOn: infrastructure)
  kustomization.yaml     # Aggregates all three entry-points
```

Flux polls the `main` branch of `https://github.com/kuseler/kimis-fluxcd.git` every minute (`interval: 1m0s`) and reconciles every 10 minutes (`interval: 10m0s`).

---

## Adding a new application

### Homelab app (standard Ingress)

1. Create `apps/homelab/<appname>/` with your manifests.
2. Add a `kustomization.yaml` listing them.
3. Add `- <appname>` to `apps/homelab/kustomization.yaml`.
4. Optionally annotate the Ingress with `gethomepage.dev/*` labels for auto-discovery.

### VPS app on port 80/443 (tunnelled through homelab)

Not directly possible — all port 80/443 TCP traffic on the VPS goes to rathole. Deploy the app on the homelab and expose it through the existing tunnel.

### VPS app on a custom port

1. Create `apps/vps/<appname>/` with a Deployment and a `LoadBalancer` Service on a free port.
2. Add `kustomization.yaml` and reference it from `apps/vps/kustomization.yaml`.
3. Add a firewall rule on the VPS to open the port if needed.

---

## Access URLs

| Service | URL | Notes |
|---------|-----|-------|
| Homepage (homelab) | `http://homepage.homelab.fritz.box` | Local network only |
| Homepage (homelab) | `http://homepage.homelab.local` | Local network only |
| Homepage (homelab, public) | `https://homepage.kimimueller.de` | Via rathole tunnel |
| Homepage (VPS) | `http://homepage.vps.kimimueller.de:3000` | Direct LoadBalancer |
| Jellyfin | `http://jellyfin.homelab.fritz.box` | Local network only |
| Jellyfin | `http://jellyfin.homelab.local` | Local network only |
| Working page | `http://homelab.fritz.box` | Quick Traefik health check |
| Working page | `http://homelab.local` | Quick Traefik health check |
