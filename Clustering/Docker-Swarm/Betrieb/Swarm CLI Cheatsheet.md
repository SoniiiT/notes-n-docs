---
tags:
  - docker-swarm
  - cli
  - cheatsheet
  - befehle
---

# Swarm CLI Cheatsheet

> [!ABSTRACT] TL;DR
> Schnellreferenz aller Docker Swarm CLI-Befehle — Cluster-Verwaltung, Services, Stacks, Nodes, Secrets, Configs und Netzwerke. Alle Kommandos mit Erklärung, Flags und Praxisbeispielen.

---

## Cluster-Verwaltung (docker swarm)

### Swarm initialisieren

```bash
# Swarm auf dem aktuellen Node starten
docker swarm init

# Mit expliziter Advertise-Adresse (empfohlen bei mehreren NICs)
docker swarm init --advertise-addr 192.168.1.10

# Mit benutzerdefiniertem Listen-Port
docker swarm init --advertise-addr 192.168.1.10 --listen-addr 0.0.0.0:2377

# Mit erzwungenem neuen Cluster (Disaster Recovery)
docker swarm init --force-new-cluster --advertise-addr 192.168.1.10
```

> [!TIP] Advertise-Adresse
> Bei Servern mit mehreren Netzwerkschnittstellen immer `--advertise-addr` angeben, damit die Nodes die richtige IP für die Cluster-Kommunikation verwenden.

### Swarm beitreten und verlassen

```bash
# Als Worker beitreten
docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377

# Als Manager beitreten
docker swarm join --token SWMTKN-1-yyy 192.168.1.10:2377

# Worker verlässt den Swarm
docker swarm leave

# Manager verlässt den Swarm (muss erzwungen werden)
docker swarm leave --force
```

> [!WARNING] Manager Force-Leave
> Ein Manager sollte nur mit `--force` entfernt werden, wenn er zuvor demoted wurde oder der Cluster aufgelöst werden soll. Bei einem 3-Manager-Cluster führt das unkontrollierte Entfernen eines Managers zum Quorum-Verlust.

### Join-Tokens verwalten

```bash
# Worker-Token anzeigen
docker swarm join-token worker

# Manager-Token anzeigen
docker swarm join-token manager

# Nur den Token-String ausgeben (ohne den vollständigen Befehl)
docker swarm join-token -q worker
docker swarm join-token -q manager

# Worker-Token rotieren (alter Token wird ungültig)
docker swarm join-token --rotate worker

# Manager-Token rotieren
docker swarm join-token --rotate manager
```

> [!IMPORTANT] Token-Rotation
> Nach dem Ausscheiden eines Mitarbeiters oder bei Verdacht auf Token-Kompromittierung sollten die Tokens sofort rotiert werden. Bestehende Nodes sind davon nicht betroffen.

### Swarm-Einstellungen aktualisieren

```bash
# Autolock aktivieren (Unlock-Key wird benötigt nach Manager-Neustart)
docker swarm update --autolock=true

# Autolock deaktivieren
docker swarm update --autolock=false

# Swarm nach Manager-Neustart entsperren
docker swarm unlock

# Aktuellen Unlock-Key anzeigen
docker swarm unlock-key

# Unlock-Key rotieren
docker swarm unlock-key --rotate

# TLS-Zertifikate rotieren
docker swarm ca --rotate

# Zertifikats-Gültigkeitsdauer auf 48h setzen
docker swarm update --cert-expiry 48h

# Task-History-Limit anpassen (Standard: 5)
docker swarm update --task-history-limit 10

# Dispatcher-Heartbeat anpassen
docker swarm update --dispatcher-heartbeat 10s
```

---

## Node-Verwaltung (docker node)

### Nodes auflisten und inspizieren

```bash
# Alle Nodes im Cluster anzeigen
docker node ls

# Mit benutzerdefiniertem Format
docker node ls --format "table {{.Hostname}}\t{{.Status}}\t{{.Availability}}\t{{.ManagerStatus}}"

# Nur Node-IDs ausgeben
docker node ls -q

# Nach Labels filtern
docker node ls --filter "node.label=env=production"

# Detailinformationen zu einem Node
docker node inspect <NODE-ID|HOSTNAME>

# Lesbares Format (Pretty Print)
docker node inspect --pretty <NODE-ID|HOSTNAME>

# Alle Tasks auf einem Node anzeigen
docker node ps <NODE-ID|HOSTNAME>

# Auch beendete Tasks anzeigen
docker node ps --no-trunc <NODE-ID|HOSTNAME>

# Tasks mit Format filtern
docker node ps <NODE-ID|HOSTNAME> --format "table {{.Name}}\t{{.Image}}\t{{.CurrentState}}\t{{.Error}}"
```

### Node-Rollen verwalten

```bash
# Worker zum Manager befördern
docker node promote <NODE-ID|HOSTNAME>

# Mehrere Nodes gleichzeitig befördern
docker node promote node-02 node-03

# Manager zum Worker degradieren
docker node demote <NODE-ID|HOSTNAME>

# Mehrere Nodes gleichzeitig degradieren
docker node demote node-02 node-03
```

> [!TIP] Manager-Anzahl
> Für Hochverfügbarkeit werden 3 oder 5 Manager empfohlen. Eine gerade Anzahl bietet keinen Vorteil gegenüber der nächstkleineren ungeraden Zahl.

| Manager-Anzahl | Tolerierte Ausfälle | Empfehlung |
|:--------------:|:--------------------:|:----------:|
| 1              | 0                    | Nur Test   |
| 2              | 0                    | Nie        |
| 3              | 1                    | Standard   |
| 5              | 2                    | Groß       |
| 7              | 3                    | Selten     |

### Node-Availability steuern

```bash
# Node auf Active setzen (Standard — nimmt neue Tasks an)
docker node update --availability active <NODE-ID|HOSTNAME>

# Node auf Pause setzen (keine neuen Tasks, bestehende laufen weiter)
docker node update --availability pause <NODE-ID|HOSTNAME>

# Node auf Drain setzen (alle Tasks werden verschoben)
docker node update --availability drain <NODE-ID|HOSTNAME>
```

> [!NOTE] Drain-Verhalten
> Beim Setzen auf `drain` werden alle laufenden Tasks beendet und auf anderen verfügbaren Nodes neu gestartet. Typischer Einsatz: Wartungsarbeiten, Kernel-Updates, Hardware-Austausch.

### Labels verwalten

```bash
# Label hinzufügen
docker node update --label-add env=production <NODE-ID|HOSTNAME>

# Mehrere Labels gleichzeitig setzen
docker node update --label-add env=production --label-add zone=eu-west <NODE-ID|HOSTNAME>

# Label entfernen
docker node update --label-rm env <NODE-ID|HOSTNAME>

# Labels eines Nodes anzeigen
docker node inspect --format '{{json .Spec.Labels}}' <NODE-ID|HOSTNAME>
```

### Node entfernen

```bash
# Node aus dem Cluster entfernen (muss vorher den Swarm verlassen haben)
docker node rm <NODE-ID|HOSTNAME>

# Nicht erreichbaren Node erzwungen entfernen
docker node rm --force <NODE-ID|HOSTNAME>
```

---

## Service-Verwaltung (docker service)

### Service erstellen

```bash
# Einfachen Service erstellen
docker service create --name web nginx:alpine

# Service mit Replicas und Port-Mapping
docker service create --name web \
  --replicas 3 \
  --publish 80:80 \
  nginx:alpine

# Service mit Ressourcen-Limits
docker service create --name api \
  --replicas 2 \
  --limit-cpu 0.5 \
  --limit-memory 256M \
  --reserve-cpu 0.25 \
  --reserve-memory 128M \
  myapp:latest

# Service mit Umgebungsvariablen
docker service create --name db \
  --env POSTGRES_PASSWORD=geheim \
  --env POSTGRES_DB=mydb \
  --env-file ./env.list \
  postgres:16

# Service mit Volumes und Mounts
docker service create --name files \
  --mount type=volume,src=data,dst=/data \
  --mount type=bind,src=/host/config,dst=/etc/app/config,readonly \
  myapp:latest

# Service mit Netzwerk und Constraint
docker service create --name web \
  --network my-overlay \
  --constraint 'node.labels.env == production' \
  --constraint 'node.role == worker' \
  nginx:alpine

# Global Service (ein Task pro Node)
docker service create --name monitoring \
  --mode global \
  node-exporter:latest

# Service mit Health-Check
docker service create --name api \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  --health-start-period 60s \
  myapp:latest

# Service mit Secrets und Configs
docker service create --name web \
  --secret db_password \
  --config source=nginx_conf,target=/etc/nginx/nginx.conf \
  nginx:alpine

# Service mit Update-Konfiguration
docker service create --name web \
  --replicas 6 \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-failure-action rollback \
  --update-order start-first \
  nginx:alpine

# Service mit Rollback-Konfiguration
docker service create --name web \
  --replicas 4 \
  --rollback-parallelism 1 \
  --rollback-delay 5s \
  --rollback-failure-action pause \
  --rollback-order stop-first \
  nginx:alpine

# Service mit Restart-Policy
docker service create --name worker \
  --restart-condition on-failure \
  --restart-delay 5s \
  --restart-max-attempts 3 \
  --restart-window 120s \
  myworker:latest

# Service mit Placement-Preferences (Spread über Zonen)
docker service create --name web \
  --replicas 6 \
  --placement-pref 'spread=node.labels.zone' \
  nginx:alpine
```

### Services auflisten und inspizieren

```bash
# Alle Services anzeigen
docker service ls

# Mit benutzerdefiniertem Format
docker service ls --format "table {{.Name}}\t{{.Replicas}}\t{{.Image}}\t{{.Ports}}"

# Nach Labels filtern
docker service ls --filter "label=app=web"

# Service-Details anzeigen
docker service inspect <SERVICE>

# Lesbares Format
docker service inspect --pretty <SERVICE>

# Nur das Image eines Services ausgeben
docker service inspect --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}' <SERVICE>

# Tasks eines Services anzeigen
docker service ps <SERVICE>

# Nur laufende Tasks anzeigen
docker service ps --filter "desired-state=running" <SERVICE>

# Tasks mit benutzerdefiniertem Format
docker service ps <SERVICE> --format "table {{.Node}}\t{{.CurrentState}}\t{{.Error}}"

# Keine abgeschnittenen IDs/Fehlermeldungen
docker service ps --no-trunc <SERVICE>
```

### Service-Logs

```bash
# Logs eines Services anzeigen
docker service logs <SERVICE>

# Logs mit Timestamps
docker service logs --timestamps <SERVICE>

# Letzte 100 Zeilen
docker service logs --tail 100 <SERVICE>

# Live-Logs (Follow)
docker service logs -f <SERVICE>

# Logs eines bestimmten Tasks
docker service logs <SERVICE>.1

# Logs seit einem Zeitpunkt
docker service logs --since 2025-01-01T10:00:00 <SERVICE>

# Logs der letzten 30 Minuten
docker service logs --since 30m <SERVICE>

# Ohne Präfix (Node/Task-ID) ausgeben
docker service logs --no-task-ids --no-trunc <SERVICE>
```

### Skalierung

```bash
# Service auf 5 Replicas skalieren
docker service scale web=5

# Mehrere Services gleichzeitig skalieren
docker service scale web=5 api=3 worker=10

# Auf 0 skalieren (Service pausieren ohne ihn zu löschen)
docker service scale web=0
```

### Service aktualisieren (Rolling Update)

```bash
# Image aktualisieren
docker service update --image nginx:1.27 web

# Umgebungsvariable hinzufügen
docker service update --env-add NEW_VAR=value web

# Umgebungsvariable entfernen
docker service update --env-rm OLD_VAR web

# Port hinzufügen
docker service update --publish-add 8080:80 web

# Port entfernen
docker service update --publish-rm 80 web

# Label hinzufügen
docker service update --label-add version=2.0 web

# Constraint ändern
docker service update --constraint-rm 'node.role == worker' \
  --constraint-add 'node.labels.env == staging' web

# Replicas ändern
docker service update --replicas 10 web

# Ressourcen-Limits anpassen
docker service update --limit-cpu 1.0 --limit-memory 512M web

# Netzwerk hinzufügen
docker service update --network-add my-new-overlay web

# Netzwerk entfernen
docker service update --network-rm old-overlay web

# Secret hinzufügen / entfernen
docker service update --secret-add new_secret --secret-rm old_secret web

# Config hinzufügen
docker service update --config-add source=app_conf,target=/etc/app.conf web

# Update erzwingen (Tasks neu starten ohne Änderung)
docker service update --force web

# Update-Parameter ändern
docker service update --update-parallelism 3 --update-delay 15s web
```

> [!TIP] Force-Update
> `docker service update --force` startet alle Tasks neu, auch wenn sich nichts geändert hat. Nützlich zum Neuverteilen von Tasks nach dem Zurückholen eines Nodes aus dem Drain-Modus.

### Rollback

```bash
# Service auf vorherige Version zurückrollen
docker service rollback <SERVICE>

# Rollback ohne Bestätigung
docker service rollback --detach <SERVICE>
```

### Service entfernen

```bash
# Einzelnen Service entfernen
docker service rm <SERVICE>

# Mehrere Services gleichzeitig entfernen
docker service rm web api worker
```

---

## Stack-Verwaltung (docker stack)

### Stack deployen

```bash
# Stack aus Compose-Datei deployen
docker stack deploy -c docker-compose.yml mystack

# Stack aus mehreren Compose-Dateien deployen
docker stack deploy \
  -c docker-compose.yml \
  -c docker-compose.prod.yml \
  mystack

# Stack mit Image-Auflösung
docker stack deploy -c docker-compose.yml mystack --resolve-image always

# Stack erneut deployen (Update)
docker stack deploy -c docker-compose.yml mystack
```

> [!NOTE] Stack vs. Compose
> `docker stack deploy` nutzt die Compose-v3-Syntax, ignoriert aber `build`, `depends_on` und andere nicht Swarm-kompatible Optionen. Swarm-spezifische Konfiguration gehört unter den `deploy`-Schlüssel.

### Stacks auflisten und inspizieren

```bash
# Alle Stacks anzeigen
docker stack ls

# Services eines Stacks anzeigen
docker stack services mystack

# Mit benutzerdefiniertem Format
docker stack services mystack --format "table {{.Name}}\t{{.Replicas}}\t{{.Image}}"

# Tasks eines Stacks anzeigen
docker stack ps mystack

# Nur laufende Tasks
docker stack ps mystack --filter "desired-state=running"

# Fehlgeschlagene Tasks anzeigen
docker stack ps mystack --filter "desired-state=shutdown" --no-trunc
```

### Stack entfernen

```bash
# Stack und alle zugehörigen Ressourcen entfernen
docker stack rm mystack

# Mehrere Stacks entfernen
docker stack rm mystack anotherstack
```

> [!WARNING] Stack entfernen
> `docker stack rm` entfernt alle Services, Netzwerke, Secrets und Configs des Stacks sofort und ohne Bestätigungsabfrage. Volumes bleiben erhalten.

---

## Secret-Verwaltung (docker secret)

### Secrets erstellen

```bash
# Secret aus Datei erstellen
docker secret create db_password ./db_password.txt

# Secret aus STDIN erstellen
echo "mein-geheimes-passwort" | docker secret create db_password -

# Secret mit Label
docker secret create --label env=production db_password ./db_password.txt

# Secret aus Pipe (sicherer, kein echo in History)
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 32 | docker secret create api_key -
```

> [!TIP] Sichere Secret-Erstellung
> Vermeide `echo` mit Klartext-Passwörtern — der Befehl landet in der Shell-History. Nutze stattdessen `read -s` oder eine Datei, die danach gelöscht wird.

### Secrets verwalten

```bash
# Alle Secrets auflisten
docker secret ls

# Secret-Details anzeigen (zeigt NICHT den Inhalt!)
docker secret inspect <SECRET>

# Lesbares Format
docker secret inspect --pretty <SECRET>

# Secret entfernen
docker secret rm <SECRET>

# Mehrere Secrets entfernen
docker secret rm secret1 secret2 secret3
```

> [!IMPORTANT] Secret-Rotation
> Secrets können nicht aktualisiert werden. Workflow für die Rotation:
> 1. Neues Secret mit neuem Namen erstellen
> 2. Service aktualisieren: `--secret-rm old --secret-add new`
> 3. Altes Secret löschen

### Secrets in Services nutzen

```bash
# Secret beim Erstellen hinzufügen
docker service create --name db \
  --secret db_password \
  postgres:16

# Secret unter benutzerdefiniertem Pfad mounten
docker service create --name db \
  --secret source=db_password,target=/run/secrets/postgres_password,uid=999,gid=999,mode=0400 \
  postgres:16

# Secret zu bestehendem Service hinzufügen
docker service update --secret-add db_password db

# Secret von Service entfernen
docker service update --secret-rm db_password db
```

---

## Config-Verwaltung (docker config)

### Configs erstellen

```bash
# Config aus Datei erstellen
docker config create nginx_conf ./nginx.conf

# Config aus STDIN erstellen
echo "max_connections = 100" | docker config create app_conf -

# Config mit Label
docker config create --label app=web nginx_conf ./nginx.conf
```

### Configs verwalten

```bash
# Alle Configs auflisten
docker config ls

# Config-Details anzeigen (zeigt den Inhalt!)
docker config inspect <CONFIG>

# Lesbares Format
docker config inspect --pretty <CONFIG>

# Nur den Config-Inhalt ausgeben (Base64-dekodiert)
docker config inspect --format '{{json .Spec.Data}}' <CONFIG> | jq -r '.' | base64 -d

# Config entfernen
docker config rm <CONFIG>
```

### Configs in Services nutzen

```bash
# Config beim Erstellen zuweisen
docker service create --name web \
  --config source=nginx_conf,target=/etc/nginx/nginx.conf \
  nginx:alpine

# Config mit Berechtigungen
docker service create --name web \
  --config source=nginx_conf,target=/etc/nginx/nginx.conf,uid=101,gid=101,mode=0440 \
  nginx:alpine

# Config zu bestehendem Service hinzufügen
docker service update --config-add source=app_conf,target=/etc/app.conf web

# Config von Service entfernen
docker service update --config-rm app_conf web
```

> [!NOTE] Configs vs. Secrets
> Configs werden unverschlüsselt im Raft-Log gespeichert und sind über `docker config inspect` einsehbar. Für sensible Daten immer Secrets verwenden.

---

## Netzwerk (Swarm-spezifisch)

### Overlay-Netzwerke erstellen

```bash
# Overlay-Netzwerk erstellen
docker network create --driver overlay my-overlay

# Mit Verschlüsselung (IPsec-Tunnel zwischen Nodes)
docker network create --driver overlay --opt encrypted my-secure-overlay

# Mit benutzerdefiniertem Subnetz
docker network create --driver overlay \
  --subnet 10.10.0.0/16 \
  --gateway 10.10.0.1 \
  my-overlay

# Attachable Overlay (auch für Standalone-Container)
docker network create --driver overlay --attachable my-overlay

# Ingress-Netzwerk neu erstellen (nach Entfernung)
docker network create --driver overlay \
  --ingress \
  --subnet 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  my-ingress
```

> [!WARNING] Netzwerk-Verschlüsselung
> Die Option `--opt encrypted` aktiviert IPsec-Verschlüsselung zwischen den Nodes. Das verursacht zusätzliche CPU-Last und reduziert den Durchsatz. Nur aktivieren, wenn der Node-zu-Node-Traffic über unsichere Netzwerke läuft.

### Netzwerke verwalten

```bash
# Alle Netzwerke anzeigen
docker network ls

# Nur Overlay-Netzwerke anzeigen
docker network ls --filter driver=overlay

# Netzwerk inspizieren
docker network inspect my-overlay

# Verbundene Services anzeigen
docker network inspect --format '{{range .Containers}}{{.Name}} {{end}}' my-overlay

# Netzwerk entfernen
docker network rm my-overlay

# Ungenutzte Netzwerke aufräumen
docker network prune
```

---

## Nützliche Kombinationen

### Cluster-Übersicht

```bash
# Kompakte Cluster-Übersicht
echo "=== NODES ===" && \
docker node ls --format "table {{.Hostname}}\t{{.Status}}\t{{.Availability}}\t{{.ManagerStatus}}" && \
echo -e "\n=== SERVICES ===" && \
docker service ls --format "table {{.Name}}\t{{.Replicas}}\t{{.Image}}" && \
echo -e "\n=== STACKS ===" && \
docker stack ls
```

### Service-Debugging

```bash
# Fehlgeschlagene Tasks eines Services finden
docker service ps <SERVICE> --no-trunc --filter "desired-state=shutdown" \
  --format "table {{.Name}}\t{{.CurrentState}}\t{{.Error}}"

# Service-Logs mit Timestamps und Task-Zuordnung
docker service logs --timestamps --tail 50 <SERVICE>

# Auf welchen Nodes läuft ein Service?
docker service ps <SERVICE> --filter "desired-state=running" \
  --format "table {{.Node}}\t{{.CurrentState}}\t{{.Ports}}"

# Inspect eines bestimmten Tasks
docker inspect <TASK-ID>
```

### Node-Wartung

```bash
# Node in den Wartungsmodus versetzen
docker node update --availability drain <NODE>

# Prüfen ob alle Tasks migriert wurden
docker node ps <NODE> --filter "desired-state=running"

# Wartungsarbeiten durchführen...

# Node wieder aktivieren
docker node update --availability active <NODE>

# Tasks gleichmäßig neu verteilen
docker service ls -q | xargs -I {} docker service update --force {}
```

### Aufräumen

```bash
# Alle Services eines Stacks auflisten und deren Logs prüfen
docker stack services mystack -q | xargs -I {} docker service logs --tail 5 {}

# Verwaiste Volumes finden
docker volume ls --filter "dangling=true"

# System-weites Aufräumen auf allen Nodes
docker system prune -af --volumes

# Alle gestoppten Tasks (Container) entfernen
docker container prune -f
```

### Monitoring-Einzeiler

```bash
# Echtzeit-Übersicht aller Services (watch)
watch -n 5 'docker service ls --format "table {{.Name}}\t{{.Replicas}}\t{{.Image}}"'

# Node-Ressourcen prüfen
docker node ls -q | xargs -I {} docker node inspect --format \
  '{{.Description.Hostname}}: CPU={{.Description.Resources.NanoCPUs}} MEM={{.Description.Resources.MemoryBytes}}' {}

# Alle Service-Endpoints auflisten
docker service ls --format '{{.Name}}: {{.Ports}}'
```

---

## Vergleich: Docker Swarm CLI vs kubectl

| Aktion | Docker Swarm | Kubernetes (kubectl) |
|--------|-------------|----------------------|
| Cluster initialisieren | `docker swarm init` | `kubeadm init` |
| Node beitreten | `docker swarm join` | `kubeadm join` |
| Nodes auflisten | `docker node ls` | `kubectl get nodes` |
| Node-Details | `docker node inspect` | `kubectl describe node` |
| Node-Drain | `docker node update --availability drain` | `kubectl drain` |
| Service erstellen | `docker service create` | `kubectl create deployment` |
| Services auflisten | `docker service ls` | `kubectl get services` |
| Skalieren | `docker service scale web=5` | `kubectl scale --replicas=5` |
| Rolling Update | `docker service update --image` | `kubectl set image` |
| Rollback | `docker service rollback` | `kubectl rollout undo` |
| Logs anzeigen | `docker service logs` | `kubectl logs` |
| Stack deployen | `docker stack deploy -c` | `kubectl apply -f` |
| Stack entfernen | `docker stack rm` | `kubectl delete -f` |
| Secret erstellen | `docker secret create` | `kubectl create secret` |
| Secrets auflisten | `docker secret ls` | `kubectl get secrets` |
| Config erstellen | `docker config create` | `kubectl create configmap` |
| Netzwerk erstellen | `docker network create --driver overlay` | (deklarativ verwaltet) |
| Status prüfen | `docker service ps` | `kubectl get pods` |
| Task/Pod inspizieren | `docker inspect <TASK>` | `kubectl describe pod` |

> [!TIP] Umstieg auf Kubernetes
> Wer Docker Swarm beherrscht, findet sich in Kubernetes schnell zurecht. Die Konzepte sind ähnlich, nur die Terminologie und der Detailgrad unterscheiden sich. Siehe [[kubectl Befehle und Cheatsheet]] für die vollständige kubectl-Referenz.

---

## Wichtige --format Vorlagen

Die Go-Template-Syntax ermöglicht flexible Ausgabeformatierung.

```bash
# Nodes — tabellarisch mit Engine-Version
docker node ls --format "table {{.ID}}\t{{.Hostname}}\t{{.Status}}\t{{.Availability}}\t{{.ManagerStatus}}\t{{.EngineVersion}}"

# Services — kompakt
docker service ls --format "{{.Name}}: {{.Replicas}} ({{.Image}})"

# Tasks — Fehlersuche
docker service ps <SERVICE> --format "table {{.ID}}\t{{.Name}}\t{{.Node}}\t{{.DesiredState}}\t{{.CurrentState}}\t{{.Error}}"

# Netzwerke — Overlay-Filter
docker network ls --filter driver=overlay --format "table {{.ID}}\t{{.Name}}\t{{.Driver}}\t{{.Scope}}"

# Secrets — mit Erstelldatum
docker secret ls --format "table {{.ID}}\t{{.Name}}\t{{.CreatedAt}}"

# Configs — mit Erstelldatum
docker config ls --format "table {{.ID}}\t{{.Name}}\t{{.CreatedAt}}"

# JSON-Ausgabe für Weiterverarbeitung
docker service inspect <SERVICE> --format '{{json .Spec}}' | jq '.'
```

> [!NOTE] Go-Templates
> Verfügbare Felder lassen sich mit `docker <command> inspect` im JSON-Output ermitteln. Die Feldnamen in `{{.Feld}}` entsprechen den JSON-Keys.

---

## Häufige Fehler und Lösungen

> [!BUG] "rpc error: code = Unavailable"
> **Ursache:** Manager-Node ist nicht erreichbar oder Quorum ist verloren.
> **Lösung:** Prüfe ob die Mehrheit der Manager-Nodes läuft. Notfalls: `docker swarm init --force-new-cluster`

> [!BUG] "No such node" beim Entfernen
> **Ursache:** Der Node hat den Swarm noch nicht verlassen.
> **Lösung:** Zuerst `docker swarm leave` auf dem Node ausführen, dann `docker node rm` auf dem Manager.

> [!BUG] "service is not converging"
> **Ursache:** Ressourcenmangel, fehlende Images oder Constraint-Konflikte.
> **Lösung:** Fehlermeldungen der Tasks mit `docker service ps <SERVICE> --no-trunc` prüfen und Placement-Constraints mit `docker service inspect --pretty <SERVICE>` verifizieren.

> [!BUG] "network not found" nach Stack-Redeployment
> **Ursache:** Overlay-Netzwerk wurde nicht vollständig entfernt.
> **Lösung:** Netzwerk manuell mit `docker network rm <NETZWERK>` entfernen, dann Stack erneut deployen.

---

## Verwandte Notizen

- [[Docker Swarm Index]] — Übersicht aller Swarm-Themen
- [[Docker CLI Cheatsheet]] — Docker Standalone CLI
- [[kubectl Befehle und Cheatsheet]] — Kubernetes CLI