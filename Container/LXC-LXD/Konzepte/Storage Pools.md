---
tags: [lxd, storage, zfs, btrfs, lvm, ceph, pools]
aliases: ["LXD Storage", "LXD Storage Pools"]
---

# Storage Pools

> [!ABSTRACT]
> LXD unterstützt mehrere Storage-Backends: ZFS, btrfs, LVM, dir und Ceph. Storage Pools verwalten den Speicher für Container-Dateisysteme, VM-Disks, Images und Custom-Volumes. Die richtige Wahl des Backends ist entscheidend für Performance und Features.

---

## Konzept: Storage Pools und Volumes

```
Storage Pool (z.B. "default", Backend: ZFS)
├── Container-Volume: mycontainer (ZFS-Dataset)
├── Container-Volume: webapp (ZFS-Dataset)
├── Image-Volume: ubuntu-24.04 (gecacht)
└── Custom-Volume: database-data (manuell erstellt)
```

- **Storage Pool:** Backend-Konfiguration (welches Laufwerk, welches Dateisystem)
- **Volume:** Einzelner Speicherbereich innerhalb des Pools
- Ein Container hat immer mindestens ein Volume (sein Root-Dateisystem)

---

## Backend-Vergleich

| Feature | dir | btrfs | lvm | zfs | ceph |
|---------|-----|-------|-----|-----|------|
| **Copy-on-Write** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Sofortige Snapshots** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Schnelle Klone** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Thin Provisioning** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Deduplizierung** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Optimierter Transfer** | ❌ | ✅ | ❌ | ✅ | ✅ |
| **Block-Support (VMs)** | ❌ | ❌ | ✅ | ❌ | ✅ |
| **Shared Storage (Cluster)** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Einfachheit** | ✅✅✅ | ✅✅ | ✅ | ✅ | ⚠️ |
| **Produktionsreife** | ⚠️ | ✅ | ✅ | ✅✅ | ✅✅ |

### Empfehlung nach Use Case

```
Einzelner Host, Produktion:     ZFS       ← beste Features
Einfaches Setup / Testing:      dir       ← kein Overhead
Container + VMs gemischt:       LVM       ← Block-Support
Multi-Node Cluster:             Ceph      ← Shared Storage
Leichtgewicht ohne Root:        btrfs     ← einfache CoW
```

---

## ZFS Backend

ZFS ist die empfohlene Wahl für Produktionsumgebungen:

### Vorteile
- Copy-on-Write: Snapshots in Millisekunden
- Checksumming: automatische Integritätsprüfung
- Kompression: transparent (lz4, zstd)
- `zfs send/receive`: optimierter Container-Transfer

### ZFS Pool erstellen

```bash
# Auf neuem Block-Device (empfohlen für Produktion)
lxc storage create production zfs source=/dev/sdb

# Loop-File für Testing (nicht für Produktion)
lxc storage create test zfs size=20GiB

# Mit Kompression
lxc storage create production zfs source=/dev/sdb
lxc storage set production zfs.pool_name mypool
```

### ZFS-Konfiguration

```bash
# Kompression aktivieren (lz4 Standard)
lxc storage set default volume.zfs.use_refquota true

# Snapshots automatisch löschen wenn kein Speicher
lxc storage set default volume.zfs.remove_snapshots true

# Blockgröße setzen
lxc storage set default zfs.blocksize 16KiB
```

### ZFS direkt verwalten

```bash
# ZFS-Pool-Status
sudo zpool status

# Datasets anzeigen
sudo zfs list

# Speicherverbrauch pro Container
sudo zfs get used,available lxdpool/containers/mycontainer
```

> [!WARNING]
> ZFS benötigt genug RAM für den ARC-Cache. Faustregel: 1 GB RAM pro 1 TB Speicher (mindestens 2 GB für ZFS).

---

## btrfs Backend

Gut für einfachere Setups mit CoW-Vorteilen:

```bash
# Pool auf Block-Device
lxc storage create mybtrfs btrfs source=/dev/sdc

# Pool als Loop-File
lxc storage create mybtrfs btrfs size=30GiB

# Auf existierendem btrfs-Dateisystem
lxc storage create mybtrfs btrfs source=/mnt/btrfs-vol
```

---

## LVM Backend

Ideal wenn Block-Devices für VMs oder traditionelle Setups benötigt werden:

```bash
# LVM Pool erstellen
lxc storage create mylvm lvm source=/dev/sdd

# Thin Pool (platzsparend)
lxc storage create mylvm lvm source=/dev/sdd
lxc storage set mylvm lvm.thinpool_name LXDThinPool

# Volume-Größe für neue Container
lxc storage set mylvm volume.size 10GiB
```

---

## dir Backend

Einfachstes Backend — kein besonderes Dateisystem nötig:

```bash
# Im Standard-Dateisystem-Verzeichnis
lxc storage create mydir dir

# Auf spezifischem Pfad
lxc storage create mydir dir source=/mnt/external-drive

# Quota mit Projektquotas (nur ext4/xfs mit Quota-Support)
lxc storage set mydir volume.size 10GiB
```

> [!NOTE]
> Kein Copy-on-Write → Snapshots/Klone sind langsam (vollständige Kopie). Nur für Testing oder einfache Setups geeignet.

---

## Ceph Backend (Cluster)

Für verteilten Storage in LXD-Clustern:

```bash
# Ceph Pool anlegen (Ceph-Cluster muss existieren)
lxc storage create ceph-pool ceph \
    ceph.cluster_name=ceph \
    ceph.osd.pool_name=lxd \
    ceph.user.name=admin

# RBD Block-Devices für VMs
lxc storage create ceph-rbd ceph \
    ceph.osd.pool_name=lxd-rbd
```

---

## Storage Pools verwalten

```bash
# Alle Pools anzeigen
lxc storage list

# Pool-Details (inkl. Belegung)
lxc storage info default

# Pool-Konfiguration anzeigen
lxc storage show default

# Pool-Konfiguration bearbeiten
lxc storage edit default

# Pool löschen (muss leer sein)
lxc storage delete old-pool
```

---

## Volumes verwalten

### Custom Volumes (unabhängig von Containern)

```bash
# Custom Volume erstellen
lxc storage volume create default mydata

# Volume einem Container zuweisen
lxc config device add mycontainer mydata disk \
    pool=default \
    source=mydata \
    path=/data

# Volume auflisten
lxc storage volume list default

# Volume löschen
lxc storage volume delete default mydata
```

### Container-Volume direkt verwalten

```bash
# Container-Volume-Größe ändern
lxc config device set mycontainer root size 50GiB

# Volume-Info
lxc storage volume show default container/mycontainer
```

---

## Snapshots

```bash
# Container-Snapshot erstellen
lxc snapshot mycontainer snap0

# Snapshot mit Beschreibung
lxc snapshot mycontainer snap-before-update

# Alle Snapshots eines Containers anzeigen
lxc info mycontainer

# Snapshot wiederherstellen
lxc restore mycontainer snap0

# Snapshot löschen
lxc delete mycontainer/snap0

# Snapshot zu neuem Container machen
lxc copy mycontainer/snap0 restored-container
```

### Automatische Snapshots

```bash
# Täglich um 02:00 Uhr
lxc config set mycontainer snapshots.schedule "0 2 * * *"

# Max. 7 Snapshots behalten
lxc config set mycontainer snapshots.expiry 7d

# Naming-Pattern
lxc config set mycontainer snapshots.pattern "snap%Y%m%d-%H%M"
```

---

## Container zwischen Hosts migrieren

```bash
# Container auf anderen Host migrieren (mit ZFS/btrfs optimiert)
lxc move mycontainer myremote:

# Mit optionalem Ziel-Storage-Pool
lxc move mycontainer myremote: --target-storage production

# Kopieren (Original bleibt bestehen)
lxc copy mycontainer myremote:mycontainer-copy
```

> [!TIP]
> Mit ZFS- oder btrfs-Backend nutzt LXD `zfs send`/`btrfs send` für effizienten Transfer — nur geänderte Blöcke werden übertragen.

---

## Verwandte Notizen

- [[LXD Installation und Setup]] — Pool beim Setup konfigurieren
- [[Profile und Konfiguration]] — Disk-Devices in Profilen
- [[LXD Cluster Betrieb]] — Ceph für Cluster-Storage
- [[LXD Troubleshooting]] — Storage-Probleme lösen
