---
tags:
  - docker
  - installation
  - setup
  - ubuntu
  - linux
aliases:
  - Docker Setup
  - Docker installieren
---

# Docker Installation

> [!ABSTRACT] TL;DR
> Vollständige Anleitung zur Installation der Docker Engine auf verschiedenen Linux-Distributionen sowie Docker Desktop für Windows und macOS. Umfasst Repository-Einrichtung, Post-Installation, Sicherheitshinweise und Grundkonfiguration.

---

## Voraussetzungen

Bevor Docker installiert werden kann, müssen folgende Systemanforderungen erfüllt sein:

| Anforderung | Minimum | Empfohlen |
|:------------|:--------|:----------|
| Architektur | 64-bit (x86_64 / amd64, arm64) | x86_64 / amd64 |
| Kernel-Version | ≥ 3.10 | ≥ 5.10+ |
| Betriebssystem | Linux, Windows 10+, macOS 11+ | Ubuntu 22.04+ LTS |
| RAM | 512 MB | ≥ 2 GB |
| Speicherplatz | 5 GB | ≥ 20 GB (je nach Images) |
| Netzwerk | Internetzugang für Registry-Pull | — |

> [!NOTE] Kernel-Module
> Docker benötigt die Kernel-Module `overlay`, `br_netfilter` und `ip_tables`. Auf modernen Distributionen sind diese standardmäßig verfügbar. Die aktuelle Kernel-Version lässt sich prüfen mit:
> ```bash
> uname -r
> ```

> [!IMPORTANT] cgroups v2
> Aktuelle Docker-Versionen unterstützen **cgroups v2** vollständig. Ältere Systeme mit ausschließlich cgroups v1 sollten aktualisiert werden. Prüfung:
> ```bash
> stat -fc %T /sys/fs/cgroup/
> # Ausgabe "cgroup2fs" → cgroups v2 aktiv
> ```

---

## Installation auf Ubuntu / Debian

### Alte Versionen entfernen

Vor der Installation sollten veraltete oder inoffizielle Docker-Pakete entfernt werden:

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
sudo apt autoremove -y
```

> [!NOTE] Keine Sorge
> Es ist kein Fehler, wenn `apt` meldet, dass keines dieser Pakete installiert ist. Dieser Schritt stellt lediglich sicher, dass keine Konflikte entstehen.

### Repository einrichten

Docker wird über das offizielle APT-Repository installiert — nicht über die Distributions-eigenen Pakete, da diese oft veraltet sind.

**1. Abhängigkeiten installieren:**

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

**2. Dockers offiziellen GPG-Key hinzufügen:**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**3. APT-Repository konfigurieren:**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

> [!TIP] Debian statt Ubuntu
> Für **Debian** einfach die URL anpassen:
> ```bash
> https://download.docker.com/linux/debian
> ```
> Der Rest bleibt identisch.

### Docker Engine installieren

```bash
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

| Paket | Beschreibung |
|:------|:-------------|
| `docker-ce` | Docker Engine (Community Edition) |
| `docker-ce-cli` | CLI-Client (`docker`-Befehl) |
| `containerd.io` | Container-Runtime |
| `docker-buildx-plugin` | Erweitertes Build-System (Multi-Platform) |
| `docker-compose-plugin` | Docker Compose v2 als CLI-Plugin |

### Post-Installation: User zur docker-Gruppe hinzufügen

```bash
sudo usermod -aG docker $USER
```

> [!WARNING] Sicherheitshinweis — docker-Gruppe
> Die Mitgliedschaft in der `docker`-Gruppe gewährt dem Benutzer **faktisch Root-Rechte** auf dem Host-System. Ein Container mit `-v /:/host` kann das gesamte Dateisystem lesen und schreiben. Nur vertrauenswürdige Benutzer sollten zur `docker`-Gruppe hinzugefügt werden.

> [!IMPORTANT] Sitzung neu starten
> Die Gruppenänderung wird erst nach einem **Logout/Login** oder Neustart wirksam:
> ```bash
> newgrp docker
> # oder: logout und erneut einloggen
> ```

---

## Installation auf RHEL / Fedora / Rocky Linux

### Repository einrichten

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/fedora/docker-ce.repo
```

> [!TIP] RHEL / Rocky Linux
> Für **RHEL** oder **Rocky Linux** die Repository-URL anpassen:
> ```bash
> https://download.docker.com/linux/rhel/docker-ce.repo
> # oder
> https://download.docker.com/linux/centos/docker-ce.repo
> ```

### Docker Engine installieren

```bash
sudo dnf install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### Docker starten und aktivieren

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### Firewall-Konfiguration

Auf RHEL-basierten Systemen blockiert `firewalld` standardmäßig den Container-Netzwerkverkehr. Docker verwaltet `iptables`-Regeln selbst, aber bei aktivierter `firewalld` können Konflikte auftreten:

```bash
# Docker-Interface in die trusted Zone aufnehmen
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --reload
```

> [!NOTE] Masquerading
> Falls Container keinen Internetzugang haben, muss Masquerading aktiviert werden:
> ```bash
> sudo firewall-cmd --permanent --zone=public --add-masquerade
> sudo firewall-cmd --reload
> ```

---

## Installation auf Arch Linux

Auf Arch Linux ist Docker direkt in den offiziellen Repositories verfügbar:

```bash
sudo pacman -S docker docker-compose docker-buildx
```

Docker-Dienst starten und aktivieren:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

Benutzer zur `docker`-Gruppe hinzufügen:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> [!TIP] AUR-Alternative
> Wer Docker aus dem AUR bauen möchte (z.B. `docker-git` für die neueste Version), kann `yay` oder `paru` verwenden. Für Produktionsumgebungen wird jedoch das offizielle Paket empfohlen.

---

## Docker Desktop vs. Docker Engine

Docker Desktop ist eine GUI-Anwendung für Windows und macOS, die Docker Engine in einer verwalteten VM ausführt.

| Merkmal | Docker Engine | Docker Desktop |
|:--------|:-------------|:---------------|
| Plattform | Linux (nativ) | Windows, macOS, Linux |
| Runtime | Nativ auf dem Host-Kernel | Leichte VM (HyperKit / WSL2 / QEMU) |
| GUI | Nein (nur CLI) | Ja (Tray-App, Dashboard) |
| Compose | Plugin (`docker compose`) | Integriert |
| Kubernetes | Manuell installieren | Eingebaut (ein Klick) |
| Lizenz | Open Source (Apache 2.0) | Kostenlos für persönliche Nutzung und kleine Unternehmen; kostenpflichtig ab 250 Mitarbeitern oder \$10M Umsatz |
| Ressourcen | Minimaler Overhead | VM-Overhead (RAM, CPU) |
| Empfehlung | Server, Produktion, Homelab | Entwicklung auf Desktop-OS |

### Docker Desktop auf Windows (WSL2-Backend)

```powershell
# 1. WSL2 aktivieren (PowerShell als Admin)
wsl --install

# 2. Docker Desktop herunterladen und installieren
# → https://docs.docker.com/desktop/install/windows-install/

# 3. In den Einstellungen WSL2-Backend aktivieren
# Settings → General → "Use the WSL 2 based engine" ✓
```

> [!NOTE] WSL2-Integration
> Docker Desktop integriert sich in WSL2-Distributionen. Der `docker`-Befehl ist dann direkt in jeder WSL2-Instanz verfügbar — ohne separate Installation innerhalb der Distribution.

### Docker Desktop auf macOS

```bash
# Via Homebrew
brew install --cask docker

# Anschließend Docker.app starten
open /Applications/Docker.app
```

---

## Post-Installation

### Docker-Gruppe einrichten

Falls noch nicht geschehen:

```bash
# Gruppe erstellen (normalerweise bei Installation angelegt)
sudo groupadd docker

# Aktuellen Benutzer hinzufügen
sudo usermod -aG docker $USER

# Gruppenmitgliedschaft prüfen
groups $USER
```

### Autostart mit systemd

```bash
# Docker beim Systemstart aktivieren
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Status prüfen
sudo systemctl status docker
```

> [!TIP] Socket-Aktivierung
> Docker kann auch über **Socket-Aktivierung** gestartet werden — der Daemon startet erst bei der ersten Nutzung des `docker`-Befehls:
> ```bash
> sudo systemctl disable docker.service
> sudo systemctl enable docker.socket
> ```
> Das spart Ressourcen auf Systemen, auf denen Docker nicht permanent läuft.

### Storage Driver prüfen

```bash
docker info | grep "Storage Driver"
# Erwartete Ausgabe: Storage Driver: overlay2
```

`overlay2` ist der empfohlene und standardmäßige Storage Driver für alle modernen Linux-Distributionen. Alternativen wie `devicemapper`, `btrfs` oder `zfs` sind nur in Spezialfällen relevant.

### daemon.json — Grundkonfiguration

Die zentrale Konfigurationsdatei des Docker Daemon liegt unter `/etc/docker/daemon.json`:

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 24
    }
  ],
  "metrics-addr": "127.0.0.1:9323",
  "experimental": false,
  "live-restore": true
}
EOF
```

| Option | Beschreibung |
|:-------|:-------------|
| `log-driver` | Log-Treiber (Standard: `json-file`) |
| `log-opts.max-size` | Maximale Größe pro Log-Datei |
| `log-opts.max-file` | Maximale Anzahl rotierter Log-Dateien |
| `storage-driver` | Storage Driver (`overlay2` empfohlen) |
| `default-address-pools` | IP-Bereiche für Docker-Netzwerke |
| `metrics-addr` | Prometheus-Metriken-Endpunkt |
| `live-restore` | Container laufen weiter, wenn der Daemon neu startet |

> [!WARNING] Log-Rotation konfigurieren
> Ohne konfigurierte Log-Rotation können Container-Logs die Festplatte füllen. Die oben gezeigte Konfiguration begrenzt jede Log-Datei auf 10 MB bei maximal 3 Dateien pro Container.

Nach Änderungen an `daemon.json` den Daemon neu starten:

```bash
sudo systemctl restart docker
```

---

## Docker testen

### Funktionstest mit Hello World

```bash
docker run hello-world
```

Erwartete Ausgabe (Auszug):

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

### Versionsinfo anzeigen

```bash
docker version
```

```text
Client: Docker Engine - Community
 Version:           28.x.x
 API version:       1.x
 ...

Server: Docker Engine - Community
 Engine:
  Version:          28.x.x
  ...
```

### Systeminfo prüfen

```bash
docker info
```

Wichtige Felder in der Ausgabe:

- **Server Version** — Installierte Docker-Version
- **Storage Driver** — Sollte `overlay2` sein
- **Cgroup Driver** — `systemd` (empfohlen) oder `cgroupfs`
- **Docker Root Dir** — Standard `/var/lib/docker`

### Erster interaktiver Container

```bash
docker run -it --rm ubuntu:latest bash
```

> [!TIP] Flags erklärt
> - `-it` — Interaktiver Modus mit Terminal (TTY)
> - `--rm` — Container wird nach dem Beenden automatisch gelöscht
> - `ubuntu:latest` — Basis-Image
> - `bash` — Auszuführender Befehl

---

## Deinstallation

### Docker Engine vollständig entfernen

**Ubuntu / Debian:**

```bash
sudo apt purge -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm -rf /etc/docker
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/keyrings/docker.gpg
```

**RHEL / Fedora / Rocky Linux:**

```bash
sudo dnf remove -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm -rf /etc/docker
```

**Arch Linux:**

```bash
sudo pacman -Rns docker docker-compose docker-buildx
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker
```

> [!DANGER] Datenverlust
> Das Löschen von `/var/lib/docker` entfernt **alle** Images, Container, Volumes und Netzwerke unwiderruflich. Vorher sicherstellen, dass keine wichtigen Daten in Docker-Volumes liegen, oder diese zuvor exportieren:
> ```bash
> # Volumes auflisten
> docker volume ls
>
> # Volume-Daten sichern
> docker run --rm -v mein_volume:/data -v $(pwd):/backup \
>   alpine tar czf /backup/volume-backup.tar.gz -C /data .
> ```

### Benutzer aus docker-Gruppe entfernen

```bash
sudo gpasswd -d $USER docker
```

---

## Verwandte Notizen

- [[Docker Index]]
- [[Was ist Docker]]
- [[Docker Architektur]]
