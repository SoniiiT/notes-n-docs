---
tags: [podman, images, container, oci, registry, buildah, dockerfile, skopeo]
aliases: ["Podman Images", "Podman Container", "OCI Images"]
---

# Container und Images

> [!ABSTRACT]
> Podman nutzt OCI-konforme Images — binär kompatibel mit Docker. Images werden aus Registries wie docker.io, quay.io oder ghcr.io bezogen. `podman build` baut Images intern über Buildah, `skopeo` ermöglicht Image-Operationen ohne Container-Runtime.

---

## OCI-Images — das Format

OCI (Open Container Initiative) Images sind der offene Standard für Container-Images. Podman, Docker, Buildah und containerd nutzen alle dasselbe Format — Images sind vollständig austauschbar.

**Aufbau eines OCI-Images:**

```
Image
├── config.json       ← Metadaten (Entrypoint, Env, Labels, ...)
└── layers/           ← Lese-nur-Layer (OverlayFS)
    ├── layer1.tar    ← Basis-OS (z.B. Ubuntu 24.04)
    ├── layer2.tar    ← Installierte Pakete
    └── layer3.tar    ← Applikations-Code
```

Jeder Layer ist ein Diff zum vorherigen. Beim Container-Start fügt die Runtime einen beschreibbaren Container-Layer oben drauf.

---

## Registries — woher kommen Images?

| Registry | URL | Schwerpunkt |
|----------|-----|-------------|
| **Docker Hub** | `docker.io` | Größte Sammlung, viele offizielle Images |
| **Quay.io** | `quay.io` | Red Hat / Open Source Fokus, automatische Security-Scans |
| **GitHub Container Registry** | `ghcr.io` | GitHub-Projekte, CI/CD-Integration |
| **Red Hat Official** | `registry.access.redhat.com` | RHEL-geprüfte Images |
| **Fedora** | `registry.fedoraproject.org` | Fedora-Images |
| **Google** | `gcr.io` | Google Cloud Images |

### Short-Name Auflösung

```bash
# Vollständiger Name (immer empfohlen)
podman pull docker.io/library/nginx:latest

# Short Name — Podman fragt interaktiv welche Registry
podman pull nginx
# ? Please select an image:
#   ▸ registry.fedoraproject.org/nginx:latest
#     registry.access.redhat.com/nginx:latest
#     docker.io/library/nginx:latest
```

```toml
# /etc/containers/registries.conf — Short-Name-Konfiguration
unqualified-search-registries = ["docker.io"]

# Short-Name-Alias definieren (kein interaktiver Prompt)
[[aliases]]
  "nginx" = "docker.io/library/nginx"
  "postgres" = "docker.io/library/postgres"
```

> [!TIP] In Scripts immer vollständige Namen
> `podman pull docker.io/library/nginx:latest` statt `podman pull nginx` — verhindert interaktive Prompts in CI/CD.

---

## Images verwalten

```bash
# Image pullen
podman pull docker.io/library/nginx:latest
podman pull quay.io/fedora/fedora:40

# Lokale Images anzeigen
podman images
podman image ls

# Image-Details
podman image inspect nginx:latest
podman image inspect --format "{{.Architecture}}" nginx:latest

# Image-History (Layer-Aufbau)
podman image history nginx:latest

# Image löschen
podman rmi nginx:latest

# Alle ungenutzten Images löschen
podman image prune
podman image prune -a   # auch nicht getaggte Images
```

---

## Container starten und verwalten

```bash
# Container im Vordergrund
podman run nginx:latest

# Container im Hintergrund (-d = detached)
podman run -d --name webserver -p 8080:80 nginx:latest

# Interaktive Shell
podman run -it --rm ubuntu:24.04 bash

# Mit Umgebungsvariablen
podman run -d \
  --name db \
  -e POSTGRES_PASSWORD=geheim \
  -e POSTGRES_DB=myapp \
  -v pgdata:/var/lib/postgresql/data:Z \
  postgres:16

# Laufende Container
podman ps

# Alle Container (auch gestoppt)
podman ps -a

# Container stoppen / starten / neustarten
podman stop webserver
podman start webserver
podman restart webserver

# Container löschen
podman rm webserver
podman rm -f webserver   # erzwingen (auch wenn laufend)
```

---

## In Container einsteigen und debuggen

```bash
# Shell in laufendem Container
podman exec -it webserver bash
podman exec -it webserver sh   # wenn kein bash vorhanden

# Befehl ausführen
podman exec webserver nginx -t   # nginx config testen

# Prozesse im Container
podman top webserver

# Dateien aus/in Container kopieren
podman cp webserver:/etc/nginx/nginx.conf ./nginx.conf
podman cp ./nginx.conf webserver:/etc/nginx/nginx.conf

# Änderungen seit Image-Start
podman diff webserver
```

---

## Images bauen — Containerfile / Dockerfile

Podman nutzt `buildah` intern — die Syntax ist identisch mit Docker.

### Einfaches Containerfile

```dockerfile
# Containerfile (oder Dockerfile — beide werden erkannt)
FROM docker.io/library/nginx:alpine

LABEL maintainer="admin@example.com"
LABEL version="1.0"

# Konfiguration kopieren
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/

# Port dokumentieren
EXPOSE 80

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

```bash
# Image bauen
podman build -t myapp:latest .
podman build -t myapp:1.0 -f Containerfile .

# Ohne Cache (sauberer Build)
podman build --no-cache -t myapp:latest .

# Squash: Alle Layer zu einem zusammenfassen
podman build --squash -t myapp:latest .

# Multi-Arch Build (via podman farm)
podman build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

### Containerfile vs. Dockerfile

Beide Dateinamen werden unterstützt — Podman bevorzugt `Containerfile` (OCI-Standard), Docker bevorzugt `Dockerfile`. Die Syntax ist identisch.

---

## Images exportieren und importieren

```bash
# Image als TAR-Archiv speichern
podman save -o nginx.tar docker.io/library/nginx:latest

# Mehrere Images in einem Archiv
podman save -o images.tar nginx:latest myapp:1.0

# TAR-Archiv laden
podman load -i nginx.tar

# Container-Filesystem exportieren (ohne Metadaten)
podman export webserver -o container.tar

# Aus TAR importieren (erstellt ein Image ohne Layer-History)
podman import container.tar myapp:imported
```

---

## Image Tags und Registry-Operationen

```bash
# Tag setzen
podman tag myapp:latest myapp:1.0
podman tag myapp:latest quay.io/meinuser/myapp:latest

# In Registry pushen
podman login quay.io   # einmalig authentifizieren
podman push quay.io/meinuser/myapp:latest

# Aus privater Registry pullen
podman login ghcr.io --username meinuser --password-stdin < token.txt
podman pull ghcr.io/meinorg/myapp:latest

# Login-Status
podman login --get-login docker.io
```

---

## Skopeo — Images ohne Container-Runtime

Skopeo ist ideal für Image-Operationen in CI/CD-Pipelines oder ohne installierte Container-Runtime.

```bash
# Image inspizieren ohne zu pullen
skopeo inspect docker://docker.io/library/nginx:latest
skopeo inspect --format "{{.Digest}}" docker://nginx:latest

# Zwischen Registries kopieren (kein lokales Speichern)
skopeo copy \
  docker://docker.io/library/nginx:latest \
  docker://quay.io/meinuser/nginx:latest

# Alle Tags einer Registry auflisten
skopeo list-tags docker://docker.io/library/nginx

# Image lokal als OCI-Directory speichern
skopeo copy docker://nginx:latest oci:./nginx-oci:latest

# Image in TAR kopieren
skopeo copy docker://nginx:latest docker-archive:./nginx.tar
```

---

## Image-Sicherheit

```bash
# Lokale Image-Signatur-Policy anzeigen
cat /etc/containers/policy.json

# Image-Trust konfigurieren
podman image trust show

# Image mit cosign verifizieren (Sigstore)
cosign verify --certificate-identity-regexp=.* \
  --certificate-oidc-issuer-regexp=.* \
  quay.io/meinuser/myapp:latest
```

> [!WARNING] Nur vertrauenswürdige Images
> Images aus docker.io können von jedem hochgeladen werden. Für Produktion empfohlen:
> - `registry.access.redhat.com` — Red Hat geprüft
> - `quay.io` — Automatische Vulnerability-Scans
> - Eigene private Registry mit `podman image trust`

---

## Verwandte Notizen

- [[Podman CLI Cheatsheet]] — Vollständige Befehlsreferenz
- [[Podman Sicherheit]] — Image-Signierung und Security-Scanning
- [[Volumes und Storage]] — Persistente Daten in Containern
- [[Podman Index]] — Übersicht aller Podman-Themen
