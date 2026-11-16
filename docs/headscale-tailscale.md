# Headscale & Tailscale VPN Setup

This repository runs a self-hosted **Headscale** control plane on the `vps` cluster (`https://headscale.kimimueller.de`) and a **Tailscale** subnet router + exit node on the `homelab` cluster advertising `192.168.1.0/24`.

## Architecture

* **VPS (`apps/vps/headscale/`)**: Runs `headscale/headscale:0.25.1` with SQLite persistence (`headscale-data` PVC) behind VPS Traefik (`certResolver: letsencrypt`).
  * `apps/vps/rathole/ingressroute.yaml` explicitly excludes `headscale.kimimueller.de` from `rathole-https` TCP passthrough so VPS Traefik terminates TLS locally for Headscale.
* **Homelab (`apps/homelab/tailscale/`)**: Runs `tailscale/tailscale:v1.82.0` in kernel mode (`/dev/net/tun`, `NET_ADMIN`) with `tailscale-state` PVC persistence, advertising `192.168.1.0/24` and exit-node routing.

---

## One-Time Bootstrap Steps (After Merging to `main`)

### 1. Create a Headscale User & Pre-Auth Key (VPS Cluster)

Wait for the `headscale` pod to become `Running`:

```bash
kubectl -n headscale get pods
```

Create a user (e.g. `homelab`) and generate a reusable pre-auth key for the homelab subnet router:

```bash
kubectl -n headscale exec deploy/headscale -- headscale users create homelab
kubectl -n headscale exec deploy/headscale -- headscale preauthkeys create --user 1 --reusable --expiration 24h
```

> **Note:** Check the user ID with `headscale users list` if needed.

### 2. Create the Auth Secret (Homelab Cluster)

Create the `tailscale-auth` secret in the `tailscale` namespace using the pre-auth key from step 1:

```bash
kubectl -n tailscale create secret generic tailscale-auth \
  --from-literal=authkey="<YOUR_HEADSCALE_PREAUTH_KEY>"
```

Restart the `tailscale-subnet-router` pod if it was waiting on the secret:

```bash
kubectl -n tailscale rollout restart deploy/tailscale-subnet-router
```

### 3. Approve the Subnet Route & Exit Node (VPS Cluster)

List the advertised routes in Headscale and enable them:

```bash
kubectl -n headscale exec deploy/headscale -- headscale routes list
kubectl -n headscale exec deploy/headscale -- headscale routes enable -r <ROUTE_ID>
```

*(Enable all route IDs shown for `192.168.1.0/24` and `0.0.0.0/0` / `::/0` if using the exit node.)*

### 4. Connect Your Devices (Phone / Laptop)

* **Linux / macOS CLI:**
  ```bash
  tailscale up --login-server=https://headscale.kimimueller.de
  ```
* **iOS / Android App:**
  * Open Tailscale settings → **Alternate Coordination Server** (tap the app version or account menu depending on platform) → set `https://headscale.kimimueller.de`.
  * Register the node in Headscale using the node key shown in the browser/app:
    ```bash
    kubectl -n headscale exec deploy/headscale -- headscale nodes register --user homelab --key <NODE_KEY>
    ```
