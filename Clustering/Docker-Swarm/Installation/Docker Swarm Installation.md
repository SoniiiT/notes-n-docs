---
tags:
  - docker-swarm
  - installation
  - setup
  - ubuntu
aliases:
  - Swarm Setup
  - Docker Swarm einrichten
---

# Docker Swarm Installation

> [!ABSTRACT] TL;DR
> Schritt-für-Schritt-Anleitung zur Einrichtung eines Docker Swarm Clusters — vom Einzelhost bis zum einfachen Manager-Worker-Setup. Umfasst Firewall-Konfiguration, Swarm-Initialisierung, Worker-Anbindung, erste Services und Node-Management.

---

## Voraussetzungen

Bevor ein Swarm Cluster aufgesetzt werden kann, müssen folgende Bedingungen auf **allen Nodes** erfüllt sein:

| Anforderung | Details |
|:------------|:--------|
| Docker Engine | ≥ 20.10 installiert auf jedem Node (siehe [[Docker Installation]]) |
| Betriebssystem | Ubuntu 22.04/24.04, Debian 12 oder RHEL 9 empfohlen |
| Architektur | 64-bit (x86_64 / amd64 oder arm64) |
| RAM | ≥ 2 GB pro Node (Manager: ≥ 4 GB empfohlen) |
| Netzwerk | Alle Nodes müssen sich gegenseitig über IP erreichen können |
| Hostname | Jeder Node braucht einen **eindeutigen Hostnamen** |
| Zeitsynchronisation | NTP / `chrony` aktiv auf allen Nodes |

### Benötigte Ports

Docker Swarm nutzt mehrere Ports für die Cluster-Kommunikation:

| Port | Protokoll | Zweck |
|:-----|:----------|:------|
| `2377` | TCP | Cluster-Management und Raft-Konsens |
| `7946` | TCP + UDP | Node-Discovery und Gossip-Protokoll |
| `4789` | UDP | VXLAN Overlay-Netzwerk (Datenverkehr) |
| IP-Protokoll `50` | ESP | Verschlüsselter Overlay-Traffic (nur bei `--opt encrypted`) |

> [!IMPORTANT] Port 2377 vs. 2376
> Port `2377` ist der Swarm-Management-Port. Nicht verwechseln mit Port `2376`, der für die Docker-Engine-API (TLS) verwendet wird. Im Swarm-Kontext ist **immer** `2377` gemeint.

### Hostnamen setzen

Eindeutige Hostnamen erleichtern die Übersicht im Cluster erheblich:

```bash
# Manager Node
sudo hostnamectl set-hostname manager1

# Worker Nodes
sudo hostnamectl set-hostname worker1
sudo hostnamectl set-hostname worker2
```

Optional die Datei `/etc/hosts` auf jedem Node pflegen, falls kein DNS vorhanden ist:

```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.1.10  manager1
192.168.1.11  worker1
192.168.1.12  worker2
EOF
```

### Zeitsynchronisation sicherstellen

Swarm-Nodes müssen eine synchrone Uhrzeit haben, da der Raft-Konsensalgorithmus zeitabhängig arbeitet:

```bash
# chrony installieren (falls nicht vorhanden)
sudo apt install -y chrony

# Status prüfen
chronyc tracking
timedatectl status
```

> [!WARNING] Zeitdrift
> Eine Zeitdrift von mehr als wenigen Sekunden zwischen Nodes kann dazu führen, dass der Raft-Konsens fehlschlägt und Manager-Nodes sich nicht einigen können. Das Cluster wird dann instabil.

---

## Firewall konfigurieren

Die Firewall muss auf **allen Nodes** die Swarm-Ports freigeben. Nachfolgend Beispiele für die drei gängigsten Firewall-Tools.

### UFW (Ubuntu / Debian)

```bash
# Cluster Management
sudo ufw allow 2377/tcp comment "Docker Swarm Management"

# Node Communication (Gossip)
sudo ufw allow 7946/tcp comment "Docker Swarm Node Discovery TCP"
sudo ufw allow 7946/udp comment "Docker Swarm Node Discovery UDP"

# Overlay Network (VXLAN)
sudo ufw allow 4789/udp comment "Docker Swarm Overlay Network"

# Regeln anwenden und Status prüfen
sudo ufw reload
sudo ufw status verbose
```

### iptables

```bash
# Cluster Management
sudo iptables -A INPUT -p tcp --dport 2377 -j ACCEPT

# Node Communication
sudo iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 7946 -j ACCEPT

# Overlay Network
sudo iptables -A INPUT -p udp --dport 4789 -j ACCEPT

# Regeln persistent machen
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

> [!TIP] Quell-IP einschränken
> In Produktionsumgebungen sollten die Regeln auf die IP-Adressen der Swarm-Nodes beschränkt werden:
> ```bash
> sudo iptables -A INPUT -p tcp --dport 2377 -s 192.168.1.0/24 -j ACCEPT
> ```
> Dadurch wird verhindert, dass beliebige Hosts dem Cluster beitreten können.

### firewalld (RHEL / Rocky Linux / Fedora)

```bash
# Ports dauerhaft freigeben
sudo firewall-cmd --permanent --add-port=2377/tcp
sudo firewall-cmd --permanent --add-port=7946/tcp
sudo firewall-cmd --permanent --add-port=7946/udp
sudo firewall-cmd --permanent --add-port=4789/udp

# Regeln neu laden
sudo firewall-cmd --reload

# Prüfen
sudo firewall-cmd --list-ports
```

> [!NOTE] Docker und firewalld
> Docker manipuliert `iptables`-Regeln direkt. Bei aktivierter `firewalld` kann es zu Konflikten kommen. Falls Container nach einem `firewall-cmd --reload` kein Netzwerk mehr haben:
> ```bash
> sudo systemctl restart docker
> ```

---

## Netzwerk-Konnektivität prüfen

Vor der Swarm-Initialisierung sollte die Erreichbarkeit zwischen allen Nodes getestet werden:

```bash
# Vom Manager aus die Worker testen
ping -c 3 worker1
ping -c 3 worker2

# Spezifisch die Swarm-Ports testen (mit nc/ncat)
nc -zv worker1 2377
nc -zv worker1 7946
```

> [!TIP] Schnelltest mit Bash-Schleife
> ```bash
> for port in 2377 7946; do
>     for host in manager1 worker1 worker2; do
>         nc -zv -w 3 $host $port 2>&1 | grep -E "succeeded|refused|timed out"
>     done
> done
> ```
> `succeeded` = Port offen, `refused` = Host erreichbar aber Port zu, `timed out` = Firewall blockiert.

---

## Swarm initialisieren

### Manager Node einrichten

Die Swarm-Initialisierung erfolgt auf dem **ersten Manager-Node**:

```bash
docker swarm init --advertise-addr 192.168.1.10
```

Erwartete Ausgabe:

```text
Swarm initialized: current node (abc123def456) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0abc...xyz-worker-token 192.168.1.10:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

> [!IMPORTANT] `--advertise-addr` — Wann und warum?
> Der Parameter `--advertise-addr` legt fest, über welche IP-Adresse der Manager für andere Nodes erreichbar ist. Er ist **zwingend erforderlich**, wenn:
> - Der Node **mehrere Netzwerk-Interfaces** hat (z.B. `eth0` + `docker0` + `br0`)
> - Der Node in einem **NAT-Netzwerk** oder hinter einem Load Balancer sitzt
> - Virtualisierte Umgebungen (Proxmox, VMware) zusätzliche virtuelle NICs haben
>
> Bei Nodes mit nur einem Interface kann `--advertise-addr` weggelassen werden — Docker wählt dann automatisch die richtige IP.

### Join-Token anzeigen

Die Token werden bei der Initialisierung einmalig ausgegeben. Sie können jederzeit erneut abgerufen werden:

```bash
# Token für Worker anzeigen
docker swarm join-token worker

# Token für zusätzliche Manager anzeigen
docker swarm join-token manager

# Nur den Token-String ausgeben (ohne Befehl)
docker swarm join-token -q worker
```

> [!WARNING] Token-Sicherheit
> Die Join-Token sind **Geheimnisse** — wer den Token kennt, kann dem Cluster beitreten. Token sollten:
> - **Niemals** in öffentliche Repositories committet werden
> - **Nicht** in Chat-Nachrichten oder Tickets im Klartext geteilt werden
> - Bei Verdacht auf Kompromittierung **sofort rotiert** werden:
> ```bash
> docker swarm join-token --rotate worker
> docker swarm join-token --rotate manager
> ```

---

## Worker Nodes hinzufügen

Auf jedem Worker-Node den Join-Befehl ausführen, der bei der Initialisierung ausgegeben wurde:

```bash
docker swarm join \
    --token SWMTKN-1-0abc...xyz-worker-token \
    192.168.1.10:2377
```

Erwartete Ausgabe:

```text
This node joined a swarm as a worker.
```

> [!NOTE] Mehrere Worker gleichzeitig
> Der gleiche Join-Befehl kann auf beliebig vielen Worker-Nodes parallel ausgeführt werden. Jeder Node meldet sich einzeln beim Manager an.

### Fehler beim Join beheben

Falls der Join fehlschlägt, die häufigsten Ursachen:

| Fehler | Ursache | Lösung |
|:-------|:--------|:-------|
| `connection refused` | Port 2377 nicht erreichbar | Firewall prüfen, Manager läuft? |
| `token is invalid` | Token falsch oder abgelaufen | Neuen Token generieren |
| `timeout` | Netzwerk-Problem | Routing, Firewall, Kabel prüfen |
| `already part of a swarm` | Node ist noch in einem alten Swarm | `docker swarm leave --force` |

```bash
# Node aus altem Swarm entfernen (auf dem Node selbst)
docker swarm leave --force

# Dann erneut joinen
docker swarm join --token <TOKEN> 192.168.1.10:2377
```

---

## Cluster verifizieren

Nach dem Hinzufügen aller Nodes den Cluster-Status auf dem **Manager-Node** prüfen.

### Node-Liste anzeigen

```bash
docker node ls
```

Erwartete Ausgabe:

```text
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
abc123def456 *                manager1   Ready     Active         Leader           28.0.1
ghi789jkl012                  worker1    Ready     Active                          28.0.1
mno345pqr678                  worker2    Ready     Active                          28.0.1
```

| Spalte | Bedeutung |
|:-------|:----------|
| `*` nach der ID | Aktueller Node (auf dem der Befehl ausgeführt wird) |
| `STATUS: Ready` | Node ist online und gesund |
| `AVAILABILITY: Active` | Node akzeptiert neue Tasks |
| `MANAGER STATUS: Leader` | Dieser Manager ist der aktive Raft-Leader |
| `MANAGER STATUS: Reachable` | Manager ist erreichbar, aber nicht Leader |

### Swarm-Status in Docker Info

```bash
docker info --format '{{.Swarm.LocalNodeState}}'
# Ausgabe: active

docker info | grep -A 10 "Swarm:"
```

### Detaillierte Node-Informationen

```bash
# Details zu einem spezifischen Node
docker node inspect --pretty worker1

# Alle Labels eines Nodes anzeigen
docker node inspect --format '{{.Spec.Labels}}' worker1
```

---

## Ersten Service deployen

Ein Swarm-Service ist die zentrale Einheit für die Lastverteilung. Im Gegensatz zu einem einzelnen Container verteilt Swarm die Replicas automatisch auf die verfügbaren Nodes.

### Nginx-Testservice erstellen

```bash
docker service create \
    --name web \
    --replicas 3 \
    --publish published=80,target=80 \
    nginx:alpine
```

| Flag | Bedeutung |
|:-----|:----------|
| `--name web` | Name des Services |
| `--replicas 3` | Drei Instanzen verteilt auf den Cluster |
| `--publish published=80,target=80` | Port 80 auf allen Nodes veröffentlichen |
| `nginx:alpine` | Verwendetes Image |

### Service-Status prüfen

```bash
# Alle Services auflisten
docker service ls
```

```text
ID             NAME   MODE         REPLICAS   IMAGE          PORTS
x1y2z3a4b5     web    replicated   3/3        nginx:alpine   *:80->80/tcp
```

```bash
# Tasks (Container) des Services anzeigen
docker service ps web
```

```text
ID             NAME    IMAGE          NODE       DESIRED STATE   CURRENT STATE
a1b2c3d4e5     web.1   nginx:alpine   manager1   Running         Running 30 seconds ago
f6g7h8i9j0     web.2   nginx:alpine   worker1    Running         Running 28 seconds ago
k1l2m3n4o5     web.3   nginx:alpine   worker2    Running         Running 28 seconds ago
```

### Service testen

Dank des **Routing Mesh** ist der Service über **jeden** Node im Cluster erreichbar — unabhängig davon, auf welchem Node die Container tatsächlich laufen:

```bash
# Von einem beliebigen Node oder externen Client
curl http://192.168.1.10
curl http://192.168.1.11
curl http://192.168.1.12
# Alle drei liefern die Nginx-Startseite
```

> [!TIP] Routing Mesh
> Docker Swarm leitet eingehende Anfragen automatisch an einen der laufenden Container weiter — auch wenn auf dem angefragten Node gar kein Container des Services läuft. Dieses Feature heißt **Ingress Routing Mesh** und ist standardmäßig aktiviert.

### Service skalieren

```bash
# Auf 5 Replicas hochskalieren
docker service scale web=5

# Mehrere Services gleichzeitig skalieren
docker service scale web=5 api=3

# Zurückskalieren
docker service scale web=1
```

### Service entfernen

```bash
docker service rm web
```

> [!WARNING] Kein Bestätigungsprompt
> `docker service rm` löscht den Service **sofort ohne Rückfrage**. Alle zugehörigen Container werden gestoppt und entfernt. Im Zweifel vorher mit `docker service ps <name>` prüfen, welche Tasks betroffen sind.

---

## Node Labels

Labels ermöglichen es, Nodes mit Metadaten zu versehen und Services gezielt auf bestimmten Nodes zu platzieren.

### Labels setzen

```bash
# Standort-Label
docker node update --label-add zone=eu-west manager1
docker node update --label-add zone=eu-west worker1
docker node update --label-add zone=eu-east worker2

# Typ-Label
docker node update --label-add type=compute worker1
docker node update --label-add type=storage worker2

# Umgebungs-Label
docker node update --label-add env=production worker1
docker node update --label-add env=staging worker2
```

### Labels anzeigen

```bash
# Labels eines Nodes anzeigen
docker node inspect --format '{{json .Spec.Labels}}' worker1 | jq .

# Alle Nodes mit Labels auflisten
docker node ls -q | xargs -I {} sh -c \
    'echo "--- $(docker node inspect --format "{{.Description.Hostname}}" {}) ---"; \
     docker node inspect --format "{{json .Spec.Labels}}" {} | jq .'
```

### Labels für Placement nutzen

```bash
# Service nur auf Nodes mit Label "type=compute" deployen
docker service create \
    --name compute-job \
    --constraint 'node.labels.type == compute' \
    --replicas 2 \
    alpine sleep 3600

# Service nur auf Nodes in Zone "eu-west"
docker service create \
    --name eu-app \
    --constraint 'node.labels.zone == eu-west' \
    --replicas 3 \
    nginx:alpine
```

### Labels entfernen

```bash
docker node update --label-rm zone worker2
```

---

## Node Management

### Node-Verfügbarkeit steuern

Jeder Node hat einen Verfügbarkeits-Status, der bestimmt, ob er neue Tasks empfängt:

| Status | Bedeutung |
|:-------|:----------|
| `active` | Node nimmt neue Tasks an (Standard) |
| `pause` | Laufende Tasks bleiben, keine neuen Tasks |
| `drain` | Alle Tasks werden auf andere Nodes verschoben |

```bash
# Node in Wartungsmodus versetzen (Drain)
docker node update --availability drain worker1

# Node wieder aktivieren
docker node update --availability active worker1

# Node pausieren (keine neuen Tasks, bestehende laufen weiter)
docker node update --availability pause worker1
```

> [!TIP] Wartungsarbeiten
> Vor einem Kernel-Update oder Reboot einen Node immer erst auf `drain` setzen. So werden alle Tasks sauber auf andere Nodes verschoben, bevor der Node offline geht:
> ```bash
> docker node update --availability drain worker1
> # Warten bis alle Tasks migriert sind
> docker node ps worker1   # Sollte leer sein oder "Shutdown"
> # Jetzt sicher rebooten
> sudo reboot
> ```

### Node aus dem Cluster entfernen

Einen Node sauber aus dem Cluster lösen — ein dreistufiger Prozess:

```bash
# 1. Tasks vom Node abziehen (auf dem Manager)
docker node update --availability drain worker2

# 2. Node verlässt den Swarm (auf dem Worker selbst)
docker swarm leave

# 3. Node aus der Cluster-Liste entfernen (auf dem Manager)
docker node rm worker2
```

> [!NOTE] Force Leave
> Falls ein Worker-Node nicht mehr erreichbar ist (Hardware-Ausfall), kann er direkt vom Manager entfernt werden:
> ```bash
> docker node rm --force worker2
> ```
> Die Tasks des ausgefallenen Nodes werden automatisch auf anderen Nodes neu gestartet.

### Node-Rollen ändern

```bash
# Worker zum Manager befördern
docker node promote worker1

# Manager zum Worker degradieren
docker node demote manager2
```

> [!WARNING] Ungerade Anzahl an Managern
> Für den Raft-Konsensalgorithmus wird eine **ungerade** Anzahl an Managern empfohlen (1, 3 oder 5). Eine gerade Anzahl bietet keinen Vorteil gegenüber der nächstniedrigeren ungeraden Zahl:
> - **1 Manager:** Kein Ausfall toleriert (nur für Entwicklung)
> - **3 Manager:** 1 Ausfall toleriert (Empfehlung für kleine Cluster)
> - **5 Manager:** 2 Ausfälle toleriert (Empfehlung für Produktion)
> - **7 Manager:** 3 Ausfälle toleriert (selten nötig, höherer Overhead)
>
> Mehr als 7 Manager sind **nicht empfohlen** — der Konsens-Overhead überwiegt den Nutzen.

---

## Swarm-Konfiguration anpassen

### Heartbeat-Intervall und Timeouts

```bash
# Dispatcher-Heartbeat anpassen (Standard: 5s)
docker swarm update --dispatcher-heartbeat 10s

# Task-History-Limit setzen (alte Tasks in docker service ps)
docker swarm update --task-history-limit 5
```

### Automatische Lock-Funktion (Autolock)

Autolock verschlüsselt die Raft-Logs auf der Festplatte. Beim Neustart eines Managers muss ein Unlock-Key eingegeben werden:

```bash
# Autolock beim Erstellen aktivieren
docker swarm init --autolock --advertise-addr 192.168.1.10

# Autolock nachträglich aktivieren
docker swarm update --autolock=true

# Unlock-Key anzeigen
docker swarm unlock-key

# Nach Manager-Neustart entsperren
docker swarm unlock
```

> [!TIP] Autolock für Produktion
> In Produktionsumgebungen mit sensiblen Secrets sollte Autolock **immer** aktiviert sein. Ohne Autolock kann jeder mit Zugriff auf die Festplatte des Managers die TLS-Schlüssel und Swarm-Secrets auslesen.

---

## Troubleshooting Installation

### Häufige Fehler und Lösungen

> [!BUG] `Error response from daemon: This node is not a swarm manager`
> Der Befehl wurde auf einem **Worker-Node** statt auf einem **Manager-Node** ausgeführt. Management-Befehle (`docker node ls`, `docker service ls`, etc.) funktionieren nur auf Managern.

> [!BUG] `Error response from daemon: rpc error: connection refused`
> - Firewall blockiert Port `2377`
> - Manager-Node ist nicht erreichbar
> - Docker-Daemon läuft nicht auf dem Manager
> ```bash
> # Prüfen
> sudo systemctl status docker
> sudo ufw status | grep 2377
> ping <MANAGER-IP>
> ```

> [!BUG] `Error: node is already part of a swarm`
> Der Node ist noch Mitglied eines vorherigen Swarms:
> ```bash
> docker swarm leave --force
> # Dann erneut joinen
> ```

### Netzwerk-Diagnose

```bash
# Overlay-Netzwerke anzeigen
docker network ls --filter driver=overlay

# Swarm-interne DNS-Auflösung testen
docker service create --name dns-test --mode global \
    alpine ping -c 3 tasks.web

# Logs eines Service-Tasks prüfen
docker service logs web --tail 50 --follow
```

### Token-Management

```bash
# Aktuellen Worker-Token anzeigen
docker swarm join-token worker

# Aktuellen Manager-Token anzeigen
docker swarm join-token manager

# Token rotieren (invalidiert alle alten Token)
docker swarm join-token --rotate worker
docker swarm join-token --rotate manager
```

> [!TIP] Regelmäßige Token-Rotation
> Als Best Practice sollten Token in Produktionsumgebungen regelmäßig rotiert werden — insbesondere wenn Teammitglieder das Projekt verlassen oder ein Node kompromittiert wurde.

### Docker-Daemon-Logs prüfen

```bash
# Systemd-Journal (Ubuntu / Debian)
sudo journalctl -u docker.service --since "1 hour ago" --no-pager

# Swarm-spezifische Einträge filtern
sudo journalctl -u docker.service | grep -i "swarm\|raft\|leader"
```

---

## Zusammenfassung — Schnellstart-Referenz

Die wichtigsten Befehle auf einen Blick:

```bash
# === SETUP ===
docker swarm init --advertise-addr <IP>           # Swarm erstellen
docker swarm join --token <TOKEN> <IP>:2377        # Node hinzufügen
docker swarm join-token worker                     # Token anzeigen

# === VERIFIZIEREN ===
docker node ls                                     # Nodes auflisten
docker info --format '{{.Swarm.LocalNodeState}}'   # Swarm-Status

# === SERVICES ===
docker service create --name web --replicas 3 -p 80:80 nginx:alpine
docker service ls                                  # Services auflisten
docker service ps web                              # Tasks anzeigen
docker service scale web=5                         # Skalieren
docker service rm web                              # Entfernen

# === NODE MANAGEMENT ===
docker node update --availability drain <NODE>     # Drain-Modus
docker node promote <NODE>                         # Worker wird Manager
docker node demote <NODE>                          # Manager wird Worker
docker node rm <NODE>                              # Entfernen
```

---

## Verwandte Notizen

- [[Docker Swarm Index]] — Übersicht aller Docker Swarm Dokumentationen
- [[Docker Swarm HA Installation]] — Hochverfügbares Multi-Manager-Setup
- [[Docker Installation]] — Docker Engine auf einzelnen Hosts installieren
- [[Swarm Services und Tasks]] — Services, Tasks und Replicas im Detail
- [[Was ist Docker Swarm]] — Konzepte, Architektur und Vergleich mit Kubernetes