---
tags:
  - virtualization
  - proxmox
  - backup
  - pbs
  - deduplication
  - disaster-recovery
aliases:
  - Proxmox Backup Server
  - PBS
  - PVE Backup
---

# Proxmox Backup Server (PBS)

> [!ABSTRACT] TL;DR
> Der **Proxmox Backup Server (PBS)** ist eine dedizierte Open-Source-Backup-Lösung für Proxmox VE. Er bietet inkrementelle Backups mit **Deduplizierung** und **Kompression**, sodass tägliche Backups kaum mehr Platz als ein Vollbackup benötigen. Backups können verschlüsselt, verifiziert und direkt aus PBS wiederhergestellt werden — einzelne Dateien ohne kompletten VM-Restore.

---

## PBS vs. vzdump (lokale Backups)

| Merkmal | **vzdump (lokal)** | **Proxmox Backup Server** |
|---|---|---|
| Speicherort | Lokales Verzeichnis / NFS | Dedizierter PBS-Server |
| Deduplizierung | ❌ | ✅ Chunk-basiert |
| Verschlüsselung | ❌ | ✅ Client-seitig (AES-256) |
| Inkrementelle Backups | ❌ (immer Vollbackup) | ✅ |
| Single-File-Restore | ❌ (kompletter Restore) | ✅ |
| Backup-Verifizierung | ❌ | ✅ automatisch |
| Prune & GC | Manuell | ✅ automatisch |
| Monitoring | Begrenzt | ✅ Dashboard |

> [!TIP] PBS für Produktion nutzen
> Für produktive Umgebungen PBS gegenüber lokalem vzdump bevorzugen. Der Speicherbedarf ist durch Deduplizierung oft 5–20x geringer als bei konventionellen Vollbackups.

---

## PBS Installation

PBS kann auf einem separaten physischen System, einer VM oder auf dem PVE-Host selbst installiert werden. **Empfohlen: separates System.**

### PBS auf separatem Debian-System installieren

```bash
# Auf dem PBS-Server (Debian 12):

# PBS-Repository hinzufügen
echo "deb [arch=amd64] http://download.proxmox.com/debian/pbs bookworm pbs-no-subscription" \
    > /etc/apt/sources.list.d/pbs-no-subscription.list

# Repository-Key importieren
wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \
    -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

# PBS installieren
apt update && apt install proxmox-backup-server -y

# Web-UI aufrufen:
# https://<PBS-IP>:8007
# Benutzer: root / Linux PAM
```

### PBS auf PVE-Host installieren (alternativ)

```bash
# PBS-Pakete auf bestehendem PVE-Host installieren
apt install proxmox-backup-server

# PBS-Web-UI läuft auf Port 8007 (PVE-UI auf 8006)
```

---

## Datastore einrichten

Ein **Datastore** ist das Backup-Verzeichnis auf dem PBS:

```bash
# Via Web-UI: Administration → Datastores → Add Datastore
# Oder per CLI:

# Verzeichnis erstellen
mkdir -p /mnt/backup-datastore

# Datastore hinzufügen
proxmox-backup-manager datastore create backups /mnt/backup-datastore

# Datastore-Status anzeigen
proxmox-backup-manager datastore list
proxmox-backup-manager datastore show backups

# Disk-Nutzung anzeigen
proxmox-backup-manager status
```

### Empfohlene Datastore-Disk-Konfiguration

| Setup | Beschreibung | Empfehlung |
|---|---|---|
| **Single HDD** | Einfach, kein RAID | Nur Homelab |
| **ZFS Mirror** | 2 HDDs, Selbstheilung | **Empfohlen für KMU** |
| **ZFS RAIDZ2** | 4+ HDDs, 2 Disk-Ausfall | Enterprise |
| **Separate SSD für chunks + HDD** | SSD als Cache | Hoher Durchsatz |

---

## Proxmox VE mit PBS verbinden

### PBS als Storage in PVE registrieren

```bash
# Via Web-UI: Datacenter → Storage → Add → Proxmox Backup Server
# Oder per CLI:
pvesm add pbs pbs-backup \
    --server 192.168.1.100 \
    --datastore backups \
    --username backup@pbs \
    --password MyPBSPassword \
    --fingerprint AA:BB:CC:...     # PBS-Server-Fingerprint (aus PBS Web-UI)

# Fingerprint auf PBS anzeigen
proxmox-backup-manager cert info | grep Fingerprint
```

### PBS-Benutzer für Backups anlegen (auf PBS)

```bash
# Benutzer erstellen
proxmox-backup-manager user create backup@pbs --password MySecurePass

# Berechtigungen setzen
proxmox-backup-manager acl update /datastore/backups backup@pbs DatastoreBackup
```

---

## Backup-Jobs konfigurieren

### Backup-Job in PVE erstellen (Web-UI)

1. **Datacenter → Backup → Add**
2. **Storage**: PBS-Datastore auswählen (`pbs-backup`)
3. **Schedule**: z.B. `daily` oder `0 2 * * *` (täglich um 02:00)
4. **Selection**: Alle VMs / ausgewählte VMs
5. **Mode**: `Snapshot` (kein Downtime), `Stop` (konsistenter), `Suspend` (selten)
6. **Retention**: Aufbewahrungsregeln setzen
7. **Encryption**: Verschlüsselung aktivieren (optional)

### Backup-Job per CLI

```bash
# Backup-Job hinzufügen
pvesh create /cluster/backup --storage pbs-backup \
    --schedule "0 2 * * *" \
    --all 1 \
    --mode snapshot \
    --compress zstd \
    --notes-template "{{guestname}} - {{node}} - {{date}}"

# Backup manuell auslösen
vzdump 100 --storage pbs-backup --mode snapshot --compress zstd

# Alle VMs und Container sichern
vzdump --all --storage pbs-backup --mode snapshot
```

### Backup-Modi

| Modus | Downtime | Konsistenz | Empfehlung |
|---|---|---|---|
| **Snapshot** | ❌ (kein Downtime) | Abhängig vom QEMU Guest Agent | ✅ Standard |
| **Stop** | ✅ VM wird gestoppt | ✅ Vollständig konsistent | Für kritische Daten |
| **Suspend** | Kurz pausiert | ✅ RAM-State inklusive | Selten genutzt |

---

## Aufbewahrungsregeln (Retention)

PBS kann Backups automatisch nach definierten Regeln löschen:

```bash
# Retention-Policy auf Datastore setzen (PBS-CLI)
proxmox-backup-manager datastore update backups \
    --keep-last 3 \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6 \
    --keep-yearly 2

# Prune manuell ausführen (löscht Backups nach Policy)
proxmox-backup-client prune --repository backup@pbs@192.168.1.100:backups \
    --keep-daily 7 --keep-weekly 4

# Garbage Collection (räumt verwaiste Chunks auf)
proxmox-backup-manager garbage-collection start backups
proxmox-backup-manager task list | grep gc
```

---

## Backup wiederherstellen

### Komplette VM wiederherstellen

```bash
# Via Web-UI: Storage (PBS) → Backups → Restore

# Via CLI:
# Liste der verfügbaren Backups
pvesh get /nodes/pve/storage/pbs-backup/content

# VM wiederherstellen (überschreibt bestehende VM 100)
qmrestore pbs-backup:backup/vm/100/2024-01-15T02:00:00Z 100 \
    --storage local-lvm \
    --force 1

# In neue VM-ID wiederherstellen
qmrestore pbs-backup:backup/vm/100/2024-01-15T02:00:00Z 150 \
    --storage local-lvm
```

### Container wiederherstellen

```bash
pct restore 200 pbs-backup:backup/ct/200/2024-01-15T02:00:00Z \
    --storage local-lvm \
    --force 1
```

### Single-File-Restore (ohne kompletten VM-Restore)

PBS ermöglicht das Mounten von Backups und das Extrahieren einzelner Dateien:

```bash
# Backup mounten (interaktiv über PBS-Web-UI oder CLI)
proxmox-backup-client mount --repository backup@pbs@192.168.1.100:backups \
    vm/100/2024-01-15T02:00:00Z /mnt/restore

# Dateien aus gemountem Backup kopieren
ls /mnt/restore/
cp /mnt/restore/var/www/html/important-file.php /root/restored/

# Unmount
proxmox-backup-client umount /mnt/restore
```

---

## Backup-Verschlüsselung

```bash
# Verschlüsselungsschlüssel erstellen (auf PVE-Host)
proxmox-backup-client key create /etc/pve/priv/backup-key.json

# Key-Fingerprint anzeigen (für Dokumentation)
proxmox-backup-client key show /etc/pve/priv/backup-key.json --fingerprint

# Backup mit Verschlüsselung erstellen
vzdump 100 --storage pbs-backup --mode snapshot \
    --encrypt 1

# WICHTIG: Key-Backup
# Ohne den Schlüssel sind verschlüsselte Backups NICHT wiederherstellbar!
# Key sicher aufbewahren (z.B. Passwort-Manager, ausgedruckt im Safe)
```

> [!DANGER] Verschlüsselungs-Key sichern
> Der Verschlüsselungsschlüssel liegt standardmäßig in `/etc/pve/priv/`. Geht dieser Schlüssel verloren (z.B. durch Host-Ausfall), sind alle verschlüsselten Backups **permanent unlesbar**. Immer mehrere Kopien an sicheren, getrennten Orten aufbewahren.

---

## PBS-Monitoring

```bash
# Task-Übersicht (auf PBS)
proxmox-backup-manager task list

# Fehlerhafte Tasks
proxmox-backup-manager task list | grep error

# Backup-Statistiken
proxmox-backup-manager datastore show backups

# Garbage-Collection-Status
proxmox-backup-manager garbage-collection status backups

# PBS-Service-Status
systemctl status proxmox-backup
journalctl -u proxmox-backup -f
```

---

## Verwandte Notizen

- [[Was ist Proxmox]]
- [[Proxmox Storage]]
- [[Proxmox Cluster und HA]]
- [[Proxmox Best Practices]]
- [[Proxmox Troubleshooting]]
