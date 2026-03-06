---
tags: [podman, rootless, user-namespaces, subuid, subgid, cgroups, sicherheit, unprivileged]
aliases: ["Rootless", "Podman Rootless", "Unprivileged Container"]
---

# Rootless Container

> [!ABSTRACT]
> Rootless Container laufen vollständig ohne Root-Rechte — der Container-Prozess erscheint im Container als UID 0, ist auf dem Host aber der normale Benutzer. Technisch basiert das auf Linux User Namespaces und subuid/subgid-Mappings. cgroups v2 ist für vollständige Resource-Limits erforderlich.

---

## Was bedeutet "rootless"?

```
Im Container:  UID 0 (root) — kann alles im Container
Auf dem Host:  UID 1000 (max) — normaler Benutzer, keine Root-Rechte
```

Root im Container ist **kein Root auf dem Host**. Das ist der fundamentale Sicherheitsgewinn von rootless Containern.

**Vergleich: rootful vs. rootless**

| Aspekt | Rootful | Rootless |
|--------|---------|---------|
| **Ausführung** | Als Root (oder docker-Gruppe) | Als normaler Benutzer |
| **Container-Prozess auf Host** | Root (UID 0) | Benutzer-UID (z.B. 1000) |
| **Angriffsfläche** | Hoch — Root-Exploit = Host-Kompromittierung | Niedrig — kein Root-Zugriff |
| **Ports < 1024** | ✅ Möglich | ❌ Eingeschränkt |
| **Macvlan-Networking** | ✅ | ❌ |
| **cgroups v1 Resource-Limits** | ✅ | ❌ (v2 erforderlich) |
| **Storage** | `/var/lib/containers/` | `~/.local/share/containers/` |
| **Netzwerk-Stack** | Netavark (Bridge) | Pasta (User-Space) |

---

## User Namespaces — das Fundament

Linux User Namespaces ermöglichen die UID-Remapping zwischen Host und Container:

```
Container-UID 0  (root)     → Host-UID 1000 (max)
Container-UID 1              → Host-UID 100001
Container-UID 2              → Host-UID 100002
...
Container-UID 65534          → Host-UID 165534
```

Das Mapping wird durch `/etc/subuid` und `/etc/subgid` definiert.

```bash
# User-Namespace-Mapping des aktuellen Users anzeigen
podman unshare cat /proc/self/uid_map
# Output:
#          0       1000          1   ← UID 0 im Namespace = UID 1000 auf Host
#          1     100000      65536   ← UID 1-65536 = Host-UID 100000-165535
```

---

## subuid und subgid konfigurieren

```bash
# Aktuelles Mapping anzeigen
cat /etc/subuid
# max:100000:65536  → Benutzer "max" kann UIDs 100000–165535 auf Host nutzen

cat /etc/subgid
# max:100000:65536

# Automatisch gesetzt bei useradd (moderne Systeme)
sudo useradd -m neueruser   # → automatisch in subuid/subgid eingetragen

# Manuell hinzufügen
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER

# Podman über neue Mappings informieren (nach Änderungen)
podman system migrate
```

> [!IMPORTANT] Neustart nach Änderungen
> Nach Änderungen an `/etc/subuid` oder `/etc/subgid` muss `podman system migrate` ausgeführt werden. Container müssen neu erstellt werden.

---

## cgroups v2 — Voraussetzung für Resource-Limits

```bash
# cgroups-Version ermitteln
stat -fc %T /sys/fs/cgroup/
# tmpfs     = cgroups v1 (eingeschränkte rootless Resource-Limits)
# cgroup2fs = cgroups v2 ✅ (vollständige Resource-Limits)

# Auf systemd-Systemen
systemctl --version | grep default-hierarchy
# default-hierarchy=unified  ← cgroups v2
```

**cgroups v2 aktivieren:**

```bash
# GRUB-Parameter setzen
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
sudo reboot

# Debian/Ubuntu (GRUB direkt)
# In /etc/default/grub:
# GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=1"
sudo update-grub && sudo reboot
```

---

## Einschränkungen von Rootless

### Ports < 1024

```bash
# Symptom
podman run -p 80:80 nginx
# Error: permission denied: bind: address already in use / cannot bind

# Lösung 1: Höheren Host-Port verwenden
podman run -p 8080:80 nginx   # Host:8080 → Container:80

# Lösung 2: Kernel-Parameter (permanent)
echo "net.ipv4.ip_unprivileged_port_start=80" | \
  sudo tee /etc/sysctl.d/99-podman-ports.conf
sudo sysctl --system
# Danach: podman run -p 80:80 nginx ✅

# Temporär (bis Reboot)
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
```

### Kein Macvlan

Macvlan erfordert das Erstellen von Kernel-Netzwerk-Interfaces — nur als Root möglich. Alternativen:
- Pasta-Networking mit Port-Forwarding
- `--network host` (verliert Isolation)
- Rootful-Modus für diesen spezifischen Container

### OverlayFS

```bash
# Kernel 5.11+ mit User-Namespace-OverlayFS (native, empfohlen)
uname -r   # ≥ 5.11 ✅

# Älter: fuse-overlayfs installieren
sudo dnf install -y fuse-overlayfs   # Fedora/RHEL
sudo apt-get install -y fuse-overlayfs  # Ubuntu/Debian

# Fallback: vfs-Treiber (langsam)
# ~/.config/containers/storage.conf
[storage]
driver = "vfs"
```

---

## Rootless-Setup prüfen und debuggen

```bash
# Rootless-Info
podman info | grep -A5 host | grep -i rootless
podman info --format "{{.Host.Security.Rootless}}"

# User-Namespace-Mapping
podman unshare cat /proc/self/uid_map
podman unshare cat /proc/self/gid_map

# Im User-Namespace arbeiten
podman unshare bash
# → jetzt ist man "root" im User-Namespace
id
# uid=0(root) gid=0(root)
# Auf Host-Perspektive testen:
cat /proc/self/uid_map
exit

# Storage prüfen
podman system info | grep -A10 store

# Rootless-spezifische Informationen
podman info | grep -E "rootless|cgroup|userns"
```

---

## Wann Rootless, wann Rootful?

| Szenario | Empfehlung | Begründung |
|----------|:----------:|-----------|
| Entwicklungsumgebung | ✅ Rootless | Kein sudo nötig, sichere Isolation |
| CI/CD Pipeline (GitHub Actions etc.) | ✅ Rootless | Jobs laufen ohne Root-Rechte |
| Multi-User-System | ✅ Rootless | Jeder User hat eigene isolierte Container |
| Webserver auf Port 80/443 | Rootless + sysctl | `ip_unprivileged_port_start=80` setzen |
| Macvlan-Networking (Pi-hole etc.) | Rootful | Kernel-Interface-Erstellung nötig |
| NFS-Volumes oder spezielle Mounts | Rootful | Root-Rechte für Mount-Operationen |
| Produktionsserver (Single-User) | Rootless | Sicherheit hat Vorrang |
| Ceph/GlusterFS Storage-Integration | Rootful | Kernel-Storage-Treiber |

---

## Rootless aktivieren / wechseln

```bash
# Podman läuft standardmäßig rootless wenn als normaler User gestartet
podman run nginx   # rootless

# Rootful explizit (als root oder sudo)
sudo podman run nginx   # rootful

# Prüfen welcher Modus aktiv ist
podman info | grep rootless
```

> [!TIP] Rootful und Rootless sind getrennte Welten
> Rootful und Rootless Container haben **separate** Image-Caches, Volumes und Netzwerke. Ein Image das rootful gepullt wurde ist nicht rootless verfügbar — und umgekehrt.

---

## lingering — Container nach Logout am Laufen halten

Standardmäßig werden User-Session-Prozesse (inkl. Container) beim Logout beendet.

```bash
# lingering aktivieren
sudo loginctl enable-linger $USER

# Status prüfen
loginctl show-user $USER | grep Linger
# Linger=yes ✅

# lingering deaktivieren
sudo loginctl disable-linger $USER
```

---

## Verwandte Notizen

- [[Podman Sicherheit]] — SELinux, Seccomp, Capabilities
- [[Volumes und Storage]] — Rootless Storage-Pfade und SELinux-Labels
- [[Podman Networking]] — Pasta-Networking für rootless
- [[Podman Installation]] — subuid/subgid Setup bei der Installation
