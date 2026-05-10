---
tags: [powershell, cheatsheet, cmdlets, referenz, admin, windows-server]
aliases: ["PowerShell Cheatsheet", "PS Cheatsheet", "PowerShell Referenz"]
---

# PowerShell Cheatsheet

> [!ABSTRACT]
> Schnellreferenz für die wichtigsten PowerShell-Befehle im Windows Server-Alltag — sortiert nach Themenbereich. Alle Befehle für Windows PowerShell 5.1 und PowerShell 7.x.

---

## System & Hardware

```powershell
# System-Info
$PSVersionTable                                     # PS-Version
Get-ComputerInfo                                    # Vollständige System-Info
hostname                                            # Computername
$env:COMPUTERNAME                                   # Computername (Variable)
[System.Environment]::OSVersion                    # OS-Version
(gcim Win32_OperatingSystem).LastBootUpTime        # Letzter Neustart
(gcim Win32_OperatingSystem).FreePhysicalMemory    # Freier RAM (KB)

# Neustart / Herunterfahren
Restart-Computer -Force
Restart-Computer -ComputerName SRV-DC01 -Force
Stop-Computer -Force

# Uptime berechnen
(Get-Date) - (gcim Win32_OperatingSystem).LastBootUpTime
```

---

## Netzwerk

```powershell
# IP-Konfiguration
Get-NetIPConfiguration
Get-NetIPAddress -AddressFamily IPv4
ipconfig /all                           # Klassisch

# DNS
Resolve-DnsName "google.de"
Resolve-DnsName "google.de" -Server 8.8.8.8
nslookup google.de                      # Klassisch

# Verbindungstest
Test-NetConnection -ComputerName SRV-DC01
Test-NetConnection -ComputerName SRV-DC01 -Port 443
Test-NetConnection -ComputerName google.de -Port 80 -InformationLevel Detailed

# Routing
Get-NetRoute
tracert google.de

# Offene Ports
Get-NetTCPConnection -State Listen
netstat -ano

# NIC
Get-NetAdapter
Set-NetAdapterAdvancedProperty -Name "Ethernet" -RegistryKeyword "SpeedDuplex" -RegistryValue 6
```

---

## Dienste

```powershell
Get-Service                             # Alle Dienste
Get-Service W*                          # Wildcard
Get-Service -Name WinRM | Select-Object *
Start-Service -Name WinRM
Stop-Service -Name Spooler -Force
Restart-Service -Name DNS
Set-Service -Name Spooler -StartupType Disabled
# StartupType: Automatic, Manual, Disabled, AutomaticDelayedStart
```

---

## Prozesse

```powershell
Get-Process                             # Alle Prozesse
Get-Process -Name lsass
Get-Process | Sort-Object CPU -Desc | Select-Object -First 10
Get-Process | Sort-Object WS -Desc | Select-Object -First 10  # RAM
Stop-Process -Name notepad -Force
Stop-Process -Id 1234
tasklist                               # Klassisch
taskkill /PID 1234 /F
```

---

## Dateisystem

```powershell
# Navigation
Set-Location C:\Windows
Get-ChildItem                           # Kurzform: ls, dir
Get-ChildItem -Filter *.log -Recurse
Get-ChildItem -Hidden

# Dateien
New-Item -Path "C:\Test\neu.txt" -ItemType File -Value "Inhalt"
New-Item -Path "C:\Test\Ordner" -ItemType Directory
Copy-Item -Path "C:\a.txt" -Destination "D:\b.txt"
Move-Item -Path "C:\a.txt" -Destination "D:\"
Remove-Item -Path "C:\Temp\*" -Recurse -Force
Get-Content "C:\Windows\System32\drivers\etc\hosts"
Set-Content -Path "C:\test.txt" -Value "Neue Zeile"
Add-Content -Path "C:\log.txt" -Value "Eintrag"

# Suche
Get-ChildItem -Path C:\ -Recurse -Filter "*.exe" -ErrorAction SilentlyContinue
Select-String -Path "C:\logs\*.log" -Pattern "Error"

# Größe
(Get-ChildItem "C:\Windows" -Recurse -ErrorAction SilentlyContinue |
    Measure-Object Length -Sum).Sum / 1GB
```

---

## Eventlog

```powershell
# Windows EventLog (klassisch)
Get-EventLog -LogName System -EntryType Error -Newest 20
Get-EventLog -LogName Application -Newest 50 -Source "Application Error"
Clear-EventLog -LogName Application

# WinEvent (modern, mächtiger)
Get-WinEvent -LogName System -MaxEvents 50
Get-WinEvent -LogName Security |
    Where-Object {$_.Id -eq 4625}       # Failed Logons
Get-WinEvent -LogName "Active Directory Web Services" -MaxEvents 20

# XPath-Filter (performant)
Get-WinEvent -FilterHashtable @{
    LogName   = "System"
    Level     = 2                       # Error
    StartTime = (Get-Date).AddDays(-1)
}
```

---

## Benutzer (Lokal)

```powershell
# Lokale Benutzer
Get-LocalUser
New-LocalUser -Name "admin2" -Password (ConvertTo-SecureString "Pass!1234" -AsPlainText -Force)
Set-LocalUser -Name "admin2" -PasswordNeverExpires $true
Disable-LocalUser -Name "Gast"
Remove-LocalUser -Name "admin2"

# Lokale Gruppen
Get-LocalGroup
Get-LocalGroupMember -Group "Administrators"
Add-LocalGroupMember -Group "Administrators" -Member "FIRMA\ITAdmin"
Remove-LocalGroupMember -Group "Administrators" -Member "admin2"
```

---

## Active Directory

```powershell
# Modul laden
Import-Module ActiveDirectory

# Benutzer
Get-ADUser -Filter * -Properties *
Get-ADUser -Identity "jdoe" -Properties MemberOf, LastLogonDate
New-ADUser -Name "Jane Doe" -SamAccountName "jdoe2" -Path "OU=User,DC=firma,DC=local"
Set-ADUser -Identity "jdoe" -Description "IT Administrator"
Disable-ADAccount -Identity "jdoe"
Enable-ADAccount -Identity "jdoe"
Unlock-ADAccount -Identity "jdoe"
Set-ADAccountPassword -Identity "jdoe" -Reset -NewPassword (Read-Host -AsSecureString)
Remove-ADUser -Identity "jdoe" -Confirm:$false

# Gruppen
Get-ADGroup -Filter * | Select-Object Name, GroupCategory
Add-ADGroupMember -Identity "IT-Admins" -Members "jdoe","jsmith"
Remove-ADGroupMember -Identity "IT-Admins" -Members "jdoe" -Confirm:$false

# Computer
Get-ADComputer -Filter * -Properties * | Select-Object Name, LastLogonDate, OperatingSystem
Get-ADComputer -Filter {OperatingSystem -like "*2022*"} | Select-Object Name

# DC
Get-ADDomainController -Filter *
repadmin /showrepl              # Replikationsstatus
dcdiag /test:replications       # DC-Diagnose
```

---

## Windows Update

```powershell
# PSWindowsUpdate-Modul (muss installiert werden)
Install-Module PSWindowsUpdate -Force

Get-WindowsUpdate                       # Updates anzeigen
Install-WindowsUpdate -AcceptAll -AutoReboot  # Alle installieren
Get-WUHistory                          # Update-Historie
```

---

## Zertifikate

```powershell
# Zertifikatsspeicher durchsuchen
Get-ChildItem Cert:\LocalMachine\My
Get-ChildItem Cert:\LocalMachine\Root
Get-ChildItem -Path Cert:\LocalMachine\My |
    Where-Object {$_.NotAfter -lt (Get-Date).AddDays(30)}  # Bald ablaufend

# Zertifikat exportieren
$cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.Subject -like "*server*"}
Export-Certificate -Cert $cert -FilePath "C:\server.cer"

# Zertifikat importieren
Import-Certificate -FilePath "C:\root-ca.cer" -CertStoreLocation Cert:\LocalMachine\Root
```

---

## Registry

```powershell
# Registry lesen
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
Get-ItemPropertyValue "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" "fDenyTSConnections"

# Registry schreiben
Set-ItemProperty -Path "HKLM:\...\Terminal Server" -Name "fDenyTSConnections" -Value 0
New-ItemProperty -Path "HKLM:\Software\Test" -Name "MeinWert" -Value "Test" -PropertyType String

# Registry-Key erstellen/löschen
New-Item -Path "HKLM:\Software\MeineFirma"
Remove-Item -Path "HKLM:\Software\MeineFirma" -Recurse
```

---

## Nützliche One-Liner

```powershell
# Offene TCP-Verbindungen mit Prozess
Get-NetTCPConnection | Where-Object State -eq "Established" |
    Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort,
    @{N="Process";E={(Get-Process -Id $_.OwningProcess).Name}}

# Gesperrte AD-Konten
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LockedOut

# Bald ablaufende AD-Passwörter
Get-ADUser -Filter {Enabled -eq $true} -Properties PasswordExpiry |
    Where-Object {$_.PasswordExpiry -lt (Get-Date).AddDays(14)} |
    Select-Object Name, PasswordExpiry

# Größte Verzeichnisse finden
Get-ChildItem "C:\" -Directory |
    ForEach-Object {
        $size = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue |
                 Measure-Object Length -Sum).Sum
        [PSCustomObject]@{Path=$_.FullName; SizeGB=[math]::Round($size/1GB,2)}
    } | Sort-Object SizeGB -Descending | Select-Object -First 10

# Alle lokalen Admins im Netzwerk abfragen
$server = Get-ADComputer -Filter * | Select-Object -ExpandProperty Name
Invoke-Command -ComputerName $server -ScriptBlock {
    Get-LocalGroupMember -Group "Administrators"
} -ErrorAction SilentlyContinue
```

---

## Verwandte Notizen

- [[PowerShell für Admins]] — Grundlagen und Scripting
- [[PowerShell Remoting]] — Remote-Administration
- [[AD Grundlagen]] — Active Directory via PowerShell
