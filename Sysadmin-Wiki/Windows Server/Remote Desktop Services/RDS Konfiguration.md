---
tags: [windows-server, rds, remote-desktop, deployment, connection-broker, load-balancing, farm]
aliases: ["RDS Konfiguration", "RDS Farm", "RDS Deployment", "RDS Load Balancing"]
---

# RDS Konfiguration

> [!ABSTRACT]
> Fortgeschrittene RDS-Konfiguration: Hochverfügbarkeit mit mehreren Session Hosts, Connection Broker im HA-Modus, Profilmanagement mit UPD/FSLogix, Drucker-Redirection und Performance-Optimierungen für Produktionsumgebungen.

---

## RDS-Deployment erstellen

```
Server Manager → Remote Desktop Services → Bereitstellungsübersicht

Schnellstart (alles auf einem Server):
1. Bereitstellungstyp: "Schnellstart"
2. Typ: "Sitzungsbasierter Desktop"
3. Server: SRV-RDS01

Standard-Deployment (produktiv, mehrere Server):
1. Bereitstellungstyp: "Standardbereitstellung"
2. RD Connection Broker:  SRV-RDCB01
3. RD Web Access:         SRV-RDWA01
4. RD Session Host:       SRV-RDSH01, SRV-RDSH02 (mehrere!)
```

---

## Session Host zur Farm hinzufügen

### GUI

```
Server Manager → Remote Desktop Services → Collections → Desktop Collection
→ Aufgaben → Eigenschaften der Sammlung
→ Sitzungshosts → Hinzufügen
  → SRV-RDSH02.firma.local auswählen → OK
```

### PowerShell

```powershell
# Session Host zu RDS-Collection hinzufügen
Add-RDSessionHost `
    -CollectionName "Desktop Collection" `
    -SessionHost "SRV-RDSH02.firma.local" `
    -ConnectionBroker "SRV-RDCB01.firma.local"

# Collection anzeigen
Get-RDSessionCollection -ConnectionBroker "SRV-RDCB01.firma.local"

# Session Hosts einer Collection
Get-RDSessionHost -CollectionName "Desktop Collection" -ConnectionBroker "SRV-RDCB01.firma.local"
```

---

## Connection Broker HA (Hochverfügbarkeit)

```powershell
# RDCB-HA benötigt: SQL Server (oder SQL Express), gemeinsame DNS-URL
# Beide RDCB müssen im SQL DB schreiben

# HA für Connection Broker aktivieren
Set-RDConnectionBrokerHighAvailability `
    -ConnectionBroker "SRV-RDCB01.firma.local" `
    -DatabaseConnectionString "DRIVER=SQL Server Native Client 11.0;SERVER=SRV-SQL01;Trusted_Connection=Yes;APP=Remote Desktop Services Connection Broker;Database=RDCB" `
    -DatabaseSecondaryConnectionString "DRIVER=SQL Server Native Client 11.0;SERVER=SRV-SQL01\SQLEXPRESS;Trusted_Connection=Yes;APP=Remote Desktop Services Connection Broker;Database=RDCB" `
    -ClientAccessName "rdbroker.firma.local"    # DNS-Eintrag mit VIP oder DNS-Round-Robin
```

---

## User Profile Disks (UPD)

### GUI

```
Server Manager → Remote Desktop Services → Collections → Desktop Collection
→ Aufgaben → Eigenschaften der Sammlung → Benutzerprofil-Datenträger

☑ Benutzerprofil-Datenträger aktivieren
  Speicherort: \\SRV-FS01\RDS-UPD
  Maximale Größe: 20 GB
→ OK
```

### PowerShell

```powershell
# UPD aktiviert (Benutzerprofile auf Netzwerkfreigabe, folgen Benutzer im Farm)
Set-RDSessionCollectionConfiguration `
    -CollectionName "Desktop Collection" `
    -ConnectionBroker "SRV-RDCB01.firma.local" `
    -EnableUserProfileDisk $true `
    -DiskPath "\\SRV-FS01\RDS-UPD" `           # UPD-Freigabe
    -MaxUserProfileDiskSizeGB 20

# UPD-Ordner vorbereiten (Freigabe und NTFS korrekt setzen)
New-Item "D:\RDS-UPD" -ItemType Directory
New-SmbShare -Name "RDS-UPD" -Path "D:\RDS-UPD" -FullAccess "FIRMA\Domain Users"
```

> [!TIP] FSLogix statt UPD
> Für Office 365/Microsoft 365 Umgebungen ist **FSLogix Profile Container** die empfohlene Alternative zu UPD. FSLogix ist in M365 inbegriffen und deutlich performanter (VHD-basiert, Outlook-Search-Index wird migriert).

---

## Drucker-Redirection

```powershell
# Drucker-Redirection per GPO konfigurieren:
# Computer Konfiguration → Richtlinien → Administrative Vorlagen →
# Windows-Komponenten → Remotedesktopdienste → RD Session Host →
# Drucker-Redirection

# "Clientdruckerredirection nicht zulassen" → Deaktiviert (erlaubt Umleitung)
# "Standarddrucker des Clients als Standarddrucker in Sitzungen" → Aktiviert

# Easy Print (Standard-Druckertreiber, kein Treiber auf RDSH nötig)
# "RD Easy Print-Druckertreiber zuerst verwenden" → Aktiviert
```

---

## Performance-Optimierungen

```powershell
# Visual Effects für beste Performance reduzieren
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\VisualEffects" `
    -Name "VisualFXSetting" -Value 2

# Per GPO (alle Benutzer):
# Benutzerkonfiguration → Administrative Vorlagen →
# Windows-Komponenten → Remotedesktopdienste → RD Session Host →
# Remote Session Environment
# → "Visuelle Effekte für Remotedesktopsitzungen optimieren"

# RDS Server für CPU/RAM-optimierung (Server-spezifisch)
# Nicht-wesentliche Dienste deaktivieren
Get-Service | Where-Object StartType -eq Automatic |
    Where-Object {$_.Name -notmatch "RDS|Terminal|UMP|SessionEnv"} |
    Select-Object Name, DisplayName
```

---

## Sicherheit: NLA und TLS

```powershell
# NLA erzwingen (Authentifizierung vor Session-Aufbau — wichtig!)
# Verhindert Angriffe auf RDP-Login-Screen

# Per GPO:
# Computer → Administrative Vorlagen → Windows-Komponenten →
# Remotedesktopdienste → RD Session Host → Sicherheit
# → "Authentifizierung auf Netzwerkebene erforderlich" → Aktiviert

# TLS-Sicherheitsebene
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
    -Name "SecurityLayer" `
    -Value 2    # 0=RDP, 1=Negotiate, 2=SSL/TLS

# Verschlüsselungsstärke
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
    -Name "MinEncryptionLevel" `
    -Value 3    # 1=Low, 2=ClientCompatible, 3=High, 4=FIPS
```

---

## Monitoring

```powershell
# Alle aktiven RDS-Sitzungen über Connection Broker
Get-RDUserSession -ConnectionBroker "SRV-RDCB01.firma.local"

# Performance-Counters für RDS
Get-Counter "\Remote Desktop Session Host\*" | Select-Object -ExpandProperty CounterSamples |
    Select-Object Path, CookedValue

# Wichtige Counters:
# \Terminal Services\Active Sessions        → Aktive Sitzungen
# \Terminal Services\Inactive Sessions      → Getrennte Sitzungen
# \Processor(_Total)\% Processor Time       → CPU-Auslastung
# \Memory\Available MBytes                  → Freier Arbeitsspeicher
```

---

## Troubleshooting

```powershell
# RDS-Dienste auf Session Host prüfen
Get-Service -Name "TermService", "SessionEnv", "UmRdpService" |
    Format-Table Name, Status, StartType

# Anmeldefehler (Event 4625)
Get-WinEvent -LogName Security |
    Where-Object {$_.Id -eq 4625 -and $_.Message -match "RDP"} |
    Select-Object -First 10

# RDS Event Log
Get-WinEvent -LogName "Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational" -MaxEvents 30

# Lizenzprobleme
Get-RDLicenseConfiguration -ConnectionBroker "SRV-RDCB01.firma.local"
Get-WinEvent -LogName "Microsoft-Windows-TerminalServices-Licensing/Admin"
```

---

## Verwandte Notizen

- [[RDS Grundlagen]] — Architektur, Lizenzierung, Grundkonfiguration
- [[RemoteApp]] — Einzelne Anwendungen veröffentlichen
- [[Group Policy]] — RDS-Einstellungen per GPO
- [[AD Grundlagen]] — AD-Gruppen für RDS-Zugriff
