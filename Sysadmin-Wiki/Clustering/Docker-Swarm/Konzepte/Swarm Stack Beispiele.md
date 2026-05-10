---
tags:
  - docker-swarm
  - stacks
  - beispiele
  - compose
  - praxis
---

# Swarm Stack Beispiele

> [!ABSTRACT] TL;DR
> Praxisnahe, produktionsreife Stack-Beispiele für Docker Swarm — vom einfachen Web-Stack über Monitoring mit Prometheus/Grafana bis hin zu komplexen Multi-Service-Anwendungen. Alle Beispiele sind direkt deploybar und enthalten Health Checks, Placement Constraints und Ressourcen-Limits.

---

## Grundstruktur eines Production-Stacks

Jeder produktionsreife Stack sollte diese Elemente enthalten:

```yaml
services:
  app:
    image: registry.example.com/app:1.2.3
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.50'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M
      placement:
        constraints:
          - node.role == worker
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - app-net
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

networks:
  app-net:
    driver: overlay
    attachable: true
```

> [!TIP] Immer feste Image-Tags verwenden
> Verwende niemals `:latest` in Produktion. Nutze immer einen konkreten Tag wie `:1.2.3` oder einen SHA-Digest wie `@sha256:abc...`, um reproduzierbare Deployments sicherzustellen.

---

## Beispiel 1: Traefik Reverse Proxy + Web-App

Ein häufiges Setup mit Traefik als Ingress-Router und einer Web-Anwendung dahinter.

### Stack-Datei

```yaml
# traefik-stack.yml
services:
  traefik:
    image: traefik:v3.2
    command:
      - "--providers.swarm=true"
      - "--providers.swarm.exposedByDefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--api.dashboard=true"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-certs:/letsencrypt
    networks:
      - proxy-net
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`)"
        - "traefik.http.routers.dashboard.service=api@internal"
        - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
        - "traefik.http.services.dashboard.loadbalancer.server.port=8080"

  webapp:
    image: registry.example.com/webapp:2.1.0
    environment:
      - DATABASE_URL=postgres://db-user:db-pass@db:5432/webapp
    networks:
      - proxy-net
      - backend-net
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.webapp.rule=Host(`app.example.com`)"
        - "traefik.http.routers.webapp.tls.certresolver=letsencrypt"
        - "traefik.http.services.webapp.loadbalancer.server.port=8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: webapp
      POSTGRES_USER: db-user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend-net
    secrets:
      - db_password
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd
      resources:
        limits:
          memory: 1G

networks:
  proxy-net:
    driver: overlay
  backend-net:
    driver: overlay
    internal: true

volumes:
  traefik-certs:
  db-data:

secrets:
  db_password:
    external: true
```

### Deployment

```bash
# Secret erstellen
echo "sicheres-passwort" | docker secret create db_password -

# Stack deployen
docker stack deploy -c traefik-stack.yml webapp

# Status prüfen
docker stack services webapp
docker service logs webapp_traefik
```

> [!NOTE] Backend-Netzwerk als `internal`
> Das `backend-net` ist als `internal: true` markiert — Container in diesem Netzwerk haben keinen direkten Internetzugang. Die Datenbank ist nur über das interne Netzwerk erreichbar.

---

## Beispiel 2: Monitoring Stack (Prometheus + Grafana)

Vollständiger Monitoring-Stack mit Prometheus, Grafana, Node Exporter und cAdvisor.

### Stack-Datei

```yaml
# monitoring-stack.yml
services:
  prometheus:
    image: prom/prometheus:v2.53.0
    volumes:
      - prometheus-data:/prometheus
    configs:
      - source: prometheus-config
        target: /etc/prometheus/prometheus.yml
    networks:
      - monitoring-net
      - proxy-net
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 2G
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.prometheus.rule=Host(`prometheus.example.com`)"
        - "traefik.http.services.prometheus.loadbalancer.server.port=9090"

  grafana:
    image: grafana/grafana:11.1.0
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin_pw
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-piechart-panel
    volumes:
      - grafana-data:/var/lib/grafana
    secrets:
      - grafana_admin_pw
    networks:
      - monitoring-net
      - proxy-net
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 512M
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.grafana.rule=Host(`grafana.example.com`)"
        - "traefik.http.services.grafana.loadbalancer.server.port=3000"

  node-exporter:
    image: prom/node-exporter:v1.8.1
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.rootfs=/rootfs"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    networks:
      - monitoring-net
    deploy:
      mode: global
      resources:
        limits:
          memory: 128M

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    networks:
      - monitoring-net
    deploy:
      mode: global
      resources:
        limits:
          memory: 256M

  alertmanager:
    image: prom/alertmanager:v0.27.0
    configs:
      - source: alertmanager-config
        target: /etc/alertmanager/alertmanager.yml
    networks:
      - monitoring-net
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 128M

networks:
  monitoring-net:
    driver: overlay
  proxy-net:
    external: true

volumes:
  prometheus-data:
  grafana-data:

configs:
  prometheus-config:
    file: ./prometheus.yml
  alertmanager-config:
    file: ./alertmanager.yml

secrets:
  grafana_admin_pw:
    external: true
```

### Prometheus-Konfiguration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    dns_sd_configs:
      - names:
          - 'tasks.node-exporter'
        type: 'A'
        port: 9100

  - job_name: 'cadvisor'
    dns_sd_configs:
      - names:
          - 'tasks.cadvisor'
        type: 'A'
        port: 8080

  - job_name: 'docker-engine'
    static_configs:
      - targets: ['172.17.0.1:9323']
```

### Deployment

```bash
# Secret erstellen
echo "admin-passwort" | docker secret create grafana_admin_pw -

# Stack deployen
docker stack deploy -c monitoring-stack.yml monitoring

# Prüfen ob alle Services laufen
docker stack services monitoring
```

> [!TIP] `mode: global` für Exporter
> Node Exporter und cAdvisor verwenden `mode: global` — so läuft automatisch eine Instanz auf **jedem** Node im Cluster, auch wenn neue Nodes hinzukommen.

---

## Beispiel 3: WordPress mit MySQL und Redis

Klassischer LAMP-Stack mit Caching-Layer.

```yaml
# wordpress-stack.yml
services:
  wordpress:
    image: wordpress:6.5-php8.3-apache
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/wp_db_password
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_REDIS_HOST', 'redis');
        define('WP_REDIS_PORT', 6379);
    volumes:
      - wp-content:/var/www/html/wp-content
    secrets:
      - wp_db_password
    networks:
      - frontend
      - backend
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 30s
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.wordpress.rule=Host(`blog.example.com`)"
        - "traefik.http.services.wordpress.loadbalancer.server.port=80"
        - "traefik.http.services.wordpress.loadbalancer.sticky.cookie=true"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/"]
      interval: 30s
      timeout: 10s
      retries: 3

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp_user
      MYSQL_PASSWORD_FILE: /run/secrets/wp_db_password
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
    volumes:
      - mysql-data:/var/lib/mysql
    secrets:
      - wp_db_password
      - mysql_root_password
    networks:
      - backend
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd
      resources:
        limits:
          memory: 1G

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    networks:
      - backend
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 256M
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true

volumes:
  wp-content:
  mysql-data:
  redis-data:

secrets:
  wp_db_password:
    external: true
  mysql_root_password:
    external: true
```

```bash
# Secrets erstellen
echo "wp-db-passwort" | docker secret create wp_db_password -
echo "root-passwort" | docker secret create mysql_root_password -

# Deployen
docker stack deploy -c wordpress-stack.yml blog

# WordPress Sticky Sessions prüfen
docker service inspect blog_wordpress --pretty
```

> [!WARNING] Sticky Sessions bei WordPress
> WordPress speichert Session-Daten lokal. Bei mehreren Replicas ist `sticky.cookie=true` im Traefik-Label essentiell, damit Benutzer immer auf denselben Container geroutet werden — oder besser: ein Session-Plugin mit Redis verwenden.

---

## Beispiel 4: GitLab Runner Stack

GitLab CI/CD Runner als Swarm-Service für automatisierte Builds.

```yaml
# gitlab-runner-stack.yml
services:
  runner:
    image: gitlab/gitlab-runner:v17.2.0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - runner-config:/etc/gitlab-runner
    configs:
      - source: runner-config-toml
        target: /etc/gitlab-runner/config.toml
    networks:
      - runner-net
    deploy:
      mode: global
      placement:
        constraints:
          - node.labels.runner == true
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
    healthcheck:
      test: ["CMD", "gitlab-runner", "verify"]
      interval: 60s
      timeout: 10s
      retries: 3

networks:
  runner-net:
    driver: overlay

volumes:
  runner-config:

configs:
  runner-config-toml:
    file: ./runner-config.toml
```

```bash
# Node labeln für Runner-Platzierung
docker node update --label-add runner=true worker-01
docker node update --label-add runner=true worker-02

# Stack deployen
docker stack deploy -c gitlab-runner-stack.yml cicd
```

---

## Beispiel 5: Logging Stack (Loki + Promtail)

Zentrales Log-Management mit Grafana Loki.

```yaml
# logging-stack.yml
services:
  loki:
    image: grafana/loki:3.1.0
    volumes:
      - loki-data:/loki
    configs:
      - source: loki-config
        target: /etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - logging-net
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 1G

  promtail:
    image: grafana/promtail:3.1.0
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    configs:
      - source: promtail-config
        target: /etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    networks:
      - logging-net
    deploy:
      mode: global
      resources:
        limits:
          memory: 128M

networks:
  logging-net:
    driver: overlay
    attachable: true

volumes:
  loki-data:

configs:
  loki-config:
    file: ./loki-config.yaml
  promtail-config:
    file: ./promtail-config.yml
```

---

## Nützliche Deployment-Patterns

### Multi-Environment mit `.env`-Dateien

```bash
# Verschiedene Umgebungen mit Environment-Variablen
ENVIRONMENT=staging docker stack deploy \
  -c base-stack.yml \
  -c staging-overrides.yml \
  myapp-staging

ENVIRONMENT=production docker stack deploy \
  -c base-stack.yml \
  -c production-overrides.yml \
  myapp-production
```

### Compose-File-Merging

Docker Swarm unterstützt mehrere `-c` Flags zum Zusammenführen:

```bash
# Basis + Umgebung + Secrets
docker stack deploy \
  -c docker-compose.yml \
  -c docker-compose.prod.yml \
  -c docker-compose.secrets.yml \
  myapp
```

```yaml
# docker-compose.yml (Basis)
services:
  app:
    image: myapp:latest
    networks:
      - app-net

# docker-compose.prod.yml (Produktion-Overrides)
services:
  app:
    deploy:
      replicas: 5
      resources:
        limits:
          memory: 1G
    environment:
      - LOG_LEVEL=warn
```

### Deployment mit Config-Versionierung

```bash
# Config mit Versionierung — Änderung erzwingt Service-Neustart
docker config create nginx-config-v1 nginx.conf

# Neue Version deployen
docker config create nginx-config-v2 nginx-updated.conf
# Stack-Datei anpassen und redeployen
docker stack deploy -c stack.yml myapp
```

---

## Checkliste: Stack für Produktion

| Punkt | Prüfung |
|-------|---------|
| ✅ Feste Image-Tags | Kein `:latest`, immer z.B. `:1.2.3` |
| ✅ Health Checks | Jeder Service hat `healthcheck` definiert |
| ✅ Ressourcen-Limits | `resources.limits` für CPU und Memory |
| ✅ Update-Strategie | `update_config` mit `failure_action: rollback` |
| ✅ Secrets statt env | Passwörter über `secrets`, nicht `environment` |
| ✅ Interne Netzwerke | Backend-Netzwerke als `internal: true` |
| ✅ Logging konfiguriert | `logging.options` mit `max-size` und `max-file` |
| ✅ Placement Constraints | Datenbanken auf Storage-Nodes, Manager geschützt |
| ✅ Rollback-Config | `rollback_config` definiert |
| ✅ Restart-Policy | `restart_policy` mit `max_attempts` |

---

## Verwandte Notizen

- [[Swarm Stacks]] — Stack-Konzepte und Direktiven
- [[Swarm Services und Tasks]] — Service-Konfiguration im Detail
- [[Swarm Secrets und Configs]] — Secrets und Configs verwalten
- [[Swarm Networking]] — Overlay-Netzwerke und Routing
- [[Swarm Best Practices]] — Produktions-Checkliste
- [[Docker Compose Beispiele]] — Beispiele für Docker Compose (ohne Swarm)
- [[Swarm CI-CD Integration]] — Automatisiertes Deployment
