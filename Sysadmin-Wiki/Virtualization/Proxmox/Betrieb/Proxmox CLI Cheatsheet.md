---
tags:
  - virtualization
  - proxmox
  - cli
  - cheatsheet
  - referenz
  - qm
  - pct
  - pvecm
aliases:
  - Proxmox CLI
  - Proxmox Kommandozeile
  - PVE Shell Referenz
---

# Proxmox CLI Cheatsheet

> [!ABSTRACT] TL;DR
> Vollständige CLI-Referenz für Proxmox VE — aufgeteilt in VM-Management (`qm`), Container-Management (`pct`), Storage (`pvesm`), Cluster (`pvecm`), Backups (`vzdump`) und System-Diagnose. Alle Befehle sind für PVE 7.x/8.x gültig.

---

## VM-Management (`qm`)

### VMs anzeigen und Status

```bash
qm list                          # Alle VMs auflisten
qm status 100                    # Status von VM 100
qm config 100                    # Vollständige Konfiguration
qm pending 100                   # Ausstehende (noch nicht angewendete) Änderungen
```

### VM starten / stoppen

```bash
qm start 100                     # VM starten
qm shutdown 100                  # Graceful shutdown (ACPI)
qm stop 100                      # Sofortiger Stopp (kein graceful)
qm reboot 100                    # Neustart
qm reset 100                     # Hard Reset (wie Reset-Taste)
qm suspend 100                   # VM pausieren (RAM auf Disk)
qm resume 100                    # VM fortsetzen
```

### VM erstellen und löschen

```bash
# Neue VM erstellen
qm create 100 --name "test-vm" --memory 2048 --cores 2 \
    --net0 virtio,bridge=vmbr0 \
    --scsi0 local-lvm:20 --scsihw virtio-scsi-single \
    --ide2 local:iso/debian.iso,media=cdrom \
    --bios ovmf --machine q35 --ostype l26

# VM löschen (nur wenn gestoppt)
qm destroy 100

# VM löschen inkl. aller Disks
qm destroy 100 --destroy-unreferenced-disks 1 --purge 1
```

### VM konfigurieren

```bash
qm set 100 --memory 4096         # RAM ändern
qm set 100 --cores 4             # CPU-Kerne ändern
qm set 100 --name "neuer-name"   # Name ändern
qm set 100 --onboot 1            # Autostart aktivieren
qm set 100 --startup order=1     # Boot-Reihenfolge (order=N, up=S, down=S)
qm set 100 --agent enabled=1     # QEMU Guest Agent aktivieren
qm set 100 --protection 1        # VM vor versehentlichem Löschen schützen
qm set 100 --description "Meine VM"

# Disk vergrößern
qm resize 100 scsi0 +20G

# Disk von Storage migrieren
qm move-disk 100 scsi0 ceph-pool --delete

# ISO auswerfen
qm set 100 --ide2 none,media=cdrom
```

### VM klonen und Templates

```bash
# VM als Template markieren (unumkehrbar)
qm template 100

# Linked Clone (spart Platz)
qm clone 100 101 --name "klon-vm"

# Full Clone (unabhängige Kopie)
qm clone 100 101 --name "klon-vm" --full 1 --storage local-lvm

# Cloud-Init-Einstellungen
qm set 101 --ciuser admin
qm set 101 --sshkey ~/.ssh/id_ed25519.pub
qm set 101 --ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1
qm cloudinit dump 101            # Cloud-Init-Config anzeigen
```

### Snapshots

```bash
qm snapshot 100 "pre-update"     # Snapshot erstellen
qm listsnapshot 100              # Snapshots anzeigen
qm rollback 100 "pre-update"     # Zurückrollen (VM muss gestoppt sein)
qm delsnapshot 100 "pre-update"  # Snapshot löschen
```

### Migration

```bash
qm migrate 100 pve2 --online              # Live Migration
qm migrate 100 pve2 --online --with-local-disks  # Mit lokalem Storage
qm migrate 100 pve2                        # Offline Migration
```

### VM-Konsole und Monitor

```bash
qm terminal 100 --iface serial0    # Serial Console
qm monitor 100                     # QMP Monitor öffnen
qm agent 100 ping                  # Guest Agent testen
qm agent 100 get-osinfo            # Gast-OS-Info
qm agent 100 network-get-interfaces  # IP-Adressen im Gast
qm agent 100 exec --command '["systemctl", "status", "nginx"]'
```

---

## Container-Management (`pct`)

### Container anzeigen und Status

```bash
pct list                         # Alle Container auflisten
pct status 200                   # Status von Container 200
pct config 200                   # Konfiguration anzeigen
```

### Container starten / stoppen

```bash
pct start 200                    # Container starten
pct shutdown 200                 # Graceful shutdown
pct stop 200                     # Sofortiger Stopp
pct reboot 200                   # Neustart
```

### Container erstellen und löschen

```bash
# Container erstellen
pct create 200 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
    --hostname webserver \
    --memory 2048 --swap 512 \
    --cores 2 \
    --rootfs local-lvm:20 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.200/24,gw=192.168.1.1 \
    --unprivileged 1 \
    --password MyPass

# Container löschen
pct destroy 200
```

### Container konfigurieren

```bash
pct set 200 --memory 4096        # RAM ändern
pct set 200 --cores 4            # CPU-Kerne
pct set 200 --onboot 1           # Autostart
pct set 200 --features nesting=1 # Docker-Support aktivieren
pct set 200 --hostname neuer-name

# Disk vergrößern
pct resize 200 rootfs +10G

# Zusätzlichen Mount-Point hinzufügen
pct set 200 --mp0 /mnt/data,mp=/data

# Netzwerkadapter hinzufügen
pct set 200 --net1 name=eth1,bridge=vmbr1,ip=10.0.0.200/24
```

### Container-Shell und Management

```bash
pct enter 200                    # Shell im Container öffnen
pct exec 200 -- systemctl status nginx  # Befehl im Container ausführen
pct push 200 /lokale/datei /ziel/pfad   # Datei in Container kopieren
pct pull 200 /pfad/im/container /lokal  # Datei aus Container kopieren
```

### Container-Snapshots und Templates

```bash
pct snapshot 200 "clean-install"
pct listsnapshot 200
pct rollback 200 "clean-install"
pct delsnapshot 200 "clean-install"

# Als Template markieren
pct template 200

# Klonen
pct clone 200 201 --hostname klon-container --full 1
```

### Templates verwalten

```bash
pveam update                     # Template-Liste aktualisieren
pveam available --section system # Verfügbare Templates
pveam available --section turnkey # TurnKey-Templates
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
pveam list local                 # Heruntergeladene Templates
pveam remove local:vztmpl/altes-template.tar.gz
```

---

## Storage-Management (`pvesm`)

```bash
pvesm status                     # Alle Storages und Auslastung
pvesm list local                 # Inhalt eines Storages

# Storage hinzufügen
pvesm add dir my-dir --path /mnt/backup --content backup,iso
pvesm add nfs nfs-storage --server 192.168.1.50 --export /backup --content backup
pvesm add zfspool my-zfs --pool datastore --content images,rootdir
pvesm add pbs pbs-backup --server 192.168.1.100 --datastore backups \
    --username backup@pbs --password MyPass --fingerprint AA:BB:CC:...

pvesm set local --content iso,vztmpl,backup  # Inhaltstypen ändern
pvesm remove my-storage          # Storage entfernen (löscht keine Daten!)
```

---

## Cluster-Management (`pvecm`)

```bash
pvecm status                     # Cluster-Status
pvecm nodes                      # Node-Liste
pvecm quorum status              # Quorum-Details

pvecm create mein-cluster        # Neuen Cluster erstellen
pvecm add 192.168.10.1           # Cluster beitreten

pvecm delnode ausgefallener-node # Node aus Cluster entfernen
pvecm expected 1                 # Erwartete Votes manuell setzen (Notfall)
```

---

## Backup und Restore (`vzdump`)

```bash
# Backup erstellen
vzdump 100 --storage pbs-backup --mode snapshot --compress zstd
vzdump 200 --storage local --mode stop --compress zstd

# Alle VMs und Container sichern
vzdump --all --storage pbs-backup --mode snapshot

# Backup-Job im Cluster-Scheduler auflisten
pvesh get /cluster/backup

# VM aus Backup wiederherstellen
qmrestore /var/lib/vz/dump/vzdump-qemu-100-2024_01_15-02_00_00.vma.zst 100
qmrestore pbs-backup:backup/vm/100/2024-01-15T02:00:00Z 100 --storage local-lvm

# Container wiederherstellen
pct restore 200 /var/lib/vz/dump/vzdump-lxc-200-2024_01_15-02_00_00.tar.zst
```

---

## HA-Management

```bash
ha-manager status                # Status aller HA-Ressourcen
ha-manager add vm:100 --group ha-gruppe --state started
ha-manager add ct:200 --state started
ha-manager set vm:100 --state disabled   # Deaktivieren (Wartung)
ha-manager remove vm:100

ha-manager groupadd ha-gruppe --nodes pve1:100,pve2:50,pve3:50
ha-manager grouplist
ha-manager groupremove ha-gruppe
```

---

## System & Diagnose

```bash
# PVE-Versionen anzeigen
pveversion -v

# Ressourcenauslastung
pvesh get /nodes/pve/status
pvesh get /nodes/pve/rrd --ds cpu --timeframe hour

# Node-Tasks
pvesh get /nodes/pve/tasks

# Cluster-Log
pvesh get /cluster/log --max 50

# Netzwerkdiagnose
pvesh get /nodes/pve/network

# Dienst-Status
systemctl status pvedaemon pveproxy pvestatd pve-cluster

# Dienste neu starten
systemctl restart pveproxy      # Web-UI Proxy
systemctl restart pvedaemon     # PVE Daemon
systemctl restart pvestatd      # Statistik-Daemon
systemctl restart pve-cluster   # Cluster-Dienst

# Logs
journalctl -u pvedaemon -f
journalctl -u pveproxy -f
journalctl -u pve-cluster -f

# Netzwerkänderungen anwenden (ohne Reboot)
ifreload -a

# ZFS-Status
zpool status
zpool list
zfs list

# Ceph-Status
ceph status
ceph osd tree
ceph df
```

---

## pvesh — REST-API via CLI

`pvesh` erlaubt Zugriff auf die vollständige PVE REST-API:

```bash
# Alle Nodes im Cluster
pvesh get /nodes

# VM-Details
pvesh get /nodes/pve/qemu/100/status/current

# Alle VMs eines Nodes
pvesh get /nodes/pve/qemu

# Task starten (VM starten)
pvesh create /nodes/pve/qemu/100/status/start

# Storage-Inhalt
pvesh get /nodes/pve/storage/local/content

# API-Dokumentation (interaktiv)
pvesh ls /
pvesh ls /nodes/pve/qemu/100/
```

---

## Verwandte Notizen

- [[Proxmox VM und Container Verwaltung]]
- [[Proxmox Storage]]
- [[Proxmox Cluster und HA]]
- [[Proxmox Troubleshooting]]
- [[Proxmox Best Practices]]
