---
tags: [podman, cli, cheatsheet, befehle, referenz, container, image, pod, volume, network]
aliases: ["Podman Befehle", "Podman Cheatsheet", "Podman CLI Referenz"]
---

# Podman CLI Cheatsheet

> [!ABSTRACT]
> Vollständige Befehlsreferenz für Podman — gegliedert nach Container, Image, Pod, Volume, Netzwerk und System. Podman ist ~95% CLI-kompatibel mit Docker. `alias docker=podman` funktioniert für die meisten Anwendungsfälle.

---

## Container-Befehle

### Starten und Stoppen

```bash
# Container starten (Vordergrund)
podman run nginx:alpine

# Hintergrund (-d), mit Name, Port-Mapping
podman run -d --name web -p 8080:80 nginx:alpine

# Interaktive Shell
podman run -it --rm ubuntu:24.04 bash

# Mit Umgebungsvariablen
podman run -d -e DB_HOST=localhost -e DB_PORT=5432 myapp

# Mit Volume
podman run -d -v mein-volume:/data:Z myapp

# Container stoppen
podman stop web
podman stop --time 30 web   # 30s Timeout vor SIGKILL

# Mehrere stoppen
podman stop web db cache

# Alle laufenden Container stoppen
podman stop -a

# Neustarten
podman restart web
```

### Container-Status

```bash
# Laufende Container
podman ps

# Alle Container (auch gestoppt)
podman ps -a

# Bestimmte Spalten
podman ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Mit Pod-Info
podman ps --pod

# Kompakteres Format
podman ps -a --format "{{.Names}}: {{.Status}}"
```

### Container verwalten

```bash
# Löschen
podman rm web
podman rm -f web           # erzwingen
podman rm -f $(podman ps -aq)   # alle löschen

# Umbenennen
podman rename web webserver

# Alle gestoppten Container löschen
podman container prune

# Container pausieren / fortsetzen
podman pause web
podman unpause web
```

### Wichtige `podman run` Flags

```bash
podman run \
  -d                          # Hintergrund
  -it                         # Interaktiv + TTY
  --rm                        # Nach Exit automatisch löschen
  --name web                  # Name vergeben
  -p 8080:80                  # Port-Mapping host:container
  -p 127.0.0.1:8080:80        # Nur auf localhost
  -v /host:/container:Z       # Bind Mount (SELinux private)
  -v mein-vol:/data:z         # Named Volume (SELinux shared)
  --tmpfs /tmp:size=64m       # tmpfs
  -e KEY=value                # Umgebungsvariable
  --env-file .env             # Env-Datei
  --network mein-netz         # Netzwerk
  --pod mein-pod              # Pod beitreten
  --user 1000:1000            # Als bestimmter User
  --workdir /app              # Arbeitsverzeichnis
  --hostname mycontainer      # Hostname setzen
  --restart always            # Neustart-Policy
  --memory 512m               # RAM-Limit
  --cpus 1.5                  # CPU-Limit
  --read-only                 # Read-Only Root-FS
  --cap-drop all              # Alle Capabilities droppen
  --cap-add NET_BIND_SERVICE  # Spezifische Capability
  --security-opt no-new-privileges  # Privilege Escalation verhindern
  --label app=myapp           # Label
  nginx:alpine                # Image
```

### Logs und Diagnose

```bash
# Logs lesen
podman logs web
podman logs -f web              # Follow
podman logs --tail=100 web      # Letzte 100 Zeilen
podman logs --since 1h web      # Letzte Stunde

# In Container einsteigen
podman exec -it web bash
podman exec -it web sh          # Falls kein bash

# Befehl ausführen
podman exec web nginx -t        # nginx config testen
podman exec web env             # Env-Variablen anzeigen

# Prozesse
podman top web
podman top web pid,ppid,comm,args

# Ressourcen
podman stats web
podman stats --no-stream        # Einmaliger Snapshot

# Dateien kopieren
podman cp web:/etc/nginx/nginx.conf ./
podman cp ./config.conf web:/etc/nginx/

# Änderungen seit Image-Start
podman diff web

# Details
podman inspect web
podman inspect --format "{{.NetworkSettings.IPAddress}}" web
podman inspect --format "{{.State.Status}}" web
```

---

## Image-Befehle

```bash
# Pull
podman pull docker.io/library/nginx:latest
podman pull quay.io/fedora/fedora:40

# Lokale Images
podman images
podman image ls

# Suchen
podman search nginx
podman search --filter=is-official nginx

# Inspizieren
podman image inspect nginx:latest
podman image inspect --format "{{.Architecture}}" nginx

# History (Layer)
podman image history nginx:latest

# Tagging
podman tag nginx:latest meinregistry.io/nginx:1.0

# Pushen
podman login meinregistry.io
podman push meinregistry.io/nginx:1.0

# Löschen
podman rmi nginx:latest
podman image prune           # dangling Images
podman image prune -a        # alle ungenutzten Images

# Bauen
podman build -t myapp:latest .
podman build -t myapp:latest -f Containerfile.prod .
podman build --no-cache -t myapp:latest .

# Export / Import
podman save -o nginx.tar nginx:latest
podman load -i nginx.tar
podman export container-id -o container.tar
podman import container.tar myimage:imported
```

---

## Pod-Befehle

```bash
# Pod erstellen
podman pod create --name meinpod
podman pod create --name webapp -p 8080:80 -p 8443:443
podman pod create --name webapp --network mein-netz -p 8080:80

# Container zum Pod hinzufügen
podman run -d --pod meinpod --name web nginx:alpine
podman run -d --pod meinpod --name app myapp:latest

# Pod-Übersicht
podman pod ps
podman pod ps --ctr-names

# Details
podman pod inspect meinpod

# Steuern
podman pod start meinpod
podman pod stop meinpod
podman pod restart meinpod
podman pod pause meinpod
podman pod unpause meinpod

# Logs
podman pod logs meinpod
podman pod logs -f meinpod

# Stats
podman pod stats meinpod
podman pod stats --no-stream

# Löschen
podman pod rm meinpod
podman pod rm --force meinpod

# Kubernetes YAML generieren
podman generate kube meinpod > meinpod.yaml
podman generate kube --service meinpod > meinpod-svc.yaml

# Kubernetes YAML deployen
podman play kube meinpod.yaml
podman play kube --down meinpod.yaml
```

---

## Volume-Befehle

```bash
# Erstellen
podman volume create mein-volume
podman volume create --label app=myapp mein-volume

# Auflisten
podman volume ls
podman volume ls --filter label=app=myapp

# Inspizieren
podman volume inspect mein-volume
podman volume inspect --format "{{.Mountpoint}}" mein-volume

# Löschen
podman volume rm mein-volume
podman volume prune           # ungenutzte

# Export / Import
podman volume export mein-volume --output backup.tar
podman volume import mein-volume backup.tar
```

---

## Netzwerk-Befehle

```bash
# Auflisten
podman network ls

# Erstellen
podman network create mein-netz
podman network create --subnet 172.20.0.0/16 mein-netz
podman network create --driver macvlan -o parent=eth0 macvlan-netz

# Inspizieren
podman network inspect mein-netz

# Verbinden / Trennen
podman network connect mein-netz container
podman network disconnect mein-netz container

# Löschen
podman network rm mein-netz
podman network prune

# Port-Anzeige
podman port container-name
podman port -a   # alle Container
```

---

## System-Befehle

```bash
# System-Info
podman info
podman version
podman system info

# Disk-Nutzung
podman system df
podman system df -v   # detailliert

# Aufräumen
podman system prune           # gestoppte Container + dangling Images + ungenutzte Netzwerke
podman system prune -a        # zusätzlich ungenutzte Images
podman system prune --volumes # zusätzlich ungenutzte Volumes

# Migration (nach subuid/subgid-Änderungen)
podman system migrate

# Komplett-Reset (ALLES löschen!)
podman system reset

# Events
podman events
podman events --filter event=start
podman events --since 1h
podman events --format json
```

---

## Secrets

```bash
# Secret erstellen
echo "geheimespasswort" | podman secret create db-password -
podman secret create tls-cert ./cert.pem

# Auflisten
podman secret ls

# Inspizieren (kein Wert, nur Metadaten)
podman secret inspect db-password

# Löschen
podman secret rm db-password

# Im Container nutzen
podman run --secret db-password,type=env,target=DB_PASSWORD myapp
podman run --secret db-password,type=mount,target=/run/secrets/db-password myapp
```

---

## Healthcheck

```bash
# Container mit Healthcheck starten
podman run -d \
  --health-cmd="curl -fs http://localhost/ || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-start-period=30s \
  --health-retries=3 \
  --name webserver nginx

# Status prüfen
podman healthcheck run webserver
podman inspect --format "{{.State.Health.Status}}" webserver
# healthy | unhealthy | starting
```

---

## Nützliche One-Liner

```bash
# Container-IP ermitteln
podman inspect -f "{{.NetworkSettings.IPAddress}}" container-name

# Alle Container-IPs
podman inspect $(podman ps -q) --format "{{.Name}}: {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}"

# Container nach Label finden
podman ps --filter label=app=myapp

# Letzte Logs aller Container
for c in $(podman ps -q); do echo "=== $c ==="; podman logs --tail=5 $c; done

# Alle Images eines Repository
podman images --format "{{.Repository}}:{{.Tag}}" | grep nginx

# Größten Images
podman images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" | sort -rh | head -10

# Docker-Kompatibilität testen
alias docker=podman
docker ps   # ✅ funktioniert
```

---

## Vergleich: Docker → Podman Befehle

| Docker | Podman | Unterschied |
|--------|--------|-------------|
| `docker run` | `podman run` | Identisch |
| `docker ps` | `podman ps` | Identisch |
| `docker build` | `podman build` | Identisch |
| `docker pull` | `podman pull` | Identisch |
| `docker push` | `podman push` | Identisch |
| `docker exec` | `podman exec` | Identisch |
| `docker logs` | `podman logs` | Identisch |
| `docker compose` | `podman compose` | Identisch |
| `docker swarm` | — | Kein Äquivalent |
| `docker service` | — | Kein Äquivalent |
| — | `podman pod` | Nur in Podman |
| — | `podman generate kube` | Nur in Podman |
| — | `podman play kube` | Nur in Podman |

---

## Verwandte Notizen

- [[Container und Images]] — Container und Image-Konzepte
- [[Pods]] — Pod-Verwaltung
- [[Podman Networking]] — Netzwerk-Befehle
- [[Volumes und Storage]] — Volume-Befehle
