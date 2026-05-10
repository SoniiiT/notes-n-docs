---
tags:
  - moc
  - index
  - proxmox
  - virtualization
  - kvm
  - lxc
aliases:
  - Proxmox Übersicht
  - Proxmox Index
  - PVE Index
---

# 🖥️ Proxmox VE Index

> [!ABSTRACT]
> Zentrale Übersicht aller Proxmox VE-Dokumentationen in dieser Vault — von der Installation über Kernkonzepte wie Netzwerk und Storage bis hin zu erweiterten Features wie Cluster, HA und Proxmox Backup Server. Inklusive Troubleshooting-Guide und CLI-Referenz für den täglichen Betrieb.

---

## 📖 Grundlagen

Einstieg in die Proxmox-Welt: Architektur, Hypervisor-Typen, Lizenzmodell und Einordnung gegenüber Hyper-V und ESXi.

| Thema | Notiz |
|-------|-------|
| Was ist Proxmox VE, Architektur, KVM vs. LXC, Lizenzmodell, Vergleich | [[Was ist Proxmox]] |

---

## 🚀 Installation

Schritt-für-Schritt-Anleitung für die Proxmox VE 8.x-Installation — von den Hardware-Mindestanforderungen über den guided Installer bis zur produktionsreifen Erstkonfiguration.

| Thema | Notiz |
|-------|-------|
| Proxmox VE 8.x installieren, Post-Install-Setup, Community Repo, Updates | [[Proxmox Installation]] |

---

## 🧩 Konzepte

Die zentralen Bausteine einer Proxmox-Umgebung: Netzwerk-Bridges, Storage-Backends, VM- und Container-Verwaltung sowie Benutzerverwaltung.

| Thema | Notiz |
|-------|-------|
| Linux Bridges, VLANs, Bonds, OVS, SDN, Firewall | [[Proxmox Netzwerk]] |
| Local, ZFS, Ceph, NFS, iSCSI, Directory-Typen, Storage-Konfiguration | [[Proxmox Storage]] |
| KVM-VMs, LXC-Container, OCI-Container, Cloud-Init, Templates, Snapshots, Migration | [[Proxmox VM und Container Verwaltung]] |
| Realms (PAM, LDAP, AD, OIDC), Benutzer, Gruppen, Rollen, ACLs, API-Tokens, 2FA | [[Proxmox Benutzerverwaltung]] |

---

## ✨ Erweiterte Features

Enterprise-Erweiterungen: Hochverfügbarkeit durch Clustering, automatisierter Failover, ZFS-Replikation, professionelles Backup und GPU-Passthrough.

| Thema | Notiz |
|-------|-------|
| Corosync Cluster, pvecm, HA Manager, Live Migration, Fencing | [[Proxmox Cluster und HA]] |
| ZFS-basierte VM/CT-Replikation auf andere Nodes, Failover, RPO | [[Proxmox Replication]] |
| PBS Installation, Deduplication, Encryption, Backup-Jobs, Restore | [[Proxmox Backup Server]] |
| VFIO, IOMMU, NVIDIA/AMD/Intel Arc, SR-IOV, vGPU, Gaming-VM | [[Proxmox GPU Passthrough]] |

---

## 🔧 Betrieb

Alltagswerkzeuge für Proxmox-Administratoren: strukturierte Fehlersuche, Shell-Referenz und Best Practices für den produktiven Betrieb.

| Thema | Notiz |
|-------|-------|
| qm, pct, pvecm, pvesm — Schnellreferenz aller wichtigen CLI-Befehle | [[Proxmox CLI Cheatsheet]] |
| Bewährte Konfigurationsempfehlungen für Host, VMs, Storage, Netzwerk & Sicherheit | [[Proxmox Best Practices]] |
| RRD-Graphen, InfluxDB/Graphite-Export, Grafana, Notifications (SMTP, Gotify) | [[Proxmox Monitoring]] |
| Log-Dateien, häufige Fehler & Lösungen, Cluster-Probleme, Performance | [[Proxmox Troubleshooting]] |

---

## 🗺️ Schnelleinstieg

```
Neu bei Proxmox?
  └─► [[Was ist Proxmox]]                         ← Konzepte & Features verstehen
       └─► [[Proxmox Installation]]                ← System einrichten
            └─► [[Proxmox Netzwerk]]               ← Netzwerk konfigurieren
                 └─► [[Proxmox Storage]]           ← Storage einrichten
                      └─► [[Proxmox VM und Container Verwaltung]]  ← Erste VM / Container
                           └─► [[Proxmox Benutzerverwaltung]]      ← Benutzer & Rollen

Bereits produktiv?
  ├─► [[Proxmox Cluster und HA]]                   ← Hochverfügbarkeit aufbauen
  ├─► [[Proxmox Replication]]                      ← DR ohne Ceph (ZFS-Replikation)
  ├─► [[Proxmox Backup Server]]                    ← Backup-Strategie umsetzen
  ├─► [[Proxmox Monitoring]]                       ← Metriken & Alerting
  ├─► [[Proxmox GPU Passthrough]]                  ← GPU/PCIe an VM durchreichen
  └─► [[Proxmox Best Practices]]                   ← Betrieb optimieren

Etwas geht nicht?
  └─► [[Proxmox Troubleshooting]]                  ← Diagnose & Lösungen
```

---

## Verwandte Notizen

- [[Hyper-V Index]] — Microsoft Hyper-V Dokumentation
- [[ESXi Index]] — VMware ESXi Dokumentation
