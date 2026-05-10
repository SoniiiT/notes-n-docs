---
tags: [lxc, lxd, grundlagen, container, linux, systemcontainer]
aliases: ["LXC Grundlagen", "LXD Grundlagen"]
---

# Was ist LXC und LXD?

> [!ABSTRACT]
> LXC (Linux Containers) und LXD sind Systemcontainer-Technologien, die eine vollständige Linux-Umgebung in einem Container bereitstellen — ohne die Overhead einer vollständigen virtuellen Maschine. Sie nutzen Kernel-Features wie Namespaces und cgroups für Isolation und Ressourcenverwaltung.

---

## Das Grundprinzip

Der Linux-Kernel bietet seit Jahren Mechanismen zur Isolation von Prozessen:

- **Namespaces** trennen Sichtbarkeit: PID, Netzwerk, Dateisystem, Benutzer, Hostname
- **cgroups** (Control Groups) begrenzen Ressourcen: CPU, RAM, I/O, Netzwerkbandbreite
- **Capabilities** steuern Kernel-Privilegien feinkörnig

LXC kombiniert diese Features zu einem vollständigen Systemcontainer. LXD fügt eine Management-Schicht darüber hinzu.

```
Anwendung
    ↓
Systemcontainer (LXC/LXD)
    ↓
Linux-Kernel (Namespaces, cgroups)
    ↓
Hardware (oder Hypervisor)
```

---

## LXC — Linux Containers

**LXC** ist das eigentliche Container-Werkzeug auf niedriger Ebene. Es:

- wurde 2008 in den Linux-Kernel integriert
- ist Teil des [Linux Containers](https://linuxcontainers.org) Projekts
- bietet eine C-Bibliothek (`liblxc`) und CLI-Tools
- läuft direkt auf dem Kernel ohne Daemon

**LXC 6.x** ist die aktuelle LTS-Serie (2024+), gepflegt von der Linux-Containers-Community.

### Was ein LXC-Container enthält

Ein LXC-Container ist de facto ein leichtgewichtiger "Mini-Rechner":

- eigener Hostname und PID-Namespace
- eigenes Netzwerk-Interface
- eigenes Root-Dateisystem
- eigener Init-Prozess (systemd, OpenRC, SysV)
- mehrere Benutzer und Dienste gleichzeitig

---

## LXD — Der System-Container-Manager

**LXD** baut auf LXC auf und fügt eine Verwaltungsschicht hinzu:

- REST-API für alle Operationen
- Daemon (`lxd`) als einziger Einstiegspunkt
- Unterstützung für **Systemcontainer** (LXC) **und** echte **VMs** (QEMU)
- Cluster-Fähigkeit für mehrere Hosts
- Profile, Storage Pools, Netzwerkverwaltung

**Aktuelle Versionen (2026):**

| Version | Typ | Maintainer |
|---------|-----|-----------|
| LXD 6.x (6.7+) | Feature-Releases | Canonical |
| LXD 5.21 | LTS | Canonical |
| **Incus 6.x** | Community-Fork | linuxcontainers.org |

> [!IMPORTANT]
> **LXD vs. Incus:** 2023 übernahm Canonical LXD vollständig. Das ursprüngliche Entwicklerteam (Stéphane Graber) gründete daraufhin **Incus** als Community-Fork. Incus (v6.21+) ist heute die empfohlene Wahl für Nutzer, die keine Bindung an Canonical wünschen. LXD bleibt die Wahl für Ubuntu-Enterprise-Umgebungen.

---

## Namespaces im Detail

```
┌─────────────────────────────────────────┐
│              Linux Kernel               │
│                                         │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Container A  │  │ Container B  │    │
│  │ PID ns:      │  │ PID ns:      │    │
│  │   PID 1 init │  │   PID 1 init │    │
│  │   PID 2 app  │  │   PID 2 sshd │    │
│  │ Net ns: eth0 │  │ Net ns: eth0 │    │
│  │ Mnt ns: /    │  │ Mnt ns: /    │    │
│  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────┘
```

| Namespace | Isoliert | Beispiel |
|-----------|---------|---------|
| `pid` | Prozess-IDs | Container hat eigene PID 1 |
| `net` | Netzwerk-Stack | eigene IPs, Routing-Tabellen |
| `mnt` | Dateisystem-Mounts | eigenes `/` Dateisystem |
| `uts` | Hostname | unabhängiger Hostname |
| `ipc` | Inter-Process-Comm. | Shared Memory, Semaphoren |
| `user` | Benutzer-IDs | UID-Remapping (unprivileged) |
| `cgroup` | cgroup-Sichtbarkeit | isolierte Ressourcengruppen |

---

## cgroups — Ressourcenkontrolle

cgroups (Control Groups) begrenzen, was Container verbrauchen dürfen:

```bash
# Beispiel: Container auf 2 CPUs und 1 GB RAM begrenzen
lxc config set mycontainer limits.cpu 2
lxc config set mycontainer limits.memory 1GiB

# Aktuellen Ressourcenverbrauch sehen
lxc info mycontainer
```

**cgroups v2** (Standard seit Ubuntu 22.04+) vereint alle Hierarchien in einem einzigen Baum.

---

## Systemcontainer vs. Anwendungscontainer

| Merkmal | Systemcontainer (LXC/LXD) | Anwendungscontainer (Docker) |
|---------|---------------------------|------------------------------|
| **Startprozess** | Init-System (systemd) | Einzelner Prozess |
| **Mehrere Dienste** | ✅ Ja | ❌ Nein (Anti-Pattern) |
| **Lebensdauer** | Langlebig (wie VMs) | Kurzlebig, ephemer |
| **Dateisystem** | Persistent, veränderlich | Immutable Images |
| **SSH-Zugang** | ✅ Standard | ⚠️ Möglich, aber unüblich |
| **Anwendungsfall** | Infrastruktur, Legacy | Microservices, CI/CD |

---

## Typische Anwendungsfälle

### Wann LXC/LXD einsetzen?

```
✅ Legacy-Anwendungen ohne Container-Support
✅ Mehrere Dienste in einer isolierten Umgebung
✅ Entwicklungsumgebungen (Homelab, Testing)
✅ Multi-Tenant-Hosting (günstige VM-Alternative)
✅ Infrastruktur-Dienste (DNS, NFS, Datenbanken)
✅ CI/CD-Build-Umgebungen mit persistentem State
✅ Migration von VMs zu Containern
```

### Wann besser Docker oder Kubernetes?

```
✅ Stateless Microservices
✅ CI/CD-Pipelines
✅ Kubernetes-Workloads
✅ Cross-Platform (Windows/macOS)
✅ Container-Registry-Workflows
```

---

## Geschichte

```
2006  → Google entwickelt cgroups (im Linux-Kernel 2.6.24)
2008  → LXC 0.1 veröffentlicht (erste Linux-Container-Implementierung)
2013  → Docker erscheint (nutzt anfangs LXC, wechselt später zu libcontainer)
2015  → LXD 0.1 von Canonical veröffentlicht (Management-Layer über LXC)
2016  → LXD 2.0 LTS — stabile REST-API, Snap-Paket
2019  → LXD 3.x — VM-Unterstützung via QEMU
2021  → LXD 4.x — Clustering, OVN-Netzwerk
2023  → Canonical übernimmt LXD vollständig → Community-Fork Incus entsteht
2024  → LXD 6.0 LTS / Incus 6.0 LTS — parallele LTS-Releases
2025+ → Incus 6.x aktive Community-Entwicklung
```

---

## Privileged vs. Unprivileged Container

Ein wichtiges Sicherheitskonzept:

**Privileged Container** (Legacy):
- Container-Root (UID 0) = Host-Root (UID 0)
- Gefährlich: Kernel-Exploit kann Host kompromittieren
- Nur für spezielle Anwendungsfälle

**Unprivileged Container** (Standard):
- UID-Remapping: Container-Root → Host-UID 100000
- Exploit im Container hat keinen Root-Zugriff auf den Host
- Standard seit LXD 3.x

```bash
# Prüfen ob Container unprivileged läuft
lxc config get mycontainer security.privileged
# Ausgabe: false = sicher (unprivileged)
```

---

## Verwandte Notizen

- [[LXC vs LXD vs Docker]] — Detaillierter Technologievergleich
- [[LXC-LXD Architektur]] — Technische Interna
- [[LXD Installation und Setup]] — Erste Schritte
- [[LXD Sicherheit]] — Isolation und Absicherung
