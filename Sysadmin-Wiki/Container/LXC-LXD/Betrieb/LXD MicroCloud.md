---
tags: [lxd, microcloud, microceph, microovn, cluster, ha, canonical]
aliases: ["MicroCloud", "LXD MicroCloud Setup"]
---

# LXD MicroCloud

> [!ABSTRACT]
> MicroCloud ist Canonicals empfohlener Weg für neue LXD-Cluster-Deployments. Ein einziger interaktiver Wizard (`microcloud init`) richtet LXD, MicroCeph (Ceph-Storage) und MicroOVN (OVN-Netzwerk) gleichzeitig ein — ohne manuelle Konfiguration jedes einzelnen Dienstes. Mindestens 3 Nodes werden benötigt.

---

## Was ist MicroCloud

MicroCloud bündelt drei Canonical-Snaps in einem gemeinsamen Setup-Prozess:

| Komponente | Aufgabe | Snap |
|------------|---------|------|
| **LXD** | Container und VM Management | `lxd` |
| **MicroCeph** | Vereinfachter Ceph-Cluster | `microceph` |
| **MicroOVN** | OVN-Netzwerkvirtualisierung | `microovn` |

Nodes entdecken sich gegenseitig automatisch via **mDNS** — keine manuelle IP-Eingabe nötig.

### MicroCloud vs. manuelles LXD-Cluster

| Kriterium | MicroCloud | Manuelles LXD Cluster |
|-----------|------------|----------------------|
| **Setup-Zeit** | ~20 Minuten | 2–4 Stunden |
| **Ceph-Konfiguration** | Automatisch | Manuell (ceph-deploy/cephadm) |
| **OVN-Konfiguration** | Automatisch | Manuell (ovn-central, ovn-host) |
| **Fehlerrisiko** | Gering | Hoch |
| **Flexibilität** | Mittel | Sehr hoch |
| **Anpassbarkeit** | Eingeschränkt | Vollständig |
| **Geeignet für** | Standard-HA-Cluster | Spezialanforderungen |

### Wann MicroCloud sinnvoll ist

- Neues Homelab oder Produktions-Cluster von Grund auf
- Kein dediziertes Storage-Team (MicroCeph ist deutlich einfacher als pures Ceph)
- Alle Nodes identische oder ähnliche Hardware
- Ubuntu 22.04 oder 24.04 auf allen Nodes

### Einschränkungen

> [!WARNING]
> MicroCloud ist **kein Kubernetes-Ersatz**. Es fehlt auto-scheduling, rolling updates und Service Mesh. Für Microservices-Orchestrierung → Kubernetes.

- MicroCloud eignet sich nicht für heterogene Hardware (unterschiedliche Disk-Layouts pro Node erschweren Ceph)
- MicroCeph ist nicht für alle Ceph-Features ausgelegt (kein RADOS Gateway, eingeschränkte Tuning-Optionen)
- Nachträgliche OVN-Änderungen erfordern manuelle `microovn`-Befehle

---

## Voraussetzungen

### Hardware-Anforderungen

| Anforderung | Minimum (Dev) | Empfohlen (Produktion) |
|-------------|---------------|----------------------|
| **Nodes** | 1 (nur LXD, kein Ceph) | 3 (Quorum) |
| **CPU pro Node** | 2 vCPU | 4–8 vCPU |
| **RAM pro Node** | 8 GB | 16–32 GB |
| **OS-Disk** | 40 GB SSD | 100 GB NVMe |
| **Ceph-Disks** | 1× dediziert | 2–3× NVMe (dediziert, NICHT die OS-Disk) |
| **Netzwerk-Interfaces** | 2 (Management + Uplink) | 2–3 (Management, Ceph, Uplink) |

> [!IMPORTANT]
> Die **Ceph-Disks müssen vollständig leer sein** (kein Partitionstable, kein Filesystem). MicroCeph verweigert die Nutzung von Disks mit vorhandenen Daten.

### Netzwerk-Voraussetzungen

```
Node 1 (192.168.1.10)    Node 2 (192.168.1.11)    Node 3 (192.168.1.12)
  eth0 = Management          eth0 = Management          eth0 = Management
  eth1 = Uplink/OVN          eth1 = Uplink/OVN          eth1 = Uplink/OVN
```

- **eth0** (Management): Statische IPs, alle Nodes erreichbar, mDNS-Multicast erlaubt
- **eth1** (Uplink): Für OVN-External-Network (physisches Netzwerk-Uplink)
- Port **8443** zwischen allen Nodes offen (LXD API)
- **mDNS/Avahi** muss im Netz funktionieren (UDP 5353, Multicast)

### Software-Voraussetzungen

```bash
# Ubuntu 22.04 oder 24.04 (HWE-Kernel empfohlen)
lsb_release -a

# Kernel-Version prüfen (empfohlen: 6.5+)
uname -r

# Sicherstellen, dass keine anderen LXD-Installationen vorhanden sind
# (kein apt-basiertes LXD mehr seit Ubuntu 24.04)
dpkg -l | grep lxd   # sollte leer sein
```

---

## Installation

> [!NOTE]
> Alle 4 Snaps auf **allen** Nodes installieren, **bevor** `microcloud init` ausgeführt wird.

```bash
# Auf JEDEM Node ausführen:
sudo snap install microcloud lxd microceph microovn

# Versionen prüfen (müssen kompatibel sein)
snap list microcloud lxd microceph microovn
```

### Snap-Channels

```bash
# Stabile Versionen (empfohlen für Produktion)
sudo snap install microcloud --channel=latest/stable
sudo snap install lxd --channel=5.21/stable
sudo snap install microceph --channel=squid/stable
sudo snap install microovn --channel=24.03/stable

# Aktuelle Versionsübersicht prüfen
snap info microcloud
```

> [!TIP]
> Alle Snaps im **gleichen Channel** halten, um Kompatibilitätsprobleme zu vermeiden. MicroCloud prüft beim Init die Versionskompatibilität automatisch.

---

## Interaktiver Setup: microcloud init

Den Wizard **nur auf Node 1** ausführen — die anderen Nodes werden automatisch erkannt.

```bash
# Auf node1 ausführen:
sudo microcloud init
```

### Phase 1 — Nodes entdecken

```
Scanning for eligible servers ...
Found the following eligible servers:
 - node2 (192.168.1.11)
 - node3 (192.168.1.12)

Select servers: [alle mit Space auswählen, Enter bestätigen]
```

MicroCloud scannt via mDNS. Alle Nodes müssen im selben L2-Segment erreichbar sein.

### Phase 2 — Management-Interface wählen

```
Select the interface that MicroCloud will use for internal traffic:
 - eth0 (192.168.1.10)
 - eth1 (10.0.0.1)

> eth0
```

Wähle das Interface, über das die Nodes sich gegenseitig erreichen (Management-Netz).

### Phase 3 — MicroCeph: Ceph-Disks auswählen

```
Configure distributed storage? (yes)

Select the disks to use for distributed storage on node1:
 - /dev/sdb (500 GB, nicht gemountet)
 - /dev/sdc (500 GB, nicht gemountet)
```

**Nur leere, nicht gemountete Disks werden angezeigt.** Mindestens 1 Disk pro Node für OSD.

```
# Das gleiche für node2 und node3:
Select the disks on node2: /dev/sdb
Select the disks on node3: /dev/sdb
```

> [!WARNING]
> Alle Daten auf den ausgewählten Disks werden **unwiderruflich gelöscht**. Sicherstellen, dass die richtigen Disks gewählt werden.

### Phase 4 — MicroOVN: Uplink konfigurieren

```
Configure distributed networking? (yes)

Select the interface to use for the uplink network on node1:
 - eth1
 
Specify the IPv4 gateway: 10.0.0.1/24
Specify the IPv4 range for LXD instances: 10.0.0.100-10.0.0.200
Specify the IPv6 gateway (optional): [Enter für Skip]
```

- **Gateway**: Router-IP des physischen Netzwerks (Uplink)
- **IP-Range**: Freier IP-Block im Uplink-Netz für Container-IPs (OVN verteilt diese direkt)

### Phase 5 — Zusammenfassung und Bestätigung

```
MicroCloud will setup:
 - LXD cluster with 3 nodes
 - MicroCeph with 3 OSDs (1 per node)
 - MicroOVN with 3 nodes
 
Proceed? (yes)
```

### Was nach microcloud init eingerichtet ist

Nach erfolgreichem Setup:

- ✅ LXD-Cluster mit 3 Nodes (dqlite/Raft aktiv)
- ✅ MicroCeph-Cluster (3 OSDs, replication factor 3)
- ✅ Ceph-Pool in LXD registriert als `remote` Storage-Pool
- ✅ MicroOVN konfiguriert mit Geneve-Tunneln zwischen Nodes
- ✅ LXD-Netzwerk `UPLINK` und `default` OVN-Netzwerk erstellt
- ✅ Default-Profil nutzt Ceph-Storage und OVN-Netzwerk
- ✅ Container bekommen automatisch IPs aus dem konfigurierten IP-Range

```bash
# Setup verifizieren:
lxc cluster list        # Alle 3 Nodes ONLINE?
microceph status        # HEALTH_OK?
microovn cluster list   # Alle 3 Nodes?
lxc storage list        # remote (ceph) vorhanden?
lxc network list        # default (ovn) vorhanden?

# Ersten Test-Container starten
lxc launch ubuntu:24.04 test-container
lxc list   # IP aus dem konfigurierten Range?
```

---

## Preseed für automatisiertes Setup

Für reproduzierbare, skriptbare Deployments:

```yaml
# microcloud-preseed.yaml — Beispiel für 3 Nodes

lookup_subnet: 192.168.1.0/24
lookup_interface: eth0

initiator: node1

systems:
  - name: node1
    ovn_uplink_interface: eth1
    storage:
      ceph:
        - path: /dev/sdb

  - name: node2
    ovn_uplink_interface: eth1
    storage:
      ceph:
        - path: /dev/sdb

  - name: node3
    ovn_uplink_interface: eth1
    storage:
      ceph:
        - path: /dev/sdb

ovn:
  ipv4_gateway: 10.0.0.1/24
  ipv4_range: 10.0.0.100-10.0.0.200
  dns_servers: 8.8.8.8,8.8.4.4
```

```bash
# Preseed-Setup starten (auf node1):
sudo microcloud init --preseed < microcloud-preseed.yaml
```

> [!TIP]
> Preseed eignet sich für CI/CD-Pipelines oder Reprovisioning. Alle Nodes müssen vor dem Preseed-Init erreichbar und mit Snaps ausgestattet sein.

---

## Cluster-Status prüfen

### Überblick aller Komponenten

```bash
# MicroCloud gesamt
microcloud cluster list

# LXD-Cluster
lxc cluster list
# Erwartete Ausgabe:
# +-------+---------------------------+-----------------+--------+
# | NAME  |            URL            |      ROLES      | STATUS |
# +-------+---------------------------+-----------------+--------+
# | node1 | https://192.168.1.10:8443 | database-leader | ONLINE |
# | node2 | https://192.168.1.11:8443 | database        | ONLINE |
# | node3 | https://192.168.1.12:8443 | database        | ONLINE |
# +-------+---------------------------+-----------------+--------+

# MicroCeph-Status
microceph cluster list
microceph status
# Erwartete Ausgabe:
# MicroCeph deployment summary:
#   node1: OSD 0 (active)
#   node2: OSD 1 (active)
#   node3: OSD 2 (active)

# MicroOVN
microovn cluster list
```

### Ceph-Health prüfen

```bash
# Ceph-Status via microceph
microceph.ceph status

# Erwartete Ausgabe bei gesundem Cluster:
# cluster:
#   id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
#   health: HEALTH_OK
#
# services:
#   mon: 3 daemons, quorum node1,node2,node3
#   osd: 3 osds: 3 up, 3 in

# Ceph-Pool-Details
microceph.ceph osd pool ls detail
```

### Netzwerk verifizieren

```bash
# OVN-Netzwerke in LXD
lxc network list
# Erwartet: UPLINK (physical), default (ovn)

# OVN-Status
microovn.ovn-nbctl show
microovn.ovn-sbctl show

# Geneve-Tunnel-Status
microovn.ovn-sbctl list Chassis
```

---

## Nodes hinzufügen (Cluster erweitern)

```bash
# Auf dem bestehenden Cluster (node1):
sudo microcloud add
```

Der Wizard erkennt neue Nodes per mDNS:

```
Scanning for eligible servers ...
Found the following new eligible servers:
 - node4 (192.168.1.13)

Select servers: [node4 auswählen]

Select the disks to use on node4: /dev/sdb
```

### Was beim Node-Hinzufügen passiert

1. **LXD**: Neuer Node joined den dqlite-Cluster (erhält `database-standby` Rolle)
2. **MicroCeph**: Neues OSD wird hinzugefügt, Ceph startet automatisch Re-Balancing
3. **MicroOVN**: Geneve-Tunnel zum neuen Node wird aufgebaut

> [!NOTE]
> **Ceph Re-Balancing** kann je nach Datenmenge mehrere Minuten dauern. Während dieser Zeit ist `HEALTH_WARN` normal. Warten bis `HEALTH_OK` zurück ist:
> ```bash
> watch microceph.ceph status
> ```

### Node entfernen

```bash
# Node evakuieren (Container migrieren)
lxc cluster evacuate node4

# Node aus MicroCloud entfernen
microcloud remove node4

# Alternativ: LXD-Node direkt entfernen (wenn MicroCloud-Entfernung fehlschlägt)
lxc cluster remove node4 --force
```

---

## Update-Strategie

### Cohort-Updates (empfohlen)

Ein Cohort stellt sicher, dass alle Nodes gleichzeitig denselben Snap-Kanal erhalten:

```bash
# Cohort-Key generieren und auf allen Nodes verwenden
COHORT=$(snap get system store.cohort-key 2>/dev/null || echo "+")

# Auf JEDEM Node gleichzeitig ausführen (z.B. via Ansible):
sudo snap refresh microovn --cohort="+"
sudo snap refresh microceph --cohort="+"
sudo snap refresh lxd --cohort="+"
sudo snap refresh microcloud --cohort="+"
```

### Empfohlene Update-Reihenfolge

```
1. microovn   → OVN-Netzwerk zuerst (Netzwerk bleibt während Update aktiv)
2. microceph  → Ceph danach (OSDs werden nacheinander gepatcht, kein Datenverlust)
3. lxd        → LXD nach Storage-Update (Container kurz pausiert während Daemon-Neustart)
4. microcloud → Management-Tool zuletzt
```

> [!WARNING]
> Niemals einzelne Nodes auf eine neuere Version updaten als den Rest des Clusters. MicroCeph und MicroOVN haben strikte Versionskompatibilitätsanforderungen zwischen Nodes.

### Automatische Updates deaktivieren (Produktion)

```bash
# Automatische Snap-Updates deaktivieren
sudo snap set system refresh.hold="forever"

# Oder auf ein Wartungsfenster einschränken
sudo snap set system refresh.schedule="sat,02:00-04:00"
```

---

## Troubleshooting

### mDNS funktioniert nicht — Nodes werden nicht gefunden

```bash
# Symptom:
# "No eligible servers found" beim microcloud init

# Prüfen: Avahi (mDNS) läuft?
systemctl status avahi-daemon

# Avahi starten falls nicht aktiv
sudo systemctl enable --now avahi-daemon

# mDNS-Traffic erlauben (Firewall)
sudo ufw allow 5353/udp

# Manuell testen: Kann node1 node2 per Avahi finden?
avahi-browse -a | grep microcloud

# Falls mDNS dauerhaft blockiert: Manuelles Preseed mit festen IPs nutzen
```

### MicroCeph — OSD nicht gefunden

```bash
# Symptom:
# "No disks available" obwohl Disks vorhanden

# Disk-Status prüfen
lsblk
sudo fdisk -l /dev/sdb

# Vorhandene Partitionstabelle oder Filesystem löschen
sudo wipefs -a /dev/sdb
sudo dd if=/dev/zero of=/dev/sdb bs=4M count=10 status=progress

# Danach microcloud init erneut versuchen
```

### MicroOVN — Uplink-Interface nicht verfügbar

```bash
# Symptom:
# eth1 wird im Wizard nicht angezeigt

# Prüfen: Interface existiert?
ip link show

# Interface darf KEINEN aktiven DHCP-Lease oder IP haben
sudo ip addr flush dev eth1

# Network-Manager entfernen falls vorhanden (Konflikte)
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
```

### Node nach Neustart nicht im Cluster

```bash
# Symptom:
# lxc cluster list zeigt Node als OFFLINE

# Snap-Services neu starten
sudo systemctl restart snap.lxd.daemon
sudo systemctl restart snap.microceph.daemon
sudo systemctl restart snap.microovn.daemon

# Dienste-Status prüfen
snap services lxd microceph microovn microcloud

# Logs der Daemons
sudo journalctl -u snap.lxd.daemon -n 50
sudo journalctl -u snap.microceph.daemon -n 50
sudo journalctl -u snap.microovn.daemon -n 50
```

### microcloud init schlägt fehl (teilweiser Setup)

```bash
# Reset: Alle Snaps auf Ausgangszustand zurücksetzen (DATENVERLUST!)
sudo microceph reset
sudo microovn reset
sudo lxc cluster rm $(hostname)  # falls bereits gejoint

# Alternativ: Snaps komplett neu installieren
sudo snap remove --purge microcloud lxd microceph microovn
sudo snap install microcloud lxd microceph microovn
```

> [!DANGER]
> `microceph reset` löscht **alle** Ceph-Daten auf diesem Node. Nur nutzen, wenn der Setup fehlgeschlagen ist und keine Produktionsdaten vorhanden sind.

### Diagnose-Übersicht

```bash
# Alle Dienste auf einen Blick
snap services

# Cluster-Zustand
lxc cluster list
microceph cluster list
microovn cluster list

# Ceph-Health
microceph.ceph status
microceph.ceph health detail

# OVN-Connectivity
microovn.ovn-sbctl list Chassis

# LXD-Daemon-Log
sudo journalctl -u snap.lxd.daemon --since "1 hour ago"
```

---

## Verwandte Notizen

- [[LXD Cluster Betrieb]] — Manuelles LXD Cluster Setup (ohne MicroCloud)
- [[LXD Netzwerk]] — OVN Grundlagen und Bridge-Networking
- [[Storage Pools]] — Ceph Storage in LXD konfigurieren
- [[LXD Installation und Setup]] — Einzelinstallation ohne Cluster
- [[LXC-LXD Index]] — Übersicht der gesamten LXC/LXD-Dokumentation
