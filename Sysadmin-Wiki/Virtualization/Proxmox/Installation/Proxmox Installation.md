---
tags:
  - virtualization
  - proxmox
  - installation
  - setup
  - debian
aliases:
  - Proxmox installieren
  - PVE Installation
  - Proxmox Setup
---

# Proxmox VE Installation

> [!ABSTRACT] TL;DR
> Proxmox VE 8.x lässt sich über einen bootfähigen USB-Stick installieren. Der Installer führt durch alle Schritte — Disk-Auswahl, Netzwerk, Root-Passwort. Nach dem Neustart sind Post-Install-Schritte nötig: Community-Repo einrichten, Enterprise-Repo deaktivieren (ohne Subscription), Updates einspielen und optional den Subscription-Hinweis entfernen.

---

## Voraussetzungen

### Hardware-Anforderungen

| Komponente | Minimum | Empfohlen |
|---|---|---|
| **CPU** | 64-Bit mit Intel VT-x oder AMD-V | AMD EPYC / Intel Xeon |
| **RAM** | 4 GB | 32 GB+ |
| **System-Disk** | 32 GB (SSD empfohlen) | 120 GB NVMe SSD |
| **VM-Storage** | Separate Disk | NVMe SSD oder HDD RAID |
| **Netzwerk** | 1x 1 GbE NIC | 2x 10 GbE (Management + Storage) |
| **BIOS/UEFI** | VT-x / AMD-V aktiviert | VT-d / AMD-Vi (für PCI-Passthrough) |

### BIOS/UEFI-Einstellungen prüfen

Vor der Installation im BIOS aktivieren:

| Einstellung | Intel | AMD |
|---|---|---|
| CPU-Virtualisierung | `Intel Virtualization Technology (VT-x)` | `AMD-V / SVM Mode` |
| IOMMU (für Passthrough) | `Intel VT-d` | `AMD-Vi / IOMMU` |
| Secure Boot | **Deaktivieren** | **Deaktivieren** |
| Boot-Reihenfolge | USB-Stick an erster Stelle | USB-Stick an erster Stelle |

> [!WARNING] Secure Boot deaktivieren
> Proxmox VE benötigt unsigned Kernel-Module (z.B. ZFS). Secure Boot muss im BIOS **deaktiviert** sein, sonst bootet das System nach der Installation nicht.

---

## Installation

### Schritt 1: ISO herunterladen und bootfähigen USB erstellen

```bash
# ISO von der offiziellen Seite herunterladen
# https://www.proxmox.com/en/downloads/proxmox-virtual-environment

# Bootfähigen USB erstellen unter Linux (sdb = USB-Laufwerk, ACHTUNG: Zieldisk korrekt prüfen!)
dd if=proxmox-ve_8.x-1.iso of=/dev/sdb bs=4M status=progress conv=fsync

# Unter Windows: Rufus oder balenaEtcher verwenden (im DD-Modus)
```

> [!DANGER] Richtige Zieldisk wählen
> `dd` überschreibt ohne Rückfrage. Mit `lsblk` oder `fdisk -l` die korrekte Disk-ID des USB-Sticks prüfen, bevor der Befehl ausgeführt wird.

### Schritt 2: Guided Installer durchlaufen

1. Von USB booten → Menü erscheint → **`Install Proxmox VE (Terminal UI)`** wählen
2. **EULA** akzeptieren
3. **Zieldisk** auswählen:
   - `Options` öffnen für erweiterte Einstellungen (Filesystem: `ext4`, `xfs`, `zfs (RAID0/1/10)`)
   - Für ZFS-Root: RAID-Level und Disks auswählen
4. **Ländereinstellungen**: Country → Germany, Timezone, Keyboard Layout
5. **Root-Passwort** und **E-Mail-Adresse** vergeben
6. **Netzwerk konfigurieren**:
   - Management Interface auswählen (NIC für Web-UI-Zugang)
   - Hostname: `pve.domain.local` (FQDN empfohlen)
   - IP-Adresse, Subnetz, Gateway, DNS
7. **Zusammenfassung prüfen** → `Install`
8. Nach Installation: USB abziehen, Neustart

> [!TIP] ZFS als Root-Filesystem
> ZFS als Boot-Filesystem empfohlen, wenn ≥ 2 System-Disks vorhanden. ZFS bietet RAID, Snapshots und automatische Selbstheilung. Bei einer einzigen SSD: `ext4` oder `xfs` verwenden.

### Schritt 3: Erste Anmeldung

```
Web-UI:  https://<IP-Adresse>:8006
Benutzer: root
Passwort: (beim Installer vergeben)
Realm:    Linux PAM standard authentication
```

> [!NOTE] SSL-Zertifikat
> Proxmox verwendet beim ersten Start ein selbst-signiertes Zertifikat. Der Browser zeigt eine Warnung — dies ist normal. Mit Let's Encrypt oder einem eigenen CA-Zertifikat kann das geändert werden.

---

## Post-Install-Konfiguration

### 1. Subscription-Repository anpassen (ohne Enterprise-Lizenz)

Standardmäßig ist das Enterprise-Repository aktiviert, das eine gültige Subscription erfordert. Für produktive Setups ohne Subscription das Community-Repo aktivieren:

```bash
# Enterprise-Repo deaktivieren
echo "# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" \
    > /etc/apt/sources.list.d/pve-enterprise.list

# Community No-Subscription-Repo hinzufügen
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
    > /etc/apt/sources.list.d/pve-no-subscription.list

# Ceph No-Subscription-Repo (falls Ceph verwendet wird)
echo "deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription" \
    > /etc/apt/sources.list.d/ceph.list

# Paketlisten aktualisieren
apt update
```

### 2. System updaten

```bash
# Alle Pakete auf den neuesten Stand bringen
apt update && apt full-upgrade -y

# Neustart nach Kernel-Update
reboot
```

### 3. Subscription-Hinweis entfernen (optional, nur Homelab)

Der „No valid subscription"-Popup beim Login kann entfernt werden:

```bash
# PVE-Manager JS-Datei patchen (nach jedem PVE-Update erneut nötig)
sed -i.bak "s/data.status !== 'Active'/false/g" \
    /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

# pveproxy neu starten
systemctl restart pveproxy
```

> [!WARNING] Bei PVE-Updates zurückgesetzt
> Der Patch wird bei jedem `pve-manager`-Update überschrieben. Für Produktionsumgebungen: Subscription kaufen statt patchen.

### 4. SSH-Zugang absichern

```bash
# SSH-Konfiguration öffnen
nano /etc/ssh/sshd_config

# Empfohlene Einstellungen:
# PermitRootLogin prohibit-password   (nur SSH-Key, kein Passwort-Login für root)
# PasswordAuthentication no           (nach SSH-Key-Setup)
# Port 22                             (optionaler Custom-Port)

# SSH neu starten
systemctl restart sshd

# SSH-Key für Root hinterlegen (vom Admin-Rechner ausführen)
ssh-copy-id root@<proxmox-ip>
```

### 5. Netzwerk-Bridges einrichten

Nach der Installation ist standardmäßig eine Bridge `vmbr0` an der Management-NIC vorhanden. Weitere Bridges nach Bedarf konfigurieren:

```bash
# Netzwerkkonfiguration öffnen
nano /etc/network/interfaces

# Konfiguration anwenden (ohne Neustart)
ifreload -a
```

Beispielkonfiguration `/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

# Physische NICs
auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

# Management Bridge (mit fester IP)
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    dns-nameservers 192.168.1.1

# VM-Bridge (kein IP-Assignment auf Host)
auto vmbr1
iface vmbr1 inet manual
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0
```

Details → [[Proxmox Netzwerk]]

### 6. Storage konfigurieren

```bash
# Verfügbare Disks anzeigen
lsblk

# PVE Storage-Übersicht
pvesm status

# ZFS-Pool erstellen (Beispiel: Mirror aus /dev/sdb und /dev/sdc)
zpool create -f datastore mirror /dev/sdb /dev/sdc

# ZFS-Pool als Storage in PVE registrieren (Web-UI: Datacenter → Storage → Add → ZFS)
# Oder per CLI:
pvesm add zfspool datastore --pool datastore --content images,rootdir
```

Details → [[Proxmox Storage]]

### 7. Zeitserver konfigurieren

```bash
# Aktuell konfigurierte NTP-Quellen anzeigen
timedatectl status

# NTP-Server setzen (in /etc/systemd/timesyncd.conf)
nano /etc/systemd/timesyncd.conf
# NTP=ntp.example.local 0.de.pool.ntp.org 1.de.pool.ntp.org

# Dienst neu starten
systemctl restart systemd-timesyncd

# Synchronisation prüfen
timedatectl show-timesync --all
```

---

## Erste VM / Container erstellen

### KVM-VM über Web-UI

1. **`Create VM`** klicken (oben rechts)
2. **General**: Node, VM-ID (z.B. 100), Name
3. **OS**: ISO hochladen oder vorhandenes ISO wählen
4. **System**: BIOS (`SeaBIOS` oder `OVMF` für UEFI), SCSI Controller (`VirtIO SCSI single` empfohlen)
5. **Disk**: Storage auswählen, Größe, Bus (`VirtIO Block` für beste Performance)
6. **CPU**: Sockets, Cores, Type (`host` für beste Performance, `kvm64` für Migration)
7. **Memory**: RAM-Menge (Ballooning für Dynamic Memory)
8. **Network**: Bridge, Model (`VirtIO` empfohlen), VLAN-Tag (optional)
9. **Confirm** → `Finish`
10. VM starten und ISO-Installation durchführen

### LXC-Container über Web-UI

1. **`Create CT`** klicken
2. **General**: Node, CT-ID (z.B. 200), Hostname, Passwort / SSH-Key
3. **Template**: Container-Template aus Proxmox-Repository wählen (vorher: `pveam update`)
4. **Root Disk**: Storage, Größe
5. **CPU**: Cores
6. **Memory**: RAM + Swap
7. **Network**: Bridge, IP-Adresse (Static oder DHCP), VLAN
8. **DNS**: Nach Host-Einstellungen oder custom
9. **Confirm** → `Finish`

```bash
# Templates herunterladen (CLI)
pveam update
pveam available --section system | grep -i debian
pveam download local debian-12-standard_12.7-1_amd64.tar.zst

# Container per CLI erstellen
pct create 200 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
    --hostname webserver \
    --memory 2048 \
    --rootfs local-lvm:20 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.200/24,gw=192.168.1.1 \
    --password MySecurePass \
    --start 1
```

---

## Nützliche Erste Schritte nach der Installation

```bash
# System-Übersicht
pveversion -v           # PVE-Version anzeigen
pvesh get /nodes        # Nodes im Cluster
pvesh get /storage      # Konfigurierte Storage-Backends

# Alle VMs anzeigen
qm list

# Alle Container anzeigen
pct list

# Ressourcenauslastung
pvesh get /nodes/pve/status

# Logs des PVE-Services
journalctl -u pveproxy -f        # Web-UI Proxy
journalctl -u pvedaemon -f       # PVE Daemon
journalctl -u pve-cluster -f     # Cluster-Service
```

---

## Verwandte Notizen

- [[Was ist Proxmox]]
- [[Proxmox Netzwerk]]
- [[Proxmox Storage]]
- [[Proxmox VM und Container Verwaltung]]
- [[Proxmox Troubleshooting]]
