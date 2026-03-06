---
tags: [powershell, windows-server, admin, cmdlet, module, execution-policy, remoting, scripting]
aliases: ["PowerShell für Admins", "PowerShell Administration", "PS Admin"]
---

# PowerShell für Admins

> [!ABSTRACT]
> PowerShell ist die primäre Verwaltungsschnittstelle für Windows Server — unerlässlich für Server Core, Automatisierung und Remote-Administration. Ab PowerShell 7 (plattformübergreifend) und Windows PowerShell 5.1 (in Windows eingebaut) stehen alle Cmdlets für Windows Server zur Verfügung.

---

## Was ist PowerShell?

PowerShell ist eine **objektorientierte Shell und Skriptsprache** von Microsoft — entwickelt als vollständiger Ersatz für die klassische Kommandozeile (`cmd.exe`) in der Server-Administration. Das Projekt wurde intern als **"Monad"** geführt, die erste öffentliche Version **Windows PowerShell 1.0** erschien 2006.

Der fundamentale Unterschied zu Unix-Shells (bash, sh): PowerShell überträgt keine **Textströme** durch die Pipeline, sondern **.NET-Objekte**. Das erlaubt es, Eigenschaften von Objekten direkt weiterzuverarbeiten ohne Text-Parsing.

### Versionen

| Version | Name | .NET-Basis | Plattform | Status |
|---------|------|------------|-----------|--------|
| **1.0–5.1** | Windows PowerShell | .NET Framework | Windows only | Maintenance (keine Features mehr) |
| **6.x** | PowerShell Core | .NET Core 2.x | Windows, Linux, macOS | Veraltet |
| **7.x** | PowerShell | .NET 5/6/7/8 | Windows, Linux, macOS | **Aktiv entwickelt** |

> [!NOTE] Zwei Versionen parallel
> Windows PowerShell 5.1 (`powershell.exe`) ist in Windows fest integriert und wird weiter mit Windows ausgeliefert, aber nicht mehr weiterentwickelt. PowerShell 7.x (`pwsh.exe`) wird separat installiert und ist die empfohlene Version für neue Skripte.

### Kernkonzepte

| Konzept | Beschreibung |
|---------|-------------|
| **Cmdlet** | Kleinste Befehlseinheit. Immer im Format `Verb-Substantiv` (z.B. `Get-Service`, `Set-Item`) |
| **Pipeline** | Objekt-Übergabe zwischen Cmdlets: `Get-Process \| Stop-Process` |
| **Provider** | Abstraktionsschicht für verschiedene Datenspeicher (Filesystem, Registry, AD, Zertifikate) |
| **Modul** | Sammlung von Cmdlets, Funktionen und Typen — per `Import-Module` oder automatisch geladen |
| **PSGallery** | Öffentliches Repository für PowerShell-Module (`Install-Module -Name Az`) |
| **Execution Policy** | Steuert, welche Skripte ausgeführt werden dürfen — kein Sicherheitsfeature, nur Schutz vor versehentlicher Ausführung |
| **Desired State Configuration (DSC)** | Deklarative Konfigurationsverwaltung: Beschreibt den Sollzustand, nicht die Schritte |

### Verb-Substantiv-Konvention

PowerShell erzwingt standardisierte Verben. Die häufigsten:

| Verb | Bedeutung | Beispiel |
|------|-----------|---------|
| `Get` | Daten lesen | `Get-ADUser`, `Get-Service` |
| `Set` | Eigenschaften ändern | `Set-ADUser`, `Set-ExecutionPolicy` |
| `New` | Objekt erstellen | `New-ADUser`, `New-Item` |
| `Remove` | Objekt löschen | `Remove-ADUser`, `Remove-Item` |
| `Enable` / `Disable` | Aktivieren / Deaktivieren | `Enable-PSRemoting` |
| `Start` / `Stop` | Prozess/Dienst steuern | `Start-Service`, `Stop-Process` |
| `Import` / `Export` | Daten laden / speichern | `Import-Csv`, `Export-Clixml` |
| `Invoke` | Befehl ausführen | `Invoke-Command`, `Invoke-WebRequest` |

---

## Grundlagen

### Versionen

| Version | Plattform | Enthalten in |
|---------|-----------|-------------|
| **Windows PowerShell 5.1** | Windows only | Windows Server 2019/2022/2025 |
| **PowerShell 7.x** | Windows, Linux, macOS | Manuell installieren |

```powershell
# Aktuelle Version prüfen
$PSVersionTable
$PSVersionTable.PSVersion
```

> [!TIP] PowerShell 7 parallel installieren
> PowerShell 7 und Windows PowerShell 5.1 können gleichzeitig vorhanden sein und laufen in verschiedenen Prozessen (`pwsh.exe` vs `powershell.exe`).

---

## Execution Policy

```powershell
# Aktuelle Policy anzeigen
Get-ExecutionPolicy
Get-ExecutionPolicy -List   # Alle Scopes

# Policies erklären:
# Restricted     → Keine Skripte erlaubt (Standard für User)
# AllSigned      → Nur signierte Skripte
# RemoteSigned   → Lokale Skripte ok, Remote-Skripte müssen signiert sein
# Unrestricted   → Alle Skripte (warnt bei Remote)
# Bypass         → Keine Einschränkungen (für CI/CD)

# Policy setzen (lokal für Admin-Session)
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

# Policy für das System (als Admin)
Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force

# Temporär für eine Session umgehen
powershell.exe -ExecutionPolicy Bypass -File meinskript.ps1
```

---

## Module

```powershell
# Alle geladenen Module anzeigen
Get-Module

# Verfügbare Module anzeigen
Get-Module -ListAvailable

# Modul importieren
Import-Module ActiveDirectory
Import-Module DHCPServer
Import-Module DnsServer

# Modul aus PowerShell Gallery installieren (braucht Internet)
Install-Module -Name Az -Force           # Azure PowerShell
Install-Module -Name PSWindowsUpdate     # Windows Update
Install-Module -Name ImportExcel         # Excel-Export

# Alle Cmdlets eines Moduls auflisten
Get-Command -Module ActiveDirectory | Select-Object Name
```

---

## Hilfe nutzen

```powershell
# Hilfe für ein Cmdlet
Get-Help Get-ADUser
Get-Help Get-ADUser -Examples
Get-Help Get-ADUser -Detailed
Get-Help Get-ADUser -Online    # Browser öffnen

# Hilfe aktualisieren (einmal nach Installation)
Update-Help -Force

# Alle Cmdlets suchen
Get-Command *AD*
Get-Command *-Service
Get-Command -Verb Get -Noun *Process*
```

---

## Wichtige Cmdlets für Admins

### System-Information

```powershell
# Systeminformationen
Get-ComputerInfo
Get-WmiObject Win32_OperatingSystem | Select-Object Caption, Version, LastBootUpTime
systeminfo   # altes Tool, aber detailliert

# CPU/RAM
Get-WmiObject Win32_Processor | Select-Object Name, NumberOfCores, MaxClockSpeed
Get-WmiObject Win32_PhysicalMemory | Measure-Object Capacity -Sum

# Festplatten
Get-PSDrive -PSProvider FileSystem
Get-Disk
Get-Volume
```

### Dienste

```powershell
# Dienste anzeigen
Get-Service
Get-Service -Name WinRM, wuauserv, ADWS

# Dienst starten/stoppen/neustarten
Start-Service -Name W32Time
Stop-Service -Name Spooler -Force
Restart-Service -Name DNS

# Starttyp ändern
Set-Service -Name Spooler -StartupType Disabled
Set-Service -Name WinRM -StartupType Automatic

# Dienst-Status aller AD-relevanten Dienste
Get-Service ADWS, Kdc, Netlogon, DNS, W32Time | Format-Table Name, Status, StartType
```

### Prozesse

```powershell
# Prozesse anzeigen
Get-Process
Get-Process -Name lsass, svchost

# Prozess beenden
Stop-Process -Name notepad -Force
Stop-Process -Id 1234 -Force

# Ressourcen-intensive Prozesse
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 10
```

### Eventlog

```powershell
# System-Log letzte 20 Fehler
Get-EventLog -LogName System -EntryType Error -Newest 20

# Application Log filtern
Get-EventLog -LogName Application -Source "MSExchange*" -Newest 50

# Windows Event Log (Get-WinEvent — mächtiger)
Get-WinEvent -LogName System -MaxEvents 50 |
    Where-Object {$_.Level -le 2} |    # 1=Critical, 2=Error
    Format-Table TimeCreated, Message -AutoSize

# Event-ID suchen
Get-WinEvent -LogName Security |
    Where-Object {$_.Id -eq 4625} |   # Failed Logon
    Select-Object -First 20
```

### Firewall

```powershell
# Firewall-Status
Get-NetFirewallProfile | Select-Object Name, Enabled

# Firewall-Regeln anzeigen
Get-NetFirewallRule | Where-Object {$_.Enabled -eq "True"} |
    Select-Object DisplayName, Direction, Action | Format-Table

# Regel erstellen
New-NetFirewallRule `
    -DisplayName "Allow HTTP" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 80 `
    -Action Allow

# Regel deaktivieren
Set-NetFirewallRule -DisplayName "Allow HTTP" -Enabled False

# Gruppe aktivieren
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

### Dateisystem

```powershell
# Navigation
Get-ChildItem C:\Windows\System32 -Filter *.exe | Measure-Object
Get-ChildItem C:\ -Recurse -Filter *.log | Sort-Object Length -Descending

# Ordner erstellen
New-Item -Path "D:\Daten\Freigaben" -ItemType Directory -Force

# Dateien kopieren/verschieben
Copy-Item -Path "C:\Scripts\*.ps1" -Destination "D:\Backup\" -Recurse
Move-Item -Path "C:\Temp\*" -Destination "D:\Archiv\"

# Dateigröße
(Get-Item "C:\pagefile.sys").Length / 1GB  # In GB

# Verzeichnisgröße berechnen
(Get-ChildItem "C:\Windows" -Recurse | Measure-Object Length -Sum).Sum / 1GB
```

---

## Skripte schreiben

### Grundstruktur

```powershell
#Requires -Version 5.1
#Requires -RunAsAdministrator

<#
.SYNOPSIS
    Kurze Beschreibung des Skripts.
.DESCRIPTION
    Ausführliche Beschreibung.
.PARAMETER Name
    Beschreibung des Parameters.
.EXAMPLE
    .\MeinSkript.ps1 -Name "Test"
#>

[CmdletBinding()]
param (
    [Parameter(Mandatory = $true)]
    [string]$Name,

    [Parameter(Mandatory = $false)]
    [int]$Count = 10,

    [switch]$Verbose
)

# Fehlerbehandlung
$ErrorActionPreference = "Stop"

try {
    Write-Host "Starte Verarbeitung für: $Name" -ForegroundColor Green
    # ... Logik hier

} catch {
    Write-Error "Fehler aufgetreten: $_"
    exit 1
} finally {
    Write-Host "Fertig." -ForegroundColor Cyan
}
```

### Logging

```powershell
# Einfaches Logging
function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $entry = "[$timestamp] [$Level] $Message"
    Add-Content -Path "C:\Logs\skript.log" -Value $entry
    Write-Host $entry
}

Write-Log "Server $env:COMPUTERNAME gestartet"
Write-Log "Fehler beim Verbinden" -Level "ERROR"
```

---

## Verwandte Notizen

- [[PowerShell Remoting]] — PSSession, Invoke-Command, WinRM
- [[PowerShell Cheatsheet]] — Befehlsreferenz auf einen Blick
- [[AD Grundlagen]] — Active Directory via PowerShell (RSAT-Modul)
