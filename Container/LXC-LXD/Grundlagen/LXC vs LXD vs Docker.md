---
tags: [lxc, lxd, docker, vergleich, container, vm]
aliases: ["LXC Docker Vergleich", "Systemcontainer vs Anwendungscontainer"]
---

# LXC / LXD vs. Docker vs. VMs

> [!ABSTRACT]
> Systemcontainer (LXC/LXD), Anwendungscontainer (Docker) und virtuelle Maschinen lösen ähnliche Probleme mit unterschiedlichen Ansätzen. Dieser Artikel erklärt die Unterschiede, Stärken und Schwächen — damit du die richtige Technologie für deinen Anwendungsfall wählst.

---

## Das Spektrum der Virtualisierung

```
Bare Metal → VM (Hypervisor) → Systemcontainer (LXD) → Anwendungscontainer (Docker)
←─────── mehr Isolation ────────────────────────── weniger Overhead ──────────────→
```

Alle vier teilen sich denselben Linux-Kernel auf dem Host — nur VMs auf einem Hypervisor haben einen eigenen Kernel.

---

## Kernunterschied: System vs. Anwendungscontainer

**Systemcontainer** (LXC/LXD) simulieren ein vollständiges Betriebssystem:

```
┌─────────────────────┐
│  Container (LXD)    │
│  ┌───────────────┐  │
│  │    systemd    │  │  ← Init-System läuft
│  │    sshd       │  │  ← Mehrere Dienste
│  │    nginx      │  │
│  │    postgresql │  │
│  └───────────────┘  │
│  eigene IPs, Users  │
└─────────────────────┘
```

**Anwendungscontainer** (Docker) kapseln einen einzelnen Prozess:

```
┌─────────────────────┐
│  Container (Docker) │
│  ┌───────────────┐  │
│  │  nginx PID 1  │  │  ← Genau ein Prozess
│  └───────────────┘  │
│  ephemer, immutable │
└─────────────────────┘
```

---

## Großer Vergleich

| Merkmal | VMs | LXD/LXC | Docker |
|---------|-----|---------|--------|
| **Kernel** | Eigener Kernel | Host-Kernel | Host-Kernel |
| **Init-System** | Vollständig | Vollständig (systemd) | Kein Init |
| **Startzeit** | 30–120 Sekunden | 2–5 Sekunden | < 1 Sekunde |
| **RAM-Overhead** | 256 MB – mehrere GB | 5–30 MB | ~1 MB |
| **Disk-Overhead** | GB (volles OS-Image) | MB (nur FS-Diffs) | MB (Layer) |
| **Isolation** | Sehr stark (Hardware-Level) | Stark (Namespace) | Mittel (Prozess-Level) |
| **Mehrere Dienste** | ✅ | ✅ | ❌ (Anti-Pattern) |
| **SSH-Zugang** | ✅ Standard | ✅ Standard | ⚠️ Möglich, unüblich |
| **Persistent** | ✅ | ✅ | ⚠️ Volumes nötig |
| **Cross-Platform** | ✅ (Hypervisor) | ❌ (nur Linux) | ✅ |
| **Live Migration** | ✅ (vMotion, etc.) | ✅ (LXD Cluster) | ❌ |
| **Snapshots** | ✅ | ✅ | ⚠️ Nur mit Volumes |
| **DevOps-Integration** | ❌ | ⚠️ begrenzt | ✅ Kubernetes, CI/CD |
| **Ökosystem** | vSphere, KVM, etc. | Ubuntu, MicroCloud | Riesig |

---

## Wann welche Technologie?

### Virtuelle Maschinen — wenn du brauchst:

```
✅ Vollständige Hardware-Emulation (andere Kernel, Windows)
✅ Maximale Isolation (Security-kritische Tenants)
✅ Anderes Betriebssystem als der Host
✅ Enterprise-Features (vMotion, vSAN, HA-Failover)
```

### LXD/LXC — wenn du brauchst:

```
✅ VM-ähnliche Erfahrung ohne den Overhead
✅ Legacy-Anwendungen (mehrere Dienste, Init-abhängig)
✅ Günstige Multi-Tenant-Umgebungen
✅ Entwicklungs- und Test-Umgebungen
✅ Homelab-Infrastruktur (DNS, NFS, Datenbanken)
✅ Migration von VMs zu Containern
✅ Build-Umgebungen mit persistentem State
```

### Docker — wenn du brauchst:

```
✅ Stateless Microservices
✅ CI/CD-Pipelines (Build, Test, Deploy)
✅ Kubernetes-Workloads
✅ Portabilität zwischen Betriebssystemen
✅ Container-Registry und Image-Lifecycle
✅ Schnelle horizontale Skalierung
```

---

## LXD vs. Docker: Detaillierter Vergleich

### Philosophie

| Aspekt | LXD | Docker |
|--------|-----|--------|
| **Einheit** | System (wie VM) | Prozess (wie App) |
| **Lebensdauer** | Langlebig, persistent | Kurzlebig, ephemer |
| **Änderungen** | In-Place (apt install) | Neues Image bauen |
| **Netzwerk** | Festes IP, eigenes Interface | Port-Mapping, Overlay-Netz |
| **Update-Workflow** | SSH + apt upgrade | docker pull + restart |

### Networking

```
LXD:
  Container bekommt eigene MAC und IP im Bridge-Netz
  Zugriff: direkt per IP (kein Port-Mapping nötig)

Docker:
  Container hat interne IP (172.17.x.x)
  Extern erreichbar nur via Port-Mapping (-p 8080:80)
```

### Dateisystem

```
LXD:
  Persistentes, veränderliches Dateisystem
  Änderungen bleiben bestehen
  Snapshots möglich

Docker:
  Immutable Image (unveränderlich)
  Änderungen gehen bei Container-Neustart verloren
  Persistenz nur via Volumes
```

---

## Kombinierter Einsatz

LXD und Docker schließen sich nicht aus — viele Setups kombinieren beide:

```
┌──────────────────────────────────────┐
│         LXD Systemcontainer          │
│  ┌───────────┐  ┌──────────────────┐ │
│  │  LXD CT 1 │  │    LXD CT 2      │ │
│  │  Infra    │  │  Docker-Host     │ │
│  │  DNS/NFS  │  │  ┌────────────┐  │ │
│  └───────────┘  │  │ App-Cont.  │  │ │
│                 │  └────────────┘  │ │
│                 └──────────────────┘ │
└──────────────────────────────────────┘
```

> [!TIP]
> **Typisches Homelab-Setup:** LXD für Infrastruktur-Dienste (Pi-hole, Nextcloud, Gitea), Docker innerhalb eines LXD-Containers für stateless Anwendungen.

---

## LXD vs. Incus

Seit dem Community-Split 2023 gibt es zwei Varianten:

| | LXD (Canonical) | Incus (Community) |
|--|-----------------|-------------------|
| **Maintainer** | Canonical | Linux Containers (Stéphane Graber) |
| **Lizenz** | Apache 2.0 | Apache 2.0 |
| **CLA** | ✅ Canonical CLA | ❌ kein CLA |
| **Fokus** | Ubuntu, Enterprise | Community, distro-agnostisch |
| **Version (2026)** | 6.x / 5.21 LTS | 6.21+ |
| **Images** | ubuntu.com | images.linuxcontainers.org |
| **Empfohlen für** | Ubuntu Pro / MicroCloud | Freie Open-Source-Setups |

> [!NOTE]
> Die Befehle von LXD und Incus sind weitgehend identisch. Migration von LXD zu Incus ist per `lxd-to-incus` Tool möglich.

---

## Ressourcenverbrauch im Vergleich (Beispiel)

| Szenario | VMs | LXD | Docker |
|----------|-----|-----|--------|
| 10 Ubuntu-Instanzen | ~5–10 GB RAM | ~500 MB RAM | ~100 MB RAM |
| Startzeit | 30–60s | 2–3s | < 1s |
| Disk (Ubuntu) | ~8 GB/VM | ~500 MB/CT | ~75 MB/Image |

> [!NOTE]
> LXD-Container teilen sich read-only den Basis-Image-Layer — tatsächlicher Verbrauch geringer als Rohdaten vermuten lassen.

---

## Verwandte Notizen

- [[Was ist LXC-LXD]] — LXC/LXD Grundlagen
- [[LXC-LXD Architektur]] — Technische Architektur
- [[Docker Index]] — Docker-Dokumentation
- [[LXD Sicherheit]] — Isolation und Absicherung
