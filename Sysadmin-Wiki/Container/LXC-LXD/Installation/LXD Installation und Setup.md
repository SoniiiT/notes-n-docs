---
tags: [lxd, installation, ubuntu, snap, setup]
aliases: ["LXD Setup", "LXD installieren"]
---

# LXD Installation und Setup

> [!ABSTRACT]
> LXD wird auf Ubuntu über Snap installiert. Nach der Installation führt `lxd init` durch einen interaktiven Setup-Wizard für Storage, Netzwerk und optionales Clustering. Diese Anleitung deckt alles ab — von der Einzelinstallation bis zum produktionsbereiten Setup.

---

## Voraussetzungen

| Anforderung | Minimum | Empfohlen |
|-------------|---------|-----------|
| **OS** | Ubuntu 20.04+ | Ubuntu 24.04 LTS |
| **CPU** | 2 Kerne | 4+ Kerne |
| **RAM** | 2 GB | 8+ GB |
| **Disk** | 20 GB | 100+ GB (separates Volume für Storage Pool) |
| **Kernel** | 5.4+ | 6.x (cgroups v2) |

> [!NOTE]
> LXD läuft auch auf Debian, Fedora und anderen Distros via Snap. Auf Nicht-Ubuntu-Systemen kann Incus die bessere Wahl sein (nativer Paketmanager).

---

## Installation

### Ubuntu (Snap — empfohlen)

```bash
# LXD via Snap installieren (latest/stable)
sudo snap install lxd

# Alternativ: LTS-Channel (5.21)
sudo snap install lxd --channel=5.21/stable

# Alternativ: Neuester Feature-Release
sudo snap install lxd --channel=latest/stable

# Aktuellen Channel prüfen
snap info lxd | grep tracking
```

### Benutzer zur lxd-Gruppe hinzufügen

```bash
# Damit lxc ohne sudo genutzt werden kann
sudo usermod -aG lxd $USER

# Neue Gruppe aktivieren (oder ausloggen/einloggen)
newgrp lxd

# Testen
lxc list
```

> [!WARNING]
> Mitglieder der `lxd`-Gruppe haben effektiv Root-Zugriff auf den Host. Nur vertrauenswürdigen Benutzern hinzufügen!

### Incus (Community-Fork, Nicht-Ubuntu)

```bash
# Debian/Ubuntu
sudo apt install incus

# Arch Linux
sudo pacman -S incus

# Nach Installation
sudo incus admin init
```

---

## lxd init — Initialisierungswizard

`lxd init` konfiguriert LXD interaktiv. Einmal ausführen nach der Installation:

```bash
sudo lxd init
```

**Typische Fragen und empfohlene Antworten:**

```
Would you like to use LXD clustering? (yes/no) [default=no]: no
Do you want to configure a new storage pool? (yes/no) [default=yes]: yes
Name of the new storage pool [default=default]: default
Name of the storage backend to use (btrfs, dir, lvm, zfs) [default=zfs]: zfs
Create a new ZFS pool? (yes/no) [default=yes]: yes
Would you like to use an existing empty block device? (yes/no) [default=no]: no
Size in GiB of the new loop device (1GiB minimum) [default=9GiB]: 50GiB
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What should the new bridge be called? [default=lxdbr0]: lxdbr0
What IPv4 address should be used? [default=auto]: auto
What IPv6 address should be used? [default=auto]: none
Would you like the LXD server to be available over the network? (yes/no) [default=no]: no
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: yes
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: yes
```

### Preseed-Datei (Automatisierung)

Für Scripts und Automatisierung:

```yaml
# lxd-preseed.yaml
config: {}
networks:
  - config:
      ipv4.address: 10.10.10.1/24
      ipv4.nat: "true"
      ipv6.address: none
    description: ""
    name: lxdbr0
    type: bridge
storage_pools:
  - config:
      size: 50GiB
    description: ""
    name: default
    driver: zfs
profiles:
  - config: {}
    description: ""
    devices:
      eth0:
        name: eth0
        network: lxdbr0
        type: nic
      root:
        path: /
        pool: default
        type: disk
    name: default
projects: []
cluster: null
```

```bash
# Preseed verwenden
cat lxd-preseed.yaml | sudo lxd init --preseed
```

---

## Ersten Container starten

```bash
# Ubuntu 24.04 Container starten
lxc launch ubuntu:24.04 mein-erster-container

# Container-Status prüfen
lxc list

# In den Container einloggen
lxc exec mein-erster-container -- bash

# Container beenden
lxc stop mein-erster-container

# Container löschen
lxc delete mein-erster-container
```

**Erwartete Ausgabe von `lxc list`:**

```
+----------------------+---------+------------------+------+-----------+-----------+
|         NAME         |  STATE  |       IPV4       | IPV6 |   TYPE    | SNAPSHOTS |
+----------------------+---------+------------------+------+-----------+-----------+
| mein-erster-container | RUNNING | 10.10.10.2 (eth0) |      | CONTAINER |         0 |
+----------------------+---------+------------------+------+-----------+-----------+
```

---

## Storage Pool konfigurieren

### ZFS auf separatem Block-Device (Produktion)

```bash
# Block-Device für ZFS vorbereiten (z.B. /dev/sdb)
sudo lxd init
# Bei "existing empty block device" → yes → /dev/sdb eingeben
```

```bash
# Nachträglich: neuen Pool auf Block-Device erstellen
lxc storage create production zfs source=/dev/sdb

# Pool als Standard für Profil setzen
lxc profile device set default root pool=production
```

### Mehrere Storage Pools

```bash
# Schneller Pool für aktive Container (NVMe)
lxc storage create fast zfs source=/dev/nvme0n1

# Langsamer Pool für Backups (HDD)
lxc storage create backup dir source=/mnt/backup-pool

# Pool bei Container-Start angeben
lxc launch ubuntu:24.04 mycontainer -s fast
```

---

## Netzwerk konfigurieren

```bash
# Standard-Bridge anzeigen
lxc network show lxdbr0

# Bridge anpassen (anderes Subnetz)
lxc network set lxdbr0 ipv4.address 192.168.100.1/24

# IPv6 aktivieren
lxc network set lxdbr0 ipv6.address auto

# DNS-Server setzen
lxc network set lxdbr0 dns.nameservers 1.1.1.1,8.8.8.8
```

---

## Remote-Server konfigurieren

```bash
# Verfügbare Remote-Server anzeigen
lxc remote list

# Typische Defaults:
# images: → images.linuxcontainers.org (Community-Images für LXC/Incus)
# ubuntu: → cloud-images.ubuntu.com (Ubuntu-Images für LXD)

# Eigenen Remote-Server hinzufügen
lxc remote add myserver https://192.168.1.100:8443

# Remote-Container starten
lxc launch myserver:ubuntu/24.04 mycontainer
```

> [!IMPORTANT]
> Seit dem LXD/Canonical-Split: LXD verwendet `ubuntu:` als Image-Quelle. Incus verwendet `images:` (linuxcontainers.org). Die verfügbaren Distributionen unterscheiden sich.

---

## LXD aktualisieren

```bash
# Snap-Update (automatisch oder manuell)
sudo snap refresh lxd

# Channel wechseln
sudo snap refresh lxd --channel=latest/stable

# Version prüfen
lxd --version
lxc --version
```

---

## Deinstallation

```bash
# Alle Container stoppen und löschen
lxc stop --all
lxc delete --all

# LXD entfernen
sudo snap remove lxd --purge

# Snap-Daten bereinigen (falls vorhanden)
sudo rm -rf /var/snap/lxd
```

---

## Nächste Schritte

Nach der Installation empfiehlt sich:

1. **Profil konfigurieren:** [[Profile und Konfiguration]] — Ressourcenlimits setzen
2. **Storage verstehen:** [[Storage Pools]] — ZFS-Features nutzen
3. **Netzwerk einrichten:** [[LXD Netzwerk]] — Bridges und macvlan
4. **CLI lernen:** [[LXD CLI Cheatsheet]] — alle wichtigen Befehle

---

## Verwandte Notizen

- [[LXD CLI Cheatsheet]] — Alle lxc-Befehle
- [[Profile und Konfiguration]] — Container-Konfiguration
- [[Storage Pools]] — Storage-Backend-Auswahl
- [[LXD Cluster Betrieb]] — Multi-Node-Setup
