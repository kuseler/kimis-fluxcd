# Kubernetes & K3s Learnings

## Enabling IPv6 (Dual-Stack) on an Existing Cluster
- **The Issue:** When you enable IPv6 on an existing single-stack (IPv4) K3s cluster (e.g., by adding `--flannel-ipv6=true`), the `k3s` service might enter a crash loop. Flannel will log errors stating it failed to register the IPv6 network because the node lacks an IPv6 `podCIDR`.
- **The Root Cause:** The `podCIDRs` field on a Kubernetes `Node` object is **immutable** once the node is created. The node was originally registered with only an IPv4 subnet, and K3s cannot automatically append the IPv6 subnet to the existing Node object.
- **The Fix:** Delete the Node object (`kubectl delete node <node-name>`). Don't worry—this does not delete your pods or workloads. The `kubelet` process (managed by K3s) will automatically re-register the node, this time successfully allocating both an IPv4 and an IPv6 `podCIDR`.
- **Selecting an IPv6 Node IP:** If you need to manually specify the `--node-ip` for IPv6 (e.g., because your network interface has multiple IPv6 addresses), avoid the temporary "privacy extension" addresses, as these change frequently. Instead, look for the EUI-64 address. This is a stable, static address derived from your interface's MAC address (it typically contains `ff:fe` in the middle of the suffix) and will persist across reboots unless the physical hardware changes.
