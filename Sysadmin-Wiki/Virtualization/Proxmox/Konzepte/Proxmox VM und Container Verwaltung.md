---
tags:
  - virtualization
  - proxmox
  - vm
  - lxc
  - container
  - cloud-init
  - qemu
aliases:
  - Proxmox VM Verwaltung
  - Proxmox Container Verwaltung
  - Proxmox KVM LXC
---

# Proxmox VM und Container Verwaltung

> [!ABSTRACT] TL;DR
> Proxmox VE verwaltet **KVM-VMs** (beliebige Gastbetriebssysteme) und **LXC-Container** (Linux only, nahezu native Performance) über eine gemeinsame Web-UI und CLI. VirtIO-Treiber sind für VMs entscheidend für gute Performance. Cloud-Init automatisiert VM-Erstkonfiguration. Snapshots, Templates und Migration runden das Verwaltungs-Portfolio ab.

---

## KVM-VMs

### VM erstellen (CLI)

```bash
# Neue VM erstellen
qm create 100 \
    --name "ubuntu-server" \
    --memory 4096 \
    --sockets 1 \
    --cores 4 \
    --cpu host \
    --bios ovmf \
    --machine q35 \
    --net0 virtio,bridge=vmbr0,tag=10 \
    --scsi0 local-lvm:32 \
    --scsihw virtio-scsi-single \
    --ide2 local:iso/ubuntu-22.04-server.iso,media=cdrom \
    --boot order=ide2;scsi0 \
    --ostype l26 \
    --agent enabled=1

# VM starten
qm start 100

# VM stoppen (graceful shutdown)
qm shutdown 100

# VM hart ausschalten
qm stop 100

# VM-Konfiguration anzeigen
qm config 100

# Alle VMs anzeigen
qm list
```

### Wichtige VM-Parameter

| Parameter | Beschreibung | Empfehlung |
|---|---|---|
| `--cpu host` | Host-CPU-Features direkt durchreichen | ✅ Beste Performance, kein Live-Migration zwischen unterschiedlichen CPU-Generationen |
| `--cpu kvm64` | Generischer KVM-CPU-Typ | ✅ Für Live Migration zwischen unterschiedlichen Hosts |
| `--machine q35` | Modern PC-Chipset mit PCIe | ✅ Für alle neuen VMs |
| `--machine i440fx` | Legacy-Chipset | Nur für alte OS (Windows XP, etc.) |
| `--bios ovmf` | UEFI (OVMF) | ✅ Für alle neuen Linux/Windows VMs |
| `--bios seabios` | Legacy BIOS | Nur für Legacy-OS |
| `--scsihw virtio-scsi-single` | VirtIO SCSI Controller | ✅ Beste Disk-Performance |
| `--agent enabled=1` | QEMU Guest Agent | ✅ Immer aktivieren |

### VM-Hardware ändern

```bash
# RAM ändern
qm set 100 --memory 8192

# CPU-Kerne ändern
qm set 100 --cores 8

# Neue Disk hinzufügen
qm set 100 --scsi1 local-lvm:50

# Disk vergrößern (nur vergrößern, nicht verkleinern möglich)
qm resize 100 scsi0 +20G

# Netzwerkadapter hinzufügen
qm set 100 --net1 virtio,bridge=vmbr0

# CD-ROM auswerfen
qm set 100 --ide2 none,media=cdrom
```

### QEMU Guest Agent

Der QEMU Guest Agent ermöglicht konsistente Snapshots, IP-Adress-Abfrage und graceful Shutdown:

```bash
# Im Gast installieren (Linux)
# Ubuntu/Debian:
apt install qemu-guest-agent
systemctl enable --now qemu-guest-agent

# RHEL/CentOS/Rocky:
dnf install qemu-guest-agent
systemctl enable --now qemu-guest-agent

# Windows:
# VirtIO-Treiber ISO einbinden → Guest Agent installieren
# Download: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/

# Agent-Status vom Host aus prüfen
qm agent 100 ping
qm agent 100 get-osinfo
qm agent 100 network-get-interfaces
```

---

## LXC Container

### Container erstellen (CLI)

```bash
# Verfügbare Templates herunterladen
pveam update
pveam available --section system

# Template herunterladen
pveam download local debian-12-standard_12.7-1_amd64.tar.zst

# Container erstellen
pct create 200 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
    --hostname webserver \
    --memory 2048 \
    --swap 512 \
    --rootfs local-lvm:20 \
    --cores 2 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.200/24,gw=192.168.1.1,tag=10 \
    --nameserver 192.168.1.1 \
    --password MySecurePass \
    --unprivileged 1 \
    --start 1

# Container starten/stoppen
pct start 200
pct shutdown 200
pct stop 200

# Container-Shell öffnen
pct enter 200

# Alle Container anzeigen
pct list

# Konfiguration anzeigen
pct config 200
```

### Privilegiert vs. Unprivilegiert

| Typ | User-Mapping | Sicherheit | Einsatz |
|---|---|---|---|
| **Unprivilegiert** (empfohlen) | Root im Container = UID 100000 auf Host | ✅ Hohe Isolation | Standard für alle Container |
| **Privilegiert** | Root im Container = Root auf Host | ❌ Geringere Isolation | Nur wenn zwingend nötig (z.B. NFS-Mount) |

### Wichtige Container-Optionen

```bash
# Ressourcen im laufenden Container ändern
pct set 200 --memory 4096
pct set 200 --cores 4

# Disk vergrößern
pct resize 200 rootfs +10G

# Weiteren Mount-Point hinzufügen
pct set 200 --mp0 /mnt/data,mp=/data

# Netzwerkadapter hinzufügen
pct set 200 --net1 name=eth1,bridge=vmbr1,ip=10.0.0.200/24

# Nesting aktivieren (für Docker in LXC)
pct set 200 --features nesting=1

# Container klonen
pct clone 200 201 --hostname webserver-2 --full 1
```

> [!WARNING] Docker in LXC
> Docker in einem unprivilegierten LXC-Container funktioniert mit `nesting=1` und ggf. `keyctl=1`. Es ist möglich, aber in Proxmox-Setups wird empfohlen, Docker-Workloads in KVM-VMs zu betreiben, wenn hohe Sicherheitsanforderungen bestehen.

---

## Snapshots

### VM-Snapshots

```bash
# Snapshot erstellen (bei laufender VM: RAM-State miteinschließen optional)
qm snapshot 100 "before-update" --description "Vor dem System-Update"

# Snapshots anzeigen
qm listsnapshot 100

# Zu Snapshot zurückkehren (VM muss gestoppt sein)
qm rollback 100 "before-update"

# Snapshot löschen
qm delsnapshot 100 "before-update"
```

> [!WARNING] Snapshots sind kein Backup
> Snapshots sind an die Basisdisk gebunden. Bei Datenverlust der Disk sind alle Snapshots ebenfalls verloren. Regelmäßige Backups mit vzdump oder PBS sind zwingend erforderlich.

### Container-Snapshots

```bash
# Container-Snapshot erstellen
pct snapshot 200 "clean-install"

# Snapshots anzeigen
pct listsnapshot 200

# Snapshot zurückrollen
pct rollback 200 "clean-install"

# Snapshot löschen
pct delsnapshot 200 "clean-install"
```

---

## Templates

### VM-Template erstellen

```bash
# VM vorbereiten (im Gast: cloud-init, SSH-Keys löschen, etc.)
# Dann: VM als Template konvertieren (unumkehrbar!)
qm template 100

# Aus Template klonen
qm clone 100 101 --name "neue-vm" --full 1 --storage local-lvm

# Linked Clone (spart Platz, teilt Basisdisk)
qm clone 100 101 --name "neue-vm"
```

### Container-Template erstellen

```bash
# Container stoppen
pct stop 200

# Als Template markieren
pct template 200

# Aus Template klonen
pct clone 200 201 --hostname neue-instanz --full 1
```

---

## Cloud-Init

Cloud-Init automatisiert die Erstkonfiguration von VMs (IP, SSH-Key, Hostname, Pakete):

### Cloud-Init-VM einrichten

```bash
# 1. Cloud-Image herunterladen (z.B. Ubuntu 22.04)
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img \
    -O /var/lib/vz/template/iso/ubuntu-22.04-cloud.img

# 2. VM erstellen
qm create 9000 \
    --name "ubuntu-cloud-template" \
    --memory 2048 \
    --cores 2 \
    --net0 virtio,bridge=vmbr0 \
    --machine q35 \
    --bios ovmf \
    --agent enabled=1 \
    --scsihw virtio-scsi-single

# 3. Cloud-Image als Disk importieren
qm importdisk 9000 /var/lib/vz/template/iso/ubuntu-22.04-cloud.img local-lvm

# 4. Disk anbinden
qm set 9000 --scsi0 local-lvm:vm-9000-disk-0,discard=on

# 5. Cloud-Init-Drive hinzufügen
qm set 9000 --ide2 local-lvm:cloudinit

# 6. Boot-Reihenfolge setzen
qm set 9000 --boot order=scsi0

# 7. Display konfigurieren
qm set 9000 --serial0 socket --vga serial0

# 8. Cloud-Init-Daten setzen
qm set 9000 --ciuser admin
qm set 9000 --sshkey ~/.ssh/id_ed25519.pub
qm set 9000 --ipconfig0 ip=dhcp

# 9. Als Template konvertieren
qm template 9000

# 10. Aus Template klonen
qm clone 9000 101 --name "webserver-01" --full 1
qm set 101 --ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1
qm start 101
```

### Cloud-Init-Werte nach dem Klonen anpassen

```bash
# IP-Adresse setzen
qm set 101 --ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1

# Hostname anpassen (via Cloud-Init ciname oder im Gast)
qm set 101 --name "webserver-01"

# SSH-Key ersetzen
qm set 101 --sshkey /root/.ssh/new_key.pub

# Cloud-Init-Konfiguration anzeigen
qm cloudinit dump 101
```

---

## OCI Container (ab PVE 8.2)

Ab Proxmox VE 8.2 können **OCI-kompatible Container-Images** (z.B. Docker Hub, GHCR, Quay.io) direkt als LXC-Container gestartet werden — ohne separates Docker oder Podman auf dem Host.

> [!NOTE] OCI ≠ Docker-in-LXC
> OCI-Container in PVE laufen als native LXC-Container — der Host-Kernel wird direkt genutzt. Es ist kein Docker-Daemon nötig. Die Images werden über `pveam` oder `pvectl` gepullt und in LXC-kompatible rootfs-Archive konvertiert.

### OCI-Image als LXC-Container starten

```bash
# OCI-Support prüfen (ab PVE 8.2)
pveversion | grep pve-manager

# Verfügbare OCI-Images (Proxmox-Registry)
pveam available --section system

# OCI-Image direkt von Docker Hub pullen und als Template speichern
pveam pull local docker://nginx:alpine \
    --os-type unmanaged

# Container aus OCI-Image erstellen
pct create 300 local:vztmpl/docker-nginx-alpine.tar.zst \
    --hostname nginx-oci \
    --memory 512 \
    --rootfs local-lvm:4 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.300/24,gw=192.168.1.1 \
    --unprivileged 1 \
    --features nesting=1 \
    --ostype unmanaged \
    --start 1
```

### Unterstützte Registries

| Registry | Format | Beispiel |
|---|---|---|
| **Docker Hub** | `docker://image:tag` | `docker://nginx:alpine` |
| **GHCR** | `docker://ghcr.io/...` | `docker://ghcr.io/home-assistant/home-assistant:stable` |
| **Quay.io** | `docker://quay.io/...` | `docker://quay.io/prometheus/prometheus:latest` |
| **Lokale Registry** | `docker://registry.local/...` | `docker://registry.local:5000/myapp:v1.0` |

> [!IMPORTANT] OCI vs. Standard-LXC
> OCI-Container haben keinen systemd/init — sie laufen einen einzelnen Prozess (wie in Docker). Standard-LXC-Templates (Debian, Ubuntu) sind vollwertige OS-Instanzen mit init. Für einfache Dienste (Nginx, Redis, Prometheus) → OCI. Für vollwertige Linux-Umgebungen → LXC-Template.

### Privilegien bei OCI-Containern

```bash
# OCI-Container brauchen oft nesting für interne Prozess-Isolation
pct set 300 --features nesting=1

# Für Container die Capabilities brauchen (z.B. net_admin)
pct set 300 --features nesting=1,keyctl=1

# Rootless-OCI (empfohlen wo möglich) — unprivileged=1 belassen
```

---

## Migration

### Live Migration (VM läuft weiter)

```bash
# VM live auf anderen Node migrieren (Shared Storage erforderlich)
qm migrate 100 pve2 --online

# Migration mit lokalem Storage (Disk wird kopiert)
qm migrate 100 pve2 --online --with-local-disks

# Container live migrieren
pct migrate 200 pve2 --online
```

> [!NOTE] Live Migration Voraussetzungen
> - Beide Nodes im selben Cluster
> - Shared Storage (Ceph, NFS) für Online-Migration ohne Disk-Kopie
> - Kompatible CPUs (gleiche Architektur, oder `--cpu kvm64` in VM)
> - Ausreichende Netzwerkbandbreite

### Offline Migration

```bash
# VM ausschalten und migrieren (kopiert Disks)
qm migrate 100 pve2

# Einzelne Disk manuell migrieren
qm move-disk 100 scsi0 ceph-pool --delete
```

---

## Nützliche Verwaltungsbefehle

```bash
# VM-Ressourcenverbrauch in Echtzeit
qm monitor 100         # QMP Monitor öffnen
# Im Monitor: info status, info pci, info block

# VM-Konsole über CLI
qm terminal 100 --iface serial0   # Serial Console
qm vncproxy 100                    # VNC Proxy

# VM-Konfiguration sichern/exportieren
qm config 100 > vm-100-config.conf

# Alle VM-IDs auflisten
qm list | awk 'NR>1 {print $1}'

# Ressourcenübersicht aller VMs
pvesh get /nodes/pve/qemu --full

# Container-Übersicht
pvesh get /nodes/pve/lxc --full
```

---

## Verwandte Notizen

- [[Was ist Proxmox]]
- [[Proxmox Netzwerk]]
- [[Proxmox Storage]]
- [[Proxmox Cluster und HA]]
- [[Proxmox Backup Server]]
- [[Proxmox CLI Cheatsheet]]
