---
tags: [podman, beispiele, praxis, nginx, postgresql, wordpress, dev-container, quadlet, compose, kubernetes]
aliases: ["Podman Beispiele", "Podman Praxis", "Praktische Beispiele"]
---

# Podman Praktische Beispiele

> [!ABSTRACT]
> Sechs vollständige, praxistaugliche Beispiele: vom einfachen Nginx-Webserver als Quadlet-Service über eine PostgreSQL-Datenbank bis zu einem WordPress-Pod mit drei Containern und einer Docker Compose → Quadlet Migration. Alle Beispiele sind rootless-ready.

---

## Beispiel 1: Nginx als rootless Systemd-Service (Quadlet)

Nginx als Webserver mit eigenem Netzwerk, Volumes und Auto-Update — komplett rootless als systemd-User-Service.

### Verzeichnisse vorbereiten

```bash
mkdir -p ~/.config/containers/systemd/
mkdir -p ~/nginx/html
mkdir -p ~/nginx/conf.d

echo '<h1>Hallo von Podman!</h1>' > ~/nginx/html/index.html
```

### Konfiguration

```ini
# ~/.config/containers/systemd/nginx.container
[Unit]
Description=Nginx Webserver (rootless)
After=network-online.target

[Container]
Image=docker.io/library/nginx:alpine
PublishPort=8080:80
Volume=%h/nginx/html:/usr/share/nginx/html:ro,z
Volume=%h/nginx/conf.d:/etc/nginx/conf.d:ro,z
Environment=NGINX_HOST=localhost
Label=app=webserver
AutoUpdate=registry
Hostname=nginx-server

[Service]
Restart=always
RestartSec=5s

[Install]
WantedBy=default.target
```

### Aktivieren

```bash
systemctl --user daemon-reload
systemctl --user enable --now nginx.service

# Testen
curl http://localhost:8080

# Logs
journalctl --user -u nginx.service -f
```

---

## Beispiel 2: PostgreSQL mit Named Volume

Produktionsreife PostgreSQL-Datenbank mit persistentem Volume und Health Check.

```bash
# Volume erstellen
podman volume create pgdata

# PostgreSQL starten
podman run -d \
  --name postgres \
  --restart always \
  -e POSTGRES_DB=mydb \
  -e POSTGRES_USER=dbuser \
  -e POSTGRES_PASSWORD=sicherespasswort \
  -v pgdata:/var/lib/postgresql/data:Z \
  -p 127.0.0.1:5432:5432 \
  --health-cmd "pg_isready -U dbuser -d mydb" \
  --health-interval 10s \
  --health-timeout 3s \
  --health-retries 5 \
  docker.io/library/postgres:16-alpine

# Health-Status prüfen
podman ps --format "{{.Names}}: {{.Status}}"
# postgres: Up 30 seconds (healthy)
```

### Als Quadlet

```ini
# ~/.config/containers/systemd/postgres.container
[Unit]
Description=PostgreSQL Datenbank
After=network-online.target

[Container]
Image=docker.io/library/postgres:16-alpine
PublishPort=127.0.0.1:5432:5432
Volume=pgdata.volume:/var/lib/postgresql/data:Z
Environment=POSTGRES_DB=mydb
Environment=POSTGRES_USER=dbuser
Secret=pg-password,type=env,target=POSTGRES_PASSWORD
HealthCmd=pg_isready -U dbuser -d mydb
HealthInterval=10s
HealthTimeout=3s
HealthRetries=5

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/pgdata.volume
[Volume]
Label=app=postgres
```

```bash
# Secret anlegen
echo "sicherespasswort" | podman secret create pg-password -

# Verbindung testen
podman exec -it postgres psql -U dbuser -d mydb -c "SELECT version();"
```

---

## Beispiel 3: WordPress-Pod (3 Container)

Vollständiger WordPress-Stack als Podman-Pod: MariaDB + WordPress + Nginx-Proxy.

```bash
# Pod erstellen
podman pod create \
  --name wordpress \
  -p 8080:80

# Netzwerk: Alle Container im gleichen Pod teilen localhost!
```

```bash
# MariaDB starten (im Pod)
podman run -d \
  --name wp-db \
  --pod wordpress \
  -e MARIADB_ROOT_PASSWORD=rootpass \
  -e MARIADB_DATABASE=wordpress \
  -e MARIADB_USER=wpuser \
  -e MARIADB_PASSWORD=wppass \
  -v wp-db-data:/var/lib/mysql:Z \
  docker.io/library/mariadb:11

# WordPress starten (verbindet sich zu 127.0.0.1:3306 — gleicher Pod!)
podman run -d \
  --name wp-app \
  --pod wordpress \
  -e WORDPRESS_DB_HOST=127.0.0.1:3306 \
  -e WORDPRESS_DB_NAME=wordpress \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD=wppass \
  -v wp-app-data:/var/www/html:Z \
  docker.io/library/wordpress:latest

# Pod-Status
podman pod ps
podman ps --pod
```

### Als Quadlet-Pod

```ini
# ~/.config/containers/systemd/wordpress.pod
[Unit]
Description=WordPress Pod

[Pod]
PublishPort=8080:80

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/wp-db.container
[Unit]
Description=WordPress MariaDB
After=wordpress-pod.service

[Container]
Image=docker.io/library/mariadb:11
Pod=wordpress.pod
Environment=MARIADB_DATABASE=wordpress
Environment=MARIADB_USER=wpuser
Environment=MARIADB_ROOT_PASSWORD=rootpass
Secret=wp-db-password,type=env,target=MARIADB_PASSWORD
Volume=wp-db.volume:/var/lib/mysql:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/wp-app.container
[Unit]
Description=WordPress App
After=wp-db.service

[Container]
Image=docker.io/library/wordpress:latest
Pod=wordpress.pod
Environment=WORDPRESS_DB_HOST=127.0.0.1:3306
Environment=WORDPRESS_DB_NAME=wordpress
Environment=WORDPRESS_DB_USER=wpuser
Secret=wp-db-password,type=env,target=WORDPRESS_DB_PASSWORD
Volume=wp-app.volume:/var/www/html:Z
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

---

## Beispiel 4: Dev-Container mit Host-Mount

Entwicklungsumgebung: Node.js-App wird im Container gebaut und gestartet, Quellcode liegt auf dem Host.

```bash
mkdir -p ~/projects/myapp
cd ~/projects/myapp

# package.json anlegen
cat > package.json << 'EOF'
{
  "name": "myapp",
  "version": "1.0.0",
  "scripts": { "start": "node server.js", "dev": "nodemon server.js" },
  "dependencies": { "express": "^4.18.0" },
  "devDependencies": { "nodemon": "^3.0.0" }
}
EOF

# Dev-Container starten (Hot-Reload via nodemon)
podman run -it --rm \
  --name myapp-dev \
  -v $(pwd):/app:Z \
  -w /app \
  -p 3000:3000 \
  -e NODE_ENV=development \
  docker.io/library/node:20-alpine \
  sh -c "npm install && npm run dev"
```

### Mit podman-compose für Dev

```yaml
# docker-compose.dev.yml
version: "3.9"
services:
  app:
    image: docker.io/library/node:20-alpine
    working_dir: /app
    volumes:
      - .:/app:Z
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    command: sh -c "npm install && npm run dev"

  db:
    image: docker.io/library/postgres:16-alpine
    environment:
      POSTGRES_DB: devdb
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
    volumes:
      - dev-pgdata:/var/lib/postgresql/data:Z
    ports:
      - "5432:5432"

volumes:
  dev-pgdata:
```

```bash
podman compose -f docker-compose.dev.yml up
```

---

## Beispiel 5: Docker Compose → Quadlet Migration

Schrittweise Migration einer bestehenden docker-compose.yml zu Quadlet-Units.

### Ausgangszustand: docker-compose.yml

```yaml
version: "3.9"
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/usr/share/nginx/html:ro
      - certdata:/etc/letsencrypt
    restart: unless-stopped
    depends_on:
      - app

  app:
    image: myapp:1.0
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    volumes:
      - appdata:/var/www/html
    restart: unless-stopped

volumes:
  certdata:
  appdata:
```

### Migrierter Quadlet

```ini
# ~/.config/containers/systemd/prod.network
[Network]
Subnet=172.20.0.0/16
```

```ini
# ~/.config/containers/systemd/certdata.volume
[Volume]
```

```ini
# ~/.config/containers/systemd/appdata.volume
[Volume]
```

```ini
# ~/.config/containers/systemd/app.container
[Unit]
Description=Anwendung Backend
After=network-online.target

[Container]
Image=myapp:1.0
Network=prod.network
Environment=DATABASE_URL=postgresql://user:pass@db:5432/mydb
Environment=REDIS_URL=redis://cache:6379
Volume=appdata.volume:/var/www/html:Z
AutoUpdate=local

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/nginx.container
[Unit]
Description=Nginx Reverse Proxy
After=app.service

[Container]
Image=docker.io/library/nginx:alpine
PublishPort=8080:80
PublishPort=8443:443
Network=prod.network
Volume=%h/nginx/nginx.conf:/etc/nginx/nginx.conf:ro,z
Volume=%h/nginx/html:/usr/share/nginx/html:ro,z
Volume=certdata.volume:/etc/letsencrypt:Z
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
# Aktivieren
systemctl --user daemon-reload
systemctl --user enable --now app.service nginx.service
systemctl --user status app.service nginx.service
```

---

## Beispiel 6: Lokales Kubernetes-Testing mit `podman play kube`

Kubernetes-YAML lokal mit Podman ausführen — kein Minikube nötig für einfache Tests.

```yaml
# webapp.yaml — Kubernetes-Manifest
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  containers:
    - name: nginx
      image: docker.io/library/nginx:alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      resources:
        limits:
          memory: "128Mi"
          cpu: "0.5"

    - name: sidecar
      image: docker.io/library/busybox:latest
      command: ["sh", "-c", "while true; do date > /html/time.txt; sleep 5; done"]
      volumeMounts:
        - name: html
          mountPath: /html

  volumes:
    - name: html
      emptyDir: {}
```

```bash
# Pod aus Kubernetes-YAML starten
podman play kube webapp.yaml

# Status
podman pod ps
podman ps

# Testen
curl http://localhost:8080   # Port-Forwarding via --publish
podman port webapp           # aktive Ports

# Aufräumen
podman play kube webapp.yaml --down

# Eigenen Container zu Kubernetes-YAML konvertieren
podman generate kube mycontainer > my-k8s.yaml
podman generate kube mypod > mypod-k8s.yaml
```

> [!TIP] Einschränkungen von podman play kube
> Für einfache Pods und Deployments geeignet. `Services`, `Ingress` und komplexe K8s-Features werden nicht vollständig unterstützt. Für vollständiges K8s-Testing: `minikube` oder `kind` verwenden.

---

## Verwandte Notizen

- [[Podman Compose]] — Compose-Dateien und Quadlet-Vergleich
- [[Systemd Integration]] — Quadlet im Detail
- [[Pods]] — Pod-Konzept und Management
- [[Podman Troubleshooting]] — Wenn etwas nicht klappt
