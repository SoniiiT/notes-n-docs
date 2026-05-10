---
tags: [lxd, best-practices, produktion, empfehlungen]
aliases: ["LXD Best Practices", "LXD Empfehlungen"]
---

# LXD Best Practices

> [!ABSTRACT]
> Bewährte Methoden für den produktiven LXD-Einsatz: von Container-Design über Ressourcenverwaltung, Update-Strategien bis hin zu Backup und Sicherheit. Orientiert an realen Homelab- und Produktionsszenarien.

---

## Container-Design

### Ein Service — ein Container

Auch wenn LXD Systemcontainer mit mehreren Diensten erlaubt, gilt als Best Practice:

```
✅ Empfohlen:
  Container "webserver"   → nginx
  Container "database"    → postgresql
  Container "cache"       → redis

⚠️ Nur wenn eng gekoppelt:
  Container "lemp-stack"  → nginx + php-fpm + mysql (Legacy-Migration)
```

**Vorteile der Trennung:**
- Unabhängige Updates und Neustarts
- Klare Ressourcenzuordnung
- Einfacheres Troubleshooting
- Einzelne Container können snapshotten/migrieren

### Container benennen

```bash
# Sprechende Namen mit Konvention
# Format: funktion-umgebung oder service-nummer
lxc launch ubuntu:24.04 nginx-prod
lxc launch ubuntu:24.04 postgres-01
lxc launch ubuntu:24.04 pihole-home
lxc launch ubuntu:24.04 dev-nextcloud

# Nicht empfohlen: generische Namen
lxc launch ubuntu:24.04 container1  # ❌
lxc launch ubuntu:24.04 test        # ❌ (was ist "test"?)
```

---

## Ressourcenlimits — immer setzen

Ohne Limits können einzelne Container den gesamten Host belasten:

```bash
# Limits direkt beim Start
lxc launch ubuntu:24.04 webapp \
    -c limits.cpu=2 \
    -c limits.memory=2GiB \
    -c limits.memory.swap=false

# Oder über Profil
lxc profile create production
lxc profile set production limits.cpu 2
lxc profile set production limits.memory 4GiB
lxc profile set production limits.memory.swap false
lxc profile set production boot.autostart true

# Limits aller Container prüfen
lxc list -c n,limits.cpu,limits.memory
```

### Faustregeln für Dimensionierung

| Anwendungsfall | CPU | RAM |
|---------------|-----|-----|
| Leichter Dienst (Pi-hole, DNS) | 1 | 256 MiB |
| Webserver (nginx, Apache) | 1–2 | 512 MiB – 1 GiB |
| Datenbank (PostgreSQL, MySQL) | 2–4 | 1–4 GiB |
| Entwicklungsumgebung | 4 | 4–8 GiB |
| Build-System | 4–8 | 4–16 GiB |

---

## Storage — die richtige Wahl

### ZFS für Produktion

```bash
# ZFS auf dediziertem Block-Device einrichten
# (Nicht auf Loop-File für Produktion!)
lxc storage create production zfs source=/dev/sdb

# ZFS-Kompression aktivieren (spart 20–40% Speicher)
sudo zfs set compression=lz4 lxdpool

# ARC-Cache begrenzen (wenn RAM knapp)
echo "options zfs zfs_arc_max=$((2 * 1024 * 1024 * 1024))" | \
    sudo tee /etc/modprobe.d/zfs.conf
sudo update-initramfs -u
```

### Disk-Limits pro Container

```bash
# Jeder Container bekommt explizites Limit
lxc config device set mycontainer root size 20GiB

# Im Profil: Standardgröße für alle Container
lxc profile device set production root size 20GiB
```

---

## Auto-Start und Startreihenfolge

```bash
# Auto-Start aktivieren
lxc config set mycontainer boot.autostart true

# Startverzögerung (Sekunden nach Host-Boot)
lxc config set mycontainer boot.autostart.delay 10

# Startreihenfolge (höher = früher)
# Datenbank zuerst, dann Webserver
lxc config set database boot.autostart.priority 10
lxc config set webserver boot.autostart.priority 5
lxc config set webserver boot.autostart.delay 15  # nach Datenbank warten
```

---

## Snapshots — Strategie

### Automatische Snapshots

```bash
# Tägliche Snapshots, 7 Tage aufbewahren
lxc config set mycontainer snapshots.schedule "0 2 * * *"
lxc config set mycontainer snapshots.expiry 7d
lxc config set mycontainer snapshots.pattern "auto-%Y%m%d"

# Snapshot vor Updates (manuell)
lxc snapshot mycontainer pre-update-$(date +%Y%m%d)
# Update durchführen...
# Wenn Update fehlschlägt:
lxc restore mycontainer pre-update-20260306
```

### Snapshot-Hygiene

```bash
# Alle Snapshots aller Container auflisten
lxc list -c nS

# Alte Snapshots automatisch bereinigen
for ct in $(lxc list --format=csv -c n); do
    # Snapshots älter als 30 Tage löschen
    lxc info $ct | grep "auto-" | awk '{print $1}' | while read snap; do
        lxc delete $ct/$snap 2>/dev/null
    done
done
```

---

## Updates und Maintenance

### LXD-Daemon aktuell halten

```bash
# Manuelles Update
sudo snap refresh lxd

# Auto-Update ist Standard (snap aktualisiert täglich)
# Auto-Update prüfen
snap get system refresh.timer

# Update-Fenster festlegen (z.B. nachts zwischen 2–4 Uhr)
sudo snap set system refresh.timer="02:00-04:00"
```

### Container-Betriebssystem aktuell halten

```bash
# In-Container-Update mit Snapshot-Schutz
lxc snapshot webserver pre-update
lxc exec webserver -- bash -c "apt update && apt upgrade -y"

# Falls Probleme:
lxc restore webserver pre-update

# Mehrere Container parallel updaten
for ct in nginx-prod postgres-01 redis-01; do
    echo "Updating $ct..."
    lxc snapshot $ct pre-update
    lxc exec $ct -- apt update -q
    lxc exec $ct -- apt upgrade -y -q
done
```

---

## Netzwerk-Best-Practices

### Netz-Segmentierung

```bash
# Trennung: Frontend, Backend, Management
lxc network create frontend-net ipv4.address=10.1.0.1/24 ipv4.nat=true
lxc network create backend-net  ipv4.address=10.2.0.1/24 ipv4.nat=false
lxc network create mgmt-net     ipv4.address=10.3.0.1/24 ipv4.nat=false

# Webserver: Frontend + Backend
lxc config device add webserver eth0 nic network=frontend-net
lxc config device add webserver eth1 nic network=backend-net

# Datenbank: nur Backend
lxc config device add database eth0 nic network=backend-net

# Statische IPs für Infrastruktur-Dienste
lxc config device set database eth0 ipv4.address 10.2.0.10
lxc config device set webserver eth1 ipv4.address 10.2.0.20
```

### MAC- und IP-Filtering immer aktiv

```bash
# In Produktionsprofil aufnehmen
lxc profile device set production eth0 security.mac_filtering true
lxc profile device set production eth0 security.ipv4_filtering true
```

---

## Sicherheits-Best-Practices

```bash
# 1. Immer unprivileged
lxc config set mycontainer security.privileged false

# 2. UID-Isolation zwischen Containern
lxc config set mycontainer security.idmap.isolated true

# 3. Keine direkten Host-Verzeichnisse einhängen (außer read-only)
lxc config device add mycontainer config disk \
    source=/etc/myapp-config \
    path=/etc/myapp \
    readonly=true          # ← immer readonly wenn möglich

# 4. lxd-Gruppe minimieren
grep lxd /etc/group        # nur vertrauenswürdige Benutzer
```

---

## Backup-Strategie

### Container-Export (Full Backup)

```bash
# Container exportieren (als .tar.gz)
lxc stop mycontainer
lxc export mycontainer /backup/mycontainer-$(date +%Y%m%d).tar.gz
lxc start mycontainer

# Wiederherstellen
lxc import /backup/mycontainer-20260306.tar.gz
lxc start mycontainer
```

### Snapshot-basiertes Backup

```bash
#!/bin/bash
# backup-lxd.sh — tägliches Backup-Script

BACKUP_DIR="/mnt/backup/lxd"
DATE=$(date +%Y%m%d)

for ct in $(lxc list --format=csv -c n,s | grep RUNNING | cut -d, -f1); do
    echo "Backing up $ct..."
    lxc snapshot $ct "backup-$DATE"
    lxc export $ct "$BACKUP_DIR/$ct-$DATE.tar.gz" \
        --instance-only --optimized-storage
    lxc delete $ct/backup-$DATE
done

# Backups älter als 7 Tage löschen
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete
```

---

## Checkliste: Neuer Container

```
Vor dem Deployment:
  [ ] Aussagekräftiger Name gewählt
  [ ] limits.cpu gesetzt
  [ ] limits.memory gesetzt
  [ ] limits.memory.swap false
  [ ] Root-Disk-Size gesetzt (root size XGiB)
  [ ] Richtiges Netzwerk zugewiesen
  [ ] security.privileged = false
  [ ] boot.autostart konfiguriert
  [ ] Snapshot-Schedule gesetzt (snapshots.schedule)
  [ ] cloud-init oder erste Konfiguration eingespielt
```

---

## Verwandte Notizen

- [[LXD Installation und Setup]] — Initiales Setup
- [[Profile und Konfiguration]] — Ressourcenlimits und Devices
- [[LXD Sicherheit]] — Sicherheitsdetails
- [[LXD Troubleshooting]] — Wenn etwas schiefläuft
