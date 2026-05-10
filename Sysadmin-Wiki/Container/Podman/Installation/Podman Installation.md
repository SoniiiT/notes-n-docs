---
tags: [podman, installation, setup, rhel, fedora, ubuntu, debian, macos, windows, podman-desktop]
aliases: ["Podman installieren", "Podman Setup"]
---

# Podman Installation

> [!ABSTRACT]
> Podman ist auf RHEL/Fedora vorinstalliert oder per Paketmanager verfügbar. Auf Ubuntu/Debian via apt, auf macOS über Homebrew + Podman Machine (leichte VM), auf Windows via Installer oder WSL2. Podman Desktop bietet eine kostenlose GUI für alle Plattformen.

---

## Überblick: Installationsmethoden

| Plattform | Methode | Podman-Version (2025/2026) |
|-----------|---------|---------------------------|
| RHEL 9/10 | `dnf install podman` | 4.x / 5.x |
| Fedora 40+ | `dnf install podman` | 5.x (vorinstalliert) |
| CentOS Stream 9/10 | `dnf install podman` | 4.x / 5.x |
| Ubuntu 22.04+ | `apt install podman` | 3.x–4.x (kubic für neuer) |
| Debian 12+ | `apt install podman` | 4.x |
| Arch Linux | `pacman -S podman` | Aktuell |
| macOS | `brew install podman` | 5.x via Podman Machine |
| Windows | Installer / WSL2 | 5.x |

---

## RHEL / CentOS Stream / Rocky Linux / AlmaLinux

```bash
# RHEL 9 / CentOS Stream 9
sudo dnf install -y podman

# Optional: podman-compose für Compose-Dateien
sudo dnf install -y podman-compose

# Version prüfen
podman --version
```

> [!NOTE] RHEL-Versionen
> - RHEL 9: Podman 4.x (AppStream-Paket)
> - RHEL 10: Podman 5.x
> - Aktuelle Version via `dnf module list container-tools`

```bash
# Container Tools Modul (bietet neuere Versionen)
sudo dnf module enable container-tools:rhel9
sudo dnf install -y container-tools
```

---

## Fedora

Podman ist seit Fedora 31 vorinstalliert.

```bash
# Aktualisieren auf neueste Version
sudo dnf upgrade -y podman

# Buildah und Skopeo (optional, Teil der Toolbox)
sudo dnf install -y buildah skopeo

# podman-compose
sudo dnf install -y podman-compose
```

---

## Ubuntu

```bash
# Ubuntu 22.04 / 24.04 — offizielle Pakete
sudo apt-get update
sudo apt-get install -y podman

# Für neuere Podman-Versionen: kubic-Repository
. /etc/os-release
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" \
  | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
curl -fsSL "https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/devel-kubic.gpg
sudo apt-get update
sudo apt-get install -y podman
```

> [!TIP] Ubuntu 24.04
> Ubuntu 24.04 LTS enthält Podman 4.x in den offiziellen Repos. Für Podman 5.x das kubic-Repository verwenden.

---

## Debian

```bash
# Debian 12 (Bookworm)
sudo apt-get update
sudo apt-get install -y podman

# Podman-Compose
sudo apt-get install -y podman-compose
```

---

## Arch Linux

```bash
# Podman
sudo pacman -S podman

# Vollständige Toolbox
sudo pacman -S podman buildah skopeo

# AUR: podman-desktop
yay -S podman-desktop
```

---

## macOS — Podman Machine

Podman läuft auf macOS über eine leichte Linux-VM, die `podman machine` verwaltet.

```bash
# Installation via Homebrew
brew install podman

# Podman Machine initialisieren (beim ersten Mal)
podman machine init

# Maschine starten
podman machine start

# Status prüfen
podman machine list

# Beispiel-Ausgabe:
# NAME        VM TYPE     CREATED     LAST UP     CPUS   MEMORY   DISK SIZE
# podman-machine-default  applehv  2 min ago   Currently running  2  2GiB  100GiB
```

### Podman Machine konfigurieren

```bash
# Maschine mit mehr Ressourcen erstellen
podman machine init --cpus 4 --memory 4096 --disk-size 60

# Apple Hypervisor (Apple Silicon, Podman 5.x — schneller)
# Wird automatisch verwendet auf Apple Silicon Macs

# rootful Machine (für Ports < 1024 etc.)
podman machine init --rootful
podman machine set --rootful

# Mehrere Maschinen verwalten
podman machine init dev-machine
podman machine start dev-machine
podman machine stop dev-machine
```

> [!INFO] Apple Hypervisor seit Podman 5.0
> Podman Machine nutzt auf Apple Silicon den nativen Apple Hypervisor statt QEMU. Das bedeutet deutlich schnellere Boot-Zeiten und bessere Performance. Dateifreigabe via virtiofs (schneller als Plan 9).

```bash
# Version und Status
podman info | grep -A3 host
podman machine inspect
```

---

## Windows

### Option A: Podman Desktop (empfohlen)

1. Download von [podman-desktop.io](https://podman-desktop.io)
2. Installer ausführen (`.exe`)
3. Podman Desktop richtet WSL2-Backend automatisch ein
4. CLI (`podman`) steht in PowerShell und CMD zur Verfügung

### Option B: Manuell via WSL2

```powershell
# WSL2 aktivieren
wsl --install -d Ubuntu-24.04

# In Ubuntu (WSL2)
sudo apt-get update
sudo apt-get install -y podman
```

### Option C: Windows Installer (ohne Podman Desktop)

```powershell
# Via winget
winget install RedHat.Podman

# Podman Machine mit WSL2-Backend
podman machine init
podman machine start
```

---

## Podman Desktop

Podman Desktop ist eine kostenlose, Open-Source-GUI für alle Plattformen — die Alternative zu Docker Desktop.

**Features:**
- Container starten, stoppen, inspizieren, Logs lesen
- Images bauen und verwalten (inkl. Buildah-Integration)
- Pods verwalten
- Volume- und Netzwerk-Verwaltung
- Kubernetes-Integration (Kind, Minikube) via Extensions
- Podman Machine-Verwaltung (macOS/Windows)
- Docker Desktop Extension Kompatibilität

```bash
# Installation Linux (Flatpak)
flatpak install flathub io.podman_desktop.PodmanDesktop

# macOS
brew install --cask podman-desktop

# Windows: Installer von podman-desktop.io
```

---

## Erstkonfiguration nach der Installation

```bash
# Version und Info
podman --version
podman info

# Erster Test (rootless)
podman run --rm hello-world

# Storage-Info
podman system info | grep -A8 store

# Registries-Konfiguration anzeigen
cat /etc/containers/registries.conf

# rootless-Info
podman info | grep rootless
```

---

## subuid / subgid für Rootless Container

Rootless Container benötigen User-Namespace-Mappings (`/etc/subuid` und `/etc/subgid`).

```bash
# Prüfen ob Mapping vorhanden
grep $USER /etc/subuid /etc/subgid

# Erwartete Ausgabe:
# /etc/subuid:max:100000:65536
# /etc/subgid:max:100000:65536

# Falls nicht vorhanden: manuell hinzufügen
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER

# Podman über neue Mappings informieren
podman system migrate
```

> [!NOTE] Automatisch bei Useranlage
> Auf modernen Systemen (RHEL 8+, Ubuntu 20.04+) werden subuid/subgid automatisch beim Anlegen von Benutzern mit `useradd` gesetzt.

---

## cgroups v2 prüfen

Rootless Container profitieren von cgroups v2 für vollständige Resource-Limits.

```bash
# cgroups-Version ermitteln
stat -fc %T /sys/fs/cgroup/
# tmpfs = cgroups v1
# cgroup2fs = cgroups v2 ✅

# Auf systemd-Systemen:
systemctl --version | grep default-hierarchy
```

> [!TIP] cgroups v2 aktivieren (falls v1)
> ```bash
> # In GRUB: systemd.unified_cgroup_hierarchy=1 hinzufügen
> sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
> sudo reboot
> ```

---

## fuse-overlayfs (ältere Systeme)

Auf Systemen mit Kernel < 5.11 oder ohne native User-Namespace-Overlayfs-Unterstützung:

```bash
# Fedora / RHEL
sudo dnf install -y fuse-overlayfs

# Ubuntu / Debian
sudo apt-get install -y fuse-overlayfs

# Aktivierung prüfen
podman info | grep graphDriverName
# graphDriverName: overlay ✅
```

---

## lingering aktivieren (Auto-Start nach Reboot)

Damit rootless Container nach einem Systemstart automatisch gestartet werden:

```bash
# Lingering aktivieren (User-Session überlebt Logout)
sudo loginctl enable-linger $USER

# Prüfen
loginctl show-user $USER | grep Linger
# Linger=yes ✅
```

---

## Verwandte Notizen

- [[Was ist Podman]] — Konzept und Grundlagen
- [[Rootless Container]] — Detaillierte Rootless-Konfiguration
- [[Podman CLI Cheatsheet]] — Wichtigste Befehle nach der Installation
- [[Podman Index]] — Übersicht aller Podman-Themen
