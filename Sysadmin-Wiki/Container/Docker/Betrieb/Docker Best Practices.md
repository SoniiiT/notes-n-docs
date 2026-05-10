---
tags:
  - docker
  - best-practices
  - produktion
  - optimierung
  - hardening
---

# Docker Best Practices

> [!ABSTRACT] TL;DR
> Gesammelte Best Practices für Docker in Produktion — von Dockerfile-Optimierung über Container-Laufzeit-Konfiguration bis hin zu Sicherheit, Logging und Ressourcen-Management. Dient als zentrale Checkliste und Nachschlagewerk.

---

## Dockerfile Best Practices

### Layer-Optimierung

```dockerfile
# SCHLECHT — jeder RUN erstellt einen Layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# GUT — ein Layer für die gesamte Installation
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*
```

### Cache-Reihenfolge

```dockerfile
# SCHLECHT — Code-Änderung invalidiert dependency-Cache
COPY . .
RUN npm ci
RUN npm run build

# GUT — Dependencies ändern sich selten
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
RUN npm run build
```

### Minimale Base-Images

| Base Image | Größe | Verwendung |
|-----------|-------|------------|
| `scratch` | 0 MB | Statisch gelinkte Binaries (Go, Rust) |
| `distroless` | ~2 MB | Binaries ohne Shell |
| `alpine` | ~7 MB | Wenn Shell benötigt wird |
| `*-slim` | ~80 MB | Wenn glibc benötigt wird |
| `ubuntu/debian` | ~120 MB | Vollständiges OS, nur wenn nötig |

> [!TIP] Alpine-Fallstricke
> Alpine verwendet **musl libc** statt glibc. Das kann bei Python-Packages mit C-Extensions (numpy, pandas) oder bei Java DNS-Lookups zu Problemen führen. Im Zweifelsfall die `-slim`-Variante verwenden.

### Feste Versionen pinnen

```dockerfile
# SCHLECHT — ändert sich ständig
FROM node:latest
FROM python

# BESSER — Major + Minor gepinnt
FROM node:22-alpine
FROM python:3.12-slim

# BEST — Exakter Digest (immutable)
FROM node:22-alpine@sha256:abc123...
```

### Non-Root User

```dockerfile
# User erstellen und verwenden
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Dateien mit korrektem Owner kopieren
COPY --chown=appuser:appgroup . .

# Zum User wechseln (VOR CMD/ENTRYPOINT)
USER appuser
```

### `.dockerignore`

```text
# Immer ausschließen
.git
.github
.gitignore
*.md
LICENSE
Dockerfile*
docker-compose*
.dockerignore
.env*

# IDE
.vscode
.idea

# Build-Artefakte
node_modules
dist
build
__pycache__
*.pyc
target
bin
obj
vendor
```

> [!IMPORTANT] `.dockerignore` ist Pflicht
> Ohne `.dockerignore` wird der gesamte Build-Context an den Docker Daemon gesendet — inklusive `.git/`, `node_modules/` und anderer großer Verzeichnisse. Das verlangsamt Builds und kann sensible Daten ins Image leaken.

---

## Container-Laufzeit

### Health Checks

```dockerfile
# Im Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 --start-period=60s \
    CMD curl -f http://localhost:8080/health || exit 1
```

```yaml
# In Docker Compose
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

**Health-Check-Empfehlungen:**

| Parameter | Empfehlung | Begründung |
|-----------|-----------|------------|
| `interval` | 15–30s | Häufig genug, ohne Last zu erzeugen |
| `timeout` | 5–10s | Sollte deutlich unter `interval` liegen |
| `retries` | 3 | Vermeidet falsche Alarme |
| `start_period` | 30–120s | Anwendung muss erst hochfahren |

### Restart-Policies

```yaml
# Docker Compose
services:
  app:
    restart: unless-stopped   # Startet automatisch, außer manuell gestoppt

  db:
    restart: always            # Startet immer, auch nach Docker-Daemon-Restart

  migration:
    restart: "no"              # Einmalige Tasks (Migrationen)

  worker:
    restart: on-failure        # Nur bei Exit-Code != 0
```

```bash
# Docker CLI
docker run --restart=unless-stopped myapp
```

| Policy | Beschreibung | Verwendung |
|--------|-------------|------------|
| `no` | Kein automatischer Restart | Einmalige Tasks, Debug-Container |
| `on-failure` | Restart bei Exit-Code ≠ 0 | Worker, Batch-Jobs |
| `unless-stopped` | Restart außer bei manuellem Stopp | Standard für die meisten Services |
| `always` | Restart immer | Kritische Services, Datenbanken |

### Ressourcen-Limits

```yaml
# Docker Compose
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M

    # Alternativ (ältere Compose-Syntax)
    mem_limit: 512m
    cpus: 1.0
```

```bash
# Docker CLI
docker run \
    --memory=512m \
    --memory-swap=1g \
    --cpus=1.0 \
    --pids-limit=100 \
    myapp
```

> [!DANGER] Ohne Memory-Limit = OOM-Risiko
> Ohne `--memory`-Limit kann ein einzelner Container den gesamten Host-Speicher verbrauchen und den OOM-Killer auslösen, der beliebige Prozesse beendet — auch Container anderer Anwendungen.

### Graceful Shutdown

```dockerfile
# ENTRYPOINT statt CMD für Signal-Weiterleitung
ENTRYPOINT ["node", "server.js"]

# Oder mit tini als Init-System
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

```bash
# Stop-Timeout erhöhen (Standard: 10 Sekunden)
docker stop --time=30 mycontainer

# In Docker Compose
services:
  app:
    stop_grace_period: 30s
```

> [!NOTE] Shell-Form vs. Exec-Form
> `CMD node server.js` (Shell-Form) startet den Prozess über `/bin/sh -c` — Signale wie SIGTERM erreichen Node.js nicht! Verwende immer die Exec-Form: `CMD ["node", "server.js"]`.

---

## Logging

### Log-Rotation konfigurieren

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "tag": "{{.Name}}"
  }
}
```

```yaml
# Pro Service in Docker Compose
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

> [!WARNING] Log-Rotation ist Pflicht
> Ohne `max-size` und `max-file` wachsen Container-Logs unbegrenzt. Ein einzelner geschwätziger Container kann die gesamte Festplatte füllen und alle Container auf dem Host lahmlegen.

### Logging Best Practices

| Empfehlung | Begründung |
|-----------|------------|
| `stdout/stderr` nutzen | Docker fängt nur stdout/stderr ab |
| JSON-Format loggen | Maschinenlesbar, parsbar |
| Keine Log-Dateien im Container | Werden bei Container-Tod gelöscht |
| Log-Level konfigurierbar machen | Via Environment-Variable |
| Sensitive Daten nicht loggen | Passwörter, Tokens, PII |

---

## Netzwerk

### Eigene Bridge-Netzwerke

```bash
# SCHLECHT — Standard-Bridge, keine DNS-Auflösung
docker run --name app myapp
docker run --name db postgres
# app kann db NICHT per Name erreichen

# GUT — Eigenes Netzwerk mit DNS
docker network create app-net
docker run --name app --network app-net myapp
docker run --name db --network app-net postgres
# app erreicht db über "db" als Hostname
```

### Netzwerk-Isolation

```yaml
# Docker Compose
services:
  traefik:
    networks:
      - frontend

  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend    # Nicht von außen erreichbar!

networks:
  frontend:
  backend:
    internal: true  # Kein Internet-Zugang
```

### Port-Binding sichern

```bash
# SCHLECHT — bindet auf 0.0.0.0 (alle Interfaces)
docker run -p 8080:80 nginx

# BESSER — nur auf localhost binden
docker run -p 127.0.0.1:8080:80 nginx

# ACHTUNG: Docker manipuliert iptables!
# -p 8080:80 umgeht die Host-Firewall (ufw, firewalld)
```

> [!DANGER] Docker umgeht die Firewall
> `docker run -p 8080:80` fügt iptables-Regeln hinzu, die **vor** UFW/firewalld-Regeln greifen. Ein Container-Port ist dann von außen erreichbar, auch wenn die Firewall den Port blockiert! Lösung: `127.0.0.1:PORT:PORT` oder den Docker-Daemon konfigurieren.

---

## Storage

### Volume-Typen richtig einsetzen

| Typ | Verwendung | Beispiel |
|-----|-----------|---------|
| Named Volume | Persistente Daten (DB, Uploads) | `db-data:/var/lib/postgresql/data` |
| Bind Mount | Entwicklung, Konfigurationsdateien | `./config:/app/config:ro` |
| tmpfs Mount | Temporäre Daten, Secrets | Sensible Zwischenergebnisse |

### Volume-Berechtigungen

```dockerfile
# Verzeichnis erstellen und Berechtigungen setzen
RUN mkdir -p /app/data && chown appuser:appgroup /app/data
VOLUME /app/data
USER appuser
```

```yaml
# Bind Mount als Read-Only
services:
  app:
    volumes:
      - ./config/app.yml:/app/config.yml:ro
      - app-data:/app/data

volumes:
  app-data:
```

---

## Sicherheit

### Security-Checkliste

| Punkt | Prüfung |
|-------|---------|
| ✅ Non-Root User | `USER appuser` im Dockerfile |
| ✅ Read-Only Filesystem | `--read-only` wo möglich |
| ✅ Keine Capabilities | `--cap-drop=ALL`, dann einzeln hinzufügen |
| ✅ Kein privilegierter Modus | Niemals `--privileged` in Produktion |
| ✅ Secrets nicht im Image | Build-Args oder Multi-Stage nutzen |
| ✅ Image-Scanning | Trivy, Docker Scout, oder Grype |
| ✅ Base-Image aktuell | Regelmäßige Updates (Dependabot/Renovate) |
| ✅ `.dockerignore` | Keine `.env`, `.git`, `node_modules` im Image |
| ✅ HTTPS für Registries | Nur TLS-geschützte Registries verwenden |
| ✅ Content Trust | `DOCKER_CONTENT_TRUST=1` für signierte Images |

### Minimale Capabilities

```bash
# Alle Capabilities entfernen, nur benötigte hinzufügen
docker run \
    --cap-drop=ALL \
    --cap-add=NET_BIND_SERVICE \
    --security-opt=no-new-privileges \
    --read-only \
    --tmpfs /tmp \
    myapp
```

```yaml
# Docker Compose
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
```

### Secrets im Build

```dockerfile
# SCHLECHT — Secret bleibt im Image-Layer
COPY .npmrc /app/.npmrc
RUN npm ci
RUN rm /app/.npmrc   # Ist trotzdem im vorherigen Layer!

# GUT — BuildKit Secret Mount
RUN --mount=type=secret,id=npmrc,target=/app/.npmrc npm ci
```

```bash
# Build mit Secret
docker buildx build --secret id=npmrc,src=.npmrc -t myapp .
```

---

## Docker Daemon Konfiguration

### Empfohlene `daemon.json`

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {"base": "172.17.0.0/12", "size": 24}
  ],
  "storage-driver": "overlay2",
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65535,
      "Soft": 65535
    }
  }
}
```

| Option | Beschreibung |
|--------|-------------|
| `live-restore` | Container laufen weiter bei Docker-Daemon-Restart |
| `userland-proxy: false` | Nutzt iptables statt userland-proxy (performanter) |
| `no-new-privileges` | Verhindert Privilege-Escalation in Containern |
| `storage-driver: overlay2` | Empfohlener Storage-Driver |

---

## Anti-Patterns

| Anti-Pattern | Problem | Lösung |
|-------------|---------|--------|
| `:latest` in Produktion | Nicht reproduzierbar | Feste Tags: `:1.2.3` |
| Root im Container | Privilege Escalation | `USER appuser` |
| Keine `.dockerignore` | Große Images, Datenlecks | `.dockerignore` pflegen |
| Secrets in ENV | Sichtbar in `docker inspect` | Build-Secrets oder Volumes |
| Keine Log-Rotation | Disk voll, Host-Ausfall | `max-size` + `max-file` |
| Shell-Form CMD | Kein Signal-Handling | Exec-Form: `["node", "app.js"]` |
| Alles in einem Container | Keine Skalierbarkeit | Ein Prozess pro Container |
| `apt-get upgrade` im Dockerfile | Nicht reproduzierbar | Base-Image updaten stattdessen |
| Kein Health Check | Fehler unerkannt | `HEALTHCHECK` definieren |
| Große Images (>1 GB) | Langsame Deployments | Multi-Stage, Alpine, Distroless |

---

## Verwandte Notizen

- [[Dockerfile]] — Dockerfile-Referenz und Direktiven
- [[Dockerfile Beispiele]] — Sprachspezifische Beispiele
- [[Docker Images]] — Image-Management und -Optimierung
- [[Multi-Stage Builds]] — Build-Optimierung
- [[Docker Security und Hardening]] — Sicherheits-Hardening im Detail
- [[Docker Monitoring und Logging]] — Monitoring und Log-Management
- [[Docker Networking]] — Netzwerk-Konfiguration
- [[Docker Volumes und Storage]] — Storage-Strategien
- [[Docker CI-CD Integration]] — CI/CD-Pipelines für Docker
- [[Swarm Best Practices]] — Best Practices für Docker Swarm
