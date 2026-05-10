---
tags: [podman, architektur, libpod, conmon, netavark, pasta, crun, containers-image]
aliases: ["Podman Internals", "Podman Architektur"]
---

# Podman Architektur

> [!ABSTRACT]
> Podman basiert auf dem Fork-Exec-Modell statt dem Daemon-Modell. Kernkomponenten sind libpod (die Bibliothek), conmon (Container-Monitor), crun/runc (OCI-Runtime), Netavark (rootful Networking), Pasta (rootless Networking) sowie containers/image und containers/storage für Image- und Layer-Verwaltung.

---

## Fork-Exec-Modell vs. Docker-Daemon-Modell

```
Docker:  CLI → dockerd (root) → containerd → runc → Container
Podman:  CLI → conmon → crun → Container
```

Bei Docker ist `dockerd` ein dauerhaft laufender Root-Prozess, der alle Container verwaltet. Fällt der Daemon aus, verliert man die Kontrolle über alle Container.

Bei Podman gibt es keinen Daemon. `podman run` führt direkt einen Fork aus, startet `conmon` als Elternprozess des Containers und beendet sich selbst. Die Container laufen weiter — unabhängig vom Podman-Prozess.

---

## Kernkomponenten

### libpod — Die Kernbibliothek

`libpod` ist die zentrale Go-Bibliothek von Podman. Sie implementiert:

- Container-Lifecycle (create, start, stop, remove, exec)
- Pod-Verwaltung (pod create, start, inspect)
- Volume- und Netzwerk-Verwaltung
- Storage-Anbindung (containers/storage)
- Image-Anbindung (containers/image)
- REST-API-Server (kompatibel mit Docker Socket API)

Alle Podman-Befehle rufen libpod-Funktionen auf. Auch andere Tools (z. B. CRI-O, der Kubernetes-Container-Runtime) nutzen Teile dieses Ökosystems.

### conmon — Container Monitor

`conmon` ist ein leichtgewichtiger C-Prozess, der **pro Container** gestartet wird:

```
user session
└─ podman (beendet sich nach Start)
   └─ conmon (läuft weiter als Elternprozess)
      └─ crun / runc (OCI-Runtime, kurzlebig)
         └─ Container-Prozess (nginx, bash, ...)
```

conmon übernimmt:
- Weiterleitung von stdio (stdout/stderr) an Logging-Backend
- Überwachung des Exit-Codes
- OOM-Detection und Reporting
- Übermittlung des Container-Status an libpod via Unix-Socket

conmon ist bewusst minimalistisch gehalten (< 2000 Zeilen C-Code), da er der einzige Prozess ist, der dauerhaft für jeden Container läuft.

### OCI-Runtime: crun und runc

Die OCI-Runtime führt tatsächlich den Container-Prozess aus — erzeugt Namespaces, setzt cgroups, führt den Entrypoint aus.

| Runtime | Sprache | Vorteile |
|---------|---------|---------|
| **crun** | C | Sehr schnell, geringer RAM-Verbrauch, empfohlen für rootless |
| **runc** | Go | Docker-Standard, breit kompatibel |
| **kata-runtime** | Go/Rust | VMs als Container (starke Isolation) |

crun ist seit Podman 4.x der empfohlene Default auf RHEL/Fedora.

```bash
# Aktive Runtime anzeigen
podman info | grep ociRuntime

# Runtime pro Container wählen
podman run --runtime crun nginx
podman run --runtime runc nginx
```

### containers/image — Image-Handling

`containers/image` ist eine Go-Bibliothek für alle Image-Operationen:
- Pull/Push von/zu Registries (docker.io, quay.io, ghcr.io, lokale Registries)
- Image-Inspektion ohne Daemon
- Konvertierung zwischen Image-Formaten (OCI, Docker V2, dir)
- Signatur-Prüfung (Sigstore, GPG)

Diese Bibliothek wird auch von Skopeo und Buildah genutzt — alle drei Tools sprechen dieselbe Image-API.

### containers/storage — Layer-Verwaltung

`containers/storage` verwaltet alle lokalen Image-Layer und Container-Rootfs:

| Kontext | Speicherpfad |
|---------|-------------|
| Rootful | `/var/lib/containers/storage/` |
| Rootless | `~/.local/share/containers/storage/` |

Unterstützte Storage-Treiber:
- **overlay** (Standard) — Kernel-native OverlayFS (Kernel 5.11+) oder fuse-overlayfs
- **vfs** — Vollständige Kopien, langsam, aber immer kompatibel (Fallback)
- **zfs** — ZFS-basierte Layer (selten genutzt)

### Netavark — Rootful Networking

Seit Podman 4.0 ersetzt **Netavark** den alten CNI-Stack:

- In Rust geschrieben — performant und memory-safe
- Erstellt Linux-Bridge-Interfaces für Container-Netzwerke
- Standard-Bridge: `podman` (10.88.0.0/16)
- Arbeitet zusammen mit **Aardvark-DNS** für Container-Name-Auflösung

```bash
# Netavark-Netzwerke anzeigen
podman network ls
# Standard: podman (bridge, 10.88.0.0/16)
```

### Pasta — Rootless Networking

Seit Podman 5.0 ist **Pasta** (basierend auf `passt.one`) der Standard-Netzwerk-Stack für rootless Container:

- User-Space-Networking — kein Kernel-Modul nötig
- Vollständige TCP/UDP/ICMP-Unterstützung
- IPv6 out-of-the-box
- VPN-kompatibel (besser als der Vorgänger slirp4netns)
- Höherer Durchsatz, geringere Latenz als slirp4netns

### Buildah — Image-Build-Engine

Podman nutzt Buildah-Bibliotheken intern für `podman build`. Buildah kann auch standalone genutzt werden:

```bash
# podman build nutzt Buildah intern
podman build -t myapp:latest .

# Buildah standalone (ohne Dockerfile)
buildah from ubuntu:24.04
buildah run container -- apt-get install -y nginx
buildah commit container myapp:latest
```

### Skopeo — Image-Transport

Skopeo ermöglicht Image-Operationen ohne Container-Runtime:

```bash
# Zwischen Registries kopieren (kein lokales Speichern)
skopeo copy docker://nginx:latest docker://quay.io/meinuser/nginx:latest

# Ohne Pull inspizieren
skopeo inspect docker://quay.io/fedora/fedora:latest

# Digest prüfen
skopeo inspect --format "{{.Digest}}" docker://nginx:latest
```

---

## Prozesshierarchie im Betrieb

### Rootful Container

```
systemd (PID 1)
└─ conmon --api-version ... (PID 1234)
   └─ nginx: master process (PID 1235)   ← Container-Prozess
      └─ nginx: worker process (PID 1236)
```

### Rootless Container

```
systemd --user (user session)
└─ conmon (PID 4001, läuft als user)
   └─ crun ... (kurzlebig, beendet sich)
      └─ nginx (PID 1 im Container = PID 4002 auf Host, UID 1000)
```

Im User-Namespace erscheint der Prozess als UID 0 (root) — auf dem Host ist er UID 1000 (normaler Benutzer).

---

## Speicher-Layout

```
# Rootful
/var/lib/containers/storage/
  overlay/              ← Image-Layer
  overlay-containers/   ← Container-Rootfs
  volumes/              ← Named Volumes

/etc/containers/
  registries.conf       ← Registry-Konfiguration
  policy.json           ← Image-Signatur-Policy
  storage.conf          ← Storage-Konfiguration

# Rootless (pro User)
~/.local/share/containers/storage/
  overlay/
  overlay-containers/
  volumes/

~/.config/containers/
  registries.conf       ← User-spezifische Overrides
  storage.conf
```

---

## Registries-Konfiguration

```bash
# Globale Konfiguration
cat /etc/containers/registries.conf
```

```toml
# Welche Registries bei Short-Names durchsucht werden
unqualified-search-registries = ["registry.fedoraproject.org", "registry.access.redhat.com", "docker.io"]

# Short-Name Aliase
[[aliases]]
  "nginx" = "docker.io/library/nginx"
  "ubuntu" = "docker.io/library/ubuntu"
```

> [!NOTE] Short-Name-Auflösung
> Wenn `podman pull nginx` ausgeführt wird und mehrere Registries konfiguriert sind, fragt Podman interaktiv welche Registry gemeint ist — es sei denn, ein Alias ist konfiguriert. In CI/CD-Umgebungen immer vollständige Image-Namen angeben: `docker.io/library/nginx:latest`.

---

## Podman Socket (Docker-API-Kompatibilität)

Podman kann einen Unix-Socket bereitstellen, der die Docker REST API emuliert:

```bash
# Rootless: systemd user unit
systemctl --user enable --now podman.socket

# Socket-Pfad
echo $XDG_RUNTIME_DIR/podman/podman.sock
# z.B. /run/user/1000/podman/podman.sock

# Docker-Tools auf Podman umleiten
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
docker ps    # nutzt jetzt Podman-Backend
docker-compose up -d  # nutzt jetzt Podman
```

Rootful Socket:
```bash
systemctl enable --now podman.socket
# Socket: /run/podman/podman.sock
export DOCKER_HOST=unix:///run/podman/podman.sock
```

---

## Verwandte Notizen

- [[Was ist Podman]] — Konzept und Einstieg
- [[Podman vs Docker]] — Detaillierter Vergleich
- [[Rootless Container]] — User Namespaces und UID-Mapping
- [[Podman Networking]] — Netavark und Pasta im Detail
