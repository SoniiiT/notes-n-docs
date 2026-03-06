---
tags:
  - docker
  - docker-compose
  - compose
  - multi-container
---

# Docker Compose

> [!ABSTRACT] TL;DR
> Docker Compose ermöglicht die Definition und Verwaltung von Multi-Container-Anwendungen über eine deklarative YAML-Datei (`compose.yaml`). Services, Volumes, Netzwerke und Secrets werden zentral beschrieben und mit einem einzigen Befehl (`docker compose up`) gestartet.

---

## Was ist Docker Compose?

Docker Compose ist ein Werkzeug zur Definition und Ausführung von **Multi-Container-Anwendungen**. Anstatt jeden Container einzeln mit langen `docker run`-Befehlen zu starten, beschreibt man die gesamte Anwendungslandschaft in einer einzigen YAML-Datei.

Ein typisches Szenario: Eine Webanwendung besteht aus einem **Frontend**, einem **Backend**, einer **Datenbank** und einem **Cache**. Ohne Compose müsste man vier separate `docker run`-Befehle mit Netzwerk-, Volume- und Umgebungsvariablen-Konfiguration ausführen. Mit Compose genügt ein `docker compose up`.

### Kernkonzepte

- **Deklarativ** — Man beschreibt den gewünschten Zustand, nicht die Schritte dorthin
- **Reproduzierbar** — Die gleiche `compose.yaml` erzeugt immer die gleiche Umgebung
- **Isoliert** — Jedes Compose-Projekt erhält ein eigenes Default-Netzwerk
- **Versioniert** — Die YAML-Datei wird zusammen mit dem Code in Git eingecheckt

### v1 vs v2 — Das Plugin-Modell

> [!IMPORTANT] Docker Compose v1 ist End-of-Life
> Docker Compose v1 (das eigenständige Python-Binary `docker-compose`) wurde im Juli 2023 offiziell eingestellt. Nur noch v2 (Go-basiertes Docker-Plugin) wird aktiv weiterentwickelt.

| Eigenschaft | Compose v1 | Compose v2 |
|:------------|:-----------|:-----------|
| Aufruf | `docker-compose` (Bindestrich) | `docker compose` (Leerzeichen) |
| Sprache | Python | Go |
| Installation | Standalone-Binary | Docker CLI-Plugin |
| Container-Namen | `projekt_service_1` | `projekt-service-1` |
| Performance | Langsamer | Deutlich schneller |
| BuildKit | Optionales Flag | Standard |
| Profiles | Nicht unterstützt | Unterstützt |
| Status | **End-of-Life** | **Aktiv** |

```mermaid
graph LR
    A[Docker CLI] --> B[compose Plugin]
    A --> C[buildx Plugin]
    A --> D[Weitere Plugins]
    B --> E[compose.yaml]
    E --> F[Container Engine]
```

---

## Installation

### Docker Desktop (Windows / macOS)

Docker Compose v2 ist in **Docker Desktop** bereits enthalten — keine separate Installation nötig.

```bash
# Prüfen ob Compose verfügbar ist
docker compose version
```

### Linux — Docker Compose Plugin

Auf Linux-Systemen wird Compose v2 als Docker-CLI-Plugin installiert:

```bash
# Über das offizielle Docker-Repository (empfohlen)
sudo apt update
sudo apt install docker-compose-plugin

# Version prüfen
docker compose version
```

> [!TIP] Manuelle Installation
> Falls das Repository nicht konfiguriert ist, kann das Plugin auch manuell installiert werden:
> ```bash
> # Neueste Version herunterladen
> DOCKER_COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep tag_name | cut -d '"' -f 4)
> sudo curl -L "https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-linux-x86_64" \
>   -o /usr/local/lib/docker/cli-plugins/docker-compose
> sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
> ```

### Kompatibilitätsalias

Für Skripte, die noch `docker-compose` (v1-Syntax) verwenden:

```bash
# Alias in ~/.bashrc oder ~/.zshrc
alias docker-compose='docker compose'
```

---

## compose.yaml Grundstruktur

> [!NOTE] Dateiname
> Der empfohlene Dateiname ist `compose.yaml`. Docker Compose erkennt auch `compose.yml`, `docker-compose.yaml` und `docker-compose.yml` — aber `compose.yaml` ist der offizielle Standard seit der Compose Specification.

### Aufbau im Überblick

```yaml
# ── Top-Level-Elemente ──────────────────────────────────

services:        # Container-Definitionen (Pflicht)
  webapp:
    image: nginx:alpine
    # ...

  database:
    image: postgres:17
    # ...

volumes:         # Benannte Volumes (Optional)
  db-data:
    driver: local

networks:        # Benutzerdefinierte Netzwerke (Optional)
  frontend:
  backend:

secrets:         # Secrets Management (Optional)
  db-password:
    file: ./secrets/db-password.txt

configs:         # Konfigurationsdateien (Optional)
  nginx-conf:
    file: ./nginx/nginx.conf
```

### Kommentiertes Praxisbeispiel

```yaml
services:
  # ── Frontend ──────────────────────────────────────────
  frontend:
    image: nginx:1.27-alpine
    ports:
      - "80:80"                    # Host:Container
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro  # Bind Mount (readonly)
      - static-files:/usr/share/nginx/html           # Named Volume
    networks:
      - frontend
    depends_on:
      backend:
        condition: service_healthy  # Warten bis Backend gesund
    restart: unless-stopped

  # ── Backend ───────────────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgres://user:pass@database:5432/app
      - REDIS_URL=redis://cache:6379
    env_file:
      - .env                       # Zusätzliche Variablen aus Datei
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

  # ── Datenbank ─────────────────────────────────────────
  database:
    image: postgres:17-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password
      POSTGRES_DB: app
    secrets:
      - db-password
    networks:
      - backend
    restart: unless-stopped

  # ── Cache ─────────────────────────────────────────────
  cache:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - cache-data:/data
    networks:
      - backend
    restart: unless-stopped

# ── Volumes ─────────────────────────────────────────────
volumes:
  db-data:
    driver: local
  cache-data:
  static-files:

# ── Netzwerke ───────────────────────────────────────────
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true                 # Kein Internetzugang

# ── Secrets ─────────────────────────────────────────────
secrets:
  db-password:
    file: ./secrets/db-password.txt
```

```mermaid
graph TD
    Client([Client]) --> Frontend[Frontend - Nginx]

    subgraph Compose-Projekt
        subgraph frontend-network
            Frontend
            Backend[Backend - App]
        end
        subgraph backend-network
            Backend
            Database[(Database - Postgres)]
            Cache[(Cache - Redis)]
        end
    end

    Frontend -->|Reverse Proxy| Backend
    Backend -->|SQL| Database
    Backend -->|Key-Value| Cache

    Database --- db-data[(db-data Volume)]
    Cache --- cache-data[(cache-data Volume)]
```

---

## Service-Konfiguration im Detail

### image / build

Container können entweder von einem **fertigen Image** oder durch einen **Build-Prozess** erstellt werden.

```yaml
services:
  # Variante 1: Fertiges Image
  nginx:
    image: nginx:1.27-alpine

  # Variante 2: Eigenes Build
  app:
    build:
      context: ./app               # Build-Kontext (Verzeichnis mit Dockerfile)
      dockerfile: Dockerfile.prod   # Alternativer Dockerfile-Name
      args:                         # Build-Argumente
        NODE_VERSION: "22"
        ENV: production
      target: production            # Multi-Stage Build Target
      cache_from:
        - myapp:cache

  # Variante 3: Image + Build (Build bevorzugt, Image als Fallback-Name)
  api:
    image: myregistry/api:latest
    build:
      context: ./api
```

### ports

```yaml
services:
  webapp:
    ports:
      # Kurzform
      - "8080:80"                  # HOST:CONTAINER
      - "443:443"

      # Nur Container-Port (zufälliger Host-Port)
      - "80"

      # Bestimmtes Interface binden
      - "127.0.0.1:8080:80"       # Nur localhost

      # Langform
      - target: 80                 # Container-Port
        published: 8080            # Host-Port
        protocol: tcp              # tcp oder udp
        mode: host                 # host oder ingress
```

> [!WARNING] Sicherheitshinweis
> `- "8080:80"` bindet auf **alle Interfaces** (`0.0.0.0`). Für lokale Dienste immer `127.0.0.1:8080:80` verwenden, um den Port nicht ins Netzwerk zu exponieren.

### volumes

```yaml
services:
  app:
    volumes:
      # Bind Mount — Verzeichnis vom Host
      - ./config:/app/config               # Relativ zum Compose-File
      - /opt/data:/app/data                # Absoluter Pfad

      # Bind Mount (readonly)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

      # Named Volume
      - app-data:/app/data

      # Langform
      - type: volume
        source: app-data
        target: /app/data
        read_only: false

      # tmpfs Mount (nur RAM, nicht persistent)
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100000000          # 100 MB

volumes:
  app-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/export/data"
```

### environment / env_file

```yaml
services:
  app:
    # Variante 1: Map-Syntax (empfohlen)
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://user:pass@db:5432/app
      DEBUG: "false"               # Booleans als String!

    # Variante 2: List-Syntax
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:pass@db:5432/app

    # Variante 3: Aus Datei laden
    env_file:
      - .env                      # Standard
      - .env.local                # Überschreibt .env-Werte
      - path: .env.production
        required: false            # Kein Fehler wenn nicht vorhanden
```

> [!DANGER] Keine Secrets in environment!
> Sensible Daten wie Passwörter, API-Keys und Tokens gehören **nicht** in `environment` oder `.env`-Dateien, die in Git eingecheckt werden. Verwende stattdessen Docker Secrets oder externe Secret-Management-Lösungen.

### depends_on

```yaml
services:
  backend:
    depends_on:
      # Einfach — wartet nur auf Container-Start
      cache:
        condition: service_started

      # Mit Healthcheck — wartet auf gesunden Zustand
      database:
        condition: service_healthy

      # Warten auf erfolgreichen Abschluss (z. B. Migrations-Container)
      migrations:
        condition: service_completed_successfully

      # Optionale Abhängigkeit mit restart
      redis:
        condition: service_healthy
        restart: true              # Backend neustarten wenn Redis neustartet
```

> [!TIP] Best Practice
> Verwende **immer** `condition: service_healthy` zusammen mit einem `healthcheck` beim Ziel-Service. Das einfache `depends_on` ohne Condition garantiert nur, dass der Container **gestartet** wurde — nicht, dass der Dienst darin bereit ist.

### restart

| Wert | Verhalten |
|:-----|:----------|
| `no` | Kein automatischer Neustart (Standard) |
| `always` | Immer neustarten, auch bei manuellem Stop |
| `unless-stopped` | Neustarten, außer manuell gestoppt |
| `on-failure` | Nur bei Fehler neustarten (Exit-Code ≠ 0) |
| `on-failure:5` | Maximal 5 Neustart-Versuche |

```yaml
services:
  webapp:
    restart: unless-stopped        # Empfehlung für Produktions-Services
```

### healthcheck

```yaml
services:
  database:
    image: postgres:17-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s               # Prüfintervall
      timeout: 5s                 # Timeout pro Prüfung
      retries: 5                  # Anzahl Fehlversuche bis "unhealthy"
      start_period: 30s           # Wartezeit nach Container-Start
      start_interval: 5s          # Intervall während start_period

  api:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

### deploy

> [!NOTE] Swarm vs. Standalone
> Der `deploy`-Block stammt ursprünglich aus Docker Swarm, aber Compose v2 unterstützt `replicas` und `resources` auch im Standalone-Modus.

```yaml
services:
  worker:
    image: myapp-worker:latest
    deploy:
      replicas: 3                  # Anzahl paralleler Instanzen

      resources:
        limits:
          cpus: "2.0"             # Maximal 2 CPU-Kerne
          memory: 512M            # Maximal 512 MB RAM
        reservations:
          cpus: "0.5"             # Mindestens 0.5 CPU-Kerne
          memory: 128M            # Mindestens 128 MB RAM

      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```

### networks

```yaml
services:
  frontend:
    networks:
      - public
      - internal

  backend:
    networks:
      internal:
        aliases:
          - api                   # Zusätzlicher DNS-Name im Netzwerk
          - backend-service
        ipv4_address: 172.28.0.10 # Feste IP (erfordert Subnet-Konfiguration)

networks:
  public:
    driver: bridge

  internal:
    driver: bridge
    internal: true                # Kein Internetzugang
    ipam:
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
```

### labels

```yaml
services:
  webapp:
    labels:
      # Traefik Reverse Proxy Konfiguration
      traefik.enable: "true"
      traefik.http.routers.webapp.rule: "Host(`app.example.com`)"
      traefik.http.routers.webapp.tls: "true"

      # Eigene Metadaten
      com.example.description: "Meine Webanwendung"
      com.example.team: "platform"
```

### command / entrypoint

```yaml
services:
  app:
    image: python:3.13-slim
    # Entrypoint überschreiben (ersetzt ENTRYPOINT im Dockerfile)
    entrypoint: ["python", "-u"]

    # Command überschreiben (ersetzt CMD im Dockerfile)
    command: ["manage.py", "runserver", "0.0.0.0:8000"]

    # Shell-Form (wird über /bin/sh -c ausgeführt)
    command: python manage.py runserver 0.0.0.0:8000

  worker:
    image: myapp:latest
    command: >-
      sh -c "
        echo 'Warte auf Datenbank...' &&
        ./wait-for-it.sh db:5432 --timeout=30 &&
        python worker.py
      "
```

---

## Compose-Befehle

### Lifecycle-Management

| Befehl | Beschreibung |
|:-------|:-------------|
| `docker compose up` | Services erstellen und starten |
| `docker compose up -d` | Im Hintergrund starten (detached) |
| `docker compose up --build` | Images vor dem Start neu bauen |
| `docker compose up --force-recreate` | Container neu erstellen |
| `docker compose up --wait` | Warten bis alle Services healthy sind |
| `docker compose down` | Services stoppen und entfernen |
| `docker compose down -v` | Inklusive Volumes entfernen |
| `docker compose down --rmi all` | Inklusive Images entfernen |
| `docker compose start` | Gestoppte Services starten |
| `docker compose stop` | Services stoppen (ohne Entfernung) |
| `docker compose restart` | Services neustarten |
| `docker compose pause` | Services pausieren |
| `docker compose unpause` | Services fortsetzen |

### Informationen und Debugging

| Befehl | Beschreibung |
|:-------|:-------------|
| `docker compose ps` | Laufende Services anzeigen |
| `docker compose ps -a` | Alle Services (auch gestoppte) |
| `docker compose logs` | Logs aller Services |
| `docker compose logs -f` | Logs live verfolgen (follow) |
| `docker compose logs -f --tail=100 backend` | Letzte 100 Zeilen eines Services |
| `docker compose top` | Laufende Prozesse anzeigen |
| `docker compose config` | Compose-Datei validieren und anzeigen |
| `docker compose images` | Verwendete Images anzeigen |
| `docker compose port webapp 80` | Gemappten Port anzeigen |

### Build und Registry

| Befehl | Beschreibung |
|:-------|:-------------|
| `docker compose build` | Alle Images bauen |
| `docker compose build --no-cache` | Ohne Cache bauen |
| `docker compose build backend` | Nur ein bestimmtes Image bauen |
| `docker compose pull` | Images aus Registry ziehen |
| `docker compose push` | Gebaute Images in Registry pushen |

### Ausführung und Interaktion

| Befehl | Beschreibung |
|:-------|:-------------|
| `docker compose exec backend bash` | Shell in laufendem Service |
| `docker compose exec -u root backend bash` | Shell als Root |
| `docker compose run --rm backend python migrate.py` | Einmaliger Befehl in neuem Container |
| `docker compose cp backend:/app/log.txt ./log.txt` | Datei aus Container kopieren |

### Praxis-Workflows

```bash
# Projekt starten (erster Start oder nach Änderungen)
docker compose up -d --build --wait

# Logs eines bestimmten Services live verfolgen
docker compose logs -f --tail=50 backend

# In die Datenbank verbinden
docker compose exec database psql -U postgres -d app

# Alles herunterfahren und aufräumen (inkl. Volumes)
docker compose down -v --rmi local

# Nur einen Service neustarten
docker compose restart backend

# Compose-Datei auf Syntax prüfen
docker compose config --quiet && echo "OK" || echo "FEHLER"

# Services nach Änderung der compose.yaml aktualisieren
docker compose up -d --remove-orphans
```

---

## Umgebungsvariablen

### Die .env-Datei

Docker Compose lädt automatisch eine `.env`-Datei aus dem Projektverzeichnis. Diese Werte stehen als **Substitutions-Variablen** in der `compose.yaml` zur Verfügung.

```bash
# .env
COMPOSE_PROJECT_NAME=mein-projekt
POSTGRES_VERSION=17
APP_PORT=8080
NODE_ENV=production
DB_PASSWORD=supersecret
```

```yaml
# compose.yaml
services:
  database:
    image: postgres:${POSTGRES_VERSION:-16}   # Fallback auf 16
    ports:
      - "${APP_PORT}:80"

  backend:
    environment:
      NODE_ENV: ${NODE_ENV}
```

### Variable Substitution — Syntax

| Syntax | Beschreibung |
|:-------|:-------------|
| `${VAR}` | Wert der Variable (Fehler wenn nicht gesetzt) |
| `${VAR:-default}` | Default-Wert wenn nicht gesetzt oder leer |
| `${VAR-default}` | Default-Wert nur wenn nicht gesetzt |
| `${VAR:?Fehlermeldung}` | Fehler mit Nachricht wenn nicht gesetzt oder leer |
| `${VAR?Fehlermeldung}` | Fehler mit Nachricht nur wenn nicht gesetzt |
| `${VAR:+replacement}` | Ersatzwert wenn gesetzt und nicht leer |

```yaml
services:
  app:
    image: myapp:${APP_VERSION:-latest}
    ports:
      - "${HOST_PORT:-8080}:${CONTAINER_PORT:-80}"
    environment:
      DATABASE_URL: postgres://${DB_USER:?DB_USER muss gesetzt sein}:${DB_PASSWORD}@db:5432/${DB_NAME:-app}
```

### Prioritätsreihenfolge

Umgebungsvariablen werden in folgender Reihenfolge aufgelöst (höchste Priorität zuerst):

```mermaid
graph TD
    A["Höchste: docker compose run -e"] --> B["Host Shell-Umgebung"]
    B --> C[".env Datei"]
    C --> D["environment: im Compose-File"]
    D --> E["Niedrigste: Dockerfile ENV"]
    style A fill:#2d5016,stroke:#4a8c29,color:#ffffff
    style B fill:#1a3a5c,stroke:#2980b9,color:#ffffff
    style C fill:#1a3a5c,stroke:#2980b9,color:#ffffff
    style D fill:#5c3a1a,stroke:#b97829,color:#ffffff
    style E fill:#5c1a1a,stroke:#b92929,color:#ffffff
```

> [!WARNING] .env nicht in Git einchecken!
> Die `.env`-Datei enthält oft sensible Daten. Erstelle stattdessen eine `.env.example` als Vorlage und füge `.env` zur `.gitignore` hinzu.

```bash
# .env.example (wird eingecheckt)
POSTGRES_VERSION=17
APP_PORT=8080
DB_PASSWORD=           # <- Hier eigenes Passwort eintragen

# .gitignore
.env
.env.local
.env.production
```

---

## Multiple Compose Files

### Override-Pattern

Docker Compose unterstützt das automatische Zusammenführen mehrerer Dateien. Die `compose.override.yaml` wird automatisch geladen, wenn sie existiert.

```
projekt/
├── compose.yaml               # Basis-Konfiguration
├── compose.override.yaml      # Lokale Entwickler-Overrides (auto-geladen)
├── compose.prod.yaml          # Produktions-Overrides
├── compose.test.yaml          # Test-Overrides
├── .env                       # Umgebungsvariablen
└── .env.example               # Vorlage für .env
```

```yaml
# compose.yaml — Basis
services:
  backend:
    image: myapp:latest
    environment:
      NODE_ENV: production
    restart: unless-stopped
```

```yaml
# compose.override.yaml — Entwicklung (wird automatisch geladen)
services:
  backend:
    build:
      context: ./backend         # Lokal bauen statt Image
    volumes:
      - ./backend/src:/app/src   # Live-Reload
    environment:
      NODE_ENV: development
      DEBUG: "true"
    ports:
      - "9229:9229"              # Debugger-Port
    restart: "no"
```

```yaml
# compose.prod.yaml — Produktion
services:
  backend:
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### Manuelle Dateiauswahl mit -f

```bash
# Nur Basis (override wird NICHT geladen bei -f)
docker compose -f compose.yaml up -d

# Basis + Produktion
docker compose -f compose.yaml -f compose.prod.yaml up -d

# Basis + Test
docker compose -f compose.yaml -f compose.test.yaml up -d

# Validierung der zusammengeführten Konfiguration
docker compose -f compose.yaml -f compose.prod.yaml config
```

> [!NOTE] Merge-Verhalten
> Beim Zusammenführen werden Werte aus späteren Dateien **überschrieben** oder **ergänzt**:
> - Skalare Werte (strings, numbers) → überschrieben
> - Listen (`ports`, `volumes`) → zusammengeführt
> - Maps (`environment`, `labels`) → zusammengeführt, bei Duplikaten überschrieben

### Profiles

Profiles ermöglichen es, bestimmte Services nur bei Bedarf zu starten:

```yaml
services:
  # ── Immer aktiv ───────────────────────────────────────
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"

  backend:
    image: myapp:latest

  database:
    image: postgres:17-alpine

  # ── Nur mit Profil "debug" ────────────────────────────
  adminer:
    image: adminer:latest
    ports:
      - "8081:8080"
    profiles:
      - debug

  mailhog:
    image: mailhog/mailhog
    ports:
      - "8025:8025"
    profiles:
      - debug

  # ── Nur mit Profil "monitoring" ───────────────────────
  prometheus:
    image: prom/prometheus
    profiles:
      - monitoring

  grafana:
    image: grafana/grafana
    profiles:
      - monitoring
```

```bash
# Nur Kern-Services starten (ohne Profile)
docker compose up -d

# Mit Debug-Tools starten
docker compose --profile debug up -d

# Mehrere Profile gleichzeitig
docker compose --profile debug --profile monitoring up -d

# Profil über Umgebungsvariable
COMPOSE_PROFILES=debug,monitoring docker compose up -d
```

### COMPOSE_PROJECT_NAME

Der Projektname beeinflusst die Benennung von Containern, Volumes und Netzwerken:

```bash
# Standard: Verzeichnisname
# Container: myproject-backend-1

# Über .env
COMPOSE_PROJECT_NAME=myapp
# Container: myapp-backend-1

# Über CLI-Flag
docker compose -p myapp up -d
```

---

## Best Practices

> [!TIP] Empfehlungen für Produktionsumgebungen
> - **Immer spezifische Image-Tags** verwenden (`postgres:17-alpine` statt `postgres:latest`)
> - **Healthchecks** für alle Services definieren
> - **Resource Limits** setzen, um Ressourcen-Hunger zu verhindern
> - **Read-only** Bind Mounts wo möglich (`:ro`)
> - **Named Volumes** statt Bind Mounts für persistente Daten
> - **Interne Netzwerke** für Backend-Kommunikation (`internal: true`)
> - **`.env.example`** als Vorlage pflegen, `.env` nie einchecken
> - **Logging-Limits** setzen, um Festplatten-Overflow zu vermeiden
> - **`restart: unless-stopped`** für alle Produktions-Services

> [!WARNING] Häufige Fehler
> - `latest`-Tag verwenden → Builds sind nicht reproduzierbar
> - Ports auf `0.0.0.0` binden → Dienste ungewollt im Netzwerk erreichbar
> - Kein `depends_on` mit Healthcheck → Race Conditions beim Start
> - Volumes vergessen → Datenverlust bei `docker compose down`
> - `.env`-Datei in Git einchecken → Credentials im Repository

---

## Troubleshooting

### Port-Konflikte

```bash
# Prüfen welcher Prozess einen Port belegt
sudo lsof -i :8080
sudo ss -tlnp | grep 8080

# Container mit anderem Port starten
APP_PORT=8081 docker compose up -d
```

### Container startet nicht

```bash
# Logs des fehlerhaften Services prüfen
docker compose logs --tail=50 backend

# Container-Details anzeigen
docker compose ps -a

# Interaktiv starten für Debugging
docker compose run --rm backend bash
```

### Volume-Berechtigungen

```bash
# User-ID im Container prüfen
docker compose exec backend id

# Host-Verzeichnis-Berechtigungen anpassen
sudo chown -R 1000:1000 ./data
```

### Compose-Datei validieren

```bash
# Syntax prüfen und aufgelöste Konfiguration anzeigen
docker compose config

# Nur Fehler anzeigen
docker compose config --quiet

# Interpolierte Variablen prüfen
docker compose config | grep -i password
```

---

## Verwandte Notizen

- [[Docker Index]]
- [[Docker Compose Beispiele]]
- [[Container Lifecycle]]
- [[Docker Volumes und Storage]]
- [[Docker Networking]]
- [[Kubernetes vs Docker und Docker Swarm]] — Von Compose zu Kubernetes-Orchestrierung
- [[Swarm Stacks]] — Docker Stacks nutzen Compose-Dateien für Swarm-Deployments