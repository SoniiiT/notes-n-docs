---
tags: [podman, compose, podman-compose, quadlet, docker-compose, yaml, multi-container]
aliases: ["Podman Compose", "podman-compose", "Quadlet Compose"]
---

# Podman Compose

> [!ABSTRACT]
> Podman bietet drei Wege für Multi-Container-Setups: `podman compose` (direkt in Podman 4.7+ integriert), `podman-compose` (Community-Python-Tool), und **Quadlet** (systemd-native `.container`-Dateien, empfohlen für Produktion). Docker Compose-Dateien funktionieren mit minimalen Anpassungen.

---

## Optionen für Compose in Podman

| Option | Beschreibung | Empfehlung |
|--------|-------------|-----------|
| `podman compose` | Ab Podman 4.7 integriert, nutzt `podman-compose` intern | Dev/Migration |
| `podman-compose` | Community-Python-Tool, docker-compose.yml-kompatibel | Dev/Migration |
| Docker Compose CLI + Podman Socket | Originales Docker Compose CLI gegen Podman-Backend | Migration |
| **Quadlet** | Native systemd-Units `.container`/`.pod`/`.network` | ✅ Produktion |

---

## podman compose (integriert, Podman 4.7+)

```bash
# Podman 4.7+ — podman compose ist direkt verfügbar
podman compose version

# Starten
podman compose up -d

# Status
podman compose ps

# Logs
podman compose logs -f

# Stoppen
podman compose down

# Mit spezifischer Compose-Datei
podman compose -f docker-compose.prod.yml up -d
```

---

## podman-compose installieren (Community-Tool)

```bash
# Via pip
pip3 install podman-compose

# Fedora/RHEL
sudo dnf install -y podman-compose

# Ubuntu/Debian
sudo apt-get install -y podman-compose

# Version prüfen
podman-compose version
```

---

## Docker Compose CLI via Podman Socket

```bash
# Podman Socket starten (User-Session)
systemctl --user enable --now podman.socket

# DOCKER_HOST setzen
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock

# Permanent in Shell-Profil
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock' >> ~/.bashrc

# Jetzt funktioniert Docker Compose CLI gegen Podman
docker compose up -d
docker compose ps
docker compose logs -f
```

---

## Beispiel: docker-compose.yml mit Podman

```yaml
# docker-compose.yml — funktioniert mit podman compose und podman-compose
version: "3.9"

services:
  web:
    image: docker.io/library/nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro,z
    depends_on:
      - app
    restart: unless-stopped

  app:
    image: docker.io/library/node:20-alpine
    working_dir: /app
    volumes:
      - ./src:/app:Z
    command: node server.js
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: docker.io/library/postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: geheim
    volumes:
      - pgdata:/var/lib/postgresql/data:Z
    restart: unless-stopped

volumes:
  pgdata:
```

> [!IMPORTANT] SELinux-Labels auf RHEL/Fedora
> Volume-Mounts immer mit `:z` (shared) oder `:Z` (private) ergänzen — sonst schlägt SELinux den Zugriff fehl.

```bash
# Starten
podman compose up -d

# Status
podman compose ps

# Logs aller Services
podman compose logs -f

# Einzelner Service
podman compose logs -f app

# Stoppen und aufräumen
podman compose down
podman compose down -v   # auch Volumes löschen
```

---

## Bekannte Einschränkungen von podman-compose

- **Docker Swarm Features** (`deploy:`, `replicas:`, `resources:`) werden ignoriert
- `depends_on: condition: service_healthy` — nur teilweise unterstützt
- `build.secrets` — nicht unterstützt
- `profiles:` — eingeschränkte Unterstützung
- Einige `logging:`-Optionen fehlen

> [!TIP] Für Produktion: Quadlet verwenden
> Compose-Dateien sind gut für Dev-Umgebungen und Migration. Für produktive Server ist **Quadlet** die bessere Wahl: native systemd-Integration, bessere Restart-Policies und journald-Logging.

---

## Quadlet — die systemd-native Alternative

Quadlet ist der empfohlene Weg für Produktions-Deployments. Keine Compose-Datei nötig — direkte systemd-Integration.

### Speicherorte

```
Rootless (User):  ~/.config/containers/systemd/
Rootful (System): /etc/containers/systemd/
```

### .container-Datei (Einzelcontainer)

```ini
# ~/.config/containers/systemd/nginx.container
[Unit]
Description=Nginx Webserver
After=network-online.target

[Container]
Image=docker.io/library/nginx:alpine
PublishPort=8080:80
Volume=%h/html:/usr/share/nginx/html:ro,z
Environment=NGINX_HOST=example.com
Label=app=webserver
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=30

[Install]
WantedBy=default.target
```

```bash
# Laden und starten
systemctl --user daemon-reload
systemctl --user enable --now nginx.service
systemctl --user status nginx.service
journalctl --user -u nginx.service -f
```

### .network-Datei

```ini
# ~/.config/containers/systemd/myapp.network
[Network]
Subnet=172.21.0.0/16
Label=app=myapp
DNS=1.1.1.1
```

### .volume-Datei

```ini
# ~/.config/containers/systemd/myapp-data.volume
[Volume]
Label=app=myapp
```

### Multi-Container mit Quadlet (.pod + .container)

```ini
# ~/.config/containers/systemd/wordpress.pod
[Unit]
Description=WordPress Pod

[Pod]
PublishPort=8080:80
Network=wordpress.network

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/wordpress-db.container
[Unit]
Description=WordPress Datenbank
After=wordpress-pod.service

[Container]
Image=docker.io/library/mariadb:11
Pod=wordpress.pod
Environment=MARIADB_ROOT_PASSWORD=rootpass
Environment=MARIADB_DATABASE=wordpress
Environment=MARIADB_USER=wpuser
Environment=MARIADB_PASSWORD=wppass
Volume=wordpress-db.volume:/var/lib/mysql:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/wordpress-app.container
[Unit]
Description=WordPress Anwendung
After=wordpress-db.service

[Container]
Image=docker.io/library/wordpress:latest
Pod=wordpress.pod
Environment=WORDPRESS_DB_HOST=localhost
Environment=WORDPRESS_DB_NAME=wordpress
Environment=WORDPRESS_DB_USER=wpuser
Environment=WORDPRESS_DB_PASSWORD=wppass
Volume=wordpress-data.volume:/var/www/html:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now wordpress-pod.service
systemctl --user start wordpress-app.service wordpress-db.service
```

---

## Quadlet-Variablen

```ini
# Nützliche Variablen in .container-Dateien
%h   # Home-Verzeichnis des Users
%u   # Benutzername
%U   # User-ID (UID)
%t   # Runtime-Verzeichnis ($XDG_RUNTIME_DIR)
```

---

## Wann Compose, wann Quadlet?

| Szenario | Empfehlung | Begründung |
|----------|:----------:|-----------|
| Vorhandene docker-compose.yml | Compose | Direkter Ersatz, wenig Umstieg |
| Neue Produktions-Deployments | ✅ Quadlet | systemd-native, bessere Kontrolle |
| Dev-Umgebung | Compose | Schnelles Auf/Ab, .env-Dateien |
| Auto-Start nach Reboot | ✅ Quadlet | Systemd-Unit mit `WantedBy=default.target` |
| Logging via journald | ✅ Quadlet | `journalctl --user -u service` |
| Auto-Update | ✅ Quadlet | `AutoUpdate=registry` in .container |
| CI/CD | Compose | Einfache Up/Down-Befehle |

---

## Verwandte Notizen

- [[Systemd Integration]] — Quadlet im Detail, podman generate systemd
- [[Pods]] — Pods mit Quadlet verwalten
- [[Volumes und Storage]] — Volume-Konfiguration in Compose
- [[Podman Praktische Beispiele]] — Vollständige Compose und Quadlet Beispiele
