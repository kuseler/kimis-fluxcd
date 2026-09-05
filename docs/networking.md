# Networking Architecture

This document comprehensively describes the network topology, domain naming, traffic routing (including NAT traversal), and DNS resolution across the `kimis-fluxcd` repository (spanning the Homelab and VPS clusters).

---

## 1. Network Topology Overview

The architecture consists of two environments securely connected via a **Rathole** tunnel:
*   **VPS Cluster**: The public edge. It receives public traffic (e.g., `*.kimimueller.de` and Wireguard UDP) and passes it blindly over a secure tunnel to the Homelab.
*   **Homelab Cluster**: The private backend (behind NAT). It terminates the public traffic, performs L7 (HTTP/HTTPS) Ingress routing, and serves local users directly via custom local domains.

---

## 2. External Traffic Flow (NAT Traversal via Rathole)

Because the Homelab is behind a NAT, public traffic enters through the VPS and is tunneled to the Homelab.

1.  **Public Edge (VPS Traefik)**:
    *   Traffic for `*.kimimueller.de` hits the VPS.
    *   Traefik on the VPS is configured with `IngressRouteTCP` (TLS Passthrough for 443, raw TCP for 80). It does **not** do L7 routing. It forwards all web traffic blindly to the Rathole server.
2.  **The Tunnel (Rathole Server & Client)**:
    *   **Server (VPS)**: Listens on ports 8080 (HTTP), 8443 (HTTPS), and 51820 (WireGuard UDP). Tunnels data over a control channel (port 2333) to the Homelab.
    *   **Client (Homelab)**: Receives the tunneled data and forwards it to local endpoints.
3.  **Homelab Ingress (Homelab Traefik)**:
    *   The Rathole Client forwards HTTP/HTTPS traffic to the Homelab's Traefik instance (`traefik.kube-system.svc.cluster.local`).
    *   Homelab Traefik acts as the actual L7 Ingress Controller. It reads the Host headers/SNI (e.g., `homepage.kimimueller.de`) and routes traffic to the correct Homelab pods based on standard Kubernetes `Ingress` resources.
4.  **Wireguard Traffic (UDP)**:
    *   UDP port 51820 hits the VPS `rathole-external` LoadBalancer, goes through Rathole, and is forwarded directly to the Fritz!Box router (`192.168.1.1:50571`) by the Rathole Client.

---

## 3. Local Domain Schemes & Internal Traffic

When accessing services from inside the Homelab network, traffic does not go through the VPS.

| Scope | Suffix / Domain | Description | Resolution Target |
| :--- | :--- | :--- | :--- |
| **Local Primary** | `*.homelab.home` | Standard internal domain for homelab services | Homelab Traefik (`192.168.1.80`) |
| **Local Alias** | `*.homelab.fritz.box` | Direct Fritz!Box-compatible alias | Homelab Traefik (`192.168.1.80`) |
| **General LAN** | `*.home` | Translated dynamically to `*.fritz.box` | Fritz!Box Router (`192.168.1.1`) |
| **Public / Edge** | `*.kimimueller.de` | Public domain routed via VPS -> Rathole -> Homelab | VPS Traefik Ingress |

---

## 4. Local DNS Resolution (CoreDNS)

Homelab DNS is handled by a custom CoreDNS deployment (`apps/homelab/coredns/`):
*   **Local Ingress Interception**: Queries matching `.*\.?homelab\.home` or `.*\.?homelab\.fritz\.box` are rewritten to `traefik-ingress.home` and statically resolved to `192.168.1.80` (Homelab Traefik LB IP).
*   **LAN Translation**: Any other `*.home` query (e.g., `nas.home`) is rewritten to `{1}.fritz.box` and resolved by the Fritz!Box.
*   **Upstream Forwarding**: Unresolved queries are forwarded to `192.168.1.1` (Fritz!Box), then Google DNS (`8.8.8.8`, `2001:4860:4860::8888`).

---

## 5. Machine-Readable Quirks & Fact-Checks

*   `traefik_dual_setup`: **FACT-CHECK**: The repo runs Traefik in *both* clusters. VPS Traefik is solely for TCP/TLS passthrough to Rathole. Homelab Traefik handles actual L7 Ingress logic.
*   `rathole_port_binding`: Rathole runs as non-root (UID 1000). Server `bind_addr` cannot use ports < 1024. Use 8080/8443 and map via k8s Service.
*   `rathole_udp_config`: UDP services must explicitly declare `type = "udp"` in BOTH server and client TOML configurations.

---

## 6. Known Quirks and Fixes (Rathole & Routing)

During the initial setup, a few issues were encountered and resolved. This documentation serves as a reference for future maintenance.

### 1. Non-Root Container Port Binding
**Symptom:** The client connects successfully to the control channel but immediately drops connections with an `early eof` error when attempting to establish data channels.
**Cause:** The official `rapiz1/rathole` Docker image runs as a non-root user (UID/GID 1000:1000) by default for security reasons. Linux prevents non-root users from binding to privileged ports (ports under 1024, like 80 and 443). When the server attempts to bind to these ports, it silently crashes the listener tasks.
**Fix:** The `bind_addr` for HTTP and HTTPS services in the server's configuration must be set to unprivileged ports (e.g., `8080` and `8443`). The Kubernetes `Service` then maps incoming traffic on standard ports (80/443) to these `targetPort`s (8080/8443).

### 2. UDP Service Configuration (e.g., WireGuard)
**Symptom:** The client logs show the warning: `Failed to run the data channel: Expect UDP traffic. Please check the configuration.`
**Cause:** By default, `rathole` assumes services are TCP. If a service is configured as UDP on the server, the client must also explicitly declare it as UDP.
**Fix:** Ensure that any UDP service (like `fritzbox-wg`) includes `type = "udp"` in the client's configuration block (`[client.services.<name>]`). Note that the server configuration also requires this `type = "udp"` declaration.
