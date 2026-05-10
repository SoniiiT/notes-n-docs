---
tags: [podman, multi-stage, build, image-size, buildah, cache, optimierung]
aliases: ["Multi-Stage Builds", "mehrstufige Builds", "Podman Build Optimierung"]
---

# Multi-Stage Builds

> [!ABSTRACT]
> Multi-Stage Builds trennen **Build-Umgebung** (mit Compiler, Dependencies, Dev-Tools) von der **Laufzeit-Umgebung** (nur das Minimum). Ergebnis: Images die 10x–100x kleiner sind. Podman unterstützt Multi-Stage-Builds nativ — kein BuildKit nötig.

---

## Warum Multi-Stage?

```
Ohne Multi-Stage:     Go-Image mit Compiler = ~1.2 GB
Mit Multi-Stage:      Scratch + Binary       = ~12 MB

Ohne Multi-Stage:     Node mit node_modules  = ~800 MB
Mit Multi-Stage:      Alpine + dist + prod-deps = ~120 MB
```

---

## Grundprinzip

```dockerfile
# Stage 1: Build (groß, viele Tools)
FROM docker.io/library/golang:1.22 AS builder
WORKDIR /build
COPY . .
RUN go build -o app .

# Stage 2: Laufzeit (minimal, nur Binary)
FROM docker.io/library/alpine:3.19
COPY --from=builder /build/app /usr/local/bin/app
CMD ["app"]
```

**Schlüssel:** `COPY --from=builder` kopiert nur was aus der vorherigen Stage gebraucht wird.

---

## Stages benennen und referenzieren

```dockerfile
# Stages per Name referenzieren
FROM docker.io/library/node:20-alpine AS deps
FROM docker.io/library/node:20-alpine AS builder
FROM docker.io/library/node:20-alpine AS tester
FROM docker.io/library/node:20-alpine AS production

# Per Index referenzieren (0, 1, 2, ...)
COPY --from=0 /app/dist ./dist      # Stage 0
COPY --from=builder /app/dist ./dist # Stage "builder" (klarer)
```

---

## Beispiel: Node.js (3 Stages)

```dockerfile
# Stage 1: Dependencies installieren
FROM docker.io/library/node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Build (TypeScript kompilieren)
FROM docker.io/library/node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # inkl. devDependencies
COPY . .
RUN npm run build && npm run test

# Stage 3: Produktion (minimal)
FROM docker.io/library/node:20-alpine AS production
WORKDIR /app

# Nur Produktions-Dependencies + Build-Output
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

RUN adduser -D -u 1001 nodeuser
USER 1001
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

```bash
# Größenvergleich
podman build --target deps -t myapp:deps .
podman build --target builder -t myapp:builder .
podman build -t myapp:prod .

podman image ls myapp
# myapp:deps      ~180 MB
# myapp:builder   ~500 MB
# myapp:prod      ~120 MB
```

---

## Beispiel: Python mit Tests-Stage

```dockerfile
FROM docker.io/library/python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Test-Stage
FROM base AS test
COPY requirements-dev.txt .
RUN pip install --no-cache-dir -r requirements-dev.txt
COPY . .
RUN pytest tests/ -v --tb=short

# Produktions-Stage
FROM base AS production
COPY --from=base /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY src/ ./src/
RUN useradd -u 1001 appuser
USER 1001
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# Nur bis Test-Stage bauen (CI: Fehler bei Test-Fehler)
podman build --target test -t myapp:test .

# Vollständiger Produktions-Build
podman build -t myapp:prod .
```

---

## Nur bestimmte Stage bauen

```bash
# --target stoppt Build nach der angegebenen Stage
podman build --target builder -t myapp:builder-only .

# Nützlich für:
# - CI: nur Tests bauen und laufen lassen
# - Debugging: in builder-Stage reinschauen
# - Caching: builder-Stage als Cache für spätere Builds

# In builder-Stage debuggen
podman run --rm -it myapp:builder-only sh
```

---

## Build-Cache nutzen

```bash
# Cache-Image als Basis verwenden (CI-Optimierung)
podman build \
  --cache-from docker.io/myrepo/myapp:cache \
  --cache-to docker.io/myrepo/myapp:cache \
  -t myapp:latest .

# Layers cachen (lokal)
podman build --layers -t myapp:latest .

# Cache vollständig überspringen
podman build --no-cache -t myapp:fresh .
```

---

## Build-Secrets in Multi-Stage

```dockerfile
# Secrets sicher in Build nutzen (werden nicht ins Image geschrieben!)
FROM docker.io/library/alpine AS downloader
RUN --mount=type=secret,id=token \
    TOKEN=$(cat /run/secrets/token) \
    wget --header="Authorization: Bearer $TOKEN" \
         https://intern.firma.de/package.tar.gz \
    -O /tmp/package.tar.gz

FROM docker.io/library/alpine
COPY --from=downloader /tmp/package.tar.gz /app/
```

```bash
# Secret übergeben (wird nicht gecacht, nicht ins Image geschrieben)
podman build \
  --secret id=token,src=$HOME/.tokens/intern-token \
  -t myapp:latest .
```

---

## Buildah für komplexe Builds

Bei sehr komplexen Build-Logiken kann Buildah (das Build-Backend von Podman) direkt in Skripten genutzt werden:

```bash
#!/bin/bash
# build.sh — Buildah-Skript als Alternative zu komplexen Containerfiles

# Stage 1: Builder
BUILDER=$(buildah from docker.io/library/golang:1.22-alpine)
buildah run $BUILDER -- go build -o /app ./...

# Stage 2: Final
FINAL=$(buildah from docker.io/library/alpine:3.19)
buildah copy --from $BUILDER $FINAL /app /usr/local/bin/app
buildah config --cmd '["app"]' $FINAL

# Commit
buildah commit $FINAL myapp:latest

# Aufräumen
buildah rm $BUILDER $FINAL

echo "Build abgeschlossen: myapp:latest"
podman image ls myapp
```

---

## Typische Image-Größen im Vergleich

| Ansatz | Beispiel | Größe |
|--------|----------|-------|
| Ohne Multi-Stage (Go) | golang:1.22 | ~1,2 GB |
| Mit Alpine (Go) | alpine + binary | ~20 MB |
| Scratch (Go, statisch) | scratch + binary | ~10 MB |
| Ohne Multi-Stage (Node) | node:20 | ~800 MB |
| Mit Alpine + prod-deps | node:20-alpine | ~150 MB |
| Distroless (Python) | distroless/python3 | ~60 MB |

---

## Verwandte Notizen

- [[Containerfile]] — Containerfile-Syntax Grundlagen
- [[Containerfile Beispiele]] — Vollständige Beispiele je Sprache
- [[Container und Images]] — Image-Verwaltung
- [[Podman Registries]] — Fertige Images veröffentlichen
