---
tags: [lxd, profile, konfiguration, devices, cloud-init, limits]
aliases: ["LXD Profile", "LXD Konfiguration"]
---

# Profile und Konfiguration

> [!ABSTRACT]
> Profile sind wiederverwendbare Konfigurationsvorlagen in LXD. Sie definieren Geräte (Netzwerk, Disk), Ressourcenlimits und andere Einstellungen. Container erben Konfiguration von Profilen — ein mächtiges System für konsistente Umgebungen.

---

## Was sind Profile?

Profile funktionieren wie Vorlagen:

```
Profile "default"        Profile "webserver"      Profile "gpu"
  - eth0: lxdbr0          - limits.cpu: 4          - GPU device
  - root: pool=default    - limits.memory: 4GiB    - nvidia driver

Container "webapp" → [default, webserver]
Container "ml-job" → [default, gpu]
```

- Container können **mehrere Profile** gleichzeitig haben
- Spätere Profile überschreiben frühere (Override-Logik)
- Direkte Container-Konfiguration überschreibt Profile

---

## Profil-Verwaltung

```bash
# Profile auflisten
lxc profile list

# Profil anzeigen
lxc profile show default

# Neues Profil erstellen
lxc profile create webserver

# Profil bearbeiten (öffnet Editor)
lxc profile edit webserver

# Profil kopieren
lxc profile copy default mein-profil

# Profil löschen
lxc profile delete mein-profil

# Container: Profile zuweisen
lxc profile assign mycontainer default,webserver

# Container: Profil hinzufügen
lxc profile add mycontainer webserver

# Container: Profil entfernen
lxc profile remove mycontainer webserver
```

---

## Das Default-Profil

Jeder neue Container erbt das `default`-Profil automatisch:

```yaml
# lxc profile show default
config: {}
description: Default LXD profile
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
```

---

## Ressourcenlimits

### CPU-Limits

```bash
# Anzahl vCPUs begrenzen
lxc config set mycontainer limits.cpu 2

# Spezifische CPU-Kerne pinnen (0 und 1)
lxc config set mycontainer limits.cpu "0,1"

# CPU-Zeit begrenzen (50% von 2 CPUs)
lxc config set mycontainer limits.cpu 2
lxc config set mycontainer limits.cpu.allowance 50%

# Im Profil setzen
lxc profile set webserver limits.cpu 4
```

### Memory-Limits

```bash
# Festes Memory-Limit
lxc config set mycontainer limits.memory 2GiB

# Soft-Limit (Empfehlung, kein Hard-Cap)
lxc config set mycontainer limits.memory.soft 1GiB

# Swap deaktivieren
lxc config set mycontainer limits.memory.swap false

# Einheiten: kB, MB, GB, KiB, MiB, GiB
```

### Disk-Limits

```bash
# Root-Filesystem begrenzen
lxc config device set mycontainer root size 10GiB

# I/O-Limits
lxc config set mycontainer limits.disk.priority 5

# Read/Write-Limits für Block-Device
lxc config device set mycontainer root limits.read 100MB
lxc config device set mycontainer root limits.write 50MB
```

### Netzwerk-Limits

```bash
# Netzwerk-Bandbreite begrenzen
lxc config device set mycontainer eth0 limits.ingress 100Mbit
lxc config device set mycontainer eth0 limits.egress 100Mbit
```

---

## Devices

Devices definieren, welche Hardware-Ressourcen einem Container zur Verfügung stehen.

### Netzwerk-Devices

```bash
# Bridge-NIC hinzufügen
lxc config device add mycontainer eth0 nic \
    nictype=bridged \
    parent=lxdbr0

# Statische IP vergeben
lxc config device set mycontainer eth0 ipv4.address 10.10.10.50

# macvlan NIC (Container direkt im physischen Netz)
lxc config device add mycontainer eth1 nic \
    nictype=macvlan \
    parent=eth0

# Port-Forwarding (Proxy Device)
lxc config device add mycontainer http-proxy proxy \
    listen=tcp:0.0.0.0:8080 \
    connect=tcp:127.0.0.1:80
```

### Disk-Devices

```bash
# Verzeichnis vom Host einhängen
lxc config device add mycontainer mydata disk \
    source=/mnt/shared-data \
    path=/data

# Nur-Lesen einhängen
lxc config device add mycontainer config-files disk \
    source=/etc/myapp-config \
    path=/etc/myapp \
    readonly=true

# Block-Device direkt weitergeben
lxc config device add mycontainer sdb disk \
    source=/dev/sdb
```

### GPU-Devices

```bash
# Alle GPUs weitergeben
lxc config device add mycontainer mygpu gpu

# Spezifische GPU (PCI-Adresse)
lxc config device add mycontainer mygpu gpu \
    pci=0000:01:00.0

# NVIDIA GPU mit GPU-ID
lxc config device add mycontainer gpu0 gpu \
    id=0

# Status prüfen
lxc exec mycontainer -- nvidia-smi
```

### USB-Devices

```bash
# Bestimmtes USB-Gerät weitergeben
lxc config device add mycontainer myusb usb \
    vendorid=0x1234 \
    productid=0x5678
```

---

## Profil-Beispiele

### Webserver-Profil

```yaml
# lxc profile edit webserver
config:
  limits.cpu: "2"
  limits.memory: 2GiB
  limits.memory.swap: "false"
description: Webserver profile
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    size: 20GiB
    type: disk
  http-proxy:
    connect: tcp:127.0.0.1:80
    listen: tcp:0.0.0.0:80
    type: proxy
  https-proxy:
    connect: tcp:127.0.0.1:443
    listen: tcp:0.0.0.0:443
    type: proxy
name: webserver
```

### Entwicklungs-Profil

```yaml
# lxc profile edit developer
config:
  limits.cpu: "4"
  limits.memory: 8GiB
  security.nesting: "true"
description: Developer workstation profile
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    size: 50GiB
    type: disk
  projects:
    path: /home/dev/projects
    source: /home/myuser/projects
    type: disk
name: developer
```

> [!NOTE]
> `security.nesting: true` erlaubt es, Docker oder weitere LXD-Instanzen innerhalb des Containers zu betreiben (Container-in-Container).

---

## Cloud-Init Integration

LXD unterstützt cloud-init für automatische Konfiguration beim ersten Start:

```bash
# cloud-init beim Launch übergeben
lxc launch ubuntu:24.04 myserver \
    -c user.user-data="$(cat cloud-init.yaml)"
```

```yaml
# cloud-init.yaml
#cloud-config
hostname: myserver
users:
  - name: admin
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3Nza... mein-key
packages:
  - nginx
  - postgresql
  - htop
runcmd:
  - systemctl enable nginx
  - systemctl start nginx
```

```bash
# cloud-init in Profil aufnehmen
lxc profile set default user.user-data "$(cat cloud-init.yaml)"

# cloud-init Log im Container prüfen
lxc exec myserver -- cloud-init status --wait
lxc exec myserver -- cat /var/log/cloud-init.log
```

---

## Container-Konfiguration direkt

Neben Profilen kann jeder Container direkt konfiguriert werden:

```bash
# Alle Konfigurationsoptionen anzeigen
lxc config show mycontainer

# Erweiterte Konfiguration (inkl. Profil-Werte)
lxc config show mycontainer --expanded

# Einzelnen Wert setzen
lxc config set mycontainer limits.cpu 4

# Einzelnen Wert lesen
lxc config get mycontainer limits.cpu

# Konfiguration als YAML bearbeiten
lxc config edit mycontainer

# Wert zurücksetzen (erbt dann vom Profil)
lxc config unset mycontainer limits.cpu
```

---

## Wichtige Konfigurationsschlüssel

| Schlüssel | Beschreibung | Beispiel |
|-----------|-------------|---------|
| `limits.cpu` | CPU-Kerne | `2` oder `"0,1"` |
| `limits.memory` | RAM-Limit | `4GiB` |
| `limits.memory.swap` | Swap erlaubt | `false` |
| `security.privileged` | Privilegierter Container | `false` |
| `security.nesting` | Container-in-Container | `true` |
| `security.idmap.isolated` | Isoliertes UID-Mapping | `true` |
| `boot.autostart` | Auto-Start beim Host-Boot | `true` |
| `boot.autostart.priority` | Start-Reihenfolge | `0` (höher = früher) |
| `environment.MY_VAR` | Umgebungsvariable setzen | `myvalue` |
| `user.user-data` | cloud-init User-Data | YAML-String |

---

## Verwandte Notizen

- [[LXD Installation und Setup]] — Initiales Setup
- [[LXD Netzwerk]] — Netzwerk-Devices im Detail
- [[Storage Pools]] — Disk-Devices und Storage
- [[LXD Sicherheit]] — security.*-Schlüssel
