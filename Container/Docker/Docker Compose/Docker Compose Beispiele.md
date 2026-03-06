---
tags:
  - docker
  - docker-compose
  - beispiele
  - praxis
---

# Docker Compose Beispiele

> [!ABSTRACT]
> Praktische, vollständig funktionierende Docker Compose Konfigurationen — vom einfachen Webserver über klassische Web-Stacks bis hin zu Monitoring und Reverse Proxy mit automatischem HTTPS. Jedes Beispiel ist sofort einsatzbereit.

---

## Einfacher Webserver

Ein minimales Setup: **Nginx** liefert eine statische Website aus. Das HTML-Verzeichnis wird als Bind Mount eingebunden, sodass Änderungen sofort sichtbar sind.

```yaml
# compose.yaml
services:
  web:
    image: nginx:alpine
    container_name: nginx-static
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    restart: unless-stopped
```

> [!TIP] Einstieg
> Das einfachste Compose-Beispiel überhaupt — ideal zum Testen, ob Docker Compose korrekt funktioniert.

```bash
# Verzeichnis und Testseite anlegen
mkdir -p html
echo '<h1>Hello from Docker Compose!</h1>' > html/index.html

# Stack starten
docker compose up -d

# Im Browser öffnen: http://localhost:8080

# Stack stoppen und aufräumen
docker compose down
```

---

## WordPress mit MariaDB

Der Klassiker: **WordPress** mit **MariaDB** als Datenbank. Persistente Volumes sichern Datenbank und WordPress-Dateien. Sensible Werte werden über eine `.env`-Datei ausgelagert.

```yaml
# compose.yaml
services:
  db:
    image: mariadb:11
    container_name: wp-mariadb
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    networks:
      - wp-network
    restart: unless-stopped

  wordpress:
    image: wordpress:6-apache
    container_name: wp-app
    depends_on:
      - db
    ports:
      - "8080:80"
    volumes:
      - wp_data:/var/www/html
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${DB_NAME}
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
    networks:
      - wp-network
    restart: unless-stopped

volumes:
  db_data:
  wp_data:

networks:
  wp-network:
    driver: bridge
```

```bash
# .env-Datei anlegen (nie in Git committen!)
cat > .env << 'EOF'
DB_ROOT_PASSWORD=supersecret-root-pw
DB_NAME=wordpress
DB_USER=wp_user
DB_PASSWORD=supersecret-user-pw
EOF

# Stack starten
docker compose up -d
```

> [!WARNING] Sicherheit
> Die `.env`-Datei gehört in die `.gitignore`. Passwörter niemals direkt in die `compose.yaml` schreiben.

---

## Node.js mit MongoDB

Eine **Express-API** mit **MongoDB** als Datenbank und **Mongo Express** als Web-basierte Admin-Oberfläche.

```yaml
# compose.yaml
services:
  api:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: node-api
    ports:
      - "3000:3000"
    environment:
      MONGO_URI: mongodb://mongo:27017/myapp
      NODE_ENV: production
    depends_on:
      - mongo
    networks:
      - app-network
    restart: unless-stopped

  mongo:
    image: mongo:7
    container_name: mongodb
    volumes:
      - mongo_data:/data/db
    environment:
      MONGO_INITDB_DATABASE: myapp
    networks:
      - app-network
    restart: unless-stopped

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-ui
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_URL: mongodb://mongo:27017/
      ME_CONFIG_BASICAUTH: "false"
    depends_on:
      - mongo
    networks:
      - app-network
    restart: unless-stopped

volumes:
  mongo_data:

networks:
  app-network:
    driver: bridge
```

```bash
# Stack starten (baut das Image bei Bedarf)
docker compose up -d --build

# API testen
curl http://localhost:3000

# Mongo Express UI: http://localhost:8081
```

---

## Python Flask mit PostgreSQL und Redis

Ein vollständiger **Python-Stack**: **Flask** als Web-Framework, **PostgreSQL** als Datenbank, **Redis** als Cache und **pgAdmin** zur Datenbankverwaltung. Mit Health Checks für zuverlässiges Startup.

```yaml
# compose.yaml
services:
  app:
    build:
      context: ./flask-app
      dockerfile: Dockerfile
    container_name: flask-app
    ports:
      - "5000:5000"
    environment:
      DATABASE_URL: postgresql://flask_user:flask_pass@postgres:5432/flask_db
      REDIS_URL: redis://redis:6379/0
      FLASK_ENV: production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    container_name: flask-postgres
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    environment:
      POSTGRES_DB: flask_db
      POSTGRES_USER: flask_user
      POSTGRES_PASSWORD: flask_pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U flask_user -d flask_db"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - backend
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: flask-redis
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: flask-pgadmin
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
      PGADMIN_CONFIG_SERVER_MODE: "False"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

volumes:
  pg_data:
  redis_data:

networks:
  backend:
    driver: bridge
```

> [!TIP] Health Checks
> Durch `condition: service_healthy` wartet Flask, bis PostgreSQL und Redis tatsächlich bereit sind — nicht nur gestartet. Das verhindert Verbindungsfehler beim Start.

```bash
# Stack starten
docker compose up -d

# Health-Status aller Services prüfen
docker compose ps

# pgAdmin: http://localhost:5050
# Flask-App: http://localhost:5000
```

---

## NGINX Reverse Proxy

**Nginx** als Reverse Proxy vor mehreren Backend-Services.

```yaml
# compose.yaml
services:
  proxy:
    image: nginx:alpine
    container_name: reverse-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    depends_on:
      - app1
      - app2
    networks:
      - proxy-network
    restart: unless-stopped

  app1:
    image: hashicorp/http-echo
    container_name: backend-app1
    command: ["-text", "Antwort von App 1"]
    networks:
      - proxy-network
    restart: unless-stopped

  app2:
    image: hashicorp/http-echo
    container_name: backend-app2
    command: ["-text", "Antwort von App 2"]
    networks:
      - proxy-network
    restart: unless-stopped

networks:
  proxy-network:
    driver: bridge
```

```nginx
# nginx/conf.d/default.conf
upstream app1_backend {
    server app1:5678;
}

upstream app2_backend {
    server app2:5678;
}

server {
    listen 80;

    location /app1 {
        proxy_pass http://app1_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /app2 {
        proxy_pass http://app2_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
# Testen
curl http://localhost/app1    # → "Antwort von App 1"
curl http://localhost/app2    # → "Antwort von App 2"
```

---

## Monitoring Stack

**Prometheus** sammelt Metriken, **Node Exporter** liefert Host-Metriken und **Grafana** visualisiert alles.

```yaml
# compose.yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"
      - "--web.enable-lifecycle"
    networks:
      - monitoring
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: "false"
    depends_on:
      - prometheus
    networks:
      - monitoring
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

> [!TIP] Grafana Dashboard
> Grafana unter `http://localhost:3000` (Login: `admin`/`admin`). Prometheus als Data Source hinzufügen (`http://prometheus:9090`), dann Dashboard **1860** (*Node Exporter Full*) importieren.

---

## Traefik mit automatischem HTTPS

**Traefik v3** als Reverse Proxy mit automatischer **Let's Encrypt**-Zertifikatsverwaltung. Neue Services werden über Docker Labels automatisch erkannt.

```yaml
# compose.yaml
services:
  traefik:
    image: traefik:v3.4
    container_name: traefik
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt_data:/letsencrypt
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
    networks:
      - traefik-public
    restart: unless-stopped

  whoami:
    image: traefik/whoami
    container_name: whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
    networks:
      - traefik-public
    restart: unless-stopped

volumes:
  letsencrypt_data:

networks:
  traefik-public:
    driver: bridge
```

> [!IMPORTANT] Voraussetzungen für Let's Encrypt
> Die Domain muss per DNS auf die öffentliche IP des Servers zeigen, Port 80 und 443 müssen erreichbar sein, und die E-Mail-Adresse muss gültig sein.

> [!TIP] Weitere Services anbinden
> Jeder neue Service braucht nur drei Labels:
> ```yaml
> labels:
>   - "traefik.enable=true"
>   - "traefik.http.routers.SERVICENAME.rule=Host(`sub.example.com`)"
>   - "traefik.http.routers.SERVICENAME.tls.certresolver=letsencrypt"
> ```

---

## Entwicklungsumgebung

Setup optimiert für **lokale Entwicklung**: Bind Mounts für Hot-Reload, Debug-Ports und Override-Dateien.

```yaml
# compose.yaml — Basis-Konfiguration (Produktion)
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: dev-app
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
    networks:
      - dev-network
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    container_name: dev-db
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: devdb
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpass
    networks:
      - dev-network
    restart: unless-stopped

volumes:
  db_data:

networks:
  dev-network:
    driver: bridge
```

```yaml
# compose.override.yaml — Wird automatisch geladen
services:
  app:
    build:
      target: development
    volumes:
      - ./src:/app/src:cached
      - /app/node_modules
    environment:
      NODE_ENV: development
      DEBUG: "app:*"
    ports:
      - "3000:3000"
      - "9229:9229"   # Node.js Debug-Port
    command: npm run dev

  db:
    ports:
      - "5432:5432"   # DB-Port nur in Entwicklung exposen
```

> [!NOTE] Override-Mechanismus
> Docker Compose lädt automatisch `compose.override.yaml`, wenn sie neben `compose.yaml` liegt. Die Override-Datei wird **zusammengeführt** (merged), nicht ersetzt.

```bash
# Entwicklung starten (lädt automatisch compose.override.yaml)
docker compose up -d --build

# Produktion starten (Override ignorieren)
docker compose -f compose.yaml up -d --build

# Alles stoppen und Volumes löschen
docker compose down -v
```

---

## Verwandte Notizen

- [[Docker Index]]
- [[Docker Compose]]
- [[Docker Volumes und Storage]]
- [[Docker Networking]]
