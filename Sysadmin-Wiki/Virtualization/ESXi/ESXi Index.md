---
tags:
  - moc
  - index
  - esxi
  - vsphere
  - vmware
  - virtualization
aliases:
  - ESXi Übersicht
  - ESXi Index
  - vSphere Index
  - VMware Index
---

# 🖥️ ESXi Index

> [!ABSTRACT]
> Zentrale Übersicht aller ESXi- und vSphere-Dokumentationen in dieser Vault — von der Installation über Kernkonzepte wie vCenter und Netzwerk/Storage bis hin zu erweiterten Enterprise-Features (vSAN, NSX, Tanzu) und dem täglichen Betrieb mit Troubleshooting und PowerCLI-Automatisierung.

---

## 📖 Grundlagen

Einstieg in die VMware-Welt: Architektur, Kernkonzepte und Lizenzierung von ESXi und vSphere — inklusive Einordnung gegenüber anderen Hypervisoren.

| Thema | Notiz |
|-------|-------|
| Was ist ESXi / vSphere, Architektur, vMotion, HA, DRS, Lizenzierung | [[ESXi Grundlagen]] |

---

## 🚀 Installation

Schritt-für-Schritt-Anleitung für die ESXi 8.x-Installation auf physischer Hardware — von den Mindestanforderungen über die DCUI-Erstkonfiguration bis zur ersten virtuellen Maschine.

| Thema | Notiz |
|-------|-------|
| ESXi 8.x installieren, DCUI, Host Client, erste VM | [[ESXi Installation]] |

---

## 🧩 Konzepte

Die zentralen Bausteine einer vSphere-Umgebung: Verwaltungsplattform, virtuelle Netzwerke, Speicher-Infrastruktur und VM-Verwaltung.

| Thema | Notiz |
|-------|-------|
| vCenter Server (VCSA), Cluster-Management, HA, DRS, vMotion | [[vCenter]] |
| VSS/VDS vSwitches, Datastores, iSCSI, NFS, VMkernel-Ports | [[ESXi Netzwerk und Storage]] |
| VM erstellen, Disk erweitern, Snapshots, Templates, Klonen, VMware Tools | [[VM Verwaltung]] |

---

## ✨ Erweiterte Features

Enterprise-Erweiterungen des vSphere-Ökosystems: hyperkonvergenter Speicher, Software-Defined Networking, nativer Kubernetes-Betrieb sowie Replikation und Disaster Recovery.

| Thema | Notiz |
|-------|-------|
| Hyperkonvergente Speicherlösung (HCI), verteilter Datastore aus lokalen Disks | [[vSAN]] |
| Software-Defined Networking, Mikrosegmentierung, verteilte Firewall, Load Balancer | [[NSX]] |
| Native Kubernetes auf vSphere, Tanzu Kubernetes Grid, Container-Workloads | [[Tanzu]] |
| Agentenlose asynchrone VM-Replikation, RPO 5 min–24 h | [[vSphere Replication]] |
| DR-Orchestrierung, automatisierter Failover/Failback, Recovery Plans | [[Site Recovery Manager]] |
| Patch- & Update-Management für ESXi-Hosts, VM Hardware und VMware Tools | [[vSphere Lifecycle Manager]] |

---

## 🔧 Betrieb

Alltagswerkzeuge für ESXi-Administratoren: strukturierte Fehlersuche, Shell-Referenz, Best Practices und PowerCLI-Automatisierung.

| Thema | Notiz |
|-------|-------|
| Log-Dateien, ESXi Shell, DCUI Notfallzugriff, häufige Fehler & Lösungen | [[ESXi Troubleshooting]] |
| Bewährte Konfigurationsempfehlungen für Host, VMs, Storage, Netzwerk & Sicherheit | [[ESXi Best Practices]] |
| esxcli, vim-cmd, vmkfstools — Schnellreferenz aller wichtigen Shell-Befehle | [[ESXi CLI Cheatsheet]] |
| PowerShell-Modul für VMware-Automatisierung (ESXi, vCenter, vSAN, NSX) | [[PowerCLI]] |

