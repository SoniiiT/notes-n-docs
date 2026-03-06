---
tags: [podman, pods, kubernetes, infra-container, multi-container, generate-kube]
aliases: ["Podman Pods", "Pods Konzept", "Pod"]
---

# Pods

> [!ABSTRACT]
> Pods sind Gruppen von Containern, die sich Netzwerk-Namespace und IPC teilen — ein Kubernetes-Konzept, das Podman als einziger lokaler Container-Manager nativ implementiert. Alle Container in einem Pod teilen dieselbe IP-Adresse und kommunizieren via localhost. Pods können direkt als Kubernetes YAML exportiert werden.

---

## Was ist ein Pod?

Ein Pod ist eine logische Einheit aus einem oder mehreren Containern, die:
- **Denselben Netzwerk-Namespace teilen** → gleiche IP, gleiche Ports
- **Denselben IPC-Namespace teilen** → gemeinsamer Shared Memory
- **Gemeinsame Volumes** haben können
- Als **Einheit** gestartet, gestoppt und verwaltet werden

```
┌─────────────────────────────────────────┐
│                  Pod                    │
│  ┌──────────┐  ┌──────────┐            │
│  │  nginx   │  │  myapp   │            │
│  │  :80     │  │  :3000   │            │
│  └──────────┘  └──────────┘            │
│                                         │
│  Shared: IP 10.88.0.5, IPC, Volumes    │
└─────────────────────────────────────────┘
        ↓ Port-Forwarding
   Host: localhost:8080 → Pod:80
```

### Wann Pods verwenden?

| Szenario | Empfehlung |
|----------|-----------|
| Eng gekoppelte Services (Frontend + Backend) | ✅ Pod |
| Sidecar-Muster (App + Log-Collector) | ✅ Pod |
| Kubernetes-Deployment lokal testen | ✅ Pod |
| Unabhängige Services ohne gemeinsame Ressourcen | ❌ Separate Container |
| Docker Compose Stack | ❌ Netzwerk reicht |

---

## Infra-Container

Jeder Pod enthält automatisch einen **Infra-Container** (`pause` Image), der:
- Den Netzwerk-Namespace des Pods hält und erhält
- Läuft, solange der Pod existiert — auch wenn alle anderen Container gestoppt sind
- Die Pod-IP-Adresse für alle Container bereitstellt

```bash
# Infra-Container ist in podman ps sichtbar
podman ps --pod
# NAME              POD           STATUS
# 3a1b2c3d4e5f-infra  meinpod   Up 5 minutes   ← Infra-Container
# nginx              meinpod   Up 5 minutes
```

---

## Pod-Lifecycle

### Pod erstellen

```bash
# Einfacher Pod
podman pod create --name meinpod

# Pod mit Port-Mapping (nur am Pod definieren!)
podman pod create --name webapp -p 8080:80 -p 8443:443

# Pod mit spezifischem Netzwerk
podman pod create --name meinpod --network mein-netzwerk

# Pod mit DNS-Namen
podman pod create --name meinpod --hostname webapp.local
```

### Container zum Pod hinzufügen

```bash
# Container im Pod starten
podman run -d --pod meinpod --name web nginx:alpine
podman run -d --pod meinpod --name app myapp:latest

# WICHTIG: Port-Mapping nur am Pod, NICHT am Container!
# Falsch:
podman run -d --pod meinpod -p 8080:80 --name web nginx  # ❌ Fehler

# Richtig:
podman pod create --name meinpod -p 8080:80
podman run -d --pod meinpod --name web nginx              # ✅
```

### Pod verwalten

```bash
# Pod-Übersicht
podman pod ps
podman pod list

# Alle Container im Pod anzeigen
podman pod ps --ctr-names

# Pod-Details
podman pod inspect meinpod

# Pod starten / stoppen / neustarten
podman pod stop meinpod
podman pod start meinpod
podman pod restart meinpod

# Pod-Logs (alle Container zusammen)
podman pod logs meinpod

# Pod-Stats (Ressourcen aller Container)
podman pod stats meinpod

# Pod löschen (stoppt alle Container)
podman pod rm meinpod
podman pod rm --force meinpod   # auch wenn laufend
```

---

## Netzwerk in Pods

Alle Container in einem Pod teilen denselben Netzwerk-Namespace — sie erreichen sich via `localhost`:

```bash
# Pod mit nginx (Port 80) und PHP-FPM (Port 9000)
podman pod create --name webserver -p 8080:80

podman run -d --pod webserver --name php php-fpm:8.3

# nginx.conf kann PHP-FPM via localhost:9000 erreichen
# (obwohl es ein separater Container ist!)
podman run -d --pod webserver --name nginx \
  -v ./nginx.conf:/etc/nginx/conf.d/default.conf:ro,z \
  nginx:alpine
```

**nginx.conf für PHP-FPM im gleichen Pod:**
```nginx
location ~ \.php$ {
    fastcgi_pass localhost:9000;   # localhost = selber Pod!
    fastcgi_index index.php;
    include fastcgi_params;
}
```

---

## Praktisches Beispiel: WordPress-Pod

```bash
# Pod erstellen
podman pod create --name wordpress -p 8080:80

# MariaDB-Container
podman run -d \
  --pod wordpress \
  --name wordpress-db \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD=wppass \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -v wordpress-db:/var/lib/mysql:Z \
  mariadb:11

# WordPress-Container (nutzt localhost für DB-Verbindung)
podman run -d \
  --pod wordpress \
  --name wordpress-app \
  -e WORDPRESS_DB_HOST=localhost \
  -e WORDPRESS_DB_NAME=wordpress \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD=wppass \
  -v wordpress-data:/var/www/html:Z \
  wordpress:latest

# Status
podman pod ps
podman ps --pod
curl http://localhost:8080
```

---

## Pod als Kubernetes YAML exportieren

Eines der mächtigsten Podman-Features: Pods direkt als Kubernetes-kompatibles YAML exportieren.

```bash
# YAML exportieren
podman generate kube meinpod > meinpod.yaml

# Mit Volumes
podman generate kube --type deployment meinpod > deployment.yaml

# Service (LoadBalancer/NodePort) generieren
podman generate kube --service meinpod > meinpod-with-service.yaml
```

### Beispiel-Output (vereinfacht)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: web
    image: docker.io/library/nginx:alpine
    ports:
    - containerPort: 80
      hostPort: 8080
  - name: app
    image: localhost/myapp:latest
```

### Kubernetes YAML lokal ausführen

```bash
# Pod aus YAML deployen (lokales Testing)
podman play kube meinpod.yaml

# Mit Volume-Mapping
podman play kube meinpod.yaml --volume myvolume:/data

# Pod stoppen und löschen
podman play kube --down meinpod.yaml
```

> [!NOTE] Kompatibilität
> Das generierte YAML ist echtes Kubernetes-kompatibles YAML und kann direkt mit `kubectl apply` genutzt werden. Einige podman-spezifische Annotationen müssen möglicherweise entfernt werden.

---

## Systemd-Service für einen Pod (Quadlet)

```ini
# ~/.config/containers/systemd/webapp.pod
[Unit]
Description=Webapp Pod

[Pod]
PublishPort=8080:80
Network=webapp-net.network

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/webapp-nginx.container
[Unit]
Description=Nginx im Webapp Pod
After=webapp-pod.service

[Container]
Image=nginx:alpine
Pod=webapp.pod
Volume=%h/html:/usr/share/nginx/html:ro,z

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user start webapp-pod.service
```

---

## Verwandte Notizen

- [[Was ist Podman]] — Einführung in das Pod-Konzept
- [[Podman Networking]] — Netzwerk-Konfiguration für Pods
- [[Systemd Integration]] — Pods als systemd-Services (Quadlet)
- [[Podman CLI Cheatsheet]] — Pod-Befehle Referenz
