---
tags: [moc, index, windows-server, active-directory, powershell, rollen, administration]
aliases: ["Windows Server Übersicht", "Windows Server Index", "WinSrv Index"]
---

# 🪟 Windows Server Index

> [!ABSTRACT]
> Umfassende Dokumentation zu Windows Server 2022 und 2025 — von der Grundinstallation über PowerShell-Administration bis zu allen wichtigen Serverrollen: Active Directory, DNS/DHCP, File Server, RDS, IIS, Hyper-V und mehr.

---

## 📚 Grundlagen

Einstieg in Windows Server: Editionen, Installation und grundlegende Verwaltungswerkzeuge.

| Thema | Notiz |
|-------|-------|
| Editionen, Versionen, Lizenzierung | [[Was ist Windows Server]] |
| Installation und Erstkonfiguration | [[Windows Server Installation]] |
| Server Manager (Rollen & Features) | [[Server Manager]] |
| Windows Admin Center (WAC) | [[Windows Admin Center]] |

---

## ⚡ PowerShell

PowerShell ist das zentrale Verwaltungswerkzeug für Windows Server — unverzichtbar für Automatisierung und Remote-Administration.

| Thema | Notiz |
|-------|-------|
| Grundlagen, Module, Execution Policy | [[PowerShell für Admins]] |
| WinRM, PSSession, Invoke-Command | [[PowerShell Remoting]] |
| Befehlsreferenz & Cheatsheet | [[PowerShell Cheatsheet]] |

---

## 🏛️ Active Directory

Das Rückgrat jeder Windows-Infrastruktur — Benutzer, Gruppen, GPOs und Authentifizierung.

| Thema | Notiz |
|-------|-------|
| Forest, Domain, OU, Trusts | [[AD Grundlagen]] |
| Domain Controller einrichten | [[AD Installation]] |
| Benutzer, Gruppen und OUs | [[Benutzer und Gruppen]] |
| Group Policy Objects (GPO) | [[Group Policy]] |
| Replikation, Kerberos, Fehlersuche | [[AD Troubleshooting]] |

---

## 🌐 DNS und DHCP

Namensauflösung und IP-Adressvergabe — häufig auf demselben Server wie AD.

| Thema | Notiz |
|-------|-------|
| Zonen, Records, Conditional Forwarder | [[Windows DNS]] |
| Scopes, Optionen, Failover | [[Windows DHCP]] |

---

## 📁 File Server

Dateifreigaben, NTFS-Berechtigungen und verteilte Dateisysteme.

| Thema | Notiz |
|-------|-------|
| SMB-Freigaben, Berechtigungen, Audit | [[SMB Freigaben]] |
| ACLs, Vererbung, icacls | [[NTFS Berechtigungen]] |
| DFS Namespace und Replikation | [[DFS]] |

---

## 🖥️ Remote Desktop Services

Terminalserver, RemoteApp und virtuelle Desktops für Remote-Zugriff.

| Thema | Notiz |
|-------|-------|
| Lizenzierung, Rollen, Architektur | [[RDS Grundlagen]] |
| Session Collections, CAL, Profile | [[RDS Konfiguration]] |
| RemoteApp veröffentlichen | [[RemoteApp]] |

---

## 🌍 IIS — Webserver

Microsoft Internet Information Services für Websites und Web-Anwendungen.

| Thema | Notiz |
|-------|-------|
| Sites, App Pools, Bindings, SSL | [[IIS Grundlagen]] |
| URL Rewrite, Reverse Proxy, Authentifizierung | [[IIS Konfiguration]] |

---

## 💻 Hyper-V

> [!NOTE] Eigener Bereich
> Hyper-V ist in separaten Bereichen dokumentiert:
> - **[[Installation Hyper-V (Windows Server 2022)]]** — Installation, Voraussetzungen, Erstkonfiguration
> - **[[Hyper-V Failover Cluster]]** — HA-Cluster, Quorum, CSV, Live Migration

---

## 🔐 PKI — AD Certificate Services

Eigene Zertifizierungsstelle für interne TLS-Zertifikate und Authentifizierung.

| Thema | Notiz |
|-------|-------|
| Root CA, Subordinate CA, Templates, Auto-Enrollment | [[PKI - AD CS]] |
| Zertifikate ausstellen, sperren, erneuern, CRL | [[Zertifikatsverwaltung]] |

---

## ⚙️ Weitere Rollen

| Rolle | Notiz |
|-------|-------|
| Windows Server Update Services | [[WSUS]] |
| RADIUS / Network Policy Server | [[RADIUS - NPS]] |
| Druckerserver und GPO-Deployment | [[Print Server]] |
| Hochverfügbarkeit mit Cluster | [[Failover Clustering]] |

---

## Verwandte Notizen

- [[Installation Hyper-V (Windows Server 2022)]] — Hyper-V in Virtualization
- [[Hyper-V Failover Cluster]] — Hyper-V Cluster in Clustering
- [[AD Grundlagen]] — Active Directory als Herzstück
