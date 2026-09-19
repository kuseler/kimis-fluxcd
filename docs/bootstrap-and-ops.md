# Cluster Bootstrap & Operational Runbook

This document details the node setup, cluster architecture, manual secret requirements, and disaster recovery procedures for both clusters in the `kimis-fluxcd` repository.

---

## 1. Cluster Environments & Node Specifications

| Cluster | OS / Platform | Distribution | Role | Primary IP / Network |
| :--- | :--- | :--- | :--- | :--- |
| **Homelab** | NixOS | K3s | Internal workloads, L7 Ingress, Storage | `192.168.1.80` (Physical node IP) |
| **VPS** | Ubuntu Linux | K3s | Public edge, Rathole tunnel endpoint, TLS passthrough | Public IPv4/IPv6 (`kimimueller.de`) |

* **Ingress IP (`192.168.1.80`)**: Directly bound to the physical network interface of the NixOS homelab node (serving K3s Traefik via host port / Service).
* **NFS Storage (`192.168.1.82`)**: NAS hosting shared persistent media volumes (`/volume2/Gemeinsame_Dateien`).

---

## 2. Cluster Bootstrapping with Flux

Both clusters are reconciled from the `main` branch of `https://github.com/kuseler/kimis-fluxcd.git`.

### 2.1 Bootstrapping Homelab
```bash
# Run with kubeconfig pointed to the Homelab (NixOS) cluster
flux bootstrap github \
  --owner=kuseler \
  --repository=kimis-fluxcd \
  --branch=main \
  --path=clusters/homelab \
  --personal
```

### 2.2 Bootstrapping VPS
```bash
# Run with kubeconfig pointed to the VPS (Ubuntu) cluster
flux bootstrap github \
  --owner=kuseler \
  --repository=kimis-fluxcd \
  --branch=main \
  --path=clusters/vps \
  --personal
```

---

## 3. Secret Management (Manual Secrets)

Secrets are intentionally not stored in Git. During cluster bootstrap or recovery, the following manual Kubernetes secrets must be created:

### 3.1 Rathole Shared Secret Token
Both clusters require a shared token secret named `rathole-token` in their respective namespaces:

#### VPS (`apps/vps/rathole` namespace: `rathole-server`):
```bash
kubectl create namespace rathole-server --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic rathole-token \
  --namespace=rathole-server \
  --from-literal=token="<YOUR_SECURE_RATHOLE_TOKEN>"
```

#### Homelab (`apps/homelab/rathole-client` namespace: `rathole-client`):
```bash
kubectl create namespace rathole-client --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic rathole-token \
  --namespace=rathole-client \
  --from-literal=token="<YOUR_SECURE_RATHOLE_TOKEN>"
```

### 3.2 Traefik ACME Let's Encrypt Email Secret
Homelab Traefik requires an ACME registration email secret to obtain Let's Encrypt TLS certificates without exposing personal email addresses in Git:

```bash
kubectl create secret generic traefik-acme-email \
  --namespace=kube-system \
  --from-literal=email="<YOUR_LETSENCRYPT_EMAIL>"
```

---

## 4. TLS & Certificate Architecture

* **Edge (VPS)**: 
  * VPS Traefik operates in **TCP/TLS Passthrough** mode (`IngressRouteTCP`). It terminates neither HTTP nor TLS connections.
* **Internal (Homelab)**:
  * Homelab Traefik handles TLS termination.
  * Let's Encrypt certificates are acquired/managed at the Homelab Traefik layer (or via ACME resolver on Homelab).

---

## 5. Routine Operations & Troubleshooting

### Check Flux Sync Status
```bash
# Check status of git sources and kustomizations
flux get sources git
flux get kustomizations

# Force immediate reconciliation
flux reconcile kustomization apps --with-source
flux reconcile kustomization infrastructure --with-source
```

### Rathole Tunnel Troubleshooting
1. Check VPS Rathole Server logs:
   ```bash
   kubectl -n rathole-server logs -l app=rathole-server -f
   ```
2. Check Homelab Rathole Client logs:
   ```bash
   kubectl -n rathole-client logs -l app=rathole-client -f
   ```
3. Common issues:
   * **`early eof`**: Non-root container tried binding to a port < 1024 on the server. Ensure `bind_addr` is set to 8080/8443.
   * **Token mismatch**: Ensure `rathole-token` secret contains identical tokens on both clusters.
   * **`Expect UDP traffic`**: A UDP service (e.g. Wireguard) is missing `type = "udp"` on either the client or server configuration.
