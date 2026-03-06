---
tags: [podman, volumes, storage, bind-mount, overlay, selinux, persistenz, named-volumes]
aliases: ["Podman Volumes", "Podman Storage"]
---

# Volumes und Storage

> [!ABSTRACT]
> Podman unterstützt Named Volumes, Bind Mounts und tmpfs. Rootful-Storage liegt unter `/var/lib/containers/storage/`, rootless unter `~/.local/share/containers/storage/`. Auf RHEL/Fedora sind SELinux-Labels (`:z`/`:Z`) bei Bind Mounts zwingend erforderlich.

---

## Storage-Typen im Überblick

| Typ | Beschreibung | Typischer Use Case |
|-----|-------------|-------------------|
| **Named Volume** | Von Podman verwaltetes Volume | Persistente Daten (DB, Uploads) |
| **Bind Mount** | Host-Verzeichnis eingebunden | Konfiguration, Quellcode, Dev |
| **tmpfs** | In-Memory-Filesystem | Temporäre Dateien, Secrets im RAM |
| **Image-Layer** | Read-only, Teil des Images | Applikations-Binaries |

---

## Named Volumes

Named Volumes werden vollständig von Podman verwaltet — Podman bestimmt den Speicherort auf dem Host.

```bash
# Volume erstellen
podman volume create mein-volume

# Volume mit Optionen
podman volume create \
  --label app=myapp \
  --opt device=tmpfs \
  --opt type=tmpfs \
  --opt o=size=100m \
  tmp-volume

# Volume beim Container-Start nutzen
podman run -d \
  --name db \
  -v mein-volume:/var/lib/postgresql/data:Z \
  postgres:16

# Volumes auflisten
podman volume ls
podman volume ls --filter label=app=myapp

# Volume inspizieren
podman volume inspect mein-volume

# Volume löschen
podman volume rm mein-volume

# Alle ungenutzten Volumes löschen
podman volume prune
```

### Speicherort von Named Volumes

```bash
# Rootful
/var/lib/containers/storage/volumes/<name>/_data/

# Rootless
~/.local/share/containers/storage/volumes/<name>/_data/

# Pfad ermitteln
podman volume inspect mein-volume --format "{{.Mountpoint}}"
```

---

## Bind Mounts

Bind Mounts binden ein Host-Verzeichnis direkt in den Container ein.

```bash
# Absoluter Pfad
podman run -v /home/user/daten:/data nginx

# Aktuelles Verzeichnis
podman run -v $(pwd)/config:/etc/nginx/conf.d nginx

# Read-only
podman run -v /etc/config:/config:ro nginx

# Mehrere Mounts
podman run \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:alpine
```

---

## SELinux-Labels — kritisch auf RHEL/Fedora

> [!WARNING] SELinux und Volume-Mounts
> Auf RHEL, Fedora und CentOS Stream blockiert SELinux standardmäßig den Zugriff auf Host-Verzeichnisse aus Containern. Ohne die richtigen Labels führt das zu `Permission denied`-Fehlern, auch wenn die Linux-Dateiberechtigungen korrekt sind.

| Label | Bedeutung | Wann verwenden |
|-------|-----------|----------------|
| `:z` | **Shared** — mehrere Container dürfen zugreifen | Wenn mehrere Container dasselbe Verzeichnis nutzen |
| `:Z` | **Private** — nur dieser Container darf zugreifen | Standard-Empfehlung für einzelne Container |
| *(kein Label)* | SELinux-Kontext unverändert | Nur wenn Verzeichnis bereits `container_file_t` hat |

```bash
# Shared (mehrere Container)
podman run -v /daten:/data:z nginx
podman run -v /daten:/data:z myapp   # ✅ beide Container können zugreifen

# Private (nur dieser Container)
podman run -v /daten:/data:Z nginx   # ✅ nur nginx kann zugreifen

# Kombiniert mit ro
podman run -v /config:/config:ro,z nginx

# Labels können kombiniert werden
podman run -v /daten:/data:rw,Z nginx
```

> [!TIP] Ubuntu/Debian
> Auf Ubuntu und Debian ist SELinux standardmäßig nicht aktiv — die `:z`/`:Z` Labels werden ignoriert, aber schaden auch nicht. In heterogenen Umgebungen schadet es nicht, sie immer anzugeben.

---

## tmpfs (In-Memory-Storage)

```bash
# tmpfs einbinden
podman run --tmpfs /tmp nginx

# Mit Größenbeschränkung
podman run --tmpfs /tmp:rw,size=100m nginx

# Mehrere tmpfs
podman run \
  --tmpfs /tmp:rw,size=64m \
  --tmpfs /run:rw,size=32m \
  myapp:latest
```

Typische Anwendungsfälle:
- Temporäre Dateien die nicht persistiert werden sollen
- Cache-Verzeichnisse
- Read-Only-Container mit beschreibbaren temporären Pfaden

---

## Read-Only-Container mit schreibbaren Ausnahmen

```bash
# Container-Root-Filesystem read-only
podman run --read-only nginx

# Bestimmte Pfade beschreibbar machen
podman run \
  --read-only \
  --tmpfs /tmp:rw,size=64m \
  --tmpfs /run:rw,size=16m \
  --tmpfs /var/cache/nginx:rw,size=32m \
  nginx:alpine
```

---

## Storage-Driver

```bash
# Aktuellen Storage-Driver anzeigen
podman info | grep -A5 store

# Konfiguration
cat ~/.config/containers/storage.conf   # rootless
cat /etc/containers/storage.conf        # rootful
```

```toml
# storage.conf — Storage-Driver konfigurieren
[storage]
driver = "overlay"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"   # für ältere Kernel
```

| Treiber | Beschreibung | Empfehlung |
|---------|-------------|-----------|
| **overlay** | Kernel-OverlayFS — effizient, Layer-Deduplication | Standard (Kernel 5.11+) |
| **fuse-overlayfs** | User-Space-Overlay — für rootless auf alten Kerneln | Fallback für Kernel < 5.11 |
| **vfs** | Vollständige Kopien — langsam, aber immer kompatibel | Nur als letzter Ausweg |
| **zfs** | ZFS-basierte Layer | Spezialfall |

---

## Volumes zwischen Containern teilen

```bash
# Volume-Container-Pattern
podman run -d \
  --name data-init \
  -v shared-data:/data \
  alpine sh -c "echo 'initialdaten' > /data/config.txt && sleep infinity"

# Anderer Container nutzt dasselbe Volume
podman run -d \
  --name app \
  -v shared-data:/app/data:z \
  myapp:latest

# Oder via --volumes-from
podman run -d \
  --volumes-from data-init \
  myapp:latest
```

---

## Volumes sichern und wiederherstellen

```bash
# Backup eines Named Volumes
podman run --rm \
  -v mein-volume:/data:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/volume-backup-$(date +%Y%m%d).tar.gz -C /data .

# Restore
podman run --rm \
  -v mein-volume:/data:Z \
  -v $(pwd):/backup:ro \
  alpine \
  tar xzf /backup/volume-backup-20260306.tar.gz -C /data

# Volume exportieren (Podman 4.0+)
podman volume export mein-volume --output volume.tar

# Volume importieren
podman volume import mein-volume volume.tar
```

---

## Volumes und Pods

```bash
# Volume beim Pod-Container definieren
podman pod create --name webapp -p 8080:80

podman run -d \
  --pod webapp \
  --name db \
  -v webapp-db:/var/lib/postgresql/data:Z \
  postgres:16

podman run -d \
  --pod webapp \
  --name app \
  -v webapp-uploads:/app/uploads:Z \
  myapp:latest
```

---

## System-Aufräumen

```bash
# Disk-Nutzung anzeigen
podman system df
podman system df -v   # detailliert

# Ungenutztes aufräumen
podman volume prune          # ungenutzte Volumes
podman image prune           # dangling Images
podman container prune       # gestoppte Container
podman system prune          # alles auf einmal
podman system prune -a       # auch ungenutzte Images
```

---

## Verwandte Notizen

- [[Rootless Container]] — Rootless-Einschränkungen beim Storage
- [[Podman Sicherheit]] — SELinux-Labels und Dateiberechtigungen
- [[Podman CLI Cheatsheet]] — Volume-Befehle Referenz
