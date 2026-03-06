---
tags: [podman, grundlagen, daemonless, rootless, container, red-hat, oci]
aliases: ["Podman Einführung", "Podman Grundlagen"]
---

# Was ist Podman

> [!ABSTRACT]
> Podman ist ein daemonloser, rootless-fähiger OCI-Container-Manager von Red Hat. Statt eines zentralen Daemon-Prozesses startet Podman jeden Container als direkten Kindprozess des aufrufenden Benutzers — sicherer, ressourcenschonender und systemd-freundlicher als Docker. Vollständig kompatibel mit OCI-Images und weitgehend CLI-kompatibel mit Docker.

---

## Das Problem mit Docker

Docker hat die Container-Welt revolutioniert — aber die zugrundeliegende Architektur hat strukturelle Schwächen, die in Produktionsumgebungen und auf Multi-User-Systemen problematisch werden.

### Der Docker-Daemon

Docker arbeitet mit einem zentralen Daemon-Prozess (`dockerd`), der dauerhaft als **Root** läuft. Jede Docker-Operation geht über diesen Prozess:

```
Docker:  CLI → dockerd (root) → containerd → runc → Container
```

Der Daemon ist immer aktiv, auch wenn kein einziger Container läuft.

### Single Point of Failure

Stürzt `dockerd` ab, verliert man die Kontrolle über alle laufenden Container. Die Container selbst können weiterlaufen, aber Docker kann sie weder überwachen noch verwalten, bis der Daemon neu startet.

### Root-Anforderung

Standardmäßig müssen Benutzer in der `docker`-Gruppe sein oder `sudo` verwenden. Die Mitgliedschaft in der `docker`-Gruppe ist sicherheitstechnisch problematisch:

> [!WARNING] Die docker-Gruppe entspricht Root-Rechten
> Ein Benutzer in der `docker`-Gruppe kann das gesamte Hostsystem kompromittieren:
> ```bash
> docker run -v /:/mnt --rm -it alpine chroot /mnt sh
> ```
> Kein `sudo` nötig — aber voller Root-Zugriff auf den Host.

### Docker Desktop Lizenzierung (ab 2022)

Seit August 2022 ist Docker Desktop für Unternehmen mit mehr als 250 Mitarbeitern **oder** mehr als 10 Mio. USD Jahresumsatz kostenpflichtig (ab ca. 21 USD/Monat/Nutzer). Das hat viele Organisationen dazu veranlasst, Alternativen zu evaluieren.

---

## Podman — die Alternative

Podman (**Pod** **Man**ager) entstand 2018 im Rahmen des `containers`-Projekts bei Red Hat als direktes Gegenstück zu Docker — mit dem Ziel, alle bekannten Schwächen zu adressieren, ohne die Kompatibilität zu opfern.

> [!INFO] Open-Source-Projekt
> Podman ist Teil der `containers`-Organisation auf GitHub (`github.com/containers`), zu der auch Buildah, Skopeo und CRI-O gehören. Das Projekt wird hauptsächlich von Red Hat entwickelt, ist aber vollständig Open Source (Apache 2.0).

| Plattform | Verfügbarkeit |
|-----------|---------------|
| Fedora / RHEL / CentOS Stream | Vorinstalliert oder `dnf install podman` |
| Debian / Ubuntu | `apt install podman` |
| Arch Linux | `pacman -S podman` |
| macOS | `brew install podman` + `podman machine init` |
| Windows | Installer + WSL2-Backend |

---

## Kernprinzipien

### 1. Daemonlos

Podman hat keinen zentralen Daemon-Prozess. Jeder `podman run`-Aufruf startet direkt einen Container — kein Hintergrunddienst, kein dauerhaft laufender Prozess:

```
Podman:  CLI → fork/exec → conmon → crun → Container
```

Jeder Container ist ein direkter Kindprozess des aufrufenden Benutzers. Podman selbst beendet sich nach dem Start — die Container laufen weiter, überwacht von `conmon` (ein schlanker C-Prozess pro Container).

```bash
# Prozessbaum bei rootless Podman
$ pstree $(whoami)
user
└─ conmon
   └─ nginx (Container-Prozess)
```

### 2. Rootless by Default

Podman wurde von Grund auf für den Betrieb ohne Root-Rechte entwickelt. Ein normaler Benutzer kann Container starten, stoppen und verwalten — ohne je `sudo` zu verwenden:

```bash
# Kein sudo nötig
podman run -d --name webserver -p 8080:80 nginx:latest
podman ps
podman logs webserver
podman stop webserver
```

Rootless-Container laufen in einem **User-Namespace**: UID 0 im Container wird auf die UID des aufrufenden Benutzers auf dem Host gemappt. Innerhalb des Containers hat der Prozess scheinbar Root-Rechte — auf dem Host ist er ein normaler Benutzer.

> [!TIP] Rootless vs. Rootful
> Podman kann auch als Root (rootful) betrieben werden — z. B. wenn Ports unter 1024 benötigt werden oder Netzwerk-Bridges angelegt werden müssen. In den meisten Anwendungsfällen ist rootless die bevorzugte und sicherere Option.

### 3. OCI-konform

Podman implementiert die Spezifikationen der **Open Container Initiative (OCI)**:

- **OCI Image Spec** — Images sind binär kompatibel mit Docker, containerd und CRI-O
- **OCI Runtime Spec** — Standard-Runtime ist `crun` (C, sehr schnell) oder `runc`
- **OCI Distribution Spec** — Kompatibel mit Docker Hub, Quay.io, GitHub Container Registry, etc.

Das bedeutet: Jedes Docker-Image läuft ohne Anpassungen unter Podman — und umgekehrt.

### 4. CLI-Kompatibilität mit Docker

Podman ist zu ca. 95 % CLI-kompatibel mit Docker. Für viele Teams reicht ein einfacher Alias für eine nahtlose Migration:

```bash
alias docker=podman
```

---

## Podman im Container-Ökosystem

| Eigenschaft | Podman | Docker | LXC / LXD |
|-------------|:------:|:------:|:---------:|
| **Daemon erforderlich** | ❌ | ✅ (`dockerd`) | ✅ (`lxd`) |
| **Rootless nativ** | ✅ | ⚠️ Eingeschränkt | ⚠️ Eingeschränkt |
| **Container-Typ** | Anwendungscontainer | Anwendungscontainer | Systemcontainer |
| **Init-System im Container** | Optional | Optional | systemd / OpenRC |
| **OCI-konform** | ✅ | ✅ | ❌ |
| **Pod-Konzept** | ✅ (wie Kubernetes) | ❌ | ❌ |
| **systemd-Integration** | ✅ Nativ (Quadlet) | ⚠️ Umständlich | ✅ |
| **Hauptplattform** | RHEL / Fedora | Plattformübergreifend | Debian / Ubuntu |

> [!NOTE] Anwendungs- vs. Systemcontainer
> Podman und Docker sind **Anwendungscontainer** — sie isolieren einzelne Prozesse oder Applikationen. LXC/LXD sind **Systemcontainer** — sie emulieren vollständige Linux-Systeme mit Init-Prozess, mehreren Diensten und eigener Paketverwaltung. Die Anwendungsfälle überschneiden sich kaum.

---

## Die Red-Hat-Toolbox

Podman ist Teil eines klar strukturierten Werkzeugkastens:

| Tool | Aufgabe | Docker-Analogie |
|------|---------|----------------|
| **Podman** | Container starten, stoppen, verwalten, Pods orchestrieren | `docker run` / `docker ps` |
| **Buildah** | Container-Images bauen (mit oder ohne Dockerfile) | `docker build` |
| **Skopeo** | Images kopieren, inspizieren, zwischen Registries transferieren | `docker pull` / `docker push` + mehr |

**Buildah** ermöglicht das Bauen von Images ohne laufenden Daemon. Podman nutzt intern Buildah-Bibliotheken für `podman build`, sodass kein zweites Tool installiert werden muss.

**Skopeo** kann Images direkt zwischen zwei Registries kopieren, ohne sie lokal zu speichern:

```bash
# Image direkt von Docker Hub zu GitHub Container Registry kopieren
skopeo copy \
  docker://docker.io/nginx:latest \
  docker://ghcr.io/meinorg/nginx:latest

# Image-Manifest inspizieren ohne zu pullen
skopeo inspect docker://docker.io/nginx:latest | jq .Architecture
```

---

## Wann Podman, wann Docker?

| Use Case | Empfehlung | Begründung |
|----------|:----------:|-----------|
| RHEL / Fedora / CentOS Stream | **Podman** | Vorinstalliert, erste Wahl von Red Hat |
| Multi-User-System (Shared Host) | **Podman** | Rootless-Isolation — jeder Nutzer hat eigene Container |
| CI/CD ohne Root (GitLab, GitHub Actions) | **Podman** | Kein Daemon nötig, funktioniert in unprivilegierten Containern |
| Kubernetes-Vorbereitung / -Migration | **Podman** | Pod-Konzept identisch, `podman generate kube` exportiert direkt |
| systemd-Service aus Container | **Podman** | Quadlet: `.container`-Unit-Files nativ in systemd |
| Sicherheitskritische Umgebungen | **Podman** | Kein Root, kein Daemon, minimale Angriffsfläche |
| Einfacher lokaler Einstieg | **Docker** | Beste Dokumentation, größte Community |
| Windows-Workloads | **Docker** | Docker Desktop mit WSL2-Integration ist ausgereifter |
| Breites Ökosystem-Tooling | **Docker** | Portainer, Watchtower, Dozzle, Traefik etc. |
| Legacy-Infrastruktur mit docker-compose | **Docker** | Vollständige Compose V2-Kompatibilität |

---

## Geschichte und Versionen

| Jahr | Version / Ereignis | Bedeutung |
|------|--------------------|-----------|
| 2018 | Erste Veröffentlichung | Daemonloses Container-Tool, Teil des `containers`-Projekts |
| 2019 | Podman 1.x | Rootless-Container stabil, Integration als Standard in RHEL 8 |
| 2020 | **Podman 2.0** | Pod-Unterstützung, REST-API (docker-kompatibel) |
| 2021 | Podman 3.x | `podman generate systemd`, Docker-Compose-Kompatibilität |
| 2022 | **Podman 4.0** | Netavark + Aardvark-DNS ersetzen CNI-Netzwerk-Stack |
| 2023 | Podman 4.4–4.6 | **Quadlet** stabil, Podman Desktop 1.0 veröffentlicht |
| 2024 | **Podman 5.0** | Pasta-Netzwerk als Standard für rootless, Machine neu gebaut |
| 2025 | Podman 5.x | Quadlet-Erweiterungen, `podman farm` für Multi-Arch-Builds |

> [!INFO] Podman Desktop
> Seit 2022 gibt es **Podman Desktop** — eine grafische Oberfläche für macOS, Windows und Linux, die Podman, Buildah, Skopeo und Kubernetes-Integration unter einer einheitlichen UI vereint. Kostenlos und Open Source — eine vollwertige Alternative zu Docker Desktop.

---

## Verwandte Notizen

- [[Podman Architektur]] — Technische Tiefe: conmon, crun, User-Namespaces, cgroups v2
- [[Podman vs Docker]] — Detaillierter Vergleich: CLI, Networking, Compose, Security, Migration
- [[Podman Installation]] — Installation auf Fedora, RHEL, Ubuntu, Debian, macOS und Windows
- [[Podman Index]] — Übersicht aller Podman-Themen in dieser Wissensdatenbank
