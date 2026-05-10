---
tags: [docker, cli, cheatsheet, befehle, referenz]
---

# Docker CLI Cheatsheet

> [!ABSTRACT] TL;DR
> Kompakte Referenz aller wichtigen Docker-Befehle — Container, Images, Volumes, Netzwerke, Compose und System-Verwaltung. Jeder Befehl mit Syntax, Flags und Beispiel.

---

## Container-Befehle

### docker run

Erstellt und startet einen neuen Container aus einem Image.

```bash
docker run [OPTIONEN] IMAGE [BEFEHL] [ARGS...]
```

| Flag | Beschreibung |
|:-----|:-------------|
| `-d, --detach` | Container im Hintergrund starten |
| `-it` | Interaktiver Modus mit TTY |
| `--name NAME` | Container benennen |
| `-p HOST:CONTAINER` | Port-Mapping |
| `-v HOST:CONTAINER` | Volume-Mount |
| `--rm` | Container nach Beenden automatisch entfernen |
| `-e KEY=VALUE` | Umgebungsvariable setzen |
| `--env-file DATEI` | Umgebungsvariablen aus Datei laden |
| `--network NETZ` | Netzwerk zuweisen |
| `--restart POLICY` | Restart-Policy (`no`, `always`, `unless-stopped`, `on-failure`) |
| `--memory, -m` | RAM-Limit setzen |
| `--cpus` | CPU-Limit setzen |
| `--hostname` | Hostname im Container setzen |
| `--user, -u` | Benutzer im Container setzen |
| `--read-only` | Root-Dateisystem schreibgeschützt |
| `--privileged` | Erweiterte Rechte (Vorsicht!) |
| `--pid` | PID-Namespace (`host` für Host-PID) |

```bash
# Nginx im Hintergrund mit Port-Mapping und Name
docker run -d --name webserver -p 8080:80 nginx:alpine

# Interaktive Shell in Ubuntu
docker run -it --rm ubuntu:24.04 /bin/bash

# Container mit Volume, Env-Vars und Restart-Policy
docker run -d \
  --name postgres-db \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=geheim \
  --restart unless-stopped \
  postgres:16
```

### docker start / stop / restart

```bash
docker start   CONTAINER [CONTAINER...]
docker stop    CONTAINER [CONTAINER...]
docker restart CONTAINER [CONTAINER...]
```

| Flag | Beschreibung |
|:-----|:-------------|
| `-t, --time SEKUNDEN` | Timeout vor SIGKILL (Standard: 10s) |
| `-a, --attach` | STDOUT/STDERR anhängen (nur `start`) |

```bash
docker stop -t 30 webserver       # 30s Graceful Shutdown
docker restart postgres-db        # Neustart
docker start -a mein-container    # Starten und Output anzeigen
```

### docker rm

Entfernt einen oder mehrere gestoppte Container.

```bash
docker rm [OPTIONEN] CONTAINER [CONTAINER...]
```

| Flag | Beschreibung |
|:-----|:-------------|
| `-f, --force` | Laufenden Container stoppen und entfernen |
| `-v, --volumes` | Zugehörige anonyme Volumes entfernen |

```bash
docker rm webserver               # Gestoppten Container entfernen
docker rm -f webserver            # Laufenden Container erzwungen entfernen
docker rm -v mein-container       # Container samt anonymer Volumes entfernen
```

### docker ps

Listet Container auf.

```bash
docker ps [OPTIONEN]
```

| Flag | Beschreibung |
|:-----|:-------------|
| `-a, --all` | Alle Container (auch gestoppte) |
| `-q, --quiet` | Nur Container-IDs ausgeben |
| `-s, --size` | Speicherverbrauch anzeigen |
| `-f, --filter` | Filtern (z. B. `status=running`) |
| `--format` | Ausgabe-Format (Go Template) |
| `-n, --last N` | Letzte N Container anzeigen |

```bash
docker ps                          # Laufende Container
docker ps -a                       # Alle Container
docker ps -aq                      # Nur IDs aller Container
docker ps -f "status=exited"       # Nur gestoppte Container
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### docker logs

Zeigt die Logs eines Containers an.

```bash
docker logs [OPTIONEN] CONTAINER
```

| Flag | Beschreibung |
|:-----|:-------------|
| `-f, --follow` | Logs in Echtzeit verfolgen |
| `--tail N` | Letzte N Zeilen anzeigen |
| `-t, --timestamps` | Zeitstempel anzeigen |
| `--since ZEIT` | Logs seit Zeitpunkt |
| `--until ZEIT` | Logs bis Zeitpunkt |

```bash
docker logs webserver              # Alle Logs
docker logs -f --tail 50 webserver # Letzte 50 Zeilen + Follow
docker logs --since 1h webserver   # Logs der letzten Stunde
docker logs -t webserver           # Logs mit Zeitstempel
```

### docker exec

Führt einen Befehl in einem laufenden Container aus.

```bash
docker exec [OPTIONEN] CONTAINER BEFEHL [ARGS...]
```

| Flag | Beschreibung |
|:-----|:-------------|
| `-it` | Interaktiver Modus mit TTY |
| `-d, --detach` | Befehl im Hintergrund ausführen |
| `-u, --user` | Benutzer für den Befehl |
| `-w, --workdir` | Arbeitsverzeichnis setzen |
| `-e, --env` | Umgebungsvariable setzen |

```bash
docker exec -it webserver /bin/sh           # Shell im Container öffnen
docker exec webserver cat /etc/nginx/nginx.conf  # Datei lesen
docker exec -u root webserver apt update    # Als Root ausführen
docker exec -w /app webserver ls -la        # In bestimmtem Verzeichnis
```

### docker inspect

Gibt detaillierte JSON-Informationen über ein Objekt zurück.

```bash
docker inspect [OPTIONEN] OBJEKT [OBJEKT...]
```

| Flag | Beschreibung |
|:-----|:-------------|
| `--format, -f` | Ausgabe mit Go Template filtern |
| `--type` | Typ erzwingen (`container`, `image`, `volume`, `network`) |

```bash
docker inspect webserver
docker inspect -f '{{.State.Status}}' webserver
docker inspect -f '{{.NetworkSettings.IPAddress}}' webserver
docker inspect -f '{{json .Config.Env}}' webserver | jq
```

### docker stats

Zeigt Live-Ressourcenverbrauch der Container.

```bash
docker stats [OPTIONEN] [CONTAINER...]
```

| Flag | Beschreibung |
|:-----|:-------------|
| `--no-stream` | Nur einmalige Ausgabe (kein Live-Update) |
| `--format` | Ausgabe-Format |
| `-a, --all` | Alle Container (auch gestoppte) |

```bash
docker stats                            # Alle laufenden Container
docker stats webserver postgres-db      # Bestimmte Container
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### docker top / cp / rename / wait

```bash
# Prozesse im Container anzeigen
docker top webserver

# Dateien zwischen Host und Container kopieren
docker cp webserver:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./config.yaml webserver:/app/config.yaml

# Container umbenennen
docker rename alter-name neuer-name

# Warten bis Container beendet wird (gibt Exit-Code zurück)
docker wait mein-container
```

### docker attach / diff / export / commit

```bash
# An laufenden Container anhängen (STDOUT/STDERR)
docker attach webserver
# Detach mit: Ctrl+P, Ctrl+Q

# Dateisystem-Änderungen gegenüber dem Image anzeigen
docker diff webserver
# A = hinzugefügt, C = geändert, D = gelöscht

# Container-Dateisystem als tar exportieren
docker export webserver > webserver-backup.tar

# Container als neues Image speichern
docker commit webserver mein-image:v1
docker commit -m "Nginx konfiguriert" webserver mein-image:v2
```

> [!WARNING] docker commit vermeiden
> `docker commit` erzeugt nicht reproduzierbare Images. Besser ein `Dockerfile` verwenden und mit `docker build` bauen.

---

## Image-Befehle

### docker images / pull / push

```bash
# Lokale Images auflisten
docker images [OPTIONEN]
docker images -a               # Alle (inkl. Zwischenlayer)
docker images -q               # Nur IDs
docker images --filter "dangling=true"   # Unbenutzte Images

# Image aus Registry herunterladen
docker pull IMAGE[:TAG]
docker pull nginx:alpine
docker pull ghcr.io/user/app:latest

# Image in Registry hochladen
docker push IMAGE[:TAG]
docker push mein-registry.de/app:v1.2
```

### docker build

Baut ein Image aus einem Dockerfile.

```bash
docker build [OPTIONEN] PFAD
```

| Flag | Beschreibung |
|:-----|:-------------|
| `-t, --tag` | Image-Name und Tag setzen |
| `-f, --file` | Alternatives Dockerfile angeben |
| `--build-arg` | Build-Argument übergeben |
| `--no-cache` | Cache ignorieren |
| `--target` | Multi-Stage Build Target |
| `--platform` | Zielplattform (`linux/amd64`, `linux/arm64`) |
| `--progress` | Fortschrittsanzeige (`auto`, `plain`, `tty`) |
| `--pull` | Basis-Image immer neu laden |

```bash
docker build -t mein-app:v1 .
docker build -t app:latest -f Dockerfile.prod .
docker build --build-arg VERSION=3.2 -t app:3.2 .
docker build --no-cache --pull -t app:fresh .
docker build --target production -t app:prod .
```

### docker tag / rmi

```bash
# Image taggen (Alias erstellen)
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
docker tag mein-app:v1 registry.example.com/app:v1
docker tag mein-app:v1 mein-app:latest

# Image löschen
docker rmi IMAGE [IMAGE...]
docker rmi nginx:alpine
docker rmi -f $(docker images -q)    # Alle Images löschen (Vorsicht!)
```

### docker history / save / load / import

```bash
# Build-Historie eines Images anzeigen
docker history nginx:alpine
docker history --no-trunc nginx:alpine   # Vollständige Befehle

# Image als tar-Archiv speichern
docker save -o backup.tar nginx:alpine
docker save nginx:alpine postgres:16 > multi-backup.tar

# Image aus tar-Archiv laden
docker load -i backup.tar
docker load < backup.tar

# Dateisystem-Archiv als Image importieren
docker import backup.tar mein-image:restored
```

### docker image prune

```bash
# Unbenutzte (dangling) Images entfernen
docker image prune

# Alle unbenutzten Images entfernen (nicht nur dangling)
docker image prune -a

# Ohne Bestätigung
docker image prune -a -f

# Älter als 24 Stunden
docker image prune -a --filter "until=24h"
```

---

## Volume-Befehle

### Übersicht

| Befehl | Beschreibung |
|:-------|:-------------|
| `docker volume create` | Volume erstellen |
| `docker volume ls` | Volumes auflisten |
| `docker volume inspect` | Volume-Details anzeigen |
| `docker volume rm` | Volume entfernen |
| `docker volume prune` | Unbenutzte Volumes entfernen |

### Beispiele

```bash
# Volume erstellen
docker volume create mein-volume
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/export/data \
  nfs-volume

# Volumes auflisten
docker volume ls
docker volume ls -q                       # Nur Namen
docker volume ls -f "dangling=true"       # Unbenutzte Volumes

# Volume-Details anzeigen
docker volume inspect mein-volume
docker volume inspect -f '{{.Mountpoint}}' mein-volume

# Volume entfernen
docker volume rm mein-volume
docker volume rm $(docker volume ls -q)   # Alle Volumes löschen (Vorsicht!)

# Unbenutzte Volumes bereinigen
docker volume prune
docker volume prune -f                    # Ohne Bestätigung
```

### Volume-Mounting bei docker run

```bash
# Named Volume
docker run -d -v mein-volume:/app/data nginx

# Bind Mount (Host-Verzeichnis)
docker run -d -v /host/pfad:/container/pfad nginx

# Read-Only Mount
docker run -d -v mein-volume:/app/data:ro nginx

# tmpfs Mount (nur im RAM)
docker run -d --tmpfs /tmp:rw,noexec,size=100m nginx
```

> [!TIP] Named Volumes bevorzugen
> Named Volumes werden von Docker verwaltet und sind portabler als Bind Mounts. Für Konfigurationsdateien sind Bind Mounts jedoch oft praktischer.

---

## Netzwerk-Befehle

### Übersicht

| Befehl | Beschreibung |
|:-------|:-------------|
| `docker network create` | Netzwerk erstellen |
| `docker network ls` | Netzwerke auflisten |
| `docker network inspect` | Netzwerk-Details anzeigen |
| `docker network rm` | Netzwerk entfernen |
| `docker network connect` | Container mit Netzwerk verbinden |
| `docker network disconnect` | Container vom Netzwerk trennen |
| `docker network prune` | Unbenutzte Netzwerke entfernen |

### Beispiele

```bash
# Netzwerk erstellen
docker network create mein-netz
docker network create --driver bridge --subnet 172.20.0.0/16 mein-netz
docker network create --driver overlay --attachable swarm-netz

# Netzwerke auflisten
docker network ls
docker network ls --filter "driver=bridge"

# Netzwerk-Details
docker network inspect mein-netz
docker network inspect -f '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}' mein-netz

# Container mit Netzwerk verbinden / trennen
docker network connect mein-netz webserver
docker network connect --ip 172.20.0.10 mein-netz webserver
docker network disconnect mein-netz webserver

# Netzwerk entfernen
docker network rm mein-netz

# Unbenutzte Netzwerke bereinigen
docker network prune -f
```

### Netzwerk-Treiber

| Treiber | Beschreibung |
|:--------|:-------------|
| `bridge` | Standard — isoliertes Netzwerk auf einem Host |
| `host` | Container nutzt Host-Netzwerk direkt |
| `none` | Kein Netzwerk |
| `overlay` | Multi-Host-Netzwerk (Swarm) |
| `macvlan` | Container erhält eigene MAC-Adresse |
| `ipvlan` | Wie macvlan, aber mit geteilter MAC |

```bash
# Container im Host-Netzwerk starten
docker run -d --network host nginx

# Container ohne Netzwerk
docker run -d --network none alpine sleep 3600

# Zwei Container im gleichen Netzwerk (können sich per Name erreichen)
docker network create app-netz
docker run -d --name api --network app-netz mein-api:latest
docker run -d --name frontend --network app-netz mein-frontend:latest
# Im frontend-Container: curl http://api:8080
```

---

## Docker Compose-Befehle

> [!NOTE] Docker Compose v2
> Seit Docker Compose v2 wird `docker compose` (ohne Bindestrich) als Plugin verwendet. Das alte `docker-compose` (mit Bindestrich) ist veraltet.

### Übersicht

| Befehl | Beschreibung |
|:-------|:-------------|
| `docker compose up` | Services erstellen und starten |
| `docker compose down` | Services stoppen und entfernen |
| `docker compose start` | Gestoppte Services starten |
| `docker compose stop` | Laufende Services stoppen |
| `docker compose restart` | Services neu starten |
| `docker compose ps` | Service-Status anzeigen |
| `docker compose logs` | Service-Logs anzeigen |
| `docker compose exec` | Befehl in laufendem Service ausführen |
| `docker compose build` | Service-Images bauen |
| `docker compose pull` | Service-Images herunterladen |
| `docker compose config` | Compose-Datei validieren und anzeigen |
| `docker compose top` | Prozesse der Services anzeigen |

### docker compose up / down

```bash
# Services starten (Vordergrund)
docker compose up

# Services im Hintergrund starten
docker compose up -d

# Bestimmte Services starten
docker compose up -d api frontend

# Images vor dem Start neu bauen
docker compose up -d --build

# Neustart erzwingen (Container neu erstellen)
docker compose up -d --force-recreate

# Mit spezifischer Compose-Datei
docker compose -f docker-compose.prod.yml up -d

# Services stoppen und Container + Netzwerke entfernen
docker compose down

# Zusätzlich Volumes entfernen
docker compose down -v

# Zusätzlich Images entfernen
docker compose down --rmi all
```

### docker compose logs / exec / ps

```bash
# Logs aller Services
docker compose logs

# Logs eines bestimmten Services (Follow)
docker compose logs -f api

# Letzte 100 Zeilen mit Timestamps
docker compose logs --tail 100 -t api

# Shell in einem Service öffnen
docker compose exec api /bin/sh

# Befehl in einem Service ausführen
docker compose exec db psql -U postgres

# Status aller Services
docker compose ps
docker compose ps -a     # Inkl. gestoppte
```

### docker compose build / pull / config

```bash
# Images bauen
docker compose build
docker compose build --no-cache api   # Ohne Cache für bestimmten Service

# Images herunterladen
docker compose pull

# Compose-Konfiguration validieren und ausgeben
docker compose config
docker compose config --services      # Nur Service-Namen
docker compose config --volumes       # Nur Volumes
```

### docker compose stop / restart / scale

```bash
# Services stoppen (ohne zu entfernen)
docker compose stop
docker compose stop api

# Services neustarten
docker compose restart
docker compose restart api

# Service skalieren
docker compose up -d --scale api=3
```

---

## System-Befehle

### docker info / version

```bash
# Detaillierte Docker-Systeminformationen
docker info

# Docker-Version anzeigen
docker version
docker version --format '{{.Server.Version}}'
```

### docker system df

Zeigt den Speicherverbrauch von Docker-Objekten.

```bash
docker system df
docker system df -v    # Detaillierte Ansicht
```

### docker system prune

Bereinigt unbenutzte Objekte.

```bash
# Gestoppte Container, dangling Images, unbenutzte Netzwerke
docker system prune

# Zusätzlich unbenutzte Volumes
docker system prune --volumes

# Alle unbenutzten Images (nicht nur dangling)
docker system prune -a

# Ohne Bestätigung + alles bereinigen
docker system prune -af --volumes

# Nur Objekte älter als 24 Stunden
docker system prune -a --filter "until=24h"
```

> [!DANGER] system prune mit Vorsicht
> `docker system prune -af --volumes` löscht **alle** ungenutzten Ressourcen inklusive Volumes. Daten in nicht gemounteten Volumes gehen unwiderruflich verloren!

### docker system events

Live-Stream aller Docker-Events.

```bash
# Alle Events in Echtzeit
docker system events

# Events eines bestimmten Containers
docker system events --filter "container=webserver"

# Nur bestimmte Event-Typen
docker system events --filter "type=container" --filter "event=start"

# Events seit einem Zeitpunkt
docker system events --since "2025-01-01T00:00:00"

# Events als JSON
docker system events --format '{{json .}}'
```

### docker login / logout

```bash
# In Docker Hub einloggen
docker login

# In private Registry einloggen
docker login registry.example.com
docker login -u benutzername -p token ghcr.io

# Ausloggen
docker logout
docker logout registry.example.com
```

---

## Nützliche Kombinationen

### Container-Operationen

```bash
# Alle laufenden Container stoppen
docker stop $(docker ps -q)

# Alle Container entfernen (auch laufende)
docker rm -f $(docker ps -aq)

# Alle gestoppten Container entfernen
docker container prune -f

# Container-ID anhand des Namens finden
docker ps -qf "name=webserver"

# Letzten erstellten Container anzeigen
docker ps -l
```

### Image-Operationen

```bash
# Alle dangling Images löschen
docker rmi $(docker images -f "dangling=true" -q)

# Alle Images löschen (Vorsicht!)
docker rmi -f $(docker images -q)

# Images nach Größe sortiert anzeigen
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | sort -k2 -h

# Unbenutzte Images älter als 7 Tage löschen
docker image prune -a --filter "until=168h" -f
```

### Monitoring und Debugging

```bash
# Logs mit Timestamps und seit bestimmter Zeit
docker logs -t --since 30m webserver

# Container-IP herausfinden
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' webserver

# Alle Container-IPs auflisten
docker ps -q | xargs -I {} docker inspect \
  -f '{{.Name}} → {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' {}

# Container-Ressourcen (einmalig, nicht live)
docker stats --no-stream

# Port-Mappings eines Containers anzeigen
docker port webserver

# Container-Umgebungsvariablen anzeigen
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' webserver

# Health-Check-Status prüfen
docker inspect -f '{{.State.Health.Status}}' webserver
```

### Aufräumen

```bash
# Komplettes Aufräumen (Nuclear Option)
docker system prune -af --volumes

# Nur alte, unbenutzte Objekte entfernen (älter als 48h)
docker system prune -af --filter "until=48h"

# Speicherverbrauch vor/nach Aufräumen prüfen
docker system df
docker system prune -af
docker system df
```

### Backup und Migration

```bash
# Volume-Backup als tar-Archiv
docker run --rm \
  -v mein-volume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mein-volume-backup.tar.gz -C /data .

# Volume-Restore aus tar-Archiv
docker run --rm \
  -v mein-volume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/mein-volume-backup.tar.gz -C /data

# Image exportieren und auf anderem Host importieren
docker save mein-app:latest | gzip > mein-app.tar.gz
# Auf dem Zielhost:
docker load < mein-app.tar.gz
```

---

## Formatierung mit --format

Docker nutzt Go Templates für die Ausgabeformatierung. Das Flag `--format` ist bei den meisten Befehlen verfügbar.

### Syntax-Grundlagen

| Ausdruck | Beschreibung |
|:---------|:-------------|
| `{{.Feld}}` | Wert eines Feldes ausgeben |
| `{{.Feld.Unterfeld}}` | Verschachtelter Wert |
| `{{json .}}` | Gesamte Ausgabe als JSON |
| `{{json .Feld}}` | Bestimmtes Feld als JSON |
| `{{range .Liste}}...{{end}}` | Über Liste iterieren |
| `{{if .Feld}}...{{end}}` | Bedingte Ausgabe |
| `{{println .Feld}}` | Wert mit Zeilenumbruch |
| `\t` | Tabulator (in `table`-Format) |
| `table` | Tabellarische Ausgabe mit Header |

### Nützliche Format-Strings

```bash
# Container — Name, Status, Ports (tabellarisch)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Container — Name und Image
docker ps --format "{{.Names}}: {{.Image}}"

# Container — Alle Infos als JSON
docker ps --format '{{json .}}' | jq

# Images — Repository, Tag, Größe
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Images — Nur Repository:Tag
docker images --format "{{.Repository}}:{{.Tag}}"

# Volumes — Name und Mountpoint
docker volume ls --format "table {{.Name}}\t{{.Mountpoint}}"

# Netzwerke — Name, Treiber, Scope
docker network ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}"

# Container-IP aus inspect
docker inspect \
  -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' webserver

# Container-Mounts auflisten
docker inspect \
  -f '{{range .Mounts}}{{.Type}}: {{.Source}} → {{.Destination}}{{"\n"}}{{end}}' webserver

# Container-Labels auflisten
docker inspect \
  -f '{{range $k, $v := .Config.Labels}}{{$k}}={{$v}}{{"\n"}}{{end}}' webserver
```

> [!TIP] JSON + jq für komplexe Abfragen
> Für komplexere Filterung die Ausgabe als JSON erzeugen und mit `jq` weiterverarbeiten:
> ```bash
> docker inspect webserver | jq '.[0].NetworkSettings.Networks'
> docker ps --format '{{json .}}' | jq -r '.Names + " " + .Status'
> ```

### Verfügbare Felder

**docker ps:**

| Feld | Beschreibung |
|:-----|:-------------|
| `.ID` | Container-ID |
| `.Image` | Image-Name |
| `.Command` | Ausgeführter Befehl |
| `.CreatedAt` | Erstellungszeitpunkt |
| `.Status` | Aktueller Status |
| `.Ports` | Port-Mappings |
| `.Names` | Container-Name |
| `.Size` | Speicherverbrauch |
| `.Mounts` | Gemountete Volumes |
| `.Networks` | Zugewiesene Netzwerke |
| `.Labels` | Container-Labels |

**docker images:**

| Feld | Beschreibung |
|:-----|:-------------|
| `.Repository` | Repository-Name |
| `.Tag` | Image-Tag |
| `.ID` | Image-ID |
| `.CreatedAt` | Erstellungszeitpunkt |
| `.CreatedSince` | Alter (relativ) |
| `.Size` | Image-Größe |
| `.Digest` | Image-Digest |

---

## Verwandte Notizen

- [[Docker Index]]
- [[Container Lifecycle]]
- [[Docker Images]]
- [[Docker Volumes und Storage]]
- [[Docker Networking]]
- [[kubectl Befehle und Cheatsheet]] — Das K8s-Pendant zur Docker CLI

---

### Qualitätscheckliste

| Kriterium | Status |
|:----------|:-------|
| Wiki-Links | ✅ `[[Docker Index]]` existiert — die anderen 4 sind als Platzhalter im [[Docker Index]] bereits verlinkt |
| Code-Blöcke | ✅ Alle mit `bash` Sprachkennung |
| Callouts | ✅ `[!ABSTRACT]`, `[!WARNING]`, `[!DANGER]`, `[!TIP]`, `[!NOTE]` korrekt |
| Tabellen | ✅ Header-Separator vorhanden, linksbündig mit `:` |
| Horizontale Linien | ✅ `---` zwischen allen Hauptabschnitten |
| Nur ein H1 | ✅ Nur `# Docker CLI Cheatsheet` |
| Frontmatter | ✅ Tags als Array |
| Deutsche Sprache | ✅ Fachbegriffe Englisch, Rest Deutsch |
| Zielgröße | ✅ ~640 Zeilen |
| Verschachtelte Fences | ✅ Innerer `bash`-Block im `[!TIP]` korrekt eingerückt |

### Zum manuellen Anlegen

```powershell
# Verzeichnis erstellen und Datei anlegen (PowerShell)
New-Item -ItemType Directory -Force -Path "C:\Development\notes-n-docs\Container\Docker\Betrieb"
# Dann den Inhalt oben in die Datei einfügen