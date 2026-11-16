# Tailscale Subnet Router & Exit Node (Homelab)

This setup runs a **Tailscale** subnet router + exit node on the `homelab` cluster (`apps/homelab/tailscale/`) advertising `192.168.1.0/24` using Tailscale's managed control plane.

## Bootstrap Steps (After Merging to `main`)

### 1. Generate an Auth Key in Tailscale
1. Open the [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys).
2. Under **Auth keys**, click **Generate auth key...** (enable **Reusable** if you want the pod to re-authenticate cleanly across rebuilds, or **Ephemeral: off** since state is persisted in `tailscale-state` PVC).

### 2. Create the Auth Secret (Homelab Cluster)
Create the `tailscale-auth` secret in the `tailscale` namespace:

```bash
kubectl -n tailscale create secret generic tailscale-auth \
  --from-literal=authkey="tskey-auth-..."
```

If the pod was already created and waiting for the secret, restart it:

```bash
kubectl -n tailscale rollout restart deploy/tailscale-subnet-router
```

### 3. Approve Subnet Route & Exit Node
1. In the [Tailscale Machines](https://login.tailscale.com/admin/machines) page, find `tailscale-subnet-router`.
2. Open its menu (`...`) → **Edit route settings...**
3. Enable **Subnet routes** (`192.168.1.0/24`) and **Exit node** (optional).

### 4. Connect Your Devices
Install the Tailscale app on your phone/laptop and sign in with the same Tailscale account. You can now reach `192.168.1.0/24` directly over Tailscale.
