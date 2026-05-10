---
tags: [podman, docker, vergleich, migration, rootless, daemonless, sicherheit]
aliases: ["Podman Docker Vergleich", "Podman oder Docker"]
---

# Podman vs Docker

> [!ABSTRACT]
> Podman und Docker sind beide OCI-kompatible Container-Tools mit weitgehend identischer CLI — aber fundamental unterschiedlicher Architektur. Podman ist daemonlos und rootless-by-default, Docker setzt auf einen zentralen Daemon. Beide haben klare Stärken; die Wahl hängt von Plattform, Sicherheitsanforderungen und vorhandenem Tooling ab.

---

## Architektur-Vergleich

| Merkmal | Podman | Docker |
|---------|--------|--------|
| **Daemon** | Kein Daemon — jeder Container ist direkter Kindprozess | `dockerd` läuft dauerhaft als Root |
| **Rootless** | ✅ Native, by default seit v1.0 | ⚠️ Möglich seit v20, aber eingeschränkter |
| **OCI-konform** | ✅ Vollständig | ✅ Vollständig |
| **Pods** | ✅ Nativ (Kubernetes-style) | ❌ Nicht vorhanden |
| **systemd-Integration** | ✅ Quadlet (First-Class) | ⚠️ Nur über externe Restart-Policies |
| **Netzwerk-Stack** | Netavark (rootful) + Pasta (rootless) | Docker-Bridge (iptables-basiert) |
| **Image-Format** | OCI + Docker V2 | Docker V2 + OCI |
| **CLI-Kompatibilität** | ~95 % Docker-kompatibel | — |
| **Compose** | `podman compose` / Quadlet | `docker compose` (V2) |
| **Desktop-Tool** | Podman Desktop (kostenlos) | Docker Desktop (kostenpflichtig ab 250 MA) |
| **Lizenz** | Apache 2.0 | Apache 2.0 (Engine) / kostenpflichtig (Desktop) |
| **Primäre Plattform** | RHEL, Fedora, CentOS Stream | Plattformübergreifend |
| **Kubernetes-Runtime** | CRI-O (selbes Ökosystem) | containerd |

---

## Sicherheitsvergleich

### Daemon als Angriffsfläche

Docker's `dockerd` läuft dauerhaft als Root. Wer den Socket `/var/run/docker.sock` lesen kann, hat effektiv Root-Zugriff auf den Host:

```bash
# Triviale Privilege Escalation mit Docker-Socket
docker run -v /:/host --rm -it alpine chroot /host sh
# → Root-Shell auf dem Host
```

Podman hat keinen solchen Socket. Ohne explizit gestarteten `podman.socket` gibt es keine Root-Angriffsfläche.

### Rootless-Implementierung

| Aspekt | Podman Rootless | Docker Rootless |
|--------|-----------------|-----------------|
| **Reife** | Seit 2018, default seit v1.0 | Seit Docker 20.10 (2020) |
| **Installation** | Kein separater Setup nötig | Separates Script (`dockerd-rootless-setuptool.sh`) |
| **cgroups v2** | Vollständig unterstützt | Vollständig unterstützt |
| **Networking** | Pasta (v5) / slirp4netns (v4) | RootlessKit + slirp4netns |
| **Overlay-Storage** | Native Kernel-Overlayfs (Kernel 5.11+) | Fuse-overlayfs |
| **Port < 1024** | Einschränkung (sysctl nötig) | Gleiche Einschränkung |

### User Namespaces

Podman nutzt User Namespaces für alle rootless Container standardmäßig. UID 0 im Container = UID des aufrufenden Benutzers auf dem Host:

```bash
# Im Container: root (UID 0)
# Auf dem Host: max (UID 1000)
podman run --rm alpine id
# uid=0(root) gid=0(root)

podman run --rm alpine cat /proc/self/status | grep NSpid
# NSpid: 1234  1  ← Host-PID vs. Container-PID
```

---

## Native Pods — Podman-Alleinstellungsmerkmal

Docker hat kein Pod-Konzept. Podman implementiert Kubernetes-style Pods nativ:

```bash
# Pod erstellen
podman pod create --name webapp -p 8080:80

# Container zum Pod hinzufügen
podman run -d --pod webapp --name frontend nginx
podman run -d --pod webapp --name backend myapp:latest

# Alle Container im Pod teilen: Netzwerk, IPC
# Frontend erreicht Backend via: localhost:3000
```

**Praxisnutzen:**
- Lokales Testen von Multi-Container-Setups ohne Compose
- Kubernetes YAML generieren: `podman generate kube webapp > webapp.yaml`
- Kubernetes YAML lokal deployen: `podman play kube webapp.yaml`

---

## systemd-Integration — Podman-Vorteil

### Docker: Externe Verwaltung nötig

Docker-Container als systemd-Services laufen zu lassen erfordert Wrapper-Scripts oder Docker Compose + `--restart always`:

```ini
# Aufwändige manuelle systemd-Unit für Docker
[Service]
ExecStart=/usr/bin/docker start -a webserver
ExecStop=/usr/bin/docker stop webserver
RemainAfterExit=yes
```

### Podman: Quadlet (First-Class systemd)

Podman Quadlet generiert systemd-Units direkt aus deklarativen `.container`-Dateien:

```ini
# ~/.config/containers/systemd/webserver.container
[Container]
Image=nginx:alpine
PublishPort=8080:80
Volume=%h/html:/usr/share/nginx/html:ro,z
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now webserver.service
journalctl --user -u webserver.service -f
```

---

## Podman Desktop vs. Docker Desktop

| Merkmal | Podman Desktop | Docker Desktop |
|---------|---------------|----------------|
| **Preis** | Kostenlos (Open Source) | Kostenlos für Einzelnutzer / kostenpflichtig für Unternehmen |
| **Lizenz** | Apache 2.0 | Proprietär |
| **Plattformen** | macOS, Windows, Linux | macOS, Windows |
| **VM-Backend (Mac)** | QEMU oder Apple Hypervisor | Apple Hypervisor |
| **GUI-Features** | Container, Images, Pods, Volumes, Extensions | Container, Images, Volumes, Extensions |
| **Kubernetes** | Kind, Minikube (via Extensions) | Integriert (Docker Kubernetes) |
| **Extensions** | ✅ | ✅ |
| **Podman-spezifisch** | Pods, Quadlet-UI | — |

> [!NOTE] Docker Desktop Lizenz
> Seit August 2022 ist Docker Desktop für Unternehmen mit >250 Mitarbeitern **oder** >10 Mio. USD Jahresumsatz kostenpflichtig (ab ~5 USD/Monat/Nutzer im Pro-Plan). Podman Desktop ist und bleibt kostenlos.

---

## Docker-Kompatibilität von Podman

### Was funktioniert

```bash
# alias docker=podman — funktioniert für die meisten Anwendungsfälle
alias docker=podman

docker run -d --name web -p 8080:80 nginx   ✅
docker ps                                    ✅
docker images                                ✅
docker build -t myapp .                      ✅
docker exec -it web bash                     ✅
docker logs web                              ✅
docker network ls                            ✅
docker volume ls                             ✅
```

### Was NICHT oder eingeschränkt funktioniert

```bash
docker swarm init          ❌  # Docker Swarm: kein Podman-Äquivalent
docker stack deploy        ❌  # Swarm-spezifisch
docker compose up          ⚠️  # Nur via Podman Socket + docker compose CLI
docker context             ⚠️  # Eingeschränkte Unterstützung
```

> [!WARNING] Short Image Names
> Podman fragt bei unqualifizierten Image-Namen interaktiv nach der Registry (`docker.io/library/nginx` vs. `quay.io/nginx`). In Scripts immer vollständige Namen nutzen:
> ```bash
> # Statt: podman run nginx
> podman run docker.io/library/nginx:latest
> ```

---

## Migration von Docker zu Podman

### Schritt 1 — Images übernehmen

```bash
# Option A: Direkt pullen (bevorzugt)
podman pull docker.io/library/nginx:latest

# Option B: Aus Docker exportieren und in Podman importieren
docker save nginx:latest | podman load
```

### Schritt 2 — Volumes migrieren

```bash
# Podman-Volume erstellen
podman volume create mydata

# Daten aus Docker-Volume kopieren
docker run --rm -v mydata:/source -v $(pwd):/backup alpine \
  tar czf /backup/mydata.tar.gz -C /source .

podman run --rm -v mydata:/target -v $(pwd):/backup alpine \
  tar xzf /backup/mydata.tar.gz -C /target
```

### Schritt 3 — docker-compose.yml umstellen

```bash
# Option A: podman compose (direkter Ersatz)
pip3 install podman-compose
podman compose up -d

# Option B: Docker Compose CLI via Podman Socket
systemctl --user enable --now podman.socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
docker compose up -d   # Docker Compose CLI → Podman Backend
```

### Schritt 4 — Systemd-Services (statt docker run in Scripts)

```bash
# Docker: cron-Job oder Script mit docker run
# Podman: Quadlet .container-Datei erstellen (empfohlen)
mkdir -p ~/.config/containers/systemd/
# → .container-Datei erstellen (siehe Systemd Integration)
systemctl --user daemon-reload && systemctl --user enable --now myservice.service
```

### Schritt 5 — SELinux-Labels (RHEL/Fedora)

```bash
# Alle Volume-Mounts mit :z oder :Z suffix versehen
# Ohne diese Labels schlägt SELinux den Zugriff fehl

# Vorher (Docker):
docker run -v /daten:/data nginx

# Nachher (Podman auf RHEL/Fedora):
podman run -v /daten:/data:z nginx   # shared
podman run -v /daten:/data:Z nginx   # private
```

### Häufige Fallstricke bei der Migration

| Problem | Ursache | Lösung |
|---------|---------|--------|
| Port < 1024 nicht bindbar | Rootless-Einschränkung | `sysctl net.ipv4.ip_unprivileged_port_start=80` |
| SELinux blockt Volume-Mount | Fehlendes `:z`/`:Z` Label | Label hinzufügen |
| DNS zwischen Containern fehlt | Unterschiedliche Netzwerke | Gemeinsames benanntes Netzwerk erstellen |
| Short image names — interaktiver Prompt | Registry nicht konfiguriert | Vollständige Image-Namen oder `registries.conf` anpassen |
| Container startet nicht nach Reboot | Kein `loginctl enable-linger` | `sudo loginctl enable-linger $USER` |

---

## Verwandte Notizen

- [[Was ist Podman]] — Grundlagen und Konzept
- [[Podman Architektur]] — Technische Unterschiede im Detail
- [[Podman Installation]] — Installation auf allen Plattformen
- [[Docker vs Podman und LXC]] — Dreier-Vergleich im Docker-Kontext
