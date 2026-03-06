---
tags: [lxd, images, remote, ubuntu, container-images]
aliases: ["LXD Images", "LXD Image-Verwaltung"]
---

# Images und Remotes

> [!ABSTRACT]
> LXD verwendet Images als Basis für Container und VMs. Images werden von Remote-Servern heruntergeladen, lokal gecacht und können selbst erstellt werden. Remotes sind konfigurierbare Image-Quellen und Remote-LXD-Server.

---

## Konzept: Images in LXD

Images in LXD sind:
- **Read-only Basis-Dateisysteme** für Container/VMs
- **Gecacht** lokal nach dem ersten Download
- **Fingerprint-verifiziert** (SHA-256)
- **Automatisch aktualisiert** (optional, täglich)

```
Remote-Server (images:, ubuntu:)
        │
        ▼ download + cache
Lokaler Image-Cache
        │
        ▼ kopieren
Container-Rootfs (eigenes Volume)
```

---

## Remote-Server

Remotes sind konfigurierte Image-Quellen und Remote-LXD-Server:

```bash
# Alle konfigurierten Remotes anzeigen
lxc remote list
```

| Remote | URL | Beschreibung |
|--------|-----|-------------|
| `ubuntu:` | cloud-images.ubuntu.com | Ubuntu-Images (offiziell, für LXD) |
| `images:` | images.linuxcontainers.org | Community-Images (für LXC/Incus) |
| `local:` | unix:// | Lokaler LXD-Daemon |

> [!NOTE]
> **LXD (Canonical)** nutzt primär `ubuntu:` — dort sind Ubuntu-Images optimiert.
> **Incus** nutzt `images:` (linuxcontainers.org) mit Unterstützung für 50+ Distributionen.

---

## Images suchen und herunterladen

```bash
# Ubuntu-Images suchen
lxc image list ubuntu: | grep 24.04

# Community-Images suchen
lxc image list images: ubuntu

# Alle verfügbaren Aliasnamen für Ubuntu 24.04
lxc image alias list ubuntu: | grep 24.04

# Debian-Images
lxc image list images: debian

# Alpine-Images
lxc image list images: alpine

# Arch Linux
lxc image list images: archlinux
```

### Häufig verwendete Image-Aliase

```bash
# Ubuntu (LXD)
lxc launch ubuntu:24.04 ct          # Ubuntu 24.04 LTS
lxc launch ubuntu:22.04 ct          # Ubuntu 22.04 LTS
lxc launch ubuntu:noble ct          # Ubuntu 24.04 (Codename)

# Community-Images (images:)
lxc launch images:debian/12 ct      # Debian 12 Bookworm
lxc launch images:alpine/3.20 ct    # Alpine Linux 3.20
lxc launch images:centos/9-Stream ct # CentOS Stream 9
lxc launch images:fedora/41 ct      # Fedora 41
lxc launch images:archlinux ct      # Arch Linux
lxc launch images:opensuse/15.6 ct  # openSUSE 15.6

# VMs (--vm Flag)
lxc launch ubuntu:24.04 myvm --vm
```

---

## Lokale Image-Verwaltung

```bash
# Lokal gecachte Images anzeigen
lxc image list local:

# Image-Details anzeigen
lxc image info ubuntu:24.04

# Image manuell herunterladen (ohne Container zu starten)
lxc image copy ubuntu:24.04 local: --alias my-ubuntu-24

# Image löschen
lxc image delete local:my-ubuntu-24

# Image als Datei exportieren
lxc image export ubuntu:24.04 ./ubuntu-24.04-export/

# Image importieren
lxc image import ./ubuntu-24.04-export/lxd.tar.xz \
    ./ubuntu-24.04-export/rootfs.squashfs \
    --alias my-custom-image
```

---

## Eigenes Image erstellen

### Methode 1: Container zu Image machen

```bash
# Container vorbereiten
lxc launch ubuntu:24.04 template-container
lxc exec template-container -- apt update
lxc exec template-container -- apt install -y nginx python3 vim
lxc exec template-container -- apt clean

# Container stoppen
lxc stop template-container

# Als Image veröffentlichen
lxc publish template-container \
    --alias myapp-base \
    description="My app base image with nginx"

# Image anzeigen
lxc image list | grep myapp-base

# Container löschen (Image bleibt)
lxc delete template-container
```

### Methode 2: Snapshot zu Image

```bash
# Snapshot erstellen
lxc snapshot template-container snap-v1

# Snapshot als Image veröffentlichen
lxc publish template-container/snap-v1 \
    --alias myapp-v1.0 \
    description="Version 1.0"
```

### Methode 3: Distrobuilder

Für vollständige Custom-Images:

```bash
# Distrobuilder installieren
sudo snap install distrobuilder --classic

# YAML-Definition
cat > ubuntu-custom.yaml << 'EOF'
image:
  distribution: ubuntu
  release: noble
  description: Custom Ubuntu 24.04

source:
  downloader: debootstrap
  url: http://archive.ubuntu.com/ubuntu
  suite: noble

packages:
  manager: apt
  update: true
  cleanup: true
  sets:
    - packages:
        - cloud-init
        - openssh-server
        - nginx
      action: install

actions:
  - trigger: post-packages
    action: |-
      #!/bin/sh
      systemctl enable nginx
      systemctl enable ssh
EOF

# Image bauen
sudo distrobuilder build-lxd ubuntu-custom.yaml
```

---

## Image-Caching und Auto-Update

```bash
# Auto-Update für gecachte Images aktivieren
lxc config set core.images_auto_update true

# Update-Intervall (Standard: 10 Stunden)
lxc config set core.images_auto_update_interval 24

# Images manuell aktualisieren
lxc image refresh

# Image als nicht-auto-update markieren
lxc image edit local:my-custom-image
# → auto_update: false
```

---

## Remote LXD-Server

Neben Image-Servern können auch Remote-LXD-Instanzen als Remotes konfiguriert werden:

```bash
# Eigenen LXD-Server als Remote hinzufügen
# Zuerst: Remote-Server auf HTTPS-Zugriff vorbereiten
lxc config set core.https_address :8443
lxc config set core.trust_password geheimespasswort  # LXD 5.x

# Auf dem Client: Remote hinzufügen
lxc remote add myserver https://192.168.1.100:8443

# LXD 6.x: Token-basierte Authentifizierung
# Auf dem Server:
lxc config trust add --name myclient
# Token kopieren und auf Client eingeben:
lxc remote add myserver https://192.168.1.100:8443

# Remote-Container starten
lxc launch ubuntu:24.04 myserver:mycontainer

# Auf Remote-Container zugreifen
lxc exec myserver:mycontainer -- bash

# Remote entfernen
lxc remote remove myserver
```

---

## Image-Metadaten

Images haben Metadaten für Konfiguration beim Start:

```bash
# Image-Metadaten anzeigen
lxc image show ubuntu:24.04

# Properties eines Images setzen
lxc image set ubuntu:24.04 \
    description="Custom Ubuntu 24.04" \
    os="Ubuntu" \
    release="Noble"
```

---

## Verwandte Notizen

- [[LXD Installation und Setup]] — Remotes beim Init
- [[LXC-LXD Architektur]] — Image-Manager-Subsystem
- [[LXD CLI Cheatsheet]] — Image-Befehle
