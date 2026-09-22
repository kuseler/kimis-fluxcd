# Authelia Forward-Auth & SSO Architecture

This document provides a comprehensive overview of the **Authelia** Forward-Auth and Single Sign-On (SSO) integration in `kimis-fluxcd`, including traffic routing, access control policies, secret setup, and service onboarding.

---

## 1. Architectural Overview & Ingress Routing (Option B)

Traffic entering from the internet passes through the VPS and the Rathole tunnel directly to Traefik on the Homelab cluster. 

We employ **Option B (Selective Traefik Middleware via Dedicated Ingress Resources)** as our best-practice architecture:

* **Single Ingress Controller**: Homelab uses the existing k3s Traefik instance in `kube-system`. We do not deploy a second redundant ingress controller.
* **Separation of Concerns**: Services split their ingress definitions into:
  1. **Remote Ingress (`*.kimimueller.de`)**: Configured with entrypoint `websecure` (port 443) and annotated with the Traefik ForwardAuth middleware (`authelia-authelia-forwardauth@kubernetescrd`).
  2. **Local Ingress (`*.homelab.home`, `*.homelab.fritz.box`)**: Configured without any auth middleware annotations.
* **Zero Overhead & Total Bypass for LAN/VPN**:
  * When accessed locally via `*.homelab.home`, Traefik matches the local Ingress router which does not have the middleware attached. The request goes straight to the backend pod without any subrequest to Authelia.
  * For defense-in-depth, Authelia's `access_control` configuration also designates the local LAN (`192.168.1.0/24`) and Kubernetes network (`10.42.0.0/16`, `10.43.0.0/16`) with `policy: bypass`.

```
[ Internet Client ]
       │  (e.g., https://whoami.kimimueller.de)
       ▼
[ VPS Traefik ] ──(TCP Passthrough)──► [ Rathole Tunnel ]
                                              │
                                              ▼
                                    [ Homelab Traefik ]
                                              │
                     ┌────────────────────────┴────────────────────────┐
                     │ (Matches Remote Ingress: whoami.kimimueller.de) │
                     ▼                                                 ▼
        [ Traefik ForwardAuth Middleware ]                   [ Matches Local Ingress: ]
                     │                                       [ whoami.homelab.home    ]
                     │ (Subrequest: /api/authz/forward-auth)           │
                     ▼                                                 │ (Direct / Bypassed)
               [ Authelia ]                                            │
                     │                                                 │
          ┌──────────┴──────────┐                                      │
     (Valid Session)       (No Session)                                │
          │                     │                                      │
          ▼                     ▼                                      ▼
     [ Target Pod ]      [ 302 Redirect to ]                     [ Target Pod ]
                     [ auth.kimimueller.de ]
```

---

## 2. Single Sign-On (SSO) Scope

Authelia manages sessions via root domain cookies:
* **Cookie Domain**: `kimimueller.de`
* **Cookie Name**: `authelia_session`
* **Portal URL**: `https://auth.kimimueller.de`
* **Session Lifecycle**: 1-hour inactivity timeout, 1-month remember-me duration.

Once a user logs into `auth.kimimueller.de`, the session cookie is valid for all subdomains (`*.kimimueller.de`). Visiting any other protected service (e.g. `homepage.kimimueller.de`, `whoami.kimimueller.de`) reuses the existing session seamlessly without re-prompting for credentials.

---

## 3. Secret Management & Cluster Bootstrap

Authelia requires several cryptographic secrets and a user database file. In accordance with this repository's security guidelines, production secrets are not committed unencrypted to Git.

### 3.1 Secret Contents

The `authelia-secrets` secret in namespace `authelia` contains four keys:

| Secret Key | Description | Format / Requirement |
| :--- | :--- | :--- |
| `identity_validation.reset_password.jwt.hmac.key` | JWT signing key for password resets | Random 32+ byte string (hex) |
| `session.encryption.key` | Encryption key for session cookies | Random 32+ byte string (hex) |
| `storage.encryption.key` | Encryption key for SQLite database | Random 32+ byte string (hex) |
| `users_database.yml` | Local file-based user accounts & groups | YAML file with Argon2id password hashes |

### 3.2 Manual Secret Creation Runbook

Run the following commands on your administrative workstation connected to the Homelab cluster:

```bash
# 1. Ensure the authelia namespace exists
kubectl create namespace authelia --dry-run=client -o yaml | kubectl apply -f -

# 2. Generate secure random secrets
JWT_SECRET=$(openssl rand -hex 32)
SESSION_KEY=$(openssl rand -hex 32)
STORAGE_KEY=$(openssl rand -hex 32)

# 3. Generate password hash for your user (e.g. using docker)
# Replace 'YourStrongPasswordHere' with your actual password:
HASHED_PASSWORD=$(docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password "YourStrongPasswordHere" | awk '{print $NF}')

# 4. Create a local users_database.yml file:
cat <<EOF > /tmp/users_database.yml
users:
  kimi:
    disabled: false
    displayname: "Kimi Müller"
    password: "${HASHED_PASSWORD}"
    email: "kimi@kimimueller.de"
    groups:
      - admins
      - dev
EOF

# 5. Apply the secret to the cluster
kubectl -n authelia create secret generic authelia-secrets \
  --from-literal=identity_validation.reset_password.jwt.hmac.key="${JWT_SECRET}" \
  --from-literal=session.encryption.key="${SESSION_KEY}" \
  --from-literal=storage.encryption.key="${STORAGE_KEY}" \
  --from-file=users_database.yml=/tmp/users_database.yml

# 6. Clean up temporary local file
rm -f /tmp/users_database.yml
```

### 3.3 Optional: GitOps Encryption with SOPS

If you wish to store `authelia-secrets` encrypted in Git:
1. Copy `apps/homelab/authelia/secrets.dummy.yaml` to `apps/homelab/authelia/secrets.enc.yaml`.
2. Fill in real values.
3. Encrypt using `sops`:
   ```bash
   sops --encrypt --age <YOUR_AGE_RECIPIENT> apps/homelab/authelia/secrets.enc.yaml > apps/homelab/authelia/secrets.yaml
   ```
4. Add `secrets.yaml` to `apps/homelab/authelia/kustomization.yaml` and ensure the Flux Kustomization in `clusters/homelab/apps.yaml` has `spec.decryption.provider: sops`.

---

## 4. How to Protect a Service with Authelia

To protect an application with Authelia, define two separate `Ingress` manifests for your service:

### 1. Remote Ingress (`ingress-remote.yaml`)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-remote
  namespace: myapp
  annotations:
    traefik.ingress.kubernetes.io/router.entryPoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Attach Authelia ForwardAuth from the 'authelia' namespace:
    traefik.ingress.kubernetes.io/router.middlewares: authelia-authelia-forwardauth@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: myapp.kimimueller.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

### 2. Local Ingress (`ingress-local.yaml`)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-local
  namespace: myapp
  annotations:
    traefik.ingress.kubernetes.io/router.entryPoints: web,websecure
spec:
  ingressClassName: traefik
  rules:
    - host: myapp.homelab.home
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
    - host: myapp.homelab.fritz.box
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

---

## 5. Reference Implementation

A fully functional example demonstrating this dual-ingress pattern is available at:
* Manifests: `apps/homelab/authelia/example/` (`deployment.yaml`, `ingress-remote.yaml`, `ingress-local.yaml`).
