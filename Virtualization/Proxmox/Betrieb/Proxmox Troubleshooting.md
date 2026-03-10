---
tags:
  - virtualization
  - proxmox
  - troubleshooting
  - fehler
  - diagnose
  - logs
aliases:
  - Proxmox Fehleranalyse
  - Proxmox Diagnose
  - PVE Troubleshooting
---

# Proxmox Troubleshooting

> [!ABSTRACT] TL;DR
> Strukturierte Fehlerdiagnose für Proxmox VE — Log-Dateipfade, häufige Fehlerbilder bei VMs, Container, Netzwerk, Storage und Cluster mit konkreten Ursachen und Lösungsschritten. Inklusive Performance-Analyse und Notfall-Zugriffsverfahren.

---

## Log-Dateien und Diagnose-Quellen

### Wichtige Log-Dateien

| Log | Pfad | Inhalt |
|---|---|---|
| PVE Daemon | `journalctl -u pvedaemon` | Allgemeine PVE-Ereignisse |
| Web-UI Proxy | `journalctl -u pveproxy` | Web-UI-Fehler, Login-Versuche |
| Cluster | `journalctl -u pve-cluster` | Cluster-Kommunikation, pmxcfs |
| Corosync | `journalctl -u corosync` | Quorum, Ring-Status |
| HA Manager | `journalctl -u pve-ha-crm` | HA Cluster Resource Manager |
| HA Local | `journalctl -u pve-ha-lrm` | HA Local Resource Manager |
| Netzwerk | `/var/log/syslog` | Bridge, Bond, VLAN-Ereignisse |
| QEMU-VM | `/var/log/pve/qemu-server/<vmid>.log` | Einzelne VM-Ereignisse |
| LXC | `/var/log/pve/lxc/<ctid>.log` | Container-Ereignisse |
| Task-Logs | `/var/log/pve/tasks/` | Alle PVE-Tasks |

### Log-Befehle

```bash
# PVE Daemon Fehler live verfolgen
journalctl -u pvedaemon -f

# Fehler der letzten 24 Stunden
journalctl -u pvedaemon -p err -S -24h

# Alle PVE-Dienste auf Fehler prüfen
for svc in pvedaemon pveproxy pve-cluster corosync; do
    echo "=== $svc ==="
    journalctl -u $svc -p err -S -7d --no-pager | tail -5
done

# VM-spezifisches Log
cat /var/log/pve/qemu-server/100.log | tail -50

# Cluster-Log (Web-UI oder CLI)
pvesh get /cluster/log --max 100

# Task-Logs auflisten
ls -lt /var/log/pve/tasks/ | head -20
```

---

## VM startet nicht

### Symptom: VM bleibt im Status "stopped" oder gibt Fehler

#### Diagnose

```bash
# VM-Log prüfen
cat /var/log/pve/qemu-server/100.log

# PVE Daemon Log
journalctl -u pvedaemon -p err -S -1h

# VM-Status
qm status 100

# VM-Konfiguration prüfen
qm config 100
```

#### Ursache A: Disk-Image nicht gefunden

```
Fehlertext: "No such file or directory" oder "unable to open block device"
```

```bash
# Disk-Pfad aus Konfiguration ermitteln
qm config 100 | grep scsi

# Prüfen ob Datei existiert
ls -la /dev/pve/vm-100-disk-0    # LVM
ls -la /mnt/datastore/images/100/ # Directory Storage

# Storage-Status prüfen
pvesm status
```

**Lösung:**
- Disk manuell im Web-UI neu verknüpfen (VM → Hardware → Disk → Edit)
- Bei LVM: `lvs | grep vm-100`
- Bei ZFS: `zfs list | grep vm-100`

#### Ursache B: Kein Speicherplatz im Storage

```bash
# Storage-Auslastung prüfen
pvesm status
df -h
zpool list    # ZFS-Pool-Auslastung
lvs           # LVM-Volumes

# Platzsparender: Snapshots löschen
qm listsnapshot 100
qm delsnapshot 100 alter-snapshot
```

#### Ursache C: CPU-Typ-Konflikt (nach Migration)

```
Fehlertext: "CPUID instruction..." oder "illegal instruction"
```

```bash
# CPU-Typ auf kompatiblen Wert setzen
qm set 100 --cpu kvm64

# Oder host-Flags einzeln deaktivieren
qm set 100 --cpu host,hv-tlbflush=off,hv-ipi=off
```

#### Ursache D: PVE-Daemon nicht bereit

```bash
# Dienst-Status prüfen
systemctl status pvedaemon
systemctl status qemu-server@100

# Daemon neu starten
systemctl restart pvedaemon
```

---

## VM hängt oder reagiert nicht

### Symptom: VM läuft, aber kein Netzwerk / kein Login möglich

```bash
# Konsole direkt verbinden (keine Netzwerk-Abhängigkeit)
qm terminal 100 --iface serial0    # Serial Console (wenn konfiguriert)

# VNC-Zugriff erzwingen (temporär)
qm set 100 --vga std
qm stop 100
qm start 100
# Dann via Web-UI → Console → noVNC

# QEMU Guest Agent Status prüfen
qm agent 100 ping
qm agent 100 get-osinfo
```

### Symptom: VM friert ein (frozen)

```bash
# VM-Zustand vom Host aus prüfen
qm status 100

# QEMU-Prozess prüfen
ps aux | grep "qemu-system.*100"

# VM-Log auf Fehler prüfen
tail -50 /var/log/pve/qemu-server/100.log

# Wenn QEMU-Prozess hängt: VM hard stoppen
qm stop 100 --skiplock
kill -9 $(pgrep -f "qemu-system-x86_64.*100.conf")
```

---

## LXC Container startet nicht

```bash
# Container-Log prüfen
cat /var/log/pve/lxc/200.log

# LXC-Konfiguration direkt prüfen
cat /etc/pve/lxc/200.conf

# Rootfs-Storage prüfen
pct config 200 | grep rootfs
lvs | grep vm-200    # Bei LVM
```

#### Ursache A: Netzwerkkonflikt

```bash
# IP-Adresse doppelt vergeben?
pct config 200 | grep net
arp-scan 192.168.1.0/24 | grep 192.168.1.200
```

#### Ursache B: Unprivilegierter Container ohne Kernel-Features

```bash
# Features anzeigen
pct config 200 | grep features

# Nötige Features setzen
pct set 200 --features nesting=1,keyctl=1  # Für Docker
```

---

## Netzwerk-Probleme

### Symptom: VM hat keine Netzwerkverbindung

```bash
# Bridge-Status prüfen
bridge link show
brctl show
ip addr show vmbr0

# Testen ob Bridge aktiv
ping -I vmbr0 192.168.1.1

# VM-seitige Netzwerkkonfiguration prüfen (via Guest Agent)
qm agent 100 network-get-interfaces

# VLAN-Konfiguration prüfen
bridge vlan show
```

**Häufige Ursachen:**
- VLAN-Tag in VM != VLAN-Tag auf Switch-Port
- Bridge-Ports fehlen: `ip link show vmbr0` → check `master`
- Firewall blockiert Traffic: Web-UI → Node/VM → Firewall

```bash
# Netzwerkkonfiguration neu laden
ifreload -a

# Einzelne Bridge neu starten
ip link set vmbr0 down && ip link set vmbr0 up
```

### Symptom: PVE Web-UI nicht erreichbar

```bash
# pveproxy Status
systemctl status pveproxy
journalctl -u pveproxy -p err -S -1h

# Port 8006 gebunden?
ss -tlnp | grep 8006

# pveproxy neu starten
systemctl restart pveproxy

# Firewall-Regeln prüfen (falls PVE-Firewall aktiv)
pve-firewall status
iptables -L INPUT -n | grep 8006

# Direkter SSH-Zugang als Fallback
ssh root@<ip>
```

---

## Storage-Probleme

### Symptom: Storage als "unavailable" markiert

```bash
# Storage-Status
pvesm status

# Spezifisch für NFS: Mount prüfen
mount | grep nfs
showmount -e 192.168.1.50   # NFS-Exports auf Server

# NFS neu mounten
umount /mnt/pve/nfs-backup
mount /mnt/pve/nfs-backup   # Aus /etc/fstab

# ZFS-Pool-Status
zpool status
zpool list
```

### Symptom: ZFS-Pool degraded

```bash
# Pool-Status anzeigen
zpool status -v datastore

# Fehlerhafte Disk identifizieren
zpool status datastore | grep -E "DEGRADED|FAULTED|OFFLINE|REMOVED"

# I/O-Fehler pro Disk
zpool iostat -v datastore

# Disk ersetzen (Disk-ID IMMER über /dev/disk/by-id/ verwenden!)
zpool replace datastore /dev/disk/by-id/alte-disk /dev/disk/by-id/neue-disk

# Resilvering-Fortschritt
zpool status datastore    # Zeigt Fortschritt in %

# Nach Resilvering: Fehler-Counter zurücksetzen
zpool clear datastore
```

### Symptom: Ceph unhealthy

```bash
# Ceph-Status
ceph status
ceph health detail

# OSD-Status
ceph osd tree
ceph osd df

# Fehlerhafte OSDs
ceph osd tree | grep -E "down|out"

# OSD manuell neu starten
systemctl restart ceph-osd@0    # OSD-ID aus ceph osd tree

# PGs (Placement Groups) prüfen
ceph pg stat
ceph pg dump | grep -v active+clean | head -20

# Volle OSDs identifizieren
ceph osd df | sort -k6 -n | tail -10
```

---

## Cluster-Probleme

### Symptom: Cluster hat kein Quorum

```
Symptom: "cluster is not quorate" Fehlermeldung
VMs können nicht gestartet werden
```

```bash
# Quorum-Status
pvecm status
pvecm quorum status

# Corosync-Ring-Status
corosync-cfgtool -s

# Alle Nodes erreichbar?
for node in pve1 pve2 pve3; do
    ping -c 1 $node && echo "$node: OK" || echo "$node: NICHT ERREICHBAR"
done

# Corosync-Log prüfen
journalctl -u corosync -p err -S -1h

# Cluster-Netzwerk prüfen (Ports 5405 UDP)
ss -ulnp | grep 5405
nc -u -z pve2 5405 && echo "OK" || echo "Blockiert"
```

**Notfall: Quorum manuell erzwingen (NUR wenn alle anderen Nodes wirklich ausgefallen)**

```bash
# GEFÄHRLICH - nur im Notfall und wenn ALLE anderen Nodes down sind
pvecm expected 1

# Nach Reparatur: Wert wieder auf korrekte Node-Anzahl zurücksetzen
pvecm expected 3
```

> [!DANGER] pvecm expected nur im absoluten Notfall
> Dieser Befehl überschreibt das Quorum-Requirement. Wenn ein "ausgefallener" Node tatsächlich noch läuft und VMs betreibt, kann dies zu Split-Brain und Datenverlust führen. Vorher **physisch sicherstellen**, dass ausgefallene Nodes wirklich ausgeschaltet sind.

### Symptom: Node verlässt Cluster

```bash
# Cluster-Memberliste
pvecm nodes
pvecm status

# pve-cluster Dienst
systemctl status pve-cluster
journalctl -u pve-cluster -p err -S -1h

# pmxcfs mounten prüfen
mountpoint /etc/pve
mount | grep pve

# Cluster-Dienst neu starten
systemctl restart pve-cluster
```

---

## Performance-Probleme

### CPU-Engpass

```bash
# Auslastung aller VMs
qm list | awk 'NR>1 {print $1}' | while read id; do
    cpu=$(qm status $id | grep cpu | awk '{print $2}')
    echo "VM $id: CPU=$cpu"
done

# Host-CPU-Last
htop
mpstat -P ALL 2

# Überbuchung prüfen: Summe vCPUs vs. physische Cores
echo "vCPUs gesamt:"
qm list | awk 'NR>1 {print $1}' | while read id; do
    qm config $id | grep "^cores:" | awk '{print $2}'
done | paste -s -d+ | bc

echo "Physische Cores:"
nproc
```

### RAM-Mangel

```bash
# RAM-Auslastung Host
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapUsed"

# RAM aller VMs prüfen
qm list | awk 'NR>1 {print $1}' | while read id; do
    mem=$(qm config $id | grep "^memory:" | awk '{print $2}')
    name=$(qm config $id | grep "^name:" | awk '{print $2}')
    echo "VM $id ($name): ${mem}MB"
done | sort -k4 -n -r

# Swap-Nutzung (sollte 0 sein auf Proxmox-Host)
swapon --show
```

### Disk-Latenz

```bash
# I/O-Statistiken in Echtzeit
iostat -x 2

# Spezifisch für ZFS
zpool iostat -v datastore 2

# SMART auf Fehler prüfen
for disk in /dev/sd?; do
    echo "=== $disk ==="
    smartctl -H $disk | grep Health
done

# I/O-Latenz-Test
fio --name=latency-test --ioengine=libaio --iodepth=1 \
    --rw=randrw --bs=4k --numjobs=1 --size=1G \
    --filename=/dev/sda --output-format=normal | grep -E "lat|iops"
```

---

## Notfall-Zugang

### Proxmox Web-UI nicht erreichbar — direkte Verwaltung

```bash
# SSH-Zugang (wenn verfügbar)
ssh root@<ip>

# Alle VMs anzeigen
qm list
pct list

# VM direkt starten/stoppen
qm start 100
qm stop 100

# Netzwerk neu konfigurieren
ifreload -a
```

### Rescue-Modus (wenn Proxmox nicht bootet)

1. Von Proxmox-ISO oder Debian-Live-ISO booten
2. ZFS importieren:
   ```bash
   zpool import -f -R /mnt rpool
   mount --bind /dev /mnt/dev
   mount --bind /proc /mnt/proc
   chroot /mnt
   ```
3. Problem beheben (z.B. Konfigurationsdatei reparieren)
4. Neustart

---

## Diagnose-Checkliste

```bash
# Schnelldiagnose — alle wichtigen Checks auf einmal
echo "=== PVE Version ===" && pveversion
echo "=== Dienste ===" && systemctl is-active pvedaemon pveproxy pve-cluster corosync
echo "=== RAM ===" && free -h | head -2
echo "=== Disk ===" && df -h | grep -v tmpfs
echo "=== ZFS ===" && zpool status -x 2>/dev/null || echo "Kein ZFS"
echo "=== Cluster ===" && pvecm status 2>/dev/null || echo "Kein Cluster"
echo "=== VMs ===" && qm list
echo "=== Container ===" && pct list
echo "=== Storage ===" && pvesm status
echo "=== Fehler (letzte 24h) ===" && journalctl -p err -S -24h --no-pager | tail -20
```

---

## Verwandte Notizen

- [[Was ist Proxmox]]
- [[Proxmox Installation]]
- [[Proxmox Netzwerk]]
- [[Proxmox Storage]]
- [[Proxmox Cluster und HA]]
- [[Proxmox CLI Cheatsheet]]
- [[Proxmox Best Practices]]
