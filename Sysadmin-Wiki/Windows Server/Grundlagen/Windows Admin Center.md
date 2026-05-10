---
tags: [windows-server, windows-admin-center, wac, web-ui, management, portal, gateway]
aliases: ["Windows Admin Center", "WAC", "Honolulu"]
---

# Windows Admin Center

> [!ABSTRACT]
> **Windows Admin Center (WAC)** ist die moderne, browserbasierte Verwaltungsoberfläche für Windows Server — kostenlos, kein Cloud-Abhängigkeit. Es ersetzt viele alte MMC-Snap-ins und ermöglicht zentrale Verwaltung von Servern, Hyper-V-Hosts, Clustern und Azure-Diensten aus einem Browser-Tab.

---

## Was ist Windows Admin Center?

- Kostenlos (in Windows Server-Lizenz enthalten)
- Läuft lokal im Browser (kein Remote-Zugriff zu Microsoft)
- Moderner Ersatz für: Server Manager, MMC, Hyper-V Manager, Failover Cluster Manager
- Erweiterbar via Extensions
- Unterstützt: Windows Server 2016–2025, Windows 10/11 (PC-Verwaltung), Azure

---

## Installation

### Download

```
https://aka.ms/WACDownload
→ WindowsAdminCenter.exe (ca. 50 MB)
```

### Installationsmodi

| Modus | Beschreibung | Geeignet für |
|-------|-------------|-------------|
| **Gateway-Modus** | WAC auf dediziertem Server, Zugriff im Netz | Produktion |
| **Desktop-Modus** | WAC auf lokalem PC (Port 6516) | Entwicklung, Test |

```powershell
# Gateway-Modus Installation (auf dediziertem Server)
# WindowsAdminCenter.exe herunterladen, dann:
msiexec /i WindowsAdminCenter.msi `
    /qn `
    /L*v log.txt `
    SME_PORT=443 `
    SSL_CERTIFICATE_OPTION=generate

# Nach Installation im Browser öffnen:
# https://SERVERNAME
```

```powershell
# Desktop-Modus (auf Windows 10/11)
# Installer ausführen → Port 6516
# Dann: https://localhost:6516
```

---

## Erste Schritte

### Browser öffnen

```
https://SERVER-NAME         (Gateway-Modus)
https://localhost:6516      (Desktop-Modus)

Anmeldung: Windows-Credentials des Administrators
```

### Server hinzufügen

```
Startseite → Hinzufügen → Server

Methoden:
- Manuell: Hostname oder IP
- Active Directory: Suche nach Servern
- Import: CSV-Datei mit Hostnamen
```

---

## Hauptfunktionen

### Übersicht (pro Server)

```
Dashboard zeigt live:
- CPU-Auslastung (Graph)
- Speicher-Auslastung
- Netzwerk-Durchsatz
- Datenträger-Aktivität
```

### Wichtige Werkzeuge in WAC

| Tool | Funktion |
|------|----------|
| **Übersicht** | Ressourcen-Dashboard, Neustart, Herunterfahren |
| **Zertifikate** | Zertifikatsspeicher verwalten |
| **Dienste** | Dienste starten/stoppen/konfigurieren |
| **Ereignisse** | Eventlog durchsuchen und filtern |
| **Dateien** | Dateisystem-Browser (mit Upload/Download) |
| **Firewall** | Windows-Firewall-Regeln |
| **Lokale Benutzer** | Lokale Accounts verwalten |
| **Netzwerk** | NICs, IP-Konfiguration |
| **PowerShell** | Browser-integriertes PowerShell-Terminal |
| **Prozesse** | Task-Manager im Browser |
| **Registrierung** | Regedit im Browser |
| **Remote Desktop** | RDP direkt im Browser (HTML5) |
| **Rollen & Features** | Serverrollen hinzufügen/entfernen |
| **Speicher** | Datenträger, Volumes, Freigaben |
| **Updates** | Windows Update verwalten |

---

## Extensions installieren

```
Einstellungen → Extensions → Verfügbare Extensions

Wichtige Extensions:
- Hyper-V (VM-Verwaltung)
- Failover Cluster Manager
- Storage Replica
- Azure Hybrid Center
- Active Directory (AD-Verwaltung)
- DHCP (DHCP-Server-Verwaltung)
- DNS (DNS-Server-Verwaltung)
```

---

## Azure-Integration

WAC kann Windows Server mit Azure-Diensten verbinden:

```
Server → Azure → Verbinden

Dienste:
- Azure Arc (zentrale Verwaltung in Azure Portal)
- Azure Backup (Server-Sicherung in Azure)
- Azure Site Recovery (Disaster Recovery)
- Azure Monitor (Metriken und Logs)
- Azure Security Center
- Azure File Sync
```

---

## WAC per HTTPS mit eigenem Zertifikat

```powershell
# Zertifikat-Thumbprint aus dem Zertifikatsspeicher
$cert = Get-ChildItem -Path Cert:\LocalMachine\My |
    Where-Object {$_.Subject -like "*server.firma.local*"}

# WAC mit eigenem Zertifikat neu installieren
msiexec /i WindowsAdminCenter.msi `
    /qn `
    SME_PORT=443 `
    SME_THUMBPRINT=$($cert.Thumbprint) `
    SSL_CERTIFICATE_OPTION=installed
```

---

## Zugriffsteuerung

```powershell
# WAC-Gruppen in lokaler Sicherheitsrichtlinie
# (werden bei Installation erstellt)
# "Windows Admin Center Administrators"  → voller Zugriff
# "Windows Admin Center Users"           → eingeschränkter Zugriff

# Benutzer hinzufügen
Add-LocalGroupMember -Group "Windows Admin Center Administrators" `
    -Member "FIRMA\IT-Admins"
```

---

## PowerShell im Browser

WAC bietet ein vollständiges PowerShell-Terminal direkt im Browser:

```
Server auswählen → Tools → PowerShell

Vorteile:
- Kein separates RDP nötig
- Läuft als angemeldeter Benutzer
- Vollständige PowerShell-Funktionalität
- Copy/Paste funktioniert
```

---

## WAC vs. Server Manager

| Funktion | Server Manager | Windows Admin Center |
|----------|:--------------:|:-------------------:|
| Rollen installieren | ✅ | ✅ |
| Remote-Server | ✅ (begrenzt) | ✅ (zentral) |
| Cluster-Verwaltung | ❌ | ✅ |
| Hyper-V | ❌ | ✅ (Extension) |
| Browser-basiert | ❌ | ✅ |
| Azure-Integration | ❌ | ✅ |
| Live-Performance | ❌ | ✅ |
| PowerShell-Terminal | ❌ | ✅ |
| RDP im Browser | ❌ | ✅ |

---

## Verwandte Notizen

- [[Server Manager]] — Klassische Verwaltungskonsole (GUI)
- [[PowerShell für Admins]] — Command-Line-Verwaltung
- [[Installation Hyper-V (Windows Server 2022)|Hyper-V Verwaltung]] — VMs über WAC verwalten
- [[Windows Server Installation]] — WAC nach der Installation einrichten
