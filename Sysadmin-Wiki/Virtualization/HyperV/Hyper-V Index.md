---
tags:
  - index
  - virtualization
  - hyper-v
---

# Hyper-V Index

> [!ABSTRACT] Übersicht
> Vollständige Dokumentation zu Microsoft Hyper-V — von der Grundlagentheorie über Installation und VM-Verwaltung bis hin zu erweiterten Features wie Live Migration und Hyper-V Replica. Geeignet für Einsteiger und erfahrene Administratoren.

---

## 🗂️ Grundlagen

| Datei | Beschreibung |
|-------|-------------|
| [[Was ist Hyper-V]] | Architektur, Hypervisor-Typen, Versionsgeschichte, Vergleich mit VMware/KVM |

---

## ⚙️ Installation

| Datei | Beschreibung |
|-------|-------------|
| [[Installation Hyper-V (Windows Server 2022)]] | Hyper-V-Rolle installieren, Erstkonfiguration, PowerShell-Setup |

---

## 📚 Konzepte

| Datei | Beschreibung |
|-------|-------------|
| [[VM Generationen und Konfigurationsversionen]] | Gen1 vs. Gen2, BIOS/UEFI, Konfigurationsversionen 5.0–12.0, Upgrade-Workflow |
| [[Hyper-V VM Verwaltung]] | VM erstellen, Dynamic Memory, Checkpoints, Disk erweitern, Templates, Integration Services |
| [[Hyper-V Netzwerk]] | Virtuelle Switches (External/Internal/Private), NIC Teaming, SR-IOV, VLANs, QoS |
| [[SET und NIC Teaming]] | SET vs. LBFO, Deprecation, RDMA/SMB Direct, Migration, Converged Fabric |
| [[Hyper-V Storage]] | VHD vs. VHDX, Disk-Typen, VHDX-Operationen, Shared VHDX, Pass-through, SMB, iSCSI |

---

## 🚀 Erweiterte Features

| Datei | Beschreibung |
|-------|-------------|
| [[Hyper-V Replica und Live Migration]] | Hyper-V Replica (Disaster Recovery), Live Migration, Quick Migration, Storage Migration |

---

## 🛠️ Betrieb

| Datei | Beschreibung |
|-------|-------------|
| [[Hyper-V PowerShell Cheatsheet]] | Vollständige PowerShell-Referenz für alle Hyper-V-Operationen |
| [[Hyper-V Best Practices]] | Empfehlungen für Host, VMs, Storage, Netzwerk, Sicherheit und Updates |
| [[Hyper-V Troubleshooting]] | Diagnose und Lösung häufiger Probleme (VM startet nicht, Netzwerk, Performance) |

---

## 🗺️ Schnelleinstieg

```
Neu bei Hyper-V?
  └─► [[Was ist Hyper-V]]                         ← Konzepte verstehen
       └─► [[Installation Hyper-V (Windows Server 2022)]]  ← System einrichten
            └─► [[VM Generationen und Konfigurationsversionen]]  ← Generation wählen
                 └─► [[Hyper-V VM Verwaltung]]     ← Erste VM erstellen

Bereits produktiv?
  ├─► [[Hyper-V Netzwerk]]                         ← VLANs, Teaming, SR-IOV
  ├─► [[Hyper-V Storage]]                          ← VHDX, iSCSI, Storage QoS
  ├─► [[Hyper-V Replica und Live Migration]]       ← HA und DR einrichten
  └─► [[Hyper-V Best Practices]]                   ← Betrieb optimieren

Etwas geht nicht?
  └─► [[Hyper-V Troubleshooting]]                  ← Diagnose & Lösungen
```

---

## Verwandte Notizen

- [[ESXi Index]] — VMware ESXi Dokumentation
- [[Was ist Hyper-V]]
