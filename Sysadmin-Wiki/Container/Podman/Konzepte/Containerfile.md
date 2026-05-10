---
tags: [podman, containerfile, dockerfile, build, oci, image, buildah, containerignore]
aliases: ["Containerfile", "Dockerfile Podman", "podman build"]
---

# Containerfile

> [!ABSTRACT]
> Ein **Containerfile** ist das Podman-Äquivalent zum Dockerfile — die Syntax ist identisch (OCI-Standard). Podman akzeptiert beide Dateinamen (`Containerfile` und `Dockerfile`). Builds werden von **Buildah** durchgeführt, das in Podman integriert ist. Kein Docker-Daemon nötig.

---

## Containerfile vs. Dockerfile

| Merkmal | Containerfile | Dockerfile |
|---------|:------------:|:---------:|
| Syntax | Identisch (OCI-Standard) | Identisch |
| Build-Tool | Buildah (in Podman integriert) | Docker BuildKit |
| Daemon nötig | ❌ Nein | ✅ Ja |
| Rootless Build | ✅ Ja | ❌ Nein (ohne Umwege) |
| Dateiname | `Containerfile` (bevorzugt) oder `Dockerfile` | `Dockerfile` |
| Registry | Jede OCI-kompatible | Jede OCI-kompatible |

---

## Grundstruktur

```dockerfile
# Containerfile — Kommentare mit #

# 1. Basis-Image
FROM docker.io/library/node:20-alpine

# 2. Metadaten
LABEL maintainer="name@beispiel.de"
LABEL version="1.0"

# 3. Umgebungsvariablen
ENV NODE_ENV=production \
    APP_PORT=3000

# 4. Arbeitsverzeichnis
WORKDIR /app

# 5. Dateien kopieren
COPY package*.json ./
RUN npm ci --only=production

COPY . .

# 6. Port deklarieren (Dokumentation)
EXPOSE 3000

# 7. Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# 8. Startbenutzer
USER node

# 9. Startbefehl
CMD ["node", "server.js"]
```

---

## Alle Anweisungen im Überblick

### FROM — Basis-Image

```dockerfile
# Einfach
FROM docker.io/library/alpine:3.19

# Mit Alias (für Multi-Stage)
FROM docker.io/library/node:20-alpine AS builder

# Scratch (leeres Image für statische Binaries)
FROM scratch
```

> [!TIP] Immer vollständige Registry-Pfade
> `FROM nginx` löst in Podman einen interaktiven Prompt aus. Immer `FROM docker.io/library/nginx:alpine` schreiben.

### RUN — Befehl während Build

```dockerfile
# Shell-Form
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    wget \
 && rm -rf /var/lib/apt/lists/*

# Exec-Form (kein Shell-Parsing)
RUN ["apt-get", "install", "-y", "curl"]

# Mehrere Befehle in einem Layer (weniger Layers = kleineres Image)
RUN dnf install -y nginx \
 && dnf clean all \
 && rm -rf /var/cache/dnf
```

### COPY — Dateien kopieren

```dockerfile
# Einfach
COPY src/ /app/src/

# Besitzer setzen (ab OCI-Spec)
COPY --chown=node:node . /app/

# Aus anderem Build-Stage (Multi-Stage)
COPY --from=builder /app/dist /app/dist

# Einzelne Datei
COPY config.yaml /etc/myapp/config.yaml
```

### ADD — Erweitert (selten nötig)

```dockerfile
# URL herunterladen
ADD https://example.com/app.tar.gz /tmp/

# Tar automatisch entpacken
ADD app.tar.gz /app/

# Tipp: COPY bevorzugen — ADD hat Überraschungseffekte
```

### ENV — Umgebungsvariablen

```dockerfile
ENV NODE_ENV=production
ENV PATH="/app/bin:${PATH}"

# Mehrere auf einmal
ENV APP_PORT=3000 \
    LOG_LEVEL=info \
    TZ=Europe/Berlin
```

### ARG — Build-Argumente

```dockerfile
ARG NODE_VERSION=20
FROM docker.io/library/node:${NODE_VERSION}-alpine

ARG APP_VERSION=1.0.0
LABEL version=${APP_VERSION}
```

```bash
# Beim Build übergeben
podman build --build-arg NODE_VERSION=22 --build-arg APP_VERSION=2.0 .
```

### WORKDIR, USER, EXPOSE

```dockerfile
WORKDIR /app                    # Arbeitsverzeichnis setzen/erstellen

USER 1001                       # Als UID 1001 ausführen (bevorzugt für rootless)
USER appuser                    # oder nach Benutzername

EXPOSE 3000                     # Dokumentiert, mappt keinen Port!
EXPOSE 3000/tcp
EXPOSE 53/udp
```

### CMD und ENTRYPOINT

```dockerfile
# CMD — Standardbefehl (überschreibbar mit podman run ... CMD)
CMD ["node", "server.js"]           # Exec-Form (empfohlen)
CMD node server.js                  # Shell-Form

# ENTRYPOINT — Hauptprogramm (nur Args überschreibbar)
ENTRYPOINT ["/usr/bin/nginx"]
CMD ["-g", "daemon off;"]

# Kombination: ENTRYPOINT + CMD
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]
```

### VOLUME, HEALTHCHECK

```dockerfile
# Volume-Deklaration (Dokumentation + Standardvolume)
VOLUME ["/data", "/logs"]

# Healthcheck
HEALTHCHECK --interval=30s \
            --timeout=5s \
            --start-period=15s \
            --retries=3 \
  CMD curl -sf http://localhost:8080/health || exit 1

# Healthcheck deaktivieren (z.B. im Basis-Image)
HEALTHCHECK NONE
```

### ONBUILD, STOPSIGNAL, SHELL

```dockerfile
# ONBUILD — wird ausgeführt wenn dieses Image als Basis genutzt wird
ONBUILD COPY . /app
ONBUILD RUN npm install

# STOPSIGNAL — Signal zum Stoppen
STOPSIGNAL SIGTERM

# SHELL — Standard-Shell für RUN
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
```

---

## .containerignore

```
# .containerignore — entspricht .dockerignore
node_modules/
.git/
.gitignore
*.log
*.tmp
dist/
.env
.env.*
tests/
docs/
README.md
Containerfile
.containerignore
```

---

## podman build — Wichtige Flags

```bash
# Einfacher Build
podman build -t myapp:1.0 .

# Mit Containerfile an anderem Ort
podman build -f /pfad/Containerfile -t myapp:1.0 .

# Mehrstufiger Build — nur bis Stage bauen
podman build --target builder -t myapp:builder .

# Build-Argumente
podman build \
  --build-arg VERSION=2.0 \
  --build-arg ENV=production \
  -t myapp:2.0 .

# Kein Cache verwenden
podman build --no-cache -t myapp:fresh .

# Cache von Registry laden
podman build \
  --cache-from docker.io/myrepo/myapp:cache \
  -t myapp:latest .

# Mehrere Plattformen (mit Manifest)
podman build \
  --platform linux/amd64,linux/arm64 \
  --manifest myapp:multi-arch .

# Labels hinzufügen
podman build \
  --label "build-date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
  --label "git-commit=$(git rev-parse --short HEAD)" \
  -t myapp:latest .

# Rootless Build (Standard in Podman!)
podman build -t myapp:latest .
# Kein sudo nötig!

# Verbose Build-Output
podman build --progress=plain -t myapp:latest .
```

---

## Best Practices für Containerfiles

### 1. Minimale Basis-Images

```dockerfile
# Schlecht — zu groß (900MB+)
FROM docker.io/library/ubuntu:latest

# Besser — minimal
FROM docker.io/library/alpine:3.19

# Noch besser — distroless für Produktion
FROM gcr.io/distroless/nodejs20-debian12
```

### 2. Layer minimieren

```dockerfile
# Schlecht — 3 Layer
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Gut — 1 Layer
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

### 3. Build-Cache optimieren (häufig ändernde Dateien zuletzt)

```dockerfile
# Gut — Dependencies zuerst (ändert sich selten)
COPY package*.json ./
RUN npm ci

# Danach Sourcecode (ändert sich oft)
COPY src/ ./src/
```

### 4. Nicht als Root laufen

```dockerfile
# User erstellen und nutzen
RUN adduser -D -u 1001 appuser
USER 1001

# oder mit vorhandener UID (für rootless-Kompatibilität)
USER 65532
```

### 5. Geheimnisse nie im Image speichern

```dockerfile
# FALSCH — Passwort landet im Image!
RUN curl -u user:passwort https://intern.firma.de/download

# RICHTIG — Build-Secret (BuildKit/Buildah)
# --mount=type=secret
RUN --mount=type=secret,id=mypassword \
    curl -u user:$(cat /run/secrets/mypassword) https://intern.firma.de/download
```

---

## Buildah — das Build-Backend

Podman nutzt Buildah intern für alle `podman build`-Operationen:

```bash
# Buildah direkt nutzen (für fortgeschrittene Builds)
buildah from docker.io/library/alpine:3.19
buildah run <container> -- apk add nginx
buildah copy <container> ./html /usr/share/nginx/html
buildah config --entrypoint '["nginx", "-g", "daemon off;"]' <container>
buildah commit <container> myapp:latest

# Image von Containerfile bauen
buildah build -t myapp:latest .

# Rootless (kein sudo nötig)
buildah bud --layers -t myapp:cached .
```

---

## Verwandte Notizen

- [[Containerfile Beispiele]] — Praxisbeispiele für verschiedene Sprachen
- [[Multi-Stage Builds]] — Mehrstufige Builds für kleinere Images
- [[Container und Images]] — Image-Management mit Podman
- [[Podman Registries]] — Images in Registries pushen
