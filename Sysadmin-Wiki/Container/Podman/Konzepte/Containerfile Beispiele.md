---
tags: [podman, containerfile, build, beispiele, nodejs, python, go, nginx, multi-stage]
aliases: ["Containerfile Beispiele", "Dockerfile Beispiele Podman", "Build Beispiele"]
---

# Containerfile Beispiele

> [!ABSTRACT]
> Praxiserprobte Containerfile-Vorlagen für die häufigsten Sprachen und Use-Cases: Node.js, Python, Go (statisch), Nginx, Java (JRE-Only) und eine PostgreSQL-Erweiterung. Alle Beispiele sind rootless-ready und optimiert für minimale Image-Größe.

---

## Node.js — Production-Ready

```dockerfile
# Containerfile — Node.js Produktions-Image
FROM docker.io/library/node:20-alpine AS deps

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# ---
FROM docker.io/library/node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build    # TypeScript/Webpack/etc.

# ---
FROM docker.io/library/node:20-alpine AS production

# Non-root User (Alpine)
RUN addgroup -g 1001 -S nodejs \
 && adduser -S -u 1001 -G nodejs nodeuser

WORKDIR /app

# Nur Produktions-Dependencies
COPY --from=deps --chown=nodeuser:nodejs /app/node_modules ./node_modules
# Nur Build-Output
COPY --from=builder --chown=nodeuser:nodejs /app/dist ./dist
COPY --chown=nodeuser:nodejs package.json ./

ENV NODE_ENV=production \
    PORT=3000

USER 1001
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

```bash
podman build -t mynode-app:1.0 .
podman run -d -p 3000:3000 --name nodeapp mynode-app:1.0
```

---

## Python — FastAPI/Flask

```dockerfile
# Containerfile — Python (FastAPI)
FROM docker.io/library/python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .

# Packages in eigenen Pfad installieren (für Copy in final stage)
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---
FROM docker.io/library/python:3.12-slim

# Sicherheits-Updates
RUN apt-get update \
 && apt-get upgrade -y \
 && rm -rf /var/lib/apt/lists/*

# Non-root User
RUN useradd -u 1001 -m appuser

WORKDIR /app

# Packages aus Builder
COPY --from=builder /install /usr/local
COPY --chown=appuser:appuser . .

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=8000

USER 1001
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Go — Statisches Binary (minimalstes Image)

```dockerfile
# Containerfile — Go statisches Binary
FROM docker.io/library/golang:1.22-alpine AS builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Statisch gelinkt, kein CGO
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -ldflags="-w -s -X main.version=$(git describe --tags --always)" \
    -o app ./cmd/server

# ---
# Scratch = leeres Image (nur Binary + CA-Zertifikate)
FROM scratch

# CA-Zertifikate für HTTPS-Requests
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Binary
COPY --from=builder /build/app /app

EXPOSE 8080

# kein Shell in scratch — Exec-Form zwingend
ENTRYPOINT ["/app"]
```

```bash
podman build -t mygo-app:1.0 .
# Resultierendes Image: ~10-15 MB statt >1 GB mit golang:latest!
podman image inspect mygo-app:1.0 --format "{{.Size}}"
```

---

## Nginx — Custom Static Site

```dockerfile
# Containerfile — Nginx mit Custom-Konfiguration
FROM docker.io/library/node:20-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build    # Erzeugt dist/ Verzeichnis

# ---
FROM docker.io/library/nginx:1.25-alpine

# Custom nginx.conf
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=build /app/dist /usr/share/nginx/html

# Nginx als non-root (Port > 1024!)
RUN sed -i 's/listen\s*80/listen 8080/g' /etc/nginx/nginx.conf \
 && chown -R nginx:nginx /usr/share/nginx/html \
 && chown -R nginx:nginx /var/cache/nginx \
 && chown -R nginx:nginx /var/log/nginx \
 && touch /var/run/nginx.pid \
 && chown nginx:nginx /var/run/nginx.pid

USER nginx
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf für rootless-Betrieb
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    server {
        listen 8080;
        root /usr/share/nginx/html;
        index index.html;

        # SPA-Routing
        location / {
            try_files $uri $uri/ /index.html;
        }

        # Cache für Assets
        location ~* \.(js|css|png|jpg|gif|ico|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

---

## Java — JRE-Only (ohne JDK)

```dockerfile
# Containerfile — Java Spring Boot
FROM docker.io/library/eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build
COPY mvnw pom.xml ./
COPY .mvn .mvn
# Abhängigkeiten zuerst cachen
RUN ./mvnw dependency:resolve -q

COPY src ./src
RUN ./mvnw package -DskipTests -q

# Layers extrahieren (Spring Boot Layer-Feature für besseren Cache)
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /layers

# ---
FROM docker.io/library/eclipse-temurin:21-jre-alpine

RUN addgroup -g 1001 -S javagroup \
 && adduser -S -u 1001 -G javagroup javauser

WORKDIR /app

# Spring Boot Layers (besseres Caching beim Rebuild)
COPY --from=builder --chown=javauser /layers/dependencies/ ./
COPY --from=builder --chown=javauser /layers/spring-boot-loader/ ./
COPY --from=builder --chown=javauser /layers/snapshot-dependencies/ ./
COPY --from=builder --chown=javauser /layers/application/ ./

ENV JAVA_OPTS="-XX:MaxRAMPercentage=75 -XX:+UseG1GC"

USER 1001
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} org.springframework.boot.loader.JarLauncher"]
```

---

## Containerfile für Entwicklung (mit Hot-Reload)

```dockerfile
# Containerfile.dev — Nur für Entwicklung, nicht für Produktion!
FROM docker.io/library/node:20-alpine

RUN apk add --no-cache git

WORKDIR /app

# Nur package.json (Source wird via Volume gemountet)
COPY package*.json ./
RUN npm install   # inkl. devDependencies

# Kein COPY . . — Source wird live via Volume gemountet
ENV NODE_ENV=development

EXPOSE 3000 9229  # 9229 = Node Inspector (Debugger)

CMD ["npm", "run", "dev"]
```

```bash
# Entwicklungs-Container mit Hot-Reload
podman run -it \
  -v $(pwd):/app:Z \
  -p 3000:3000 \
  -p 9229:9229 \
  --name myapp-dev \
  myapp:dev
```

---

## Distroless — Maximale Sicherheit

```dockerfile
# Containerfile — Distroless (kein Shell, kein Paketmanager)
FROM docker.io/library/python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt
COPY . .

# ---
FROM gcr.io/distroless/python3-debian12:nonroot

WORKDIR /app
COPY --from=builder /install/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /app .

EXPOSE 8000

# kein Shell → exec-Form zwingend
CMD ["/app/main.py"]
```

> [!NOTE] Distroless-Images
> Enthalten kein Shell, keinen Paketmanager, keine Tools — nur Runtime und Anwendung. Extrem sicher, aber Debugging erfordert ephemeral Container: `podman debug container-id`

---

## .containerignore Vorlage (universal)

```gitignore
# .containerignore — für alle Projekte

# Version Control
.git/
.gitignore
.gitattributes

# Dependency Verzeichnisse
node_modules/
vendor/
__pycache__/
*.pyc
.venv/
venv/
env/

# Build-Output
dist/
build/
target/
out/

# Test-Dateien
tests/
test/
__tests__/
*.test.js
*.spec.ts
coverage/

# IDE/Editor
.vscode/
.idea/
*.swp
*.swo

# Dokumentation
docs/
*.md
!README.md   # README behalten

# Umgebung
.env
.env.*
!.env.example

# Logs
*.log
logs/

# Container-Dateien selbst
Containerfile
Dockerfile
docker-compose*.yml
.containerignore
.dockerignore
```

---

## Verwandte Notizen

- [[Containerfile]] — Containerfile-Syntax und alle Anweisungen
- [[Multi-Stage Builds]] — Mehrstufige Builds im Detail
- [[Podman Best Practices]] — Image-Optimierung
- [[Podman Registries]] — Images in Registries veröffentlichen
