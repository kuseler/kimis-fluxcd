
Ich betreibe ein k3s-Cluster in meinem lokalen Homelab, das über FluxCD (GitOps) verwaltet wird. 
Bestimmte Services werden über einen 'rathole'-Tunnel über einen Remote-Server/VPS öffentlich exponiert, während das restliche Cluster nur im LAN/VPN erreichbar bleiben soll.

Erstelle mir ein vollständiges, produktionsreifes FluxCD-Setup für Authelia als Forward-Auth-Provider mit folgenden Bedingungen:

1. Bereitstellungsziel (Scope):
- Authelia soll via HelmRelease (offizielles Chart) durch FluxCD verwaltet werden.
- WICHTIG: Authelia soll NUR für Ingress-Ressourcen greifen, die über den rathole-Tunnel remote exponiert sind.
- Lokale Zugriffe (z. B. per LAN-Subnetz/VPN) dürfen NICHT durch Authelia blockiert werden.

2. Architektur & Routing:
- Zeige die empfohlene Ingress-Architektur für Traefik oder Ingress-NGINX:
  - Option A: Zwei Ingress-Controller / IngressClasses (eine für lokal, eine für remote/rathole mit globalem Forward-Auth Middleware-Hook).
  - Option B: Traefik Middleware / Ingress-Annotationen (`auth-url` / `forward-auth`), die gezielt nur an den remote-exponierten Ingress-Ressourcen angehängt werden.
  - Gib mir die sauberere Best-Practice-Variante (bevorzugt Traefik Middleware oder dedizierte Ingress-Annotation).

3. Konkrete Manifeste (GitOps-Struktur):
- `HelmRepository` und `HelmRelease` für Authelia (minimales, funktionales Setup mit lokaler SQLite/File-Storage und `users_database.yml` als Secret).
- Definition der Forward-Auth-Middleware (z. B. Traefik `Middleware` CRD oder NGINX External Auth Snippet).
- Beispiel für einen remote-exponierten Ingress (mit Authelia-Schutz).
- Beispiel für denselben Service als lokaler Ingress (ohne Authelia-Schutz / bypassed).

4. Secrets & Security:
- Platziere Dummy-Werte für JWT Secret, Session Secret und Encryption Key, markiere aber klar, wie diese für SOPS / SealedSecrets vorbereitet werden sollten.
