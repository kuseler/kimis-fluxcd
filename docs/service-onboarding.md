# Service Onboarding Runbook

This runbook outlines the exact steps required to deploy a new application or service in `kimis-fluxcd`, configure its internal local access, and optionally expose it to the public edge via Rathole and Traefik.

---

## Decision Matrix: Access Scope

Before writing manifests, determine where the service should be reachable:

1. **Internal-Only (Homelab LAN)**:
   * Accessible only inside the local network via `*.homelab.home` or `*.homelab.fritz.box`.
   * Requires only: Pod/Deployment, Service, and Homelab Traefik Ingress.
   * **No VPS or Rathole configuration required** (CoreDNS automatically intercepts the domain).

2. **Public (Internet via VPS)**:
   * Accessible worldwide via `<app>.kimimueller.de`.
   * Standard web traffic (HTTP 80 / HTTPS 443): Routed through existing Rathole HTTP/HTTPS tunnels; requires Ingress rules on Homelab Traefik.
   * Non-HTTP custom TCP/UDP ports (e.g. WireGuard, game servers): Requires a new dedicated Rathole tunnel service definition and VPS Service port mapping.

---

## Step 1: Create Application Manifests (`apps/homelab/<app>/`)

1. Create directory `apps/homelab/<service-name>/`.
2. Add your deployment or HelmRelease:
   * For Helm charts: create `sources.yaml` (HelmRepository) and `<service>.yaml` (HelmRelease).
   * For standard manifests: create `deployment.yaml`, `service.yaml`, and `ingress.yaml`.
3. In your `ingress.yaml` (or Helm chart `ingress` section), specify `ingressClassName: traefik` and define the required hostnames:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-app
  namespace: default
spec:
  ingressClassName: traefik
  rules:
    # Public domain (if public exposure is desired)
    - host: example.kimimueller.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-app
                port:
                  number: 8080
    # Internal LAN domains
    - host: example.homelab.home
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-app
                port:
                  number: 8080
    - host: example.homelab.fritz.box
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-app
                port:
                  number: 8080
```

4. Create `apps/homelab/<service-name>/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

5. Register the directory in `apps/homelab/kustomization.yaml`:
```yaml
resources:
  - coredns
  - rathole-client
  - homepage
  - media
  - <service-name>
```

---

## Step 2: Routing Custom TCP/UDP Ports (If Applicable)

> [!NOTE]
> If your service is a regular HTTP/HTTPS web application, Rathole already proxies all port 80/443 traffic to Homelab Traefik. You **do not** need to edit Rathole configs for web apps.

If your service uses a custom TCP or UDP port (like WireGuard or a dedicated protocol):

### 1. Server Configuration (`apps/vps/rathole/rathole-server.toml`)
Add a new service block:
```toml
[server.services.<service-name>]
bind_addr = "0.0.0.0:<unprivileged-port>"  # Must be > 1024 (e.g. 51820)
type = "udp"                               # Omit or set to "tcp" for TCP
```

### 2. VPS Service Exposure (`apps/vps/rathole/service-external.yaml` or `service.yaml`)
Expose the port on the VPS LoadBalancer/NodePort service so external traffic reaches Rathole.

### 3. Client Configuration (`apps/homelab/rathole-client/rathole-client.toml`)
Configure the client to route forwarded packets to the local target:
```toml
[client.services.<service-name>]
local_addr = "<target-internal-ip-or-dns>:<target-port>"
type = "udp"                               # MUST match server type
```

---

## Step 3: Verification & Reconciliation

1. Validate Kustomize builds locally:
```bash
kustomize build apps/homelab/<service-name>
```
2. Commit and push your changes to `main`.
3. Check Flux status or force an immediate sync:
```bash
flux reconcile kustomization apps --with-source
```
4. Verify DNS and ingress response:
```bash
# Verify internal resolution:
dig @192.168.1.80 <service-name>.homelab.home

# Test HTTP access:
curl -Iv http://<service-name>.homelab.home
```
