# Rathole Deployment Notes

This directory contains the deployment configuration for the `rathole` server, which acts as a reverse proxy for NAT traversal, allowing services in the homelab to be exposed via the VPS.

## Known Quirks and Fixes

During the initial setup, a few issues were encountered and resolved. This documentation serves as a reference for future maintenance.

### 1. Non-Root Container Port Binding
**Symptom:** The client connects successfully to the control channel but immediately drops connections with an `early eof` error when attempting to establish data channels.
**Cause:** The official `rapiz1/rathole` Docker image runs as a non-root user (UID/GID 1000:1000) by default for security reasons. Linux prevents non-root users from binding to privileged ports (ports under 1024, like 80 and 443). When the server attempts to bind to these ports, it silently crashes the listener tasks.
**Fix:** The `bind_addr` for HTTP and HTTPS services in the server's `configmap.yaml` must be set to unprivileged ports (e.g., `8080` and `8443`). The Kubernetes `Service` (`service.yaml`) then maps incoming traffic on standard ports (80/443) to these `targetPort`s (8080/8443).

### 2. UDP Service Configuration (e.g., WireGuard)
**Symptom:** The client logs show the warning: `Failed to run the data channel: Expect UDP traffic. Please check the configuration.`
**Cause:** By default, `rathole` assumes services are TCP. If a service is configured as UDP on the server, the client must also explicitly declare it as UDP.
**Fix:** Ensure that any UDP service (like `fritzbox-wg`) includes `type = "udp"` in the client's configuration block (`[client.services.<name>]`). Note that the server configuration also requires this `type = "udp"` declaration.
