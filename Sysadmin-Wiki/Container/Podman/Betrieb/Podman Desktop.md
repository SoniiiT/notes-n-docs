---
tags: [podman, podman-desktop, gui, desktop, grafisch, extensions, kubernetes, windows, macos]
aliases: ["Podman Desktop", "Podman GUI", "Podman Desktop App"]
---

# Podman Desktop

> [!ABSTRACT]
> **Podman Desktop** ist eine kostenlose, Open-Source GUI-Anwendung für Container-Management auf Windows, macOS und Linux. Es ist die empfohlene grafische Alternative zu Docker Desktop — vollständig kostenlos (auch für Unternehmen), unterstützt Podman, Docker und Kubernetes in einem Interface.

---

## Was ist Podman Desktop?

Podman Desktop bietet eine grafische Oberfläche für:
- Container und Images verwalten
- Pods erstellen und überwachen
- Volumes und Netzwerke konfigurieren
- Kubernetes via Kind oder Minikube
- Extensions (Kompose, Lima, Dev Containers, ...)

**Kostenlos und Open Source** — kein Lizenzmodell, keine kommerziellen Einschränkungen.

---

## Installation

### Windows

```powershell
# Winget (empfohlen)
winget install RedHat.PodmanDesktop

# Oder: Installer von podman-desktop.io herunterladen
# https://podman-desktop.io/downloads
```

### macOS

```bash
# Homebrew (empfohlen)
brew install podman-desktop

# Oder: DMG-Datei von podman-desktop.io
```

### Linux

```bash
# Flatpak (empfohlen für Linux)
flatpak install flathub io.podman_desktop.PodmanDesktop

# AppImage
# Herunterladen von: https://podman-desktop.io/downloads
chmod +x podman-desktop-*.AppImage
./podman-desktop-*.AppImage

# Fedora/RHEL via DNF
sudo dnf install -y podman-desktop
```

---

## Erste Schritte

### Podman-Engine einrichten (Windows/macOS)

Auf Windows und macOS benötigt Podman eine Linux-VM:

```
1. Podman Desktop öffnen
2. "Onboarding" folgen
3. Podman Machine erstellen:
   - Name: podman-machine-default
   - CPUs: 2 (oder mehr)
   - Memory: 4 GB (oder mehr)
   - Disk: 20 GB
4. Machine starten (läuft im Hintergrund)
```

```bash
# Alternativ via CLI
podman machine init --cpus 2 --memory 4096 --disk-size 20
podman machine start

# Status
podman machine list
podman machine ssh   # SSH in die VM
```

---

## Benutzeroberfläche

### Hauptbereiche

| Bereich | Funktion |
|---------|----------|
| **Containers** | Alle Container anzeigen, starten, stoppen, Logs |
| **Pods** | Pods verwalten (Podman-spezifisch) |
| **Images** | Images pullen, bauen, löschen |
| **Volumes** | Volume-Verwaltung |
| **Kubernetes** | Cluster-Verwaltung (Kind, Minikube) |
| **Settings** | Registries, Proxies, Extensions |

### Container-Management

In der Container-Ansicht:
- **Status-Badge**: Grün = running, Grau = stopped
- **Quick-Actions**: Start, Stop, Restart, Delete
- **Logs**: Direkte Log-Ansicht im Browser-Style
- **Terminal**: interaktiver Shell-Zugriff
- **Inspect**: JSON-Konfiguration des Containers

---

## Images bauen

```
1. Images → Build
2. Containerfile auswählen (Browse...)
3. Build-Context setzen (Verzeichnis)
4. Image-Name und Tag vergeben
5. Build starten
6. Build-Logs in Echtzeit
```

---

## Podman Desktop Extensions

Extensions erweitern die Funktionalität erheblich:

### Wichtige Extensions

| Extension | Funktion |
|-----------|----------|
| **Compose** | Docker Compose / podman-compose UI |
| **Kind** | Lokale Kubernetes-Cluster |
| **Minikube** | Minikube-Cluster-Management |
| **Lima** | Lima VM-Management (macOS) |
| **Bootc** | Bootable Container Images |
| **Dev Environments** | Dev Container Support |
| **OpenShift** | OpenShift Local (CRC) |
| **Headlamp** | Kubernetes Dashboard |

```
Extensions installieren:
Settings → Extensions → Extension Catalog → Installieren
```

---

## Kubernetes-Integration

### Kind-Cluster erstellen

```
1. Kind Extension installieren
2. Kubernetes → Create new cluster
3. Provider: Kind
4. Cluster-Name vergeben
5. Erstellen
6. Cluster erscheint in Kubernetes-Ansicht
```

### Kubernetes-Ressourcen verwalten

```
Kubernetes-Ansicht zeigt:
- Pods
- Deployments
- Services
- ConfigMaps, Secrets
- Namespaces
```

### `podman play kube` in Podman Desktop

```
Images → Kubernetes YAML → Deploy
```

---

## Docker-Kompatibilität

Podman Desktop kann auch **Docker** als Engine nutzen:

```
Settings → Docker Desktop Extension aktivieren
oder:
Settings → Resources → Docker Engine hinzufügen
```

```bash
# Docker Socket in Podman Desktop verfügbar
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
```

---

## Podman Machine (Windows/macOS Details)

Die Podman Machine ist eine QEMU/HyperV/Hyperkit-VM:

```bash
# Machine-Management via CLI
podman machine list
podman machine start
podman machine stop
podman machine ssh
podman machine info

# Machine anpassen
podman machine set --cpus 4 --memory 8192

# Rootful-Mode (für Docker-Kompatibilität)
podman machine set --rootful
podman machine restart
```

> [!NOTE] Windows: WSL2-Backend
> Podman Desktop nutzt auf Windows standardmäßig WSL2 als Backend. QEMU ist ebenfalls verfügbar.

---

## Registries konfigurieren

```
Settings → Registries → Add Registry

Felder:
- Server URL: docker.io / quay.io / ghcr.io / ...
- Username
- Password / Token
```

---

## Vergleich: Podman Desktop vs. Docker Desktop

| Feature | Podman Desktop | Docker Desktop |
|---------|:--------------:|:--------------:|
| Kosten | **Kostenlos** | ❌ kostenpflichtig für Unternehmen |
| Open Source | ✅ | ❌ |
| Rootless | ✅ | ❌ |
| Pods | ✅ | ❌ |
| Kubernetes | ✅ (Kind/Minikube) | ✅ (eingebaut) |
| Extensions | ✅ | ✅ |
| Docker-Kompatibilität | ✅ (Socket) | ✅ nativ |
| Dev Environments | ✅ (Extension) | ✅ |
| Windows WSL2 | ✅ | ✅ |
| macOS VM | ✅ (QEMU) | ✅ (VirtIO) |

> [!TIP]
> Für Unternehmen mit mehr als 250 Mitarbeitern oder >10M USD Umsatz ist Docker Desktop kostenpflichtig (~$21/Monat/User). Podman Desktop ist für alle kostenlos.

---

## Verwandte Notizen

- [[Was ist Podman]] — Podman-Grundlagen
- [[Podman Installation]] — Podman ohne GUI installieren
- [[Rootless Container]] — Rootless-Konzept
- [[Podman vs Docker]] — Ausführlicher Vergleich
