---
tags:
  - docker-swarm
  - upgrades
  - backups
  - disaster-recovery
  - betrieb
---

# Swarm Upgrades und Backups

> [!ABSTRACT] TL;DR
> Docker Swarm Cluster erfordern regelmäßige Wartung — Docker-Engine-Upgrades, Raft-Log-Backups und Disaster-Recovery-Strategien. Diese Dokumentation deckt den gesamten Lifecycle ab: Rolling Upgrades der Docker-Engine, Backup und Restore des Swarm-State, sowie Notfallwiederherstellung bei Cluster-Verlust.

---

## Swarm-State verstehen

Bevor ein Backup oder Upgrade durchgeführt wird, ist es wichtig zu verstehen, **was** gesichert werden muss.

### Was speichert der Swarm-State?

Der gesamte Cluster-Zustand wird im **Raft-Log** auf den Manager-Nodes gespeichert:

| Daten | Speicherort | Beschreibung |
|-------|-------------|--------------|
| Service-Definitionen | Raft-Log | Alle `docker service create`-Konfigurationen |
| Netzwerk-Konfiguration | Raft-Log | Overlay-Netzwerke, Ingress-Routing |
| Secrets | Raft-Log (verschlüsselt) | Verschlüsselt at rest seit Docker 1.13 |
| Configs | Raft-Log | Konfigurationsdateien für Services |
| Node-Informationen | Raft-Log | Node-Rollen, Labels, Availability |
| Task-Zustand | Raft-Log | Aktuelle Task-Verteilung |
| Unlock-Key | Raft-Log (verschlüsselt) | Autolock-Schlüssel (falls aktiviert) |

### Speicherort auf der Festplatte

```bash
# Swarm-Daten liegen standardmäßig in:
/var/lib/docker/swarm/

# Struktur:
/var/lib/docker/swarm/
├── certificates/    # TLS-Zertifikate (CA, Node-Cert, Key)
├── raft/           # Raft-Konsens-Log und Snapshots
│   ├── snap-v3-encrypted/   # Verschlüsselte Snapshots
│   └── wal-v3-encrypted/    # Write-Ahead-Log
├── state.json      # Lokaler Node-State
└── worker/         # Worker-spezifische Daten (auf Worker-Nodes)
```

> [!DANGER] Raft-Verzeichnis niemals manuell ändern
> Das `/var/lib/docker/swarm/raft/`-Verzeichnis darf **niemals** manuell bearbeitet werden. Änderungen korrumpieren den Raft-Konsens und können den gesamten Cluster zerstören.

---

## Backup-Strategien

### Methode 1: Docker Swarm Snapshot (empfohlen)

Die sicherste Methode — Docker wird gestoppt und das gesamte Swarm-Verzeichnis gesichert.

```bash
#!/bin/bash
# swarm-backup.sh — Swarm-State sichern
set -euo pipefail

BACKUP_DIR="/opt/swarm-backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/swarm-backup-${TIMESTAMP}.tar.gz"
MANAGER_HOSTNAME=$(hostname)

echo "=== Swarm Backup auf ${MANAGER_HOSTNAME} ==="

# 1. Prüfen ob dieser Node ein Manager ist
if ! docker node inspect self --format '{{ .Spec.Role }}' | grep -q manager; then
    echo "FEHLER: Backup muss auf einem Manager-Node ausgeführt werden!"
    exit 1
fi

# 2. Docker stoppen (verhindert Raft-Änderungen während des Backups)
echo "Docker Engine stoppen..."
systemctl stop docker

# 3. Swarm-Verzeichnis sichern
echo "Backup erstellen: ${BACKUP_FILE}"
mkdir -p "${BACKUP_DIR}"
tar -czf "${BACKUP_FILE}" -C /var/lib/docker swarm

# 4. Docker wieder starten
echo "Docker Engine starten..."
systemctl start docker

# 5. Warten bis der Node wieder am Cluster teilnimmt
echo "Warte auf Cluster-Reconnect..."
sleep 10
docker node ls

# 6. Alte Backups aufräumen (älter als 30 Tage)
find "${BACKUP_DIR}" -name "swarm-backup-*.tar.gz" -mtime +30 -delete

echo "=== Backup abgeschlossen: ${BACKUP_FILE} ==="
echo "Größe: $(du -h ${BACKUP_FILE} | cut -f1)"
```

> [!IMPORTANT] Docker-Stopp ist notwendig
> Das Stoppen von Docker ist essentiell für ein konsistentes Backup. Bei einem laufenden Docker können Raft-Writes das Backup inkonsistent machen. In einem HA-Cluster mit 3+ Managern übernehmen die anderen Manager während des kurzen Stopps.

### Methode 2: Live-Backup (ohne Docker-Stopp)

Für Situationen, in denen Docker nicht gestoppt werden kann — weniger sicher, aber ausreichend für regelmäßige Snapshots.

```bash
#!/bin/bash
# swarm-live-backup.sh — Live-Backup ohne Docker-Stopp
set -euo pipefail

BACKUP_DIR="/opt/swarm-backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Raft-Snapshot erzwingen (intern)
docker node inspect self > /dev/null

# Swarm-Verzeichnis kopieren (rsync für Konsistenz)
rsync -a --delete /var/lib/docker/swarm/ \
    "${BACKUP_DIR}/swarm-live-${TIMESTAMP}/"

echo "Live-Backup erstellt: ${BACKUP_DIR}/swarm-live-${TIMESTAMP}/"
```

> [!WARNING] Live-Backups sind nicht garantiert konsistent
> Ein Live-Backup kann einen inkonsistenten Zustand enthalten, wenn während des Kopierens Raft-Writes stattfinden. Verwende diese Methode nur als Ergänzung, nicht als alleinige Backup-Strategie.

### Methode 3: Secrets und Configs separat sichern

Zusätzlich zum Raft-Backup sollten Secrets und Configs auch einzeln gesichert werden:

```bash
#!/bin/bash
# export-configs.sh — Alle Configs und Stack-Dateien exportieren
set -euo pipefail

EXPORT_DIR="/opt/swarm-exports/$(date +%Y%m%d)"
mkdir -p "${EXPORT_DIR}/configs" "${EXPORT_DIR}/services"

# Alle Configs exportieren
for config in $(docker config ls --format '{{.Name}}'); do
    docker config inspect "${config}" --pretty > \
        "${EXPORT_DIR}/configs/${config}.json"
done

# Alle Service-Definitionen exportieren
for service in $(docker service ls --format '{{.Name}}'); do
    docker service inspect "${service}" --pretty > \
        "${EXPORT_DIR}/services/${service}.json"
done

# Netzwerke exportieren
docker network ls --filter driver=overlay --format '{{.Name}}' > \
    "${EXPORT_DIR}/overlay-networks.txt"

echo "Export abgeschlossen: ${EXPORT_DIR}"
```

> [!NOTE] Secrets können nicht exportiert werden
> Docker Secrets sind by-design nicht auslesbar — sie können nur innerhalb von Containern als Datei unter `/run/secrets/` gelesen werden. Das Raft-Backup enthält die verschlüsselten Secrets. Bewahre die Originalwerte separat in einem Passwort-Manager auf.

### Automatisierung mit Cron

```bash
# Tägliches Backup um 3:00 Uhr nachts
echo "0 3 * * * root /opt/scripts/swarm-backup.sh >> /var/log/swarm-backup.log 2>&1" \
    > /etc/cron.d/swarm-backup

# Wöchentliches Export von Configs
echo "0 4 * * 0 root /opt/scripts/export-configs.sh >> /var/log/swarm-export.log 2>&1" \
    > /etc/cron.d/swarm-export
```

---

## Restore und Disaster Recovery

### Szenario 1: Einzelnen Manager wiederherstellen

Wenn ein einzelner Manager ausfällt, aber das Quorum noch besteht:

```bash
# 1. Alten Node aus dem Cluster entfernen (von einem funktionierenden Manager)
docker node demote <ausgefallener-node>
docker node rm --force <ausgefallener-node>

# 2. Neuen Node aufsetzen und Docker installieren
# (siehe Docker Swarm Installation)

# 3. Neuen Node als Manager joinen
docker swarm join --token <manager-token> <manager-ip>:2377

# 4. Zum Manager promoten (falls als Worker gejoined)
docker node promote <neuer-node>
```

### Szenario 2: Quorum verloren — Cluster wiederherstellen

Wenn die Mehrheit der Manager ausgefallen ist und kein Quorum mehr besteht:

```bash
# Auf dem EINZIGEN verbliebenen Manager:
# Force-Reinitialisierung des Raft-Clusters mit diesem Node als einzigem Member
docker swarm init --force-new-cluster

# Cluster-Status prüfen
docker node ls

# Neue Manager-Nodes hinzufügen
docker swarm join-token manager
# → Token auf neuen Nodes verwenden
```

> [!DANGER] `--force-new-cluster` nur im Notfall
> Dieser Befehl erstellt ein neues Single-Manager-Cluster aus dem aktuellen Raft-State. **Alle anderen Manager werden ungültig** und müssen den Cluster neu joinen. Nur verwenden, wenn kein Quorum mehr hergestellt werden kann.

### Szenario 3: Vollständiger Cluster-Verlust — Restore aus Backup

Wenn der gesamte Cluster verloren ist:

```bash
# 1. Docker auf dem neuen Manager-Node installieren
apt-get install -y docker-ce

# 2. Docker stoppen
systemctl stop docker

# 3. Backup wiederherstellen
tar -xzf /opt/swarm-backups/swarm-backup-20260301_030000.tar.gz \
    -C /var/lib/docker/

# 4. Docker starten
systemctl start docker

# 5. Neues Cluster aus dem wiederhergestellten State erzwingen
docker swarm init --force-new-cluster

# 6. Cluster-Zustand prüfen
docker node ls
docker service ls
docker secret ls
docker config ls

# 7. Weitere Manager und Worker hinzufügen
docker swarm join-token manager
docker swarm join-token worker
```

### Szenario 4: Autolock-Cluster nach Restart entsperren

Bei aktiviertem Autolock muss der Cluster nach jedem Docker-Neustart entsperrt werden:

```bash
# Unlock-Key anzeigen (VOR dem Neustart, auf laufendem Manager)
docker swarm unlock-key

# Nach dem Neustart: Cluster entsperren
docker swarm unlock
# → Unlock-Key eingeben

# Unlock-Key rotieren
docker swarm unlock-key --rotate
```

> [!IMPORTANT] Unlock-Key sicher aufbewahren
> Ohne den Unlock-Key kann ein gestoppter Manager nicht mehr am Cluster teilnehmen. Speichere den Key an einem sicheren Ort außerhalb des Clusters (z.B. Passwort-Manager, verschlüsselter USB-Stick).

---

## Docker-Engine Upgrades

### Vorbereitung

```bash
# 1. Aktuelle Version prüfen
docker version

# 2. Swarm-Status prüfen — alle Nodes müssen "Ready" sein
docker node ls

# 3. Backup erstellen (IMMER vor einem Upgrade!)
/opt/scripts/swarm-backup.sh

# 4. Release Notes lesen
# https://docs.docker.com/engine/release-notes/
```

### Rolling Upgrade — Worker-Nodes

Worker zuerst upgraden, dann Manager:

```bash
# === Auf jedem Worker-Node ===

# 1. Node in Drain-Modus setzen (von einem Manager aus)
docker node update --availability drain <worker-node>

# 2. Warten bis alle Tasks migriert sind
watch docker node ps <worker-node>

# 3. Docker upgraden (auf dem Worker-Node selbst)
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io

# 4. Docker-Version prüfen
docker version

# 5. Node wieder aktivieren (von einem Manager aus)
docker node update --availability active <worker-node>

# 6. Prüfen ob Tasks zurück verteilt werden
docker node ps <worker-node>
```

### Rolling Upgrade — Manager-Nodes

> [!WARNING] Manager einzeln upgraden
> **Niemals** mehrere Manager gleichzeitig upgraden! Das Quorum muss zu jedem Zeitpunkt erhalten bleiben. Bei 3 Managern darf nur **einer** gleichzeitig offline sein.

```bash
# === Manager-Upgrade-Reihenfolge ===
# 1. Nicht-Leader-Manager zuerst
# 2. Leader-Manager zuletzt

# Leader identifizieren
docker node inspect self --format '{{ .ManagerStatus.Leader }}'

# Auf jedem Nicht-Leader-Manager:
# 1. Drain setzen
docker node update --availability drain <manager-node>

# 2. Docker upgraden
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io

# 3. Cluster-Gesundheit prüfen
docker node ls

# 4. Node wieder aktivieren
docker node update --availability active <manager-node>

# 5. Warten bis Raft-Konsens wiederhergestellt ist
sleep 30
docker node ls
```

### Upgrade-Reihenfolge (3-Manager-Cluster)

```mermaid
graph LR
    A[Worker 1] --> B[Worker 2]
    B --> C[Worker N]
    C --> D[Manager Follower 1]
    D --> E[Manager Follower 2]
    E --> F[Manager Leader]

    style A fill:#4CAF50,color:#fff
    style B fill:#4CAF50,color:#fff
    style C fill:#4CAF50,color:#fff
    style D fill:#FF9800,color:#fff
    style E fill:#FF9800,color:#fff
    style F fill:#f44336,color:#fff
```

**Grün** = Zuerst (Worker) → **Orange** = Dann (Follower-Manager) → **Rot** = Zuletzt (Leader)

### Major-Version-Upgrade

Bei großen Versionssprüngen (z.B. Docker 24 → 27):

```bash
# 1. Kompatibilitätsmatrix prüfen
# Docker Engine Versionen im Cluster müssen maximal
# eine Major-Version auseinanderliegen

# 2. Testumgebung aufsetzen und Stack testen
docker stack deploy -c stack.yml test-stack

# 3. Schrittweise upgraden — NICHT von 24 direkt auf 27
# 24.x → 25.x → 26.x → 27.x

# 4. Nach jedem Schritt alle Services prüfen
docker service ls
docker service ps <service> --filter "desired-state=running"
```

---

## Node-Wartung

### Geplante Wartung (Reboot, Kernel-Update)

```bash
# 1. Node drainieren
docker node update --availability drain <node>

# 2. Warten bis alle Tasks migriert sind
while docker node ps <node> --filter "desired-state=running" -q | grep -q .; do
    echo "Warte auf Task-Migration..."
    sleep 5
done

# 3. Wartungsarbeiten durchführen
sudo reboot
# oder: apt-get upgrade -y && reboot

# 4. Nach dem Reboot: Node wieder aktivieren
docker node update --availability active <node>
```

### Node aus dem Cluster entfernen

```bash
# 1. Node drainieren
docker node update --availability drain <node>

# 2. Auf dem Node selbst: Swarm verlassen
docker swarm leave          # Worker
docker swarm leave --force  # Manager (erfordert --force)

# 3. Auf einem Manager: Node entfernen
docker node rm <node>
```

---

## Backup-Checkliste

| Was | Wie oft | Methode | Aufbewahrung |
|-----|---------|---------|--------------|
| Swarm-State (Raft) | Täglich | Docker-Stopp + tar | 30 Tage |
| Stack-Compose-Dateien | Bei Änderung | Git-Repository | Unbegrenzt |
| Secrets (Originalwerte) | Bei Erstellung | Passwort-Manager | Unbegrenzt |
| Configs | Wöchentlich | `docker config inspect` | 90 Tage |
| Service-Definitionen | Wöchentlich | `docker service inspect` | 90 Tage |
| Application-Volumes | Täglich | Volume-Backup-Job | Nach Bedarf |
| TLS-Zertifikate | Bei Rotation | Teil des Raft-Backups | Mit Raft |
| Unlock-Key | Bei Rotation | Passwort-Manager | Unbegrenzt |

### Volume-Backup als Swarm-Job

```bash
# Einmaliger Backup-Job für ein Volume
docker service create \
    --name backup-job \
    --mode replicated \
    --replicas 1 \
    --restart-condition none \
    --mount type=volume,source=db-data,target=/data,readonly \
    --mount type=bind,source=/opt/backups,target=/backup \
    alpine \
    tar czf /backup/db-data-$(date +%Y%m%d).tar.gz -C /data .
```

---

## Verwandte Notizen

- [[Docker Swarm Architektur]] — Raft-Konsens und Manager-Kommunikation
- [[Docker Swarm HA Installation]] — Hochverfügbare Cluster aufsetzen
- [[Swarm Security]] — Autolock, TLS-Rotation, Encryption
- [[Swarm Monitoring und Maintenance]] — Cluster-Überwachung
- [[Swarm Best Practices]] — Produktions-Checkliste
- [[Swarm Troubleshooting]] — Fehlersuche bei Backup/Restore-Problemen
