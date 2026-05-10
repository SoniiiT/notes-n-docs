---
tags: [windows-server, server-manager, rollen, features, gui, dashboard, pools]
aliases: ["Server Manager", "Rollen hinzufügen", "Server-Manager"]
---

# Server Manager

> [!ABSTRACT]
> Der **Server Manager** ist die zentrale Verwaltungskonsole für Windows Server mit Desktop Experience. Er ermöglicht das Hinzufügen und Entfernen von Serverrollen und Features, zeigt Ereignisse und Dienste-Status aller verwalteten Server im Überblick und kann mehrere Remote-Server gleichzeitig verwalten.

---

## Überblick

```
Server Manager öffnen:
- Startet automatisch bei Anmeldung
- Taskleiste: Server-Manager-Symbol
- Start → Server Manager
- PowerShell: ServerManager
```

### Dashboard-Bereiche

| Bereich | Funktion |
|---------|----------|
| **Dashboard** | Übersicht, Willkommens-Aktionen |
| **Lokaler Server** | Hostname, IP, Updates, Firewall, RDP |
| **Alle Server** | Verwaltete Remote-Server |
| **Rollen** (z.B. AD DS, DNS) | Rollenstatus und Ereignisse |
| **Datei- und Speicherdienste** | Volumes, Freigaben |

---

## Serverrolle hinzufügen (GUI)

```
1. Server Manager → Verwalten → Rollen und Features hinzufügen
2. Installationstyp: "Rollenbasierte oder featurebasierte Installation"
3. Zielserver auswählen (lokal oder remote)
4. Serverrollen auswählen → z.B. "Active Directory Domain Services"
   ↳ Abhängige Features werden automatisch vorgeschlagen → Hinzufügen
5. Features auswählen (optional)
6. Rollendienste konfigurieren (rollenspezifisch)
7. Bestätigung → Installieren
8. Nach Installation: Konfigurationsaufgabe ausführen (gelbe Flagge oben rechts)
```

---

## Serverrolle hinzufügen (PowerShell)

```powershell
# Verfügbare Rollen anzeigen
Get-WindowsFeature | Where-Object {$_.InstallState -eq "Available"} | Format-Table Name, DisplayName

# Rolle installieren
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Mit allen Abhängigkeiten und Tools
Install-WindowsFeature -Name Web-Server -IncludeAllSubFeature -IncludeManagementTools

# Mehrere Rollen gleichzeitig
Install-WindowsFeature DNS, DHCP -IncludeManagementTools

# Auf Remote-Server installieren
Install-WindowsFeature -Name AD-Domain-Services -ComputerName SRV-DC02 -IncludeManagementTools

# Rolle deinstallieren
Uninstall-WindowsFeature -Name Web-Server -IncludeManagementTools -Restart
```

---

## Wichtige Rollen und ihre Feature-Namen

| Rolle | Feature-Name (PowerShell) |
|-------|--------------------------|
| Active Directory Domain Services | `AD-Domain-Services` |
| DNS-Server | `DNS` |
| DHCP-Server | `DHCP` |
| Dateidienste (File Server) | `FS-FileServer` |
| DFS Namespace | `FS-DFS-Namespace` |
| DFS Replikation | `FS-DFS-Replication` |
| Remote Desktop Services | `Remote-Desktop-Services` |
| IIS Webserver | `Web-Server` |
| Hyper-V | `Hyper-V` |
| Failover Clustering | `Failover-Clustering` |
| AD Certificate Services | `AD-Certificate` |
| WSUS | `UpdateServices` |
| Netzwerkrichtlinien-Server (NPS) | `NPAS` |
| Druckserver | `Print-Server` |
| Windows Admin Center (Gateway) | separate Installation |

```powershell
# Alle installierten Rollen anzeigen
Get-WindowsFeature | Where-Object {$_.InstallState -eq "Installed"} | Format-Table DisplayName, Name
```

---

## Remote-Server hinzufügen

Server Manager kann mehrere Server zentral verwalten:

```
Server Manager → Verwalten → Server hinzufügen

Methoden:
- Active Directory: Server aus AD suchen
- DNS: Hostname eingeben
- Importieren: TXT-Liste mit Hostnamen
```

```powershell
# Firewall für Remote-Verwaltung auf Zielserver aktivieren
Enable-PSRemoting -Force
netsh advfirewall firewall set rule group="Remote Administration" new enable=yes

# Alternativ per GPO: Computer Configuration → Policies →
# Windows Settings → Security Settings → Windows Firewall
```

---

## Lokaler Server — Konfiguration

Im Abschnitt "Lokaler Server" können direkt konfiguriert werden:

| Einstellung | Aktion |
|-------------|--------|
| Computername | Klick → Umbenennen |
| Arbeitsgruppe/Domain | Klick → Domain beitreten |
| Windows Firewall | Klick → Firewall-Einstellungen |
| Remote Desktop | Klick → RDP aktivieren/deaktivieren |
| Windows Update | Klick → Update-Einstellungen |
| Netzwerk | Klick → NIC-Eigenschaften |
| Zeitzone | Klick → Datum/Uhrzeit |
| IE Enhanced Security | Klick → IE ESC deaktivieren (für Browser-Zugriff) |

---

## Ereignisse und Warnungen

```
Rote/Gelbe Symbole im Dashboard bedeuten:
- Kritische Ereignisse (Eventlog-Fehler)
- Dienste-Ausfälle
- Konfigurationsaufgaben ausstehend

Klick auf Zahl → zeigt gefilterte Ereignisse
```

```powershell
# Kritische Ereignisse der letzten 24h per PowerShell
Get-EventLog -LogName System -EntryType Error,Warning `
    -Newest 50 | Format-Table TimeGenerated, Source, Message -AutoSize

# AD DS-spezifische Ereignisse
Get-EventLog -LogName "Directory Service" -Newest 20
```

---

## Verwandte Notizen

- [[Windows Admin Center]] — Moderne Web-UI als Server-Manager-Nachfolger
- [[Windows Server Installation]] — Grundkonfiguration nach Installation
- [[PowerShell für Admins]] — Alles was der Server Manager kann, auch per CLI
