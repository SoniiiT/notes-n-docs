---
tags: [lxc, lxd, architektur, daemon, rest-api, cluster]
aliases: ["LXD Architektur", "LXC Architektur"]
---

# LXC-LXD Architektur

> [!ABSTRACT]
> LXD besteht aus einem Daemon (`lxd`), einer REST-API, einem CLI-Client (`lxc`) und mehreren Subsystemen für Storage, Netzwerk und Clustering. Das Verständnis der Architektur hilft bei Konfiguration, Debugging und Cluster-Betrieb.

---

## Überblick

```
┌─────────────────────────────────────────────────────┐
│                   Benutzer / Tooling                │
│  lxc CLI   REST-API   Terraform   Ansible   Web-UI  │
└───────────────────────┬─────────────────────────────┘
                        │ HTTPS / Unix Socket
┌───────────────────────▼─────────────────────────────┐
│                    LXD Daemon                       │
│                                                     │
│  ┌──────────┐ ┌──────────┐ ┌───────────────────┐   │
│  │ Instance │ │ Storage  │ │     Netzwerk       │   │
│  │ Manager  │ │ Manager  │ │     Manager        │   │
│  └────┬─────┘ └────┬─────┘ └────────┬──────────┘   │
│       │            │                │               │
│  ┌────▼─────┐ ┌────▼───────┐ ┌─────▼────────────┐  │
│  │   LXC    │ │ZFS/btrfs   │ │ Bridge/OVN/macvlan│  │
│  │  (Cont.) │ │LVM/dir/Ceph│ │                  │  │
│  │   QEMU   │ └────────────┘ └──────────────────┘  │
│  │  (VMs)   │                                       │
│  └──────────┘                                       │
└─────────────────────────────────────────────────────┘
```

---

## Komponenten im Detail

### LXD Daemon (`lxd`)

Der Daemon ist das Herzstück:

- läuft als Systemdienst (`systemctl status snap.lxd.daemon`)
- stellt eine **Unix-Socket** (`/var/snap/lxd/common/lxd/unix.socket`) zur Verfügung
- optional: **HTTPS-Endpunkt** auf Port `8443` für Remote-Zugriff und Clustering
- verwaltet alle Ressourcen: Container, VMs, Images, Storage, Netzwerk, Profile

```bash
# Daemon-Status prüfen
systemctl status snap.lxd.daemon

# Daemon-Log anzeigen
journalctl -u snap.lxd.daemon -f

# Daemon-Konfiguration
lxc config show
```

### LXC CLI (`lxc`)

Der Client kommuniziert ausschließlich über die REST-API:

```bash
# Lokale Verbindung (Unix-Socket)
lxc list

# Remote-Verbindung (HTTPS)
lxc list myremote:

# Direkte API-Abfrage
lxc query /1.0/instances
```

> [!TIP]
> `lxc --debug <befehl>` zeigt alle REST-API-Aufrufe — ideal zum Lernen der API-Struktur.

---

## REST-API

Die LXD REST-API ist das Fundament aller Operationen:

| Aspekt | Details |
|--------|---------|
| **Protokoll** | HTTPS (TLS) oder Unix-Socket |
| **Port** | 8443 (Remote) |
| **Auth** | Client-Zertifikate oder OIDC (LXD 6.x) |
| **Format** | JSON |
| **Versionierung** | `/1.0/` — stabile API |

**Wichtige API-Endpunkte:**

| Endpunkt | Beschreibung |
|----------|-------------|
| `/1.0` | Server-Info, Konfiguration |
| `/1.0/instances` | Container und VMs |
| `/1.0/images` | Image-Verwaltung |
| `/1.0/networks` | Netzwerk-Konfiguration |
| `/1.0/storage-pools` | Storage Pools |
| `/1.0/profiles` | Profile |
| `/1.0/cluster` | Cluster-Member |

```bash
# API direkt abfragen
lxc query /1.0
lxc query /1.0/instances
lxc query /1.0/instances/mycontainer

# API mit curl (Unix-Socket)
curl --unix-socket /var/snap/lxd/common/lxd/unix.socket \
     a/1.0/instances
```

---

## Instance-Typen

LXD unterstützt zwei Instance-Typen:

### Systemcontainer (Typ: `container`)

```
LXD Daemon
    ↓
liblxc (LXC-Bibliothek)
    ↓
Kernel-Namespaces + cgroups
    ↓
Container-Prozesse
```

- teilen sich den Host-Kernel
- Start in 1–5 Sekunden
- minimaler Overhead

### Virtuelle Maschinen (Typ: `virtual-machine`)

```
LXD Daemon
    ↓
QEMU/KVM
    ↓
Eigener Kernel + eigenes Userspace
    ↓
VM-Prozesse
```

- eigener Kernel (stärkere Isolation)
- Start in 10–30 Sekunden
- höherer Overhead
- für Windows-Guests oder kernel-spezifische Workloads

```bash
# Container starten
lxc launch ubuntu:24.04 mycontainer

# VM starten
lxc launch ubuntu:24.04 myvm --vm
```

---

## Datenbank und State

LXD verwendet **SQLite** (lokal) bzw. **dqlite** (Cluster):

```
/var/snap/lxd/common/lxd/
├── database/
│   ├── global/      ← dqlite Cluster-Datenbank
│   └── local/       ← lokale SQLite (Hilfsdaten)
├── unix.socket      ← lokaler API-Socket
├── server.crt       ← TLS-Zertifikat
└── logs/            ← Container-Logs
```

**dqlite** ist eine verteilte SQLite-Implementierung mit Raft-Konsensus — garantiert Konsistenz im Cluster ohne externe Datenbank (kein etcd oder Postgres nötig).

---

## Cluster-Architektur

```
┌─────────────────────────────────────────────────────────┐
│                    LXD Cluster                          │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │    │
│  │  (Leader)   │  │  (Follower) │  │  (Follower) │    │
│  │             │  │             │  │             │    │
│  │  lxd daemon │  │  lxd daemon │  │  lxd daemon │    │
│  │  dqlite     │◄─►  dqlite     │◄─►  dqlite     │    │
│  │  (Raft)     │  │  (Raft)     │  │  (Raft)     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│        │8443            │8443            │8443          │
│        └────────────────┴────────────────┘             │
│                  Shared API                             │
└─────────────────────────────────────────────────────────┘
```

- **Kein externer Load Balancer nötig** — jeder Node kennt den Leader
- **Mindest-Quorum:** 3 Nodes (Raft-Konsensus)
- **Shared Storage:** Ceph oder NFS für Container-Migration zwischen Nodes

---

## Subsysteme

### Storage Manager

Verwaltet Storage Pools und Volumes:

```
LXD Storage Manager
├── Pool-Backend-Auswahl (ZFS, btrfs, LVM, dir, Ceph)
├── Volume-Lifecycle (create, clone, snapshot, delete)
└── Image-Caching (lokale Kopien von Remote-Images)
```

### Netzwerk Manager

```
LXD Netzwerk Manager
├── Managed Bridges (lxdbr0, etc.)
├── macvlan / ipvlan
├── SR-IOV
├── OVN (Open Virtual Network) für Cluster
└── Proxy-Devices (Port-Forwarding)
```

### Image Manager

```
LXD Image Manager
├── Remote-Server (images:, ubuntu:, etc.)
├── Lokaler Cache
├── Fingerprint-Verifikation
└── Auto-Update (täglich optional)
```

---

## Sicherheitsarchitektur

```
┌─────────────────────────────────────┐
│          LXD Daemon                 │
│                                     │
│  AppArmor-Profile pro Container     │
│  Seccomp-Filter (syscall-Block)     │
│  UID/GID-Remapping (unprivileged)   │
│  Capabilities-Beschränkung         │
│                                     │
│  ┌──────────────────────────────┐   │
│  │      Container               │   │
│  │  UID 0 → Host UID 100000    │   │
│  │  UID 1000 → Host UID 101000 │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

---

## Datenfluss bei `lxc launch`

```
lxc launch ubuntu:24.04 mycontainer
    │
    ▼
lxc CLI
    │ REST POST /1.0/instances
    ▼
LXD Daemon
    │
    ├─ Image vom Remote-Server laden (falls nicht gecacht)
    ├─ Storage-Volume erstellen (ZFS-Dataset o.ä.)
    ├─ Netzwerk konfigurieren (veth-Pair, Bridge)
    ├─ AppArmor-Profil laden
    ├─ cgroup-Limits setzen
    └─ LXC-Container starten
            │
            ▼
        Container-Init (systemd)
```

---

## Verwandte Notizen

- [[Was ist LXC-LXD]] — Grundlagen und Konzepte
- [[LXD Netzwerk]] — Netzwerk-Subsystem im Detail
- [[Storage Pools]] — Storage-Backends
- [[LXD Cluster Betrieb]] — Cluster-Setup und -Betrieb
