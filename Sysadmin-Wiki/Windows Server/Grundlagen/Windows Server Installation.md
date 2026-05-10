---
tags: [windows-server, installation, setup, konfiguration, ipconfig, sconfig, core, gui]
aliases: ["Windows Server Installation", "WinSrv Setup", "Server einrichten"]
---

# Windows Server Installation

> [!ABSTRACT]
> Schritt-für-Schritt Installation von Windows Server 2022/2025 — sowohl Desktop Experience (GUI) als auch Server Core. Nach der Installation: Grundkonfiguration mit `sconfig` (Core) oder Server Manager (GUI), statische IP, Computername, Windows Update und Remote-Verwaltung aktivieren.

---

## Vor der Installation

### Checkliste

```
☐ Lizenz-Key bereit
☐ ISO heruntergeladen (microsoft.com/evalcenter für Evaluation)
☐ Bootmedium erstellt (USB via Rufus, oder ISO in Hyper-V/VMware eingebunden)
☐ Hardware-Kompatibilität geprüft (HCL)
☐ BIOS/UEFI: Virtualisierung aktiviert (für Hyper-V)
☐ Bootreihenfolge auf USB/ISO gesetzt
☐ Netzwerk-Informationen bereit (IP, Gateway, DNS)
```

---

## Installation (GUI-Wizard)

```
1. Sprache, Zeit, Tastatur → Weiter
2. Jetzt installieren
3. Lizenzschlüssel eingeben (oder überspringen für Evaluation)
4. Edition wählen:
   → "Windows Server 2025 Standard (Desktop Experience)"  — mit GUI
   → "Windows Server 2025 Standard"                       — Server Core
5. Lizenzbedingungen akzeptieren
6. Installationstyp: "Benutzerdefiniert" (Neuinstallation)
7. Datenträger auswählen / Partitionen erstellen
8. Installation startet (~10-20 Minuten)
9. Administrator-Passwort setzen (min. Komplexität!)
```

> [!WARNING] Partitionierung
> Empfehlung: OS-Partition (C:) mindestens **100 GB**. Daten auf separate Partition/Laufwerk (D:). System-Volume niemals für Daten nutzen.

---

## Grundkonfiguration mit sconfig (Server Core)

`sconfig` ist das interaktive Konfigurations-Tool für Server Core:

```powershell
# sconfig starten
sconfig
```

```
Serverseite:
1) Domäne/Arbeitsgruppe     → Domain beitreten oder Workgroup setzen
2) Computername             → Hostname setzen
3) Lokalen Administrator    → Admin-Konto konfigurieren
4) Remoteverwaltung         → WinRM aktivieren
5) Windows Update           → Updates konfigurieren und installieren
6) Downloads von Updates    → Manuell/Automatisch
7) Remotedesktopdienste     → RDP aktivieren
8) Netzwerkeinstellungen    → IP-Konfiguration
9) Datum und Uhrzeit        → Zeitzone
10) Telemetrie             → Datenschutz-Einstellungen
11) Windows-Aktivierung     → Lizenz aktivieren
12) Abmelden               → Session beenden
13) Neustart                → Neu starten
14) Herunterfahren          → Ausschalten
15) Beenden                → CMD/PowerShell zurück
```

---

## Grundkonfiguration per PowerShell

```powershell
# 1. Computername setzen
Rename-Computer -NewName "SRV-DC01" -Restart

# 2. Statische IP konfigurieren
$iface = Get-NetAdapter | Where-Object {$_.Status -eq "Up"} | Select-Object -First 1
New-NetIPAddress `
    -InterfaceAlias $iface.Name `
    -IPAddress "192.168.1.10" `
    -PrefixLength 24 `
    -DefaultGateway "192.168.1.1"

Set-DnsClientServerAddress `
    -InterfaceAlias $iface.Name `
    -ServerAddresses "192.168.1.10","192.168.1.11"

# 3. Zeitzone setzen
Set-TimeZone -Id "W. Europe Standard Time"

# 4. Windows Update aktivieren (automatisch)
Set-Service -Name wuauserv -StartupType Automatic
Start-Service wuauserv

# 5. Sofort nach Updates suchen
Install-Module PSWindowsUpdate -Force
Get-WindowsUpdate -Install -AcceptAll -AutoReboot

# 6. WinRM (Remoteverwaltung) aktivieren
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force

# 7. Firewall: Ping erlauben (für Tests)
Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)"
```

---

## Netzwerkkonfiguration im Detail

```powershell
# Aktuelle Konfiguration anzeigen
Get-NetIPConfiguration
Get-NetIPAddress
Get-NetRoute

# IP ändern (statisch)
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet" -Confirm:$false -ErrorAction SilentlyContinue

New-NetIPAddress `
    -InterfaceAlias "Ethernet" `
    -IPAddress "10.0.0.10" `
    -PrefixLength 24 `
    -DefaultGateway "10.0.0.1"

# DNS-Server setzen
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses "10.0.0.10","10.0.0.11"

# NIC umbenennen (hilfreich bei mehreren NICs)
Rename-NetAdapter -Name "Ethernet" -NewName "LAN"
Rename-NetAdapter -Name "Ethernet 2" -NewName "iSCSI"

# Konfiguration prüfen
Test-NetConnection -ComputerName "google.de" -Port 443
Resolve-DnsName "google.de"
```

---

## Domain beitreten

```powershell
# Domain beitreten (Neustart erforderlich)
Add-Computer `
    -DomainName "firma.local" `
    -Credential (Get-Credential) `
    -Restart

# Mit OU-Pfad
Add-Computer `
    -DomainName "firma.local" `
    -OUPath "OU=Server,OU=Computer,DC=firma,DC=local" `
    -Credential (Get-Credential) `
    -Restart

# Domain-Beitritt prüfen
(Get-WmiObject -Class Win32_ComputerSystem).PartOfDomain
(Get-WmiObject -Class Win32_ComputerSystem).Domain
```

---

## Remote Desktop (RDP) aktivieren

```powershell
# RDP aktivieren
Set-ItemProperty `
    -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" `
    -Name "fDenyTSConnections" `
    -Value 0

# Firewall-Regel aktivieren
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# NLA (Network Level Authentication) aktivieren (empfohlen)
Set-ItemProperty `
    -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
    -Name "UserAuthentication" `
    -Value 1

# Prüfen
(Get-ItemProperty "HKLM:\System\CurrentControlSet\Control\Terminal Server").fDenyTSConnections
# 0 = RDP aktiviert
```

---

## Windows-Aktivierung

```powershell
# Produktschlüssel eingeben
slmgr.vbs /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX

# Online aktivieren
slmgr.vbs /ato

# KMS-Server konfigurieren
slmgr.vbs /skms kms.firma.local:1688
slmgr.vbs /ato

# Aktivierungsstatus prüfen
slmgr.vbs /xpr     # Ablaufdatum
slmgr.vbs /dlv     # Detaillierte Lizenzinfo
```

---

## Post-Installation Checkliste

```powershell
# Vollständige Post-Install-Checkliste als PowerShell-Skript

# System-Info ausgeben
$info = @{
    "Computername"    = $env:COMPUTERNAME
    "OS-Version"      = (Get-WmiObject Win32_OperatingSystem).Caption
    "IP-Adresse"      = (Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.PrefixOrigin -ne "WellKnown"}).IPAddress
    "Domain"          = (Get-WmiObject Win32_ComputerSystem).Domain
    "Zeitzone"        = (Get-TimeZone).DisplayName
    "Letzter Reboot"  = (Get-WmiObject Win32_OperatingSystem).LastBootUpTime
}
$info.GetEnumerator() | Format-Table -AutoSize

# Windows Update Status
(New-Object -ComObject Microsoft.Update.AutoUpdate).Settings | Select-Object NotificationLevel

# WinRM Status
Get-Service WinRM | Select-Object Status, StartType

# RDP Status
(Get-ItemProperty "HKLM:\System\CurrentControlSet\Control\Terminal Server").fDenyTSConnections
```

---

## Verwandte Notizen

- [[Was ist Windows Server]] — Editionen und Anforderungen
- [[Server Manager]] — Rollen und Features über GUI hinzufügen
- [[PowerShell für Admins]] — Alles per PowerShell verwalten
- [[AD Installation]] — Server zum Domain Controller promoten
