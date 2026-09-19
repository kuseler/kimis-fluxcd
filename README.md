# kimis-fluxcd

Multi-cluster GitOps repository powered by **FluxCD**, managing infrastructure and application workloads across a local **Homelab** cluster and an edge **VPS** cluster.

---

## Architecture Overview

* **Target Public Domain**: `kimimueller.de`
* **Edge Cluster (`vps`)**:
  * Public-facing gateway / ingress point.
  * Runs Traefik configured for TCP/TLS passthrough.
  * Hosts the **Rathole server** for secure NAT-traversal tunneling to the homelab.
* **Internal Cluster (`homelab`)**:
  * Private Kubernetes cluster located behind NAT.
  * Hosts the **Rathole client**, establishing an outbound tunnel to the VPS.
  * Runs local Traefik acting as the L7 Ingress Controller for all applications.
  * Runs split-horizon DNS via custom **CoreDNS** mapping `.homelab.home` and `.homelab.fritz.box` directly to internal ingress (`192.168.1.80`).

```
                [ Internet / Public Traffic ]
                              │
                    *.kimimueller.de :80/:443
                              ▼
        ┌──────────────────────────────────────────┐
        │               VPS Cluster                │
        │  ┌────────────────────────────────────┐  │
        │  │  Traefik (TCP / TLS Passthrough)   │  │
        │  └─────────────────┬──────────────────┘  │
        │                    ▼                     │
        │  ┌────────────────────────────────────┐  │
        │  │  Rathole Server (:2333, :8080/8443)│  │
        │  └─────────────────┬──────────────────┘  │
        └────────────────────┼─────────────────────┘
                             │ Rathole Tunnel
                             ▼
        ┌──────────────────────────────────────────┐
        │             Homelab Cluster              │
        │  ┌────────────────────────────────────┐  │
        │  │          Rathole Client            │  │
        │  └─────────────────┬──────────────────┘  │
        │                    ▼                     │
        │  ┌────────────────────────────────────┐  │
        │  │  Traefik (L7 Ingress Controller)   │◀─┼─ Local LAN DNS (*.homelab.home)
        │  └──────┬──────────────────────┬──────┘  │
        │         ▼                      ▼         │
        │  [ Homepage Pod ]       [ Jellyfin Pod ] │
        └──────────────────────────────────────────┘
```

For comprehensive network, routing, and DNS details, see [docs/networking.md](docs/networking.md).

---

## Directory Structure

```text
kimis-fluxcd/
├── clusters/                   # Cluster entry points & root Flux definitions
│   ├── homelab/                # Homelab (K3s on NixOS) flux-system, apps.yaml, infra
│   └── vps/                    # VPS (K3s on Ubuntu) flux-system, apps.yaml, infra
├── infrastructure/             # Foundational cluster services & storage
│   ├── homelab/                # NFS PersistentVolumes, PVCs, namespaces
│   └── vps/                    # VPS infrastructure dependencies
├── apps/                       # Workloads deployed per environment
│   ├── homelab/                # CoreDNS, Homepage, Jellyfin, Rathole Client
│   └── vps/                    # Rathole Server, Traefik IngressRouteTCP
├── docs/                       # Architecture documentation and operational runbooks
│   ├── networking.md           # Network topology, Rathole NAT traversal & DNS rules
│   ├── service-onboarding.md   # Step-by-step guide to expose and deploy services
│   └── bootstrap-and-ops.md    # Cluster bootstrap, manual secrets & troubleshooting
└── AGENTS.md                   # Memory, guidelines, and operational quirks for AI agents
```

---

## Operational Documentation

* [Cluster Bootstrap & Operations](docs/bootstrap-and-ops.md): K3s on NixOS/Ubuntu setup, Flux bootstrap commands, manual secrets, and runbooks.
* [Networking Architecture & NAT Traversal](docs/networking.md): Details on Rathole ports, Traefik dual-setup, and CoreDNS rewrites.
* [Adding a New Service](docs/service-onboarding.md): Step-by-step checklist to deploy an app with internal and/or public access.
* [Agent Guidelines & Operational Memory](AGENTS.md): Hard-won operational discoveries, non-root quirks, and rules for AI assistants.
