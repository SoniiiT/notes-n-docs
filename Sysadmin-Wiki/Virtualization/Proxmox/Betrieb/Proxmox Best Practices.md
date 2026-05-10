---
tags:
  - virtualization
  - proxmox
  - best-practices
  - security
  - performance
  - hardening
aliases:
  - Proxmox Best Practices
  - PVE Empfehlungen
  - Proxmox Absicherung
---

# Proxmox Best Practices

> [!ABSTRACT] TL;DR
> Sammlung bewährter Empfehlungen für produktiven Proxmox-Betrieb — von der Host-Absicherung über Storage-Design und Netzwerk-Segmentierung bis hin zu Backup-Strategien und Performance-Tuning. Diese Best Practices gelten für PVE 7.x/8.x in KMU- und Homelab-Umgebungen.

---

## Host-Konfiguration

### Updates und Patch-Management

```bash
# Regelmäßige Updates einplanen (wöchentlich oder nach Sicherheitsupdates)
apt update && apt full-upgrade -y

# Automatische Sicherheitsupdates aktivieren (nur für nicht-breaking Updates)
apt install unattended-upgrades
dpkg-reconfigure unattended-upgrades

# Nach Kernel-Updates: Neustart einplanen (im Wartungsfenster)
# Kernel-Version prüfen
uname -r
dpkg -l | grep pve-kernel
```

> [!IMPORTANT] Keine spontanen Reboots in Produktion
> Kernel-Updates erfordern einen Neustart. In Produktionsumgebungen immer Wartungsfenster einplanen — VMs vorher auf andere Nodes migrieren (bei Cluster) oder rechtzeitig herunterfahren.

### Swap deaktivieren oder minimieren

```bash
# Swap deaktivieren (auf dedizierten Virtualisierungs-Hosts empfohlen)
swapoff -a
# In /etc/fstab die swap-Zeile auskommentieren

# Swappiness reduzieren (wenn Swap behalten)
echo "vm.swappiness=10" >> /etc/sysctl.conf
sysctl -p
```

> [!NOTE] Warum kein Swap?
> Wenn der Host anfängt zu swappen, werden alle VMs gleichzeitig langsam. Es ist besser, VMs zu migrieren oder zu stoppen, bevor der Host unter Speichermangel leidet.

### Watchdog konfigurieren

```bash
# Softdog-Watchdog für HA aktivieren
echo "softdog" >> /etc/modules-load.d/watchdog.conf
modprobe softdog

# Hardware-Watchdog bevorzugen, falls verfügbar
ls /dev/watchdog*
# z.B. /dev/watchdog0 → Hardware-Watchdog vorhanden
```

---

## Netzwerk

### Management-Netzwerk trennen

```bash
# Best Practice: Separate NICs für verschiedene Netzwerke
# - vmbr0: Management-Netzwerk (Web-UI, SSH)
# - vmbr1: VM-Produktionsnetzwerk
# - vmbr2: Storage-Netzwerk (Ceph, NFS, iSCSI)
# - vmbr3: Migration-Netzwerk (Live Migration)
```

Beispielkonfiguration:

```
# Management (vmbr0)
auto vmbr0
iface vmbr0 inet static
    address 10.0.1.10/24
    gateway 10.0.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

# Storage-Netzwerk (vmbr2) - kein Gateway, reines L2
auto vmbr2
iface vmbr2 inet static
    address 10.0.3.10/24
    bridge-ports eno3
    bridge-stp off
    bridge-fd 0
```

### Proxmox Firewall aktivieren

```bash
# Datacenter-Firewall aktivieren (Web-UI: Datacenter → Firewall → Options → Firewall: Yes)
# Mindest-Regelset für Host:

# /etc/pve/nodes/pve1/host.fw
[OPTIONS]
enable: 1
log_level_in: nolog
log_level_out: nolog

[RULES]
# SSH nur aus Managementnetz
IN ACCEPT -proto tcp -dport 22 -source 10.0.1.0/24
# PVE Web-UI nur aus Managementnetz
IN ACCEPT -proto tcp -dport 8006 -source 10.0.1.0/24
# Cluster-Kommunikation zwischen Nodes erlauben
IN ACCEPT -source 10.0.1.0/24 -dest 10.0.1.0/24
# ICMP erlauben
IN ACCEPT -proto icmp
# Alles andere ablehnen
IN DROP
```

### NTP-Synchronisation sicherstellen

```bash
# Alle Nodes müssen zeitsynchron sein (für Cluster-Quorum und Logs!)
timedatectl status
timedatectl set-ntp true

# NTP-Server konfigurieren
nano /etc/systemd/timesyncd.conf
# NTP=ntp.example.local 0.de.pool.ntp.org

systemctl restart systemd-timesyncd
timedatectl show-timesync
```

---

## Storage

### 3-2-1 Backup-Regel einhalten

| Regel | Bedeutung | Umsetzung in Proxmox |
|---|---|---|
| **3** Kopien der Daten | Produktionsdaten + 2 Backups | PBS + zusätzliche Offsite-Kopie |
| **2** verschiedene Medien | NVMe + HDD + Cloud | ZFS lokal + PBS remote |
| **1** Offsite-Kopie | Außerhalb des Rechenzentrums | PBS auf separatem Standort / Cloud |

### ZFS-Best-Practices

```bash
# Kompression immer aktivieren
zfs set compression=lz4 datastore

# Recordsize für VM-Workloads optimieren
zfs set recordsize=64K datastore/vms      # VMs mit großen sequentiellen I/Os
zfs set recordsize=8K datastore/databases  # Datenbanken

# ARC-Größe begrenzen (max. 50 % des RAM)
# In /etc/modprobe.d/zfs.conf:
echo "options zfs zfs_arc_max=8589934592" > /etc/modprobe.d/zfs.conf
# 8589934592 = 8 GB

# ZFS scrub regelmäßig ausführen (monatlich)
zpool scrub datastore

# Scrub per Cron automatisieren
echo "0 2 1 * * root zpool scrub datastore" >> /etc/cron.d/zfs-scrub

# SMART-Monitoring aktivieren
apt install smartmontools
systemctl enable --now smartd
```

### Disk-Trennung

| Verwendung | Empfohlenes Medium | Begründung |
|---|---|---|
| Proxmox OS | 2x SSD (ZFS Mirror) | Selbstheilung, Redundanz |
| VM-Storage | NVMe SSD (ZFS) | Latenz, IOPS |
| Backup-Storage | HDD RAID (ZFS) | Kapazität, Kosten |
| Ceph OSD | SSD oder HDD (je nach Budget) | Je nach Performance-Anforderung |

---

## VM-Konfiguration

### VirtIO-Treiber immer nutzen

| Gerät | Falsch | Richtig |
|---|---|---|
| Netzwerk | `e1000` (emuliert) | `virtio` |
| Disk Controller | `IDE`, `SATA` | `virtio-scsi-single` |
| Disk Bus | `IDE`, `SATA` | `VirtIO Block` oder `SCSI` (via virtio-scsi) |
| Display | `std` (QXL) | `virtio` (oder `none` + serial) |

```bash
# Netzwerk auf VirtIO umstellen
qm set 100 --net0 virtio,bridge=vmbr0

# SCSI-Controller auf virtio-scsi-single
qm set 100 --scsihw virtio-scsi-single
```

### CPU-Typ richtig wählen

| CPU-Typ | Wann nutzen |
|---|---|
| `host` | Maximale Performance, kein Live-Migration zwischen unterschiedlichen CPUs |
| `kvm64` | Live Migration zwischen unterschiedlichen CPU-Generationen |
| `x86-64-v2-AES`, `x86-64-v3` | Guter Kompromiss: moderne Features, bessere Migration als `host` |

### RAM-Ballooning

```bash
# Ballooning aktiviert: Host kann RAM dynamisch zwischen VMs aufteilen
qm set 100 --memory 8192 --balloon 4096
# VM startet mit 4 GB, kann auf bis zu 8 GB wachsen

# Für Performance-kritische VMs: festen RAM zuweisen (Ballooning deaktivieren)
qm set 100 --memory 8192 --balloon 0
```

### QEMU Guest Agent überall aktivieren

```bash
# In PVE: QEMU Guest Agent aktivieren
qm set 100 --agent enabled=1,fstrim_cloned_disks=1

# Im Gast installieren
# Linux: apt install qemu-guest-agent && systemctl enable --now qemu-guest-agent
# Windows: VirtIO-Driver-ISO einbinden → Guest Agent installieren
```

### VM-Schutz aktivieren

```bash
# Verhindert versehentliches Löschen produktiver VMs
qm set 100 --protection 1

# Autostart konfigurieren (mit Reihenfolge)
qm set 100 --onboot 1 --startup order=1,up=30,down=120
# order=1: Bootreihenfolge (niedrige Nummer = zuerst)
# up=30: Warte 30 Sekunden nach Start bevor nächste VM
# down=120: Warte 120 Sekunden beim Herunterfahren
```

---

## Sicherheit

### SSH absichern

```bash
# Passwort-Authentifizierung deaktivieren (nach SSH-Key-Setup)
nano /etc/ssh/sshd_config

# Empfohlene Einstellungen:
# PermitRootLogin prohibit-password
# PasswordAuthentication no
# MaxAuthTries 3
# LoginGraceTime 30

systemctl restart sshd

# SSH-Key auf PVE einrichten
ssh-copy-id root@pve-host
```

### PVE-Benutzer und Rollen

```bash
# Keinen Root direkt nutzen — eigene PVE-Benutzer anlegen
pveum user add admin@pve --password SecurePass --email admin@example.com

# Rollen zuweisen
pveum acl modify / -user admin@pve -role Administrator

# Verfügbare Rollen anzeigen
pveum role list

# Audit-Log aktivieren
# Web-UI: Datacenter → Options → Log entries per page
```

### Two-Factor Authentication (2FA)

```bash
# 2FA im Web-UI aktivieren:
# Benutzer → Einstellungen → Two Factor Authentication
# TOTP (z.B. Google Authenticator) oder Recovery Keys

# Via CLI:
pveum user tfa add admin@pve totp
```

### Subscription-Repository verwenden (Produktion)

Für Produktionsumgebungen: Enterprise-Subscription kaufen und Enterprise-Repo nutzen — nur dort werden Pakete ausgiebig getestet vor dem Release.

---

## Performance-Tuning

### CPU-Überbuchung vermeiden

```bash
# vCPU-Verhältnis berechnen: vCPUs gesamt / physische Kerne
# Empfehlung: max. 3:1 für durchschnittliche Workloads
# Für CPU-intensive Workloads: max. 1:1

# CPU-Nutzung aller VMs anzeigen
qm list | awk 'NR>1 {print $1}' | xargs -I{} sh -c \
    'echo -n "VM {}: "; qm config {} | grep cores'
```

### I/O-Scheduler optimieren

```bash
# Für SSDs: mq-deadline oder none (kein Scheduler)
# Für HDDs: mq-deadline oder bfq

# Aktuellen Scheduler anzeigen
cat /sys/block/sda/queue/scheduler

# Scheduler dauerhaft setzen (udev-Regel)
echo 'ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"' \
    > /etc/udev/rules.d/60-scheduler.rules
udev adm trigger

# ZFS: Kein Scheduler auf Raw-Disks (ZFS hat eigenen Cache)
```

### Hugepages für VMs aktivieren

```bash
# Hugepages verbessern Performance RAM-intensiver VMs
# 1 GB Hugepages allokieren (2048 x 2 MB):
echo 2048 > /proc/sys/vm/nr_hugepages

# Dauerhaft in /etc/sysctl.conf:
echo "vm.nr_hugepages=2048" >> /etc/sysctl.conf

# In VM aktivieren:
qm set 100 --hugepages 2   # 2 MB Hugepages
```

---

## Monitoring und Alerting

```bash
# Proxmox hat integriertes Monitoring (Graphs in Web-UI)
# Für erweiterte Metriken: Telegraf + InfluxDB + Grafana

# SMTP-Benachrichtigungen konfigurieren
# Web-UI: Datacenter → Options → Email from address
# Node → Shell:
nano /etc/postfix/main.cf
# relayhost = [smtp.example.com]:587

# Backups und Tasks per Email benachrichtigen:
# Web-UI: Datacenter → Backup → Add → Send Email To
```

---

## Regelmäßige Wartungsaufgaben

| Aufgabe | Häufigkeit | Befehl |
|---|---|---|
| System-Updates | Wöchentlich | `apt update && apt full-upgrade` |
| Backup-Verifikation | Wöchentlich | PBS: Verify-Job einrichten |
| ZFS Scrub | Monatlich | `zpool scrub datastore` |
| SMART-Test | Monatlich | `smartctl -t long /dev/sda` |
| Logs prüfen | Wöchentlich | `journalctl -p err -S -7d` |
| Backup-Restore-Test | Vierteljährlich | Test-VM aus Backup wiederherstellen |
| Disk-Auslastung prüfen | Täglich | `pvesm status` / `zpool list` |
| HA-Status prüfen | Täglich | `ha-manager status` |

---

## Verwandte Notizen

- [[Was ist Proxmox]]
- [[Proxmox Installation]]
- [[Proxmox Netzwerk]]
- [[Proxmox Storage]]
- [[Proxmox Backup Server]]
- [[Proxmox Troubleshooting]]
- [[Proxmox CLI Cheatsheet]]
