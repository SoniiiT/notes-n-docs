---
tags: [lxd, cluster, ha, hochverfügbarkeit, multi-node, dqlite, ovn, live-migration]
aliases: ["LXD Cluster", "LXD HA"]
---

# LXD Cluster Betrieb

> [!ABSTRACT]
> LXD-Cluster verbinden mehrere Hosts zu einem einzigen verwalteten System. Der Cluster nutzt dqlite (verteiltes SQLite mit Raft-Konsensus) ohne externe Datenbank, OVN für Cross-Node-Netzwerke und CRIU für Live-Migration von Containern. Ab 3 Nodes ist Quorum und Verwaltungs-HA möglich.

---

## Cluster-Architektur

```
┌──────────────────────────────────────────────────────────────┐
│                        LXD Cluster                           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │    node1     │  │    node2     │  │    node3     │      │
│  │  (Leader)    │  │  (Follower)  │  │  (Follower)  │      │
│  │              │  │              │  │              │      │
│  │  lxd daemon  │  │  lxd daemon  │  │  lxd daemon  │      │
│  │  dqlite/Raft │◄─►  dqlite/Raft │◄─►  dqlite/Raft │      │
│  │  :8443       │  │  :8443       │  │  :8443       │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  Shared Resources:                                           │
│  - Ceph Storage (empfohlen für Container-Migration)         │
│  - OVN Netzwerk (empfohlen für Cross-Node-Connectivity)     │
└──────────────────────────────────────────────────────────────┘
```

**Cluster-Voraussetzungen:**

| Anforderung | Details |
|-------------|---------|
| **Nodes** | Mindestens 3 (Quorum), empfohlen ungerade Anzahl |
| **LXD-Version** | Identisch auf allen Nodes |
| **Netzwerk** | Port 8443 zwischen allen Nodes erreichbar |
| **Zeit** | NTP synchronisiert (max. 1s Abweichung) |
| **Hostname** | Eindeutig pro Node, DNS-auflösbar |
| **Hardware (empfohlen)** | 4 vCPU, 8 GB RAM, 40 GB OS, 300 GB Storage |

---

## dqlite und Raft — der Konsensus-Mechanismus

LXD verwendet **dqlite** (distributed SQLite) als Cluster-Datenbank. Kein externes etcd, kein Postgres — dqlite läuft direkt im LXD-Daemon.

### Wie Raft funktioniert

```
┌─────────────────────────────────────────────────────────┐
│                    Raft Konsensus                        │
│                                                         │
│   node1 (Leader)   node2 (Follower)   node3 (Follower)  │
│                                                         │
│   1. Client schreibt → Leader                          │
│   2. Leader repliziert → alle Follower                 │
│   3. Majority (2/3) bestätigt → Commit                 │
│   4. Leader antwortet Client: "erfolgreich"            │
│                                                         │
│   Bei Leader-Ausfall:                                   │
│   1. Follower bemerken: kein Heartbeat                 │
│   2. Election-Timeout → Candidate                      │
│   3. Abstimmung → neuer Leader gewählt                 │
│   4. Cluster weiter funktionsfähig (2/3 Quorum)        │
└─────────────────────────────────────────────────────────┘
```

**Wichtige Eigenschaften:**
- **Schreib-Operationen** immer über den Leader
- **Lese-Operationen** können von jedem Node beantwortet werden
- **Heartbeat** alle 500ms — fällt ein Node aus, wird er nach ~2s erkannt
- **Leader Election** dauert 1–3 Sekunden

### dqlite-Datenbankpfad

```bash
# Datenbank-Dateien auf dem Host
ls /var/snap/lxd/common/lxd/database/

# Lokale Datenbank (Metadaten dieses Nodes)
ls /var/snap/lxd/common/lxd/database/local/

# Globale Cluster-Datenbank (Raft-Log + State)
ls /var/snap/lxd/common/lxd/database/global/

# Cluster-Datenbank-Status (Debug)
lxc query /internal/cluster/state
```

---

## Cluster-Setup: Manuell

### Ersten Node initialisieren

```bash
# Auf node1: Cluster-Setup mit lxd init
sudo lxd init

# Interaktive Fragen:
# Would you like to use LXD clustering? → yes
# What name should be used to identify this node? → node1
# What IP address or DNS name should be used to reach this node? → 192.168.1.10
# Are you joining an existing cluster? → no (erster Node)
# Create a new local database → yes

# Oder mit Preseed:
cat > cluster-node1.yaml << 'EOF'
config:
  core.https_address: 192.168.1.10:8443
cluster:
  server_name: node1
  enabled: true
networks:
  - config:
      bridge.mode: standard
      ipv4.address: 10.10.0.1/24
      ipv4.nat: "true"
    name: lxdbr0
    type: bridge
storage_pools:
  - config: {}
    driver: dir
    name: local
profiles:
  - config: {}
    devices:
      eth0:
        name: eth0
        network: lxdbr0
        type: nic
      root:
        path: /
        pool: local
        type: disk
    name: default
EOF

cat cluster-node1.yaml | sudo lxd init --preseed
```

### Weitere Nodes hinzufügen

#### Token-basiertes Joining (LXD 6.x — empfohlen)

```bash
# Auf node1: Join-Token generieren
lxc cluster add node2
# → Gibt Token aus, z.B.: eyJzZXJ2ZXJfbmFtZSI6I...

# Auf node2: LXD installieren, dann joinen
sudo snap install lxd
sudo lxd init

# Interaktive Fragen:
# Would you like to use LXD clustering? → yes
# What name should be used to identify this node? → node2
# Are you joining an existing cluster? → yes
# Please provide join token → <Token von node1 einfügen>
```

#### Passwort-basiertes Joining (LXD 5.x)

```bash
# Auf node1: Trust-Passwort setzen
lxc config set core.trust_password "geheim123"

# Auf node2: Join
sudo lxd init
# → yes für clustering, IP von node1, Passwort eingeben
```

### Automatisiertes Cluster-Setup (Preseed für alle Nodes)

```yaml
# node2-preseed.yaml — Join-Preseed
cluster:
  server_name: node2
  enabled: true
  cluster_address: 192.168.1.10:8443
  cluster_token: "eyJzZXJ2ZXJfbmFtZSI6I..."  # Token von lxc cluster add
  member_config:
    - entity: storage-pool
      name: local
      key: source
      value: ""
    - entity: network
      name: lxdbr0
      key: bridge.external_interfaces
      value: ""
```

```bash
cat node2-preseed.yaml | sudo lxd init --preseed
```

---

## Cluster-Status verwalten

```bash
# Cluster-Übersicht
lxc cluster list

# Beispiel-Ausgabe:
# +-------+---------------------------+------------------+--------+
# | NAME  |             URL           |      ROLES       | STATUS |
# +-------+---------------------------+------------------+--------+
# | node1 | https://192.168.1.10:8443 | database-leader  | ONLINE |
# | node2 | https://192.168.1.11:8443 | database         | ONLINE |
# | node3 | https://192.168.1.12:8443 | database         | ONLINE |
# +-------+---------------------------+------------------+--------+

# Node-Details (inkl. Ressourcen)
lxc cluster show node1

# Cluster-Konfiguration
lxc cluster config show

# Cluster-Konfiguration anpassen
lxc cluster config set scheduler.instance density
```

### Rollen im Cluster

| Rolle | Beschreibung | Max. Anzahl |
|-------|-------------|-------------|
| `database-leader` | Aktueller Raft-Leader, koordiniert Schreiboperationen | 1 |
| `database` | Vollwertiges Cluster-Member, nimmt an Wahlen teil | empfohlen 3 oder 5 |
| `database-standby` | Kann bei Ausfall zum Voter werden | optional |
| *(keine Rolle)* | Worker-Only Node, kein DB-Voting, nur Container-Host | beliebig |

```bash
# Rollen anzeigen
lxc cluster list -c nR

# Node-Rolle manuell anpassen
lxc cluster edit node4
# → roles: [] für reinen Worker-Node
```

> [!TIP]
> Große Cluster: 3–5 Nodes als Datenbank-Voter, Rest als Worker. So skaliert die Worker-Kapazität beliebig ohne Raft-Overhead zu erhöhen.

---

## Container im Cluster verwalten

```bash
# Container auf bestimmtem Node starten
lxc launch ubuntu:24.04 webapp --target node2

# Container-Verteilung anzeigen (N=Name, L=Location, s=Status)
lxc list -c nLs4

# Alle Container auf einem Node
lxc list --target node2

# Container nach Ressourcen platzieren (manuell)
# node3 hat mehr RAM → Datenbank-Container dort starten
lxc launch ubuntu:24.04 postgres --target node3 \
    -c limits.memory=4GiB

# Scheduler: gleichmäßige Verteilung (Standard)
lxc cluster config set scheduler.instance all-nodes
```

---

## Live Migration

Live Migration verschiebt laufende Container ohne Downtime zwischen Nodes.

### Voraussetzungen für Live Migration

```
Ohne Shared Storage:   Container muss gestoppt werden (Cold Migration)
Mit Ceph als Storage:  Container kann live migriert werden
Mit CRIU:              Container-Prozesse werden eingefroren, übertragen, fortgesetzt
```

### Cold Migration (ohne Shared Storage)

```bash
# Container stoppen, migrieren, starten
lxc stop webapp
lxc move webapp --target node3
lxc start webapp

# Oder in einem Schritt
lxc move webapp --target node3
# LXD stoppt, überträgt, startet automatisch
```

### Live Migration mit CRIU (Container-in-Container)

**CRIU** (Checkpoint/Restore in Userspace) friert Prozesse ein, serialisiert den Prozesszustand und stellt ihn auf dem Ziel-Node wieder her.

```bash
# Live Migration (Container bleibt laufend)
lxc move webapp --target node3
# LXD nutzt CRIU automatisch wenn verfügbar

# Status während Migration prüfen
watch -n 1 lxc list

# Stateless Migration (State wird NICHT übertragen, schneller)
lxc move webapp --target node3 --stateless
```

**Was CRIU überträgt:**
- Prozess-State (Register, Stack, Heap)
- Offene Dateideskriptoren
- Netzwerk-Verbindungen (TCP-Sockets)
- Memory-Mapping

**Einschränkungen:**
- Nicht alle Anwendungen sind CRIU-kompatibel
- Prozesse mit privilegierten Sockets (z.B. Docker-in-LXD) scheitern
- Empfehlung: mit `--stateless` testen, danach mit voller Migration

### Live Migration mit Ceph

Mit Ceph-Storage läuft die Migration am zuverlässigsten:

```bash
# Container auf Ceph-Storage migrieren
lxc move webapp --target node3
# Ceph-Block wird sofort auf node3 verfügbar (kein Datentransfer!)
# Nur Prozess-State wird übertragen

# Downtime: typisch 1–3 Sekunden
```

```
node2 (Quelle)              node3 (Ziel)
CRIU: Prozess einfrieren
State serialisieren  ──────► State deserialisieren
                             Prozess fortsetzen
Ceph RBD: node3 übernimmt ► I/O auf Ceph weiterhin verfügbar
```

---

## OVN Networking im Cluster

OVN (Open Virtual Network) ermöglicht Containern auf verschiedenen Nodes, in einem gemeinsamen virtuellen Netz zu kommunizieren — ohne manuelle Tunnel-Konfiguration.

### OVN-Architektur im LXD-Cluster

```
┌─────────────────────────────────────────────────────────────────┐
│                        LXD Cluster                              │
│                                                                 │
│  node1                    node2                    node3        │
│  ┌───────────────┐        ┌───────────────┐        ┌────────┐  │
│  │  Container A  │        │  Container B  │        │  CT C  │  │
│  │  10.20.0.10   │        │  10.20.0.20   │        │10.20.  │  │
│  └───────┬───────┘        └───────┬───────┘        └───┬────┘  │
│          │ OVN vNIC              │ OVN vNIC            │        │
│  ┌───────▼───────────────────────▼─────────────────────▼────┐  │
│  │              OVN Logical Switch (ovn-net)                │  │
│  │           10.20.0.0/24 — alle Container im selben Netz   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                     Geneve Tunnel (UDP 6081)                    │
│                    zwischen physischen Nodes                    │
└─────────────────────────────────────────────────────────────────┘
```

### OVN installieren und konfigurieren

```bash
# Auf ALLEN Nodes:
sudo apt install ovn-host

# Nur auf den ersten 3 Nodes (OVN Central):
sudo apt install ovn-central

# OVN-Cluster starten (auf node1)
sudo ovn-ctl start_nb_ovsdb
sudo ovn-ctl start_sb_ovsdb
sudo ovn-ctl start_northd

# OVN-Verbindung konfigurieren
sudo ovs-vsctl set open_vswitch . \
    external_ids:ovn-remote=tcp:192.168.1.10:6642 \
    external_ids:ovn-encap-type=geneve \
    external_ids:ovn-encap-ip=192.168.1.10

# Auf node2 und node3:
sudo ovs-vsctl set open_vswitch . \
    external_ids:ovn-remote=tcp:192.168.1.10:6642 \
    external_ids:ovn-encap-type=geneve \
    external_ids:ovn-encap-ip=192.168.1.11  # jeweilige Node-IP
```

### OVN-Netzwerk in LXD erstellen

```bash
# Uplink-Netzwerk (physisches Interface)
lxc network create UPLINK \
    --type=physical \
    parent=eth1 \
    ipv4.gateway=192.168.2.1/24 \
    ipv4.ovn.ranges=192.168.2.100-192.168.2.200

# OVN-Netzwerk für Container
lxc network create ovn-net \
    --type=ovn \
    network=UPLINK \
    ipv4.address=10.20.0.1/24 \
    ipv4.nat=true

# Container in OVN-Netz starten (auf verschiedenen Nodes!)
lxc launch ubuntu:24.04 ct-a --target node1
lxc config device add ct-a eth0 nic network=ovn-net

lxc launch ubuntu:24.04 ct-b --target node2
lxc config device add ct-b eth0 nic network=ovn-net

# ct-a und ct-b können sich direkt erreichen, obwohl auf verschiedenen Nodes
lxc exec ct-a -- ping ct-b.lxd
```

---

## Shared Storage mit Ceph

### Ceph manuell einrichten (für LXD-Cluster)

```bash
# Ceph-Pool auf dem Cluster erstellen (muss pro Node initialisiert werden)
lxc storage create ceph-pool ceph \
    ceph.cluster_name=ceph \
    ceph.osd.pool_name=lxd-containers \
    ceph.user.name=admin \
    --target node1

lxc storage create ceph-pool ceph \
    ceph.cluster_name=ceph \
    ceph.osd.pool_name=lxd-containers \
    ceph.user.name=admin \
    --target node2

lxc storage create ceph-pool ceph \
    ceph.cluster_name=ceph \
    ceph.osd.pool_name=lxd-containers \
    ceph.user.name=admin \
    --target node3

# Zweiten Schritt: Pool cluster-weit finalisieren (ohne --target)
lxc storage create ceph-pool ceph \
    ceph.cluster_name=ceph \
    ceph.osd.pool_name=lxd-containers

# Ceph als Standard-Pool im Profil setzen
lxc profile device set default root pool=ceph-pool
```

### Ceph-Schlüssel an LXD-Nodes verteilen

```bash
# Ceph-Schlüssel exportieren
sudo ceph auth get-or-create client.lxd \
    mon 'profile rbd' \
    osd 'profile rbd pool=lxd-containers' \
    -o /etc/ceph/ceph.client.lxd.keyring

# Schlüssel auf alle Nodes kopieren
for node in node1 node2 node3; do
    scp /etc/ceph/ceph.client.lxd.keyring $node:/etc/ceph/
done
```

---

## Node-Wartung

### Node evakuieren (vor Wartung/Shutdown)

```bash
# Alle Container des Nodes auf andere Nodes migrieren
lxc cluster evacuate node2

# Warten bis Evakuierung abgeschlossen
watch lxc cluster list  # node2 sollte "EVACUATED" zeigen

# Wartung durchführen
ssh node2
sudo apt upgrade -y
sudo reboot

# Warten bis node2 wieder ONLINE ist
watch lxc cluster list

# Node wieder in Betrieb nehmen (Container zurückmigieren)
lxc cluster restore node2
```

### Cluster skalieren — Node hinzufügen

```bash
# Neuen Node einrichten (wie bei initialem Join)
lxc cluster add node4
# → Token generiert

# Auf node4: LXD installieren und joinen
sudo snap install lxd
# Token beim lxd init eingeben

# Rolle zuweisen
# Nur Worker (kein Datenbank-Voter):
lxc cluster edit node4
# roles: []  ← leer = reiner Worker
```

### Node entfernen

```bash
# Node sanft entfernen (Container müssen vorher migriert/gestoppt sein)
lxc cluster remove node3

# Node forciert entfernen (bei ausgefallenen Nodes)
lxc cluster remove node3 --force

# Danach: Ceph kennt den Node noch → manuell bereinigen
sudo ceph osd out <osd-ids>
sudo ceph osd purge <osd-ids>
```

---

## Quorum und Ausfallsicherheit

| Nodes (Voter) | Max. Ausfälle | Quorum |
|--------------|--------------|--------|
| 3 | 1 | 2 |
| 5 | 2 | 3 |
| 7 | 3 | 4 |

```
Cluster mit 3 Voter-Nodes:
  node1 ✅  node2 ✅  node3 ✅  → Normalbetrieb
  node1 ✅  node2 ✅  node3 ❌  → 2/3 Quorum → weiter funktionsfähig
  node1 ✅  node2 ❌  node3 ❌  → Kein Quorum → Read-Only, kein neue Container
```

> [!WARNING]
> Ohne Quorum ist der Cluster **read-only**. Neue Container können nicht gestartet, Konfigurationen nicht geändert werden. Laufende Container bleiben unberührt.

---

## Disaster Recovery

### Szenario 1: Ein Node fällt aus (Quorum erhalten)

```bash
# Cluster läuft weiter — einfach warten bis Node zurückkommt
# Node neu starten:
ssh node2
sudo snap restart lxd

# Falls Node dauerhaft weg → entfernen und ersetzen
lxc cluster remove node2 --force

# Neuen Node hinzufügen
lxc cluster add node2-new
```

### Szenario 2: Quorum verloren (2+ Nodes weg)

```bash
# Auf dem verbleibenden Node — Quorum manuell wiederherstellen
sudo lxd cluster recover-from-quorum-loss

# LXD neu starten
sudo snap restart lxd

# Cluster-Status prüfen
lxc cluster list

# Andere Nodes müssen neu gejoint werden!
lxc cluster add node2
lxc cluster add node3
```

> [!DANGER]
> `recover-from-quorum-loss` deklariert den verbleibenden Node als alleinige Wahrheitsquelle. Alle Änderungen, die auf den ausgefallenen Nodes seit dem letzten Raft-Commit gemacht wurden, gehen verloren.

### Szenario 3: Alle Nodes ausgefallen (Complete Cluster Loss)

```bash
# Aus Backup wiederherstellen
# 1. LXD auf einem Node installieren
# 2. dqlite-Backup einspielen
sudo snap stop lxd
sudo cp -r /backup/lxd-db/ /var/snap/lxd/common/lxd/database/
sudo snap start lxd

# 3. Container aus Backup-Exports wiederherstellen
lxc import /backup/mycontainer-20260306.tar.gz
```

### Backup der Cluster-Datenbank

```bash
#!/bin/bash
# Tägliches Cluster-DB-Backup
BACKUP_DIR="/mnt/backup/lxd-cluster"
DATE=$(date +%Y%m%d-%H%M)

sudo snap stop lxd
sudo tar -czf "$BACKUP_DIR/lxd-db-$DATE.tar.gz" \
    /var/snap/lxd/common/lxd/database/
sudo snap start lxd

# Backups älter als 14 Tage löschen
find $BACKUP_DIR -name "lxd-db-*.tar.gz" -mtime +14 -delete
```

---

## TLS und Authentifizierung im Cluster

```bash
# Cluster-Zertifikat anzeigen
lxc config show | grep -A 5 certificate

# Client-Zertifikat für externen Zugriff hinzufügen
lxc config trust add admin.crt --name "Admin Workstation"

# Token-basierter Zugriff (LXD 6.x)
lxc config trust add --name "remote-client"
# → Gibt Einmal-Token aus

# Alle vertrauenswürdigen Clients anzeigen
lxc config trust list

# Zertifikat widerrufen
lxc config trust revoke <fingerprint>

# Cluster-interner TLS (automatisch, keine Konfiguration nötig)
# Nodes authentifizieren sich gegenseitig via Cluster-Zertifikat
lxc query /1.0/cluster | python3 -m json.tool | grep -A 5 server_name
```

---

## Cluster-Monitoring und Health Checks

```bash
# ── Cluster-Status ───────────────────────────────────────

lxc cluster list                          # Alle Nodes und Rollen
lxc cluster show node1                    # Node-Details
lxc cluster info                          # Cluster-weite Infos

# ── Container-Verteilung ─────────────────────────────────

lxc list -c nLs4                          # Name, Location, Status, IP
lxc list --target node2                   # Nur Containers auf node2

# ── Ressourcen pro Node ──────────────────────────────────

for node in node1 node2 node3; do
    echo "=== $node ==="
    lxc info --resources --target $node | grep -E "CPU|Memory|Disk"
done

# ── Events cluster-weit ──────────────────────────────────

lxc monitor --type=lifecycle              # Start/Stop/Migrate Events
lxc monitor --type=logging               # Log-Events

# ── Prometheus ───────────────────────────────────────────

# Metrics von jedem Node separat scrapen
# prometheus.yml:
# - targets: ['node1:9090', 'node2:9090', 'node3:9090']

# ── Automatischer Health Check ───────────────────────────

#!/bin/bash
# cluster-health.sh
EXPECTED_NODES=3
ONLINE=$(lxc cluster list --format=csv | grep -c ONLINE)

if [ "$ONLINE" -lt "$EXPECTED_NODES" ]; then
    echo "WARNUNG: Nur $ONLINE von $EXPECTED_NODES Nodes ONLINE!"
    lxc cluster list
fi
```

---

## Cluster mit MicroCloud (empfohlen)

Für einfacheres Cluster-Setup mit integriertem Ceph und OVN → [[LXD MicroCloud]]

```bash
# Schnellübersicht:
sudo snap install microcloud lxd microceph microovn

# Interaktiver Setup-Wizard (auf Node 1)
sudo microcloud init
# → erkennt andere Nodes per mDNS automatisch
# → konfiguriert Ceph, OVN und LXD in einem Schritt
```

---

## Verwandte Notizen

- [[LXD Installation und Setup]] — Einzelinstallation als Basis
- [[Storage Pools]] — Ceph für Cluster
- [[LXD Netzwerk]] — OVN für Cluster
- [[LXD MicroCloud]] — Automatisiertes Cluster-Setup
- [[LXD Troubleshooting]] — Cluster-Diagnose
