---
tags:
  - docker
  - troubleshooting
  - debugging
  - fehler
---

# Docker Troubleshooting

> [!ABSTRACT] TL;DR
> Systematischer Leitfaden zur Fehlerdiagnose und -behebung in Docker-Umgebungen — von nicht startenden Containern über Netzwerkprobleme bis hin zu Daemon-Ausfällen. Enthält Exit-Code-Referenz, Debug-Befehle und bewährte Lösungsstrategien.

---

## Allgemeine Diagnose

Bevor man sich in spezifische Probleme vertieft, sollte man sich einen Überblick über den Gesamtzustand der Docker-Installation verschaffen. Die folgenden Befehle liefern die wichtigsten Systeminformationen auf einen Blick.

### Docker-Version und Systeminfo

```bash
# Docker Version (Client + Server)
docker version

# Ausführliche Systeminformationen
docker info

# Speicherverbrauch (Images, Container, Volumes, Build Cache)
docker system df

# Detaillierte Aufschlüsselung
docker system df -v
```

### Docker Daemon Logs

```bash
# Systemd-basierte Systeme (Ubuntu, Debian, RHEL, etc.)
journalctl -u docker.service --no-pager -n 100

# Live-Logs verfolgen
journalctl -u docker.service -f

# Logs seit dem letzten Boot
journalctl -u docker.service -b
```

### Schneller Gesundheitscheck

```bash
# Prüfen ob der Daemon läuft
systemctl is-active docker

# Docker-Socket vorhanden?
ls -la /var/run/docker.sock

# Testcontainer starten
docker run --rm hello-world
```

> [!TIP] Erste Anlaufstelle
> Bei unklaren Problemen immer zuerst `docker info` und `journalctl -u docker` prüfen. Viele Fehler sind bereits in den Daemon-Logs dokumentiert, bevor sie sich in Containern manifestieren.

---

## Container startet nicht

Ein Container, der nicht startet oder sofort beendet wird, ist eines der häufigsten Probleme. Der Exit Code gibt dabei den entscheidenden Hinweis auf die Ursache.

### Exit Codes Referenz

| Exit Code | Signal | Bedeutung | Häufige Ursache |
|:---------:|:------:|-----------|-----------------|
| `0` | — | Normales Ende | Prozess hat seine Aufgabe erledigt und beendet sich |
| `1` | — | Allgemeiner Fehler | Anwendungsfehler, fehlende Abhängigkeiten, Syntaxfehler |
| `2` | — | Shell-Fehler | Falscher Befehl im Entrypoint/CMD |
| `126` | — | Nicht ausführbar | Binary existiert, ist aber nicht ausführbar (Permissions) |
| `127` | — | Befehl nicht gefunden | Binary/Skript existiert nicht im Image |
| `137` | `SIGKILL` | Erzwungener Abbruch | OOM-Killer, `docker kill`, Memory Limit erreicht |
| `139` | `SIGSEGV` | Segmentation Fault | Speicherzugriffsfehler in der Anwendung |
| `143` | `SIGTERM` | Ordentliches Beenden | `docker stop`, Shutdown-Signal empfangen |

### Diagnose-Workflow

```bash
# Container-Status und Exit Code prüfen
docker ps -a --filter "name=mein-container"

# Letzte Logs abrufen
docker logs mein-container

# Logs mit Zeitstempel und Tail
docker logs --timestamps --tail 50 mein-container

# Live-Logs verfolgen
docker logs -f mein-container

# Container-Details inspizieren (State, Config, Mounts, etc.)
docker inspect mein-container

# Nur den Exit Code extrahieren
docker inspect --format='{{.State.ExitCode}}' mein-container

# OOM-Kill prüfen
docker inspect --format='{{.State.OOMKilled}}' mein-container
```

### Häufige Ursachen und Lösungen

#### Fehlende Umgebungsvariablen

```bash
# Welche Env-Vars sind gesetzt?
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' mein-container

# Container mit fehlenden Vars starten (Fehler reproduzieren)
docker run --rm -e DB_HOST=localhost -e DB_PORT=5432 mein-image
```

> [!WARNING] Env-Vars in Compose
> In `docker-compose.yml` werden Variablen aus `.env`-Dateien **nicht** in den Container durchgereicht, wenn sie nur unter `environment:` ohne expliziten Wert stehen. Prüfe immer mit `docker compose config`, ob die Variablen korrekt aufgelöst werden.

#### Falsche Port-Konfiguration

```bash
# Welche Ports sind exponiert?
docker inspect --format='{{range $p, $conf := .NetworkSettings.Ports}}{{$p}} -> {{$conf}}{{println}}{{end}}' mein-container

# Port-Binding prüfen
docker port mein-container
```

#### Permission-Probleme

```bash
# Als welcher User läuft der Prozess?
docker inspect --format='{{.Config.User}}' mein-container

# In gestoppten Container schauen (neuen starten mit Shell)
docker run --rm -it --entrypoint /bin/sh mein-image

# Dateiberechtigungen im Container prüfen
docker run --rm --entrypoint /bin/sh mein-image -c "ls -la /app"
```

> [!TIP] Entrypoint überschreiben
> Wenn ein Container sofort crasht, kann man ihn mit `--entrypoint /bin/sh` oder `--entrypoint /bin/bash` starten, um interaktiv darin zu debuggen. Damit umgeht man den fehlerhaften Startbefehl.

---

## Netzwerk-Probleme

Netzwerkprobleme in Docker sind vielfältig und oft schwer zu diagnostizieren. Die meisten lassen sich auf DNS, Firewall-Regeln oder falsche Netzwerkkonfiguration zurückführen.

### Container können sich nicht erreichen

```bash
# Netzwerke auflisten
docker network ls

# Details eines Netzwerks anzeigen (inkl. verbundener Container)
docker network inspect bridge

# Prüfen, in welchem Netzwerk ein Container ist
docker inspect --format='{{range $key, $value := .NetworkSettings.Networks}}{{$key}}: {{$value.IPAddress}}{{println}}{{end}}' mein-container

# Konnektivität von Container zu Container testen
docker exec container-a ping -c 3 container-b

# DNS-Auflösung innerhalb eines Containers testen
docker exec mein-container nslookup anderer-container
```

> [!WARNING] Default Bridge vs. Custom Bridge
> Container im **Default-Bridge-Netzwerk** (`bridge`) können sich gegenseitig **nicht über den Containernamen** ansprechen — nur über IP-Adressen. Die DNS-basierte Service Discovery funktioniert **ausschließlich in Custom-Bridge-Netzwerken**.

```bash
# Custom Bridge erstellen (mit DNS-Auflösung)
docker network create mein-netzwerk

# Container im Custom Network starten
docker run -d --name app --network mein-netzwerk mein-image
docker run -d --name db --network mein-netzwerk postgres:16
```

### Port bereits belegt

```bash
# Welcher Prozess belegt den Port? (Linux)
ss -tulnp | grep :8080
# Alternativ
netstat -tulnp | grep :8080

# Alle von Docker belegten Ports anzeigen
docker ps --format "table {{.Names}}\t{{.Ports}}"

# Container auf anderem Port starten
docker run -d -p 8081:80 mein-image
```

> [!TIP] Port-Konflikte vermeiden
> In Compose-Dateien den Host-Port variabel halten: `${APP_PORT:-8080}:80`. So kann der Port via `.env`-Datei geändert werden, ohne die Compose-Datei anzupassen.

### DNS-Auflösung funktioniert nicht

```bash
# DNS-Server im Container prüfen
docker exec mein-container cat /etc/resolv.conf

# Manuelle DNS-Auflösung testen
docker exec mein-container nslookup google.com
docker exec mein-container dig google.com

# Container mit explizitem DNS-Server starten
docker run --dns 8.8.8.8 --dns 1.1.1.1 mein-image

# DNS global für den Daemon konfigurieren
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
EOF
sudo systemctl restart docker
```

### Kein Internet im Container

```bash
# Internet-Konnektivität testen
docker run --rm alpine ping -c 3 8.8.8.8
docker run --rm alpine wget -qO- https://ifconfig.me

# IP-Forwarding aktiviert?
sysctl net.ipv4.ip_forward
# Falls 0 → aktivieren
sudo sysctl -w net.ipv4.ip_forward=1

# iptables-Regeln prüfen (Docker-Chain)
sudo iptables -L -n -v | grep -A 5 DOCKER

# NAT-Regeln prüfen
sudo iptables -t nat -L -n -v
```

> [!WARNING] Firewall-Interaktion
> Docker manipuliert `iptables`-Regeln direkt. Firewalls wie `ufw` oder `firewalld` können mit Docker in Konflikt geraten. Wenn nach einem Firewall-Update Container kein Internet mehr haben, Docker-Netzwerk neu erstellen oder den Daemon neustarten.

---

## Volume- und Storage-Probleme

### Permission Denied (UID/GID Mismatch)

Das häufigste Volume-Problem: Der Prozess im Container läuft unter einem anderen Benutzer als der Besitzer der gemounteten Dateien auf dem Host.

```bash
# UID/GID des Container-Prozesses prüfen
docker exec mein-container id

# Besitzer auf dem Host prüfen
ls -ln /pfad/zum/volume

# Lösung 1: Besitzer auf dem Host anpassen
sudo chown -R 1000:1000 /pfad/zum/volume

# Lösung 2: Container mit bestimmter UID starten
docker run --user 1000:1000 mein-image

# Lösung 3: Init-Script im Container, das Permissions setzt
docker run -v /host/data:/data -e PUID=1000 -e PGID=1000 mein-image
```

> [!TIP] PUID/PGID Pattern
> Viele populäre Images (Linuxserver.io, etc.) unterstützen `PUID` und `PGID` als Umgebungsvariablen. Damit passt der Container die Dateirechte beim Start automatisch an den Host-User an.

### Disk Full — Speicherplatz erschöpft

```bash
# Docker-Speicherverbrauch analysieren
docker system df

# Detailliert nach Image, Container, Volume
docker system df -v

# Ungenutzte Ressourcen aufräumen (ungefährlich)
docker system prune

# Aggressive Bereinigung (inkl. unbenutzter Images und Volumes)
docker system prune -a --volumes

# Nur gestoppte Container entfernen
docker container prune

# Nur unbenutzte Images entfernen
docker image prune -a

# Nur verwaiste Volumes entfernen
docker volume prune

# Dangling Build-Cache entfernen
docker builder prune
```

> [!DANGER] `docker system prune -a --volumes`
> Dieser Befehl entfernt **alle** nicht laufenden Container, **alle** nicht verwendeten Images und **alle** nicht gemounteten Volumes. In Produktionsumgebungen nur gezielt einzelne Ressourcen bereinigen, niemals blind alles löschen.

### Volume nicht gemountet

```bash
# Mounts eines Containers anzeigen
docker inspect --format='{{range .Mounts}}{{.Type}}: {{.Source}} -> {{.Destination}}{{println}}{{end}}' mein-container

# Prüfen ob der Host-Pfad existiert
ls -la /host/pfad/zum/volume

# Named Volume inspizieren
docker volume inspect mein-volume

# Inhalt eines Named Volumes prüfen
docker run --rm -v mein-volume:/data alpine ls -la /data
```

---

## Performance-Probleme

### Ressourcenverbrauch überwachen

```bash
# Live-Statistiken aller Container
docker stats

# Statistiken eines bestimmten Containers
docker stats mein-container

# Einmalige Momentaufnahme (kein Stream)
docker stats --no-stream

# Formatierte Ausgabe
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
```

### CPU- und Memory-Limits

```bash
# Container mit Limits starten
docker run -d \
  --name app \
  --cpus="2.0" \
  --memory="512m" \
  --memory-swap="1g" \
  mein-image

# Aktuelle Limits eines laufenden Containers prüfen
docker inspect --format='CPU: {{.HostConfig.NanoCpus}} | Mem: {{.HostConfig.Memory}}' mein-container

# Limits live ändern (ohne Neustart)
docker update --cpus="4.0" --memory="1g" mein-container
```

> [!WARNING] OOM-Killer
> Wenn ein Container sein Memory-Limit überschreitet, wird er vom OOM-Killer mit Exit Code `137` beendet. Prüfe mit `docker inspect --format='{{.State.OOMKilled}}'` ob ein OOM-Kill stattgefunden hat. Die Lösung ist entweder das Limit erhöhen oder den Speicherverbrauch der Anwendung optimieren.

### I/O-Probleme und Overlay2 Cleanup

```bash
# Overlay2-Speicherverbrauch auf dem Host prüfen
du -sh /var/lib/docker/overlay2/

# Größte Layer identifizieren
du -sh /var/lib/docker/overlay2/* | sort -rh | head -20

# Dangling Images entfernen (häufigste Ursache für Overlay2-Bloat)
docker image prune

# Docker-Root-Verzeichnis prüfen
docker info --format '{{.DockerRootDir}}'
```

> [!TIP] Storage Driver prüfen
> Falls `overlay2` Probleme macht, prüfe ob das Dateisystem `xfs` mit `d_type=true` oder `ext4` verwendet wird. Overlay2 benötigt `d_type`-Support. Prüfbar mit: `xfs_info /var/lib/docker | grep ftype` — der Wert muss `1` sein.

---

## Image-Build-Probleme

### Build-Cache und Invalidierung

```bash
# Build ohne Cache (erzwingt kompletten Neuaufbau)
docker build --no-cache -t mein-image .

# Nur ab einem bestimmten Stage ohne Cache
docker build --no-cache --target build-stage -t mein-image .

# Build-Cache anzeigen
docker builder du

# Build-Cache komplett leeren
docker builder prune -a
```

> [!TIP] Cache-effiziente Dockerfiles
> Selten geänderte Schichten (z.B. `apt install`) gehören an den Anfang des Dockerfiles, häufig geänderte (z.B. `COPY . .`) ans Ende. Jede `RUN`-, `COPY`- oder `ADD`-Anweisung erstellt einen Layer — wird einer davon invalidiert, werden **alle nachfolgenden** Layer ebenfalls neu gebaut.

### Build-Context zu groß

```bash
# Build-Context-Größe prüfen (wird beim Build angezeigt)
docker build -t mein-image . 2>&1 | head -5
# Ausgabe: "Sending build context to Docker daemon  2.1GB"

# .dockerignore erstellen, um unnötige Dateien auszuschließen
cat <<'EOF' > .dockerignore
.git
node_modules
*.log
*.md
.env
.vscode
__pycache__
dist
build
EOF
```

> [!WARNING] Fehlende `.dockerignore`
> Ohne `.dockerignore` wird das gesamte Verzeichnis (inkl. `.git`, `node_modules`, Logs) als Build-Context gesendet. Das kann den Build um Minuten verlangsamen und das Image unnötig aufblähen.

### BuildKit Debugging

```bash
# BuildKit aktivieren (falls nicht Standard)
export DOCKER_BUILDKIT=1

# Verbose Build-Output mit BuildKit
docker build --progress=plain -t mein-image .

# Build mit zusätzlichen Debug-Informationen
docker build --progress=plain --no-cache -t mein-image . 2>&1 | tee build.log

# Multi-Stage Build — nur eine bestimmte Stage bauen
docker build --target development -t mein-image:dev .
```

---

## Docker Compose-Probleme

### depends_on wartet nicht auf Readiness

`depends_on` stellt nur sicher, dass ein Container **gestartet** wird — nicht, dass der Service darin **bereit** ist (z.B. Datenbank akzeptiert Verbindungen).

```yaml
# docker-compose.yml — Lösung mit healthcheck (Compose V2)
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 10
    environment:
      POSTGRES_PASSWORD: secret

  app:
    image: mein-app
    depends_on:
      db:
        condition: service_healthy
    environment:
      DB_HOST: db
```

> [!TIP] Healthcheck-Beispiele
> - **PostgreSQL:** `pg_isready -U postgres`
> - **MySQL/MariaDB:** `mysqladmin ping -h localhost`
> - **Redis:** `redis-cli ping`
> - **HTTP-Service:** `curl -f http://localhost:8080/health`
> - **TCP-Port:** `nc -z localhost 5432`

### Orphan Container

```bash
# Warnung: "Found orphan containers"
# Container aus einer früheren Compose-Konfiguration existieren noch

# Orphan Container anzeigen und entfernen
docker compose down --remove-orphans

# Alles herunterfahren inkl. Volumes
docker compose down --remove-orphans --volumes

# Compose-Projekt-Name prüfen (verursacht oft Orphans)
docker compose ls
```

### Netzwerk-Konflikte in Compose

```bash
# Compose erstellt automatisch ein Netzwerk: <projektname>_default
# Konflikte entstehen, wenn Subnet-Bereiche kollidieren

# Alle Docker-Netzwerke mit Subnets anzeigen
docker network ls -q | xargs -I {} docker network inspect {} \
  --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

```yaml
# docker-compose.yml — Custom Subnet
networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

services:
  app:
    image: mein-app
    networks:
      - backend
```

> [!WARNING] Subnet-Konflikte mit dem Host-Netzwerk
> Docker verwendet standardmäßig Subnetze aus dem Bereich `172.17.0.0/16` bis `172.31.0.0/16`. Falls diese Bereiche im lokalen Netzwerk (VPN, Firmennetz) bereits belegt sind, kommt es zu Routing-Konflikten. In `/etc/docker/daemon.json` einen anderen Bereich festlegen:
> ```json
> { "default-address-pools": [{"base": "10.10.0.0/16", "size": 24}] }
> ```

---

## Docker Daemon-Probleme

### Daemon startet nicht

```bash
# Status prüfen
systemctl status docker

# Detaillierte Fehlermeldung
journalctl -xeu docker.service

# Konfiguration validieren
dockerd --validate

# Daemon manuell im Vordergrund starten (für Debugging)
sudo dockerd --debug

# Häufige Ursache: fehlerhafte daemon.json
sudo cat /etc/docker/daemon.json
# JSON-Syntax prüfen
python3 -c "import json; json.load(open('/etc/docker/daemon.json'))"
```

### Storage Driver Fehler

```bash
# Aktuellen Storage Driver prüfen
docker info --format '{{.Driver}}'

# Storage Driver explizit setzen
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
EOF
sudo systemctl restart docker
```

> [!DANGER] `/var/lib/docker` löschen
> Das Löschen von `/var/lib/docker` entfernt **alle** Images, Container, Volumes und Netzwerke unwiderruflich. Nur als absolut letzter Ausweg verwenden und vorher alle wichtigen Volumes sichern.

### Docker Socket Permissions

```bash
# Fehlermeldung: "permission denied while trying to connect to the Docker daemon socket"

# Prüfen ob der User in der docker-Gruppe ist
groups $USER

# User zur docker-Gruppe hinzufügen
sudo usermod -aG docker $USER

# Gruppenänderung aktivieren (ohne Logout)
newgrp docker

# Socket-Berechtigungen prüfen
ls -la /var/run/docker.sock
# Erwartete Ausgabe: srw-rw---- 1 root docker ...
```

> [!WARNING] Sicherheitsimplikation der `docker`-Gruppe
> Mitglieder der `docker`-Gruppe haben effektiv **Root-Zugriff** auf den Host, da sie Container mit vollen Privilegien starten können. In Produktionsumgebungen Rootless Docker oder Podman in Erwägung ziehen.

---

## Registry und Image-Pull-Fehler

### Authentifizierungsprobleme

```bash
# Login-Status prüfen
cat ~/.docker/config.json | grep -A2 "auths"

# Erneut einloggen
docker login registry.example.com
docker login ghcr.io -u USERNAME --password-stdin <<< "$GITHUB_TOKEN"

# Credential Helper prüfen
docker-credential-desktop list 2>/dev/null || echo "Kein Credential Helper"
```

| Fehler | Ursache | Lösung |
|--------|---------|--------|
| `unauthorized: authentication required` | Nicht eingeloggt oder Token abgelaufen | `docker login` erneut ausführen |
| `denied: requested access is not authorized` | Keine Berechtigung für dieses Image | Repository-Zugriff prüfen (Org-Mitgliedschaft, Token-Scopes) |
| `certificate signed by unknown authority` | Selbstsigniertes Zertifikat | CA-Cert in `/etc/docker/certs.d/<registry>/` hinterlegen |
| `x509: certificate has expired` | Abgelaufenes TLS-Zertifikat | Zertifikat erneuern |

### Image-Pull-Fehler

```bash
# Image manuell pullen für bessere Fehlermeldung
docker pull registry.example.com/app:1.2.3

# Manifest prüfen (Architektur-Check)
docker manifest inspect registry.example.com/app:1.2.3

# Multi-Arch: Spezifische Plattform pullen
docker pull --platform linux/amd64 registry.example.com/app:1.2.3
```

| Fehler | Ursache | Lösung |
|--------|---------|--------|
| `manifest unknown` | Tag existiert nicht | Tag-Name prüfen, `docker search` nutzen |
| `no matching manifest for linux/arm64` | Kein ARM-Image vorhanden | AMD64-Image nutzen oder Multi-Arch bauen |
| `i/o timeout` | Registry nicht erreichbar | DNS, Firewall, Proxy-Einstellungen prüfen |
| `toomanyrequests` (Docker Hub) | Pull-Rate-Limit (100/6h anonym) | Docker Hub Login oder Registry-Mirror nutzen |

### Registry-Mirror konfigurieren

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

---

## Security-Troubleshooting

### Permission Denied im Container

```bash
# User im Container prüfen
docker exec mein-container whoami
docker exec mein-container id

# Dateiberechtigungen prüfen
docker exec mein-container ls -la /app/

# Container mit Root debuggen (temporär!)
docker exec -u 0 mein-container bash
```

| Fehler | Ursache | Lösung |
|--------|---------|--------|
| `Permission denied` auf Volumes | UID/GID-Mismatch | `chown` im Dockerfile oder `--user` anpassen |
| `read-only file system` | `--read-only` aktiv | `tmpfs`-Mount für beschreibbare Verzeichnisse |
| `Operation not permitted` | Fehlende Capability | `--cap-add=<CAP>` hinzufügen |
| `cannot open shared object file` | Library fehlt | Abhängigkeiten im Dockerfile installieren |

### Capability-Probleme

```bash
# Aktive Capabilities eines Containers prüfen
docker inspect --format '{{.HostConfig.CapAdd}}' mein-container
docker inspect --format '{{.HostConfig.CapDrop}}' mein-container

# Container mit minimalen Capabilities testen
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# Welche Capability braucht mein Prozess?
# Im Container: apt-get install libcap2-bin
docker exec mein-container capsh --print
```

| Capability | Benötigt für |
|-----------|-------------|
| `NET_BIND_SERVICE` | Ports < 1024 binden |
| `SYS_PTRACE` | Debugging (strace, gdb) |
| `DAC_OVERRIDE` | Dateiberechtigungen überschreiben |
| `CHOWN` | Datei-Ownership ändern |
| `NET_RAW` | Raw-Sockets (ping) |

### AppArmor / SELinux Konflikte

```bash
# AppArmor-Status prüfen (Ubuntu/Debian)
sudo aa-status
docker inspect --format '{{.HostConfig.SecurityOpt}}' mein-container

# AppArmor temporär deaktivieren für Test
docker run --security-opt apparmor=unconfined myapp

# SELinux-Status prüfen (RHEL/CentOS)
getenforce
# Bei Problemen: SELinux-Label auf Volume setzen
docker run -v /data:/data:Z myapp     # :Z für private Label
docker run -v /data:/data:z myapp     # :z für shared Label
```

---

## Nützliche Debug-Befehle

### Schnellreferenz

| Aufgabe | Befehl |
|---------|--------|
| Container-Status | `docker ps -a` |
| Container-Logs | `docker logs --tail 100 -f <name>` |
| In Container einsteigen | `docker exec -it <name> /bin/bash` |
| Container-Details | `docker inspect <name>` |
| Exit Code prüfen | `docker inspect -f '{{.State.ExitCode}}' <name>` |
| OOM-Kill prüfen | `docker inspect -f '{{.State.OOMKilled}}' <name>` |
| Ressourcenverbrauch | `docker stats --no-stream` |
| Netzwerk inspizieren | `docker network inspect <netzwerk>` |
| Port-Belegung | `docker port <name>` |
| Mounts anzeigen | `docker inspect -f '{{json .Mounts}}' <name> \| jq` |
| Prozesse im Container | `docker top <name>` |
| Dateisystem-Änderungen | `docker diff <name>` |
| Events live verfolgen | `docker events` |
| Speicherverbrauch | `docker system df -v` |
| Alles aufräumen | `docker system prune -a --volumes` |

### Erweiterte Diagnose

```bash
# Alle Container mit ihrem Status und Exit Code
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}"

# Alle Container eines Compose-Projekts
docker compose ps -a

# Events in Echtzeit überwachen (sehr nützlich bei Debugging)
docker events --filter type=container \
  --format '{{.Time}} {{.Action}} {{.Actor.Attributes.name}}'

# Dateisystem-Diff seit Container-Start
docker diff mein-container
# A = Added, C = Changed, D = Deleted

# Container als Image exportieren (für Analyse)
docker export mein-container > container-snapshot.tar

# Laufende Prozesse im Container
docker top mein-container

# Container-IP schnell ermitteln
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mein-container
```

### Debugging mit temporären Containern

```bash
# Netzwerk-Debugging mit nicolaka/netshoot
docker run --rm -it --network container:mein-container nicolaka/netshoot

# DNS-Debugging im selben Netzwerk
docker run --rm --network mein-netzwerk alpine nslookup mein-service

# Dateisystem eines Images untersuchen
docker run --rm -it mein-image /bin/sh

# Volume-Inhalte inspizieren
docker run --rm -v mein-volume:/data alpine ls -laR /data
```

> [!TIP] nicolaka/netshoot
> Das Image `nicolaka/netshoot` enthält alle gängigen Netzwerk-Tools (`curl`, `dig`, `nslookup`, `tcpdump`, `iperf`, `netstat`, etc.) und eignet sich hervorragend für Netzwerk-Debugging in Docker-Umgebungen. Mit `--network container:<name>` teilt es sich den Netzwerk-Namespace eines laufenden Containers.

---

## Verwandte Notizen

- [[Docker Index]]
- [[Container Lifecycle]]
- [[Docker Networking]]
- [[Docker CLI Cheatsheet]]
- [[Docker Monitoring und Logging]]
- [[Troubleshooting]] — Kubernetes-Troubleshooting-Methoden