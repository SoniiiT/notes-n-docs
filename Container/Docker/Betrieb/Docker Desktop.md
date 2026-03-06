---
tags: [docker, docker-desktop, gui, desktop, grafisch, extensions, dev-environments, windows, macos, lizenz]
aliases: ["Docker Desktop", "Docker GUI", "Docker Desktop App"]
---

# Docker Desktop

> [!ABSTRACT]
> **Docker Desktop** ist die offizielle GUI-Anwendung von Docker für Windows und macOS. Sie bündelt Docker Engine, Docker Compose, Docker CLI, Kubernetes und einen grafischen Manager in einer Anwendung. Seit 2022 ist Docker Desktop für Unternehmen mit mehr als 250 Mitarbeitern **kostenpflichtig**.

---

## Was ist Docker Desktop?

Docker Desktop bietet:
- Docker Engine (Linux-VM auf Windows/macOS)
- Docker Compose (integriert)
- Kubernetes (eingebaut, ein Klick)
- Grafisches Dashboard für Container und Images
- Dev Environments (reproduzierbare Entwicklungsumgebungen)
- Extensions-Marketplace

> [!IMPORTANT] Lizenz-Änderung seit August 2022
> Docker Desktop ist **nicht mehr kostenlos** für:
> - Unternehmen mit mehr als 250 Mitarbeitern
> - Unternehmen mit mehr als 10 Millionen USD Jahresumsatz
>
> Für diese gilt: Subscription erforderlich (Pro: ~$9/Monat, Team: ~$15/Monat, Business: ~$21/Monat).
> **Kostenlose Alternative:** [[Podman Desktop]] — vollständig kostenlos, auch für Unternehmen.

---

## Installation

### Windows

```powershell
# Winget
winget install Docker.DockerDesktop

# Oder: Installer herunterladen
# https://www.docker.com/products/docker-desktop/
```

**Voraussetzungen Windows:**
- Windows 10 Version 1903+ oder Windows 11
- WSL2 aktiviert (empfohlen) oder Hyper-V
- Virtualisierung im BIOS aktiviert

```powershell
# WSL2 installieren (PowerShell als Admin)
wsl --install
wsl --set-default-version 2
```

### macOS

```bash
# Homebrew
brew install --cask docker

# Oder: DMG von docker.com
```

**Voraussetzungen macOS:**
- macOS 12.0 (Monterey) oder neuer
- Chip: Intel oder Apple Silicon (M1/M2/M3)

### Linux

> [!NOTE] Docker Desktop auf Linux
> Docker Desktop ist auf Linux verfügbar (Deb/RPM), aber unüblich. Auf Linux wird in der Regel direkt Docker Engine ohne GUI verwendet.

```bash
# Ubuntu (DEB-Paket)
# Herunterladen von: https://docs.docker.com/desktop/install/linux-install/
sudo dpkg -i docker-desktop-*.deb

# Fedora (RPM)
sudo rpm -i docker-desktop-*.rpm
```

---

## Erste Schritte

### Docker Desktop starten

```
1. Docker Desktop öffnen
2. Docker Engine startet automatisch (ca. 30 Sek.)
3. Statusleiste: "Docker Desktop is running"
4. Tray-Icon erscheint (Systemleiste)
```

```bash
# Prüfen ob Docker läuft
docker version
docker info
```

---

## Benutzeroberfläche

### Hauptbereiche

| Bereich | Funktion |
|---------|----------|
| **Containers** | Alle Container, Compose-Stacks |
| **Images** | Lokale Images, Hub-Suche |
| **Volumes** | Volume-Verwaltung |
| **Dev Environments** | Reproduzierbare Dev-Umgebungen |
| **Extensions** | Extension Marketplace |
| **Settings** | Engine, Resources, Docker Hub |

### Container-Manager

- **Status**: Visuell mit Farb-Indikatoren
- **Quick-Actions**: Start, Stop, Restart, Delete, Open in Terminal
- **Logs**: Echtzeit-Log-Ansicht
- **Port-Links**: Klickbare Links für exponierte Ports
- **Compose**: Stack-Ansicht für docker-compose Projekte

---

## Kubernetes (eingebaut)

Docker Desktop hat Kubernetes out-of-the-box:

```
Settings → Kubernetes → Enable Kubernetes → Apply & Restart
```

```bash
# Nach Aktivierung
kubectl version
kubectl get nodes
# NAME             STATUS   ROLES
# docker-desktop   Ready    control-plane

# Context wechseln
kubectl config use-context docker-desktop
```

> [!NOTE] Single-Node Cluster
> Der eingebaute Kubernetes-Cluster ist ein einzelner Node — gut für lokales Testing, nicht für Multi-Node-Szenarien. Für mehr: Kind oder Minikube nutzen.

---

## Dev Environments

Dev Environments erstellen reproduzierbare Entwicklungsumgebungen:

```
Dev Environments → Create → From Git Repository
URL: https://github.com/username/projekt
```

Docker Desktop klont das Repo, analysiert die Konfiguration und startet einen Container mit allen Abhängigkeiten — direkt in VS Code oder JetBrains öffnen.

### devcontainer.json

```json
// .devcontainer/devcontainer.json
{
  "name": "Node.js Entwicklungsumgebung",
  "image": "mcr.microsoft.com/devcontainers/node:20",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode"
      ]
    }
  },
  "postCreateCommand": "npm install",
  "forwardPorts": [3000]
}
```

---

## Extensions

Docker Desktop bietet einen Extension Marketplace:

```
Extensions → Browse → Installieren
```

### Beliebte Extensions

| Extension | Funktion |
|-----------|----------|
| **Disk Usage** | Speicherverbrauch visualisieren |
| **Logs Explorer** | Erweiterte Log-Suche |
| **Resource Usage** | CPU/Memory-Charts |
| **Portainer** | Erweiterte Container-Verwaltung |
| **Volumes Backup** | Volume-Backup und -Restore |
| **Trivy** | Security-Scanning direkt in Desktop |
| **Snyk** | Vulnerability-Scanning |
| **Telepresence** | Kubernetes-Service lokal debuggen |

---

## Ressourcen konfigurieren

```
Settings → Resources

CPU: 4 (von verfügbaren CPUs)
Memory: 8 GB (von verfügbarem RAM)
Swap: 1 GB
Disk image size: 64 GB
Virtual disk location: C:\...
```

> [!TIP] Memory-Einstellung wichtig
> Docker Desktop begrenzt standardmäßig den RAM der Linux-VM. Für ressourcenhungrige Container oder Kubernetes: Memory erhöhen.

---

## WSL2-Integration (Windows)

Docker Desktop integriert sich tief in WSL2:

```
Settings → Resources → WSL Integration

Aktivierte Distros erhalten automatisch:
- docker CLI
- docker compose
- Zugriff auf laufende Container
```

```bash
# In WSL2-Terminal (nach Integration)
docker ps
docker run -d nginx

# Dateien im Container-Pfad
ls /var/lib/docker/   # Nur in Docker Desktop VM verfügbar
```

---

## Docker Hub Integration

```
Account oben rechts → Sign in

Features nach Login:
- Images direkt auf Docker Hub pushen
- Private Repositories
- Scout (Security-Scanning)
- Build-History
```

---

## Häufige Probleme

### Docker Desktop startet nicht

```powershell
# Windows: WSL2-Backend zurücksetzen
wsl --shutdown
# Docker Desktop neu starten

# Windows: Hyper-V prüfen
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

### Speicher zurückgewinnen

```bash
# Unbenutzte Ressourcen bereinigen
docker system prune -a

# In Docker Desktop:
Settings → Troubleshoot → Clean / Purge data
```

### Proxy-Konfiguration

```
Settings → Resources → Proxies

HTTP-Proxy: http://proxy.firma.de:3128
HTTPS-Proxy: http://proxy.firma.de:3128
No Proxy: localhost,127.0.0.1,.firma.de
```

---

## Lizenzmodell (2025)

| Plan | Preis | Zielgruppe |
|------|-------|-----------|
| Personal | Kostenlos | Einzelpersonen, Open-Source |
| Pro | ~$9/Monat | Entwickler, kleine Teams |
| Team | ~$15/Monat/User | Teams, CI/CD-Features |
| Business | ~$21/Monat/User | Unternehmen, SSO, Management |

**Kostenlos erlaubt** für:
- Bildungseinrichtungen
- Open-Source-Projekte
- Persönliche Nutzung
- Unternehmen < 250 MA und < $10M Umsatz

---

## Verwandte Notizen

- [[Podman Desktop]] — Kostenlose Alternative (auch für Unternehmen)
- [[Docker vs Podman und LXC]] — Ausführlicher Vergleich
- [[Docker Installation]] — Docker ohne Desktop-App
- [[Docker Compose]] — Compose-Projekte in Docker Desktop
