# AGENTS.md — Agent Guidelines & Repository Memory

Welcome to `kimis-fluxcd`. This file provides instructions, architecture context, and operational rules for AI agents working in this repository.

---

## 1. Repository Architecture & Layout

This is a multi-cluster GitOps repository managed by **FluxCD**:

* **`clusters/`**: Cluster bootstrap and root Flux configurations.
  * `clusters/homelab/`: Local homelab cluster definitions (`apps.yaml`, `infrastructure.yaml`, `flux-system/`).
  * `clusters/vps/`: Public VPS cluster definitions (`apps.yaml`, `infrastructure.yaml`, `flux-system/`).
* **`infrastructure/`**: Foundational cluster resources (storage classes, persistent volume claims, namespaces).
* **`apps/`**: Workloads and services split across environments:
  * `apps/homelab/`: Local services (CoreDNS, Jellyfin media, Homepage, Rathole client).
  * `apps/vps/`: Public edge services (Rathole server, Traefik ingress).
* **`docs/`**: Centralized documentation, architecture notes, and troubleshooting runbooks.

### Network Architecture
* **Public Domain**: Target address is `kimimueller.de`.
* **NAT Traversal**: Services hosted on the `homelab` cluster are tunneled to the public `vps` cluster using **Rathole**.
* **Ingress (Dual-Traefik Setup)**: 
  * **VPS Traefik**: Terminates TLS locally for the apex domain `kimimueller.de` (Hugo Blog) via Let's Encrypt. Forwards all other web traffic (`*.kimimueller.de`) via TCP/TLS passthrough to the Rathole server.
  * **Homelab Traefik**: Acts as the L7 Ingress Controller for homelab services, receiving tunneled traffic from the Rathole client and terminating TLS locally.

---

## 2. Core Operational Rules

1. **Declarative & Valid Kustomize Hierarchy**:
   * Every new resource (`Deployment`, `Service`, `ConfigMap`, etc.) must be referenced in its directory's `kustomization.yaml`.
   * Parent `kustomization.yaml` files must include child directories.
   * Do not commit invalid YAML or broken references.
2. **Cluster Context Awareness**:
   * Distinguish between `homelab` (internal network, behind NAT) and `vps` (public IP, edge routing).
   * Do not assume public cluster ingress is directly reachable on homelab workloads; use Rathole client/server pairs.
3. **Documentation Placement**:
   * Place architecture guides, setup walkthroughs, and operational runbooks in `docs/`.
   * Keep manifest directories clean of standalone documentation files unless explicitly requested.

---

## 3. Self-Improvement & Documentation Evolution

Agents are explicitly authorized and expected to keep this repository self-documenting and maintainable:

1. **Document Known Quirks & Solutions**:
   * Whenever you diagnose an obscure issue, edge case, or environment quirk (e.g. non-root container permissions, protocol mismatches, DNS forwarder behaviors), document it in a relevant file inside `docs/` (or update existing docs such as `docs/networking.md`).
2. **Update This File (`AGENTS.md`)**:
   * If you discover a reusable pattern, design principle, or recurring mistake that future agents must avoid, add it to the **"Learned Patterns & Operational Quirks"** section below.
   * If an instruction in this file becomes obsolete due to architecture changes, update or prune it.
3. **Propose Improvements**:
   * When making changes, identify areas where documentation or repository layout can be clarified, and propose or implement clean documentation updates.

---

## 4. Learned Patterns & Operational Quirks

*(Agents: append new hard-won operational discoveries and rules below as they are verified.)*

* **Rathole Port Binding**:
  * Rathole Docker images run as non-root (UID/GID 1000).
  * Do not bind `bind_addr` directly to ports < 1024 inside the container. Use unprivileged ports (e.g. `8080`, `8443`) and let the Kubernetes `Service` map port `80`/`443` to targetPort `8080`/`8443`.
* **Rathole Protocol Definitions**:
  * Any UDP service (e.g., WireGuard) must explicitly declare `type = "udp"` in both server and client configurations; otherwise Rathole defaults to TCP and drops data channels.
* **CoreDNS Forwarding**:
  * CoreDNS forwarders should support dual-stack (IPv4 & IPv6) where applicable.
* **Traefik Edge Routing Split (VPS)**:
  * When hosting edge workloads (like Hugo) on the VPS alongside Rathole TLS passthrough, the TCP passthrough router (`rathole-https`) must not use a broad `HostSNI('*')`. It must explicitly match subdomains (`HostSNI('*.kimimueller.de', '*.*.kimimueller.de')`) so that apex domain TLS connections are terminated by Traefik rather than forwarded blindly.
