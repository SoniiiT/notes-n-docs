---
tags:
  - docker
  - multi-stage
  - dockerfile
  - build
  - optimierung
---

# Multi-Stage Builds

> [!ABSTRACT]
> Multi-Stage Builds ermöglichen es, mehrere `FROM`-Anweisungen in einem einzigen Dockerfile zu verwenden. Dadurch werden Build-Werkzeuge, Compiler und Dev-Dependencies sauber vom finalen Image getrennt — das Ergebnis sind schlanke, sichere Production-Images ohne unnötigen Ballast.

---

## Das Problem: Aufgeblähte Docker Images

Ein klassisches Dockerfile für eine kompilierte Anwendung sieht oft so aus:

```dockerfile
FROM golang:1.24
WORKDIR /app
COPY . .
RUN go mod download
RUN go build -o /app/server .
CMD ["/app/server"]
```

Das funktioniert — aber das resultierende Image enthält **alles**, was zum Bauen nötig war:

- Den kompletten **Go-Compiler** (~500 MB)
- Sämtliche **Build-Tools** (gcc, make, git)
- Den **Quellcode** und die Module
- **Caches** und temporäre Build-Artefakte

> [!WARNING] Das Ergebnis
> Ein Go-Binary von 10 MB steckt in einem Image von **1.2 GB**. Das ist Verschwendung von Speicher, Bandbreite und vergrößert die Angriffsfläche.

Dasselbe Problem tritt bei praktisch jeder Sprache auf:

| Sprache | Build-Image enthält | Davon tatsächlich benötigt |
|---------|---------------------|---------------------------|
| Go | Compiler, Toolchain, Source | Statisches Binary |
| Node.js | npm, devDependencies, Source | Build-Output (HTML/JS/CSS) |
| Java | JDK, Maven/Gradle, Source | JAR + JRE |
| Python | gcc, Build-Headers, pip | Wheels + Python Runtime |
| Rust | Compiler, Cargo, Source | Statisches Binary |

---

## Was sind Multi-Stage Builds?

Multi-Stage Builds lösen dieses Problem durch **mehrere `FROM`-Anweisungen** in einem einzigen Dockerfile. Jede `FROM`-Anweisung startet eine neue **Stage** (Baustufe) mit einem eigenen Base-Image.

Der entscheidende Mechanismus: Mit `COPY --from=<stage>` können **Artefakte gezielt zwischen Stages kopiert** werden. Nur die letzte Stage wird zum finalen Image — alle vorherigen Stages werden verworfen.

```mermaid
graph LR
    A["Quellcode"] --> B["Build-Stage"]
    B --> C["Binary"]
    C --> D["Runtime-Stage"]
    D --> E["Finales Image ~15 MB"]

    style B fill:#f4a261,stroke:#e76f51,color:#000
    style D fill:#2a9d8f,stroke:#264653,color:#fff
    style E fill:#06d6a0,stroke:#028090,color:#000
```

### Grundprinzip

```dockerfile
# === Stage 1: Build ===
FROM golang:1.24 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# === Stage 2: Runtime ===
FROM alpine:3.21
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

> [!NOTE] Wie funktioniert `COPY --from`?
> Die Anweisung `COPY --from=builder` greift auf das Dateisystem der Stage mit dem Namen `builder` zu und kopiert nur die angegebenen Dateien in die aktuelle Stage. Der Rest der Build-Stage wird **nicht** ins finale Image übernommen.

---

## Benannte Stages

Jede Stage kann mit `AS <name>` benannt werden. Das macht das Dockerfile lesbarer und ermöglicht gezielte Referenzierung.

```dockerfile
FROM node:22-alpine AS dependencies
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS production
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> [!TIP] Namenskonventionen
> Verwende sprechende Stage-Namen wie `builder`, `dependencies`, `build`, `test`, `production` oder `runtime`. Das macht das Dockerfile selbstdokumentierend.

---

## Praktische Beispiele

### Go-Anwendung

Go eignet sich hervorragend für Multi-Stage Builds, da es statisch gelinkte Binaries erzeugt, die **kein Runtime-Image** benötigen.

```dockerfile
# === Build-Stage ===
FROM golang:1.24-alpine AS builder

RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app/server ./cmd/server

# === Runtime-Stage ===
FROM scratch

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

COPY --from=builder /app/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

> [!NOTE] Warum `scratch`?
> `scratch` ist ein **leeres** Image — 0 Bytes. Da Go statische Binaries erzeugt, braucht es kein Betriebssystem. Das finale Image enthält nur das Binary, TLS-Zertifikate und Zeitzonen-Daten — typischerweise **5–15 MB**.

### Node.js-Anwendung (Frontend SPA)

Bei Node.js-Frontends wird die Anwendung gebaut und dann mit einem leichtgewichtigen Webserver ausgeliefert.

```dockerfile
# === Stage 1: Dependencies ===
FROM node:22-alpine AS dependencies
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production=false

# === Stage 2: Build ===
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build
RUN npm prune --production

# === Stage 3: Production ===
FROM nginx:1.27-alpine AS production

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -q --spider http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

> [!TIP] Drei Stages für Node.js
> Die Trennung in `dependencies` → `build` → `production` nutzt den Docker Layer-Cache optimal: Wenn sich nur der Quellcode ändert, werden die Dependencies nicht neu installiert.

### Python-Anwendung

```dockerfile
# === Build-Stage ===
FROM python:3.13-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# === Runtime-Stage ===
FROM python:3.13-slim AS runtime

COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app
COPY . .

RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Java / Spring Boot

```dockerfile
# === Build-Stage ===
FROM eclipse-temurin:21-jdk-alpine AS builder

RUN apk add --no-cache maven

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline -B

COPY src ./src
RUN mvn package -DskipTests -B

RUN java -Djarmode=tools -jar target/*.jar extract --layers --destination extracted

# === Runtime-Stage ===
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN addgroup -S spring && adduser -S spring -G spring
USER spring

WORKDIR /app

COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "application.jar"]
```

---

## Größenvergleich

| Anwendung | Ohne Multi-Stage | Mit Multi-Stage | Base-Image (Runtime) | Reduktion |
|-----------|-----------------:|----------------:|----------------------|----------:|
| Go API | ~1.200 MB | ~12 MB | `scratch` | **~99 %** |
| Go API | ~1.200 MB | ~18 MB | `alpine:3.21` | **~98 %** |
| Node.js SPA | ~1.100 MB | ~45 MB | `nginx:1.27-alpine` | **~96 %** |
| Python API | ~950 MB | ~180 MB | `python:3.13-slim` | **~81 %** |
| Java Spring | ~800 MB | ~290 MB | `temurin:21-jre-alpine` | **~64 %** |
| Rust CLI | ~1.500 MB | ~8 MB | `scratch` | **~99 %** |

---

## Fortgeschrittene Techniken

### Target-Stages mit `--target`

Mit `docker build --target <stage>` kann gezielt eine **bestimmte Stage** gebaut werden. Das ist nützlich für Development- vs. Production-Builds.

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM base AS development
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS build
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS production
COPY --from=build /app/dist /usr/share/nginx/html
CMD ["nginx", "-g", "daemon off;"]
```

```bash
# Nur die Development-Stage bauen
docker build --target development -t myapp:dev .

# Production-Build (default — letzte Stage)
docker build -t myapp:prod .
```

### Externe Images als Stage-Quelle

`COPY --from` akzeptiert nicht nur Stage-Namen, sondern auch **beliebige externe Images**:

```dockerfile
FROM alpine:3.21

COPY --from=docker/buildx-bin:latest /buildx /usr/libexec/docker/cli-plugins/docker-buildx
COPY --from=docker:27-cli /usr/local/bin/docker /usr/local/bin/docker
COPY --from=hashicorp/terraform:1.10 /bin/terraform /usr/local/bin/terraform
```

### Cache-Mounts mit BuildKit

Docker BuildKit ermöglicht **persistente Caches** zwischen Builds:

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.24-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./

RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o server .

FROM scratch
COPY --from=builder /app/server /server
CMD ["/server"]
```

Cache-Mounts für andere Sprachen:

```dockerfile
# Python pip-Cache
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Node.js npm-Cache
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Maven-Repository-Cache
RUN --mount=type=cache,target=/root/.m2/repository \
    mvn package -DskipTests
```

### Test-Stage integrieren

```dockerfile
FROM golang:1.24-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM builder AS test
RUN go test -v -race ./...

FROM alpine:3.21 AS production
COPY --from=builder /app/server /server
CMD ["/server"]
```

```bash
# Nur Tests ausführen
docker build --target test -t myapp:test .

# Production-Build (Tests werden NICHT automatisch ausgeführt)
docker build --target production -t myapp:prod .
```

> [!WARNING] Tests und die letzte Stage
> Wenn die Test-Stage **nicht** im Abhängigkeitspfad der letzten Stage liegt, wird sie bei einem normalen `docker build` **nicht** ausgeführt. Verwende `--target test` explizit in der CI-Pipeline.

---

## Best Practices

| Praxis | Beschreibung |
|--------|-------------|
| **Minimale Runtime-Images** | `scratch`, `alpine`, `*-slim` statt vollwertiger Images |
| **Benannte Stages** | `AS builder` statt Index-Referenzierung |
| **Dependencies zuerst** | Manifeste vor Source für Layer-Cache |
| **Non-Root User** | Im Runtime-Image immer `USER` setzen |
| **`.dockerignore`** | Build-Context klein halten |
| **`--no-cache-dir`** | Package-Manager-Caches nicht ins Image |
| **HEALTHCHECK** | Im Production-Image immer definieren |
| **`-ldflags="-w -s"`** | Go-Binaries ohne Debug-Symbole (kleinere Größe) |
| **`CGO_ENABLED=0`** | Go statisch linken für `scratch`-Kompatibilität |

> [!IMPORTANT] `.dockerignore` nicht vergessen
> Ohne `.dockerignore` wird das gesamte Projektverzeichnis (inkl. `.git`, `node_modules`, etc.) an den Docker-Daemon gesendet. Das verlangsamt den Build und kann sensible Daten ins Image leaken.

---

## Verwandte Notizen

- [[Docker Index]]
- [[Dockerfile]]
- [[Docker Images]]
- [[GitOps]] — Optimierte Images für CI/CD-Pipelines
