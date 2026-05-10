---
tags: [active-directory, installation, domain-controller, dc-promotion, dcpromo, forest, adds]
aliases: ["AD Installation", "Domain Controller einrichten", "DC Promotion", "AD DS Installation"]
---

# AD Installation

> [!ABSTRACT]
> Schritt-für-Schritt: Einen neuen Forest erstellen (erste Domain), einen Domain Controller zu einer bestehenden Domain hinzufügen und eine Child-Domain einrichten. Voraussetzung: statische IP und funktionierendes DNS.

---

## Voraussetzungen

```powershell
# Vor der Installation prüfen:

# 1. Statische IP gesetzt
Get-NetIPAddress -AddressFamily IPv4

# 2. Computername gesetzt (wird zum DC-Namen!)
hostname

# 3. Zeit korrekt (Kerberos ist zeitkritisch, max. 5 Min. Abweichung)
w32tm /query /status

# 4. Alle Updates installiert
```

> [!WARNING] Computername vor Promotion setzen!
> Nach der DC-Promotion kann der Computername nur mit erheblichem Aufwand geändert werden. Den gewünschten Namen **vorher** setzen.

---

## Neuen Forest erstellen (erste Domain)

### Schritt 1: AD DS Rolle installieren

#### GUI

```
Server Manager → Verwalten → Rollen und Features hinzufügen
→ Serverrollen → Active Directory Domain Services
→ Features: (automatisch ausgewählt)
→ Installieren (noch nicht konfigurieren!)
```

#### PowerShell

```powershell
# AD DS Rolle installieren (ohne gleich zu konfigurieren)
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

### Schritt 2: Forest erstellen

#### GUI

```
Server Manager → Benachrichtigung (Flagge) →
"Server zu einem Domänencontroller heraufstufen"

Bereitstellungskonfiguration:
→ "Neue Gesamtstruktur hinzufügen"
  Stammdomänenname: firma.local
→ Weiter

Domänencontrolleroptionen:
  Gesamtstruktur-Funktionsebene: Windows Server 2016
  Domänen-Funktionsebene:        Windows Server 2016
  ✓ DNS-Server
  ✓ Globaler Katalog (GC)
  DSRM-Kennwort: ************  ← WICHTIG: sicher aufbewahren!
→ Weiter

DNS-Optionen: Weiter (Delegierungswarnung ignorieren bei neuem Forest)

Weitere Optionen:
  NetBIOS-Domänenname: FIRMA  (automatisch befüllt)
→ Weiter

Pfade: (Standard oder anpassen)
  NTDS: C:\Windows\NTDS
  SYSVOL: C:\Windows\SYSVOL
→ Weiter → Weiter → Installieren
→ Server startet automatisch neu
```

#### PowerShell

```powershell
# Forest und erste Domain erstellen
Install-ADDSForest `
    -DomainName "firma.local" `
    -DomainNetbiosName "FIRMA" `
    -DomainMode "WinThreshold" `     # Windows Server 2016 FFL
    -ForestMode "WinThreshold" `     # Windows Server 2016 FFL
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "DSRM-Pass!2024" -AsPlainText -Force) `
    -Force
# Server startet automatisch neu
```

```
DomainMode / ForestMode Werte:
WinThreshold = Windows Server 2016 (empfohlen für 2022/2025)
Win2012R2    = Windows Server 2012 R2
Win2016      = Windows Server 2016 (veraltet, WinThreshold nutzen)
```

> [!IMPORTANT] DSRM-Passwort sicher aufbewahren!
> Das Directory Services Restore Mode Passwort wird für Offline-Reparaturen benötigt. Sicher und getrennt vom normalen Admin-Passwort aufbewahren.

### Schritt 3: Nach Neustart prüfen

```powershell
# Domain prüfen
Get-ADDomain
Get-ADForest

# DC-Status
Get-ADDomainController

# DNS prüfen (automatisch konfiguriert)
Get-DnsServerZone
nslookup firma.local

# SYSVOL-Freigabe prüfen
net share | findstr SYSVOL
```

---

## Weiteren DC zur Domain hinzufügen (Empfohlen: mind. 2 DCs!)

### GUI

```
Voraussetzung: DNS des neuen Servers zeigt auf vorhandenen DC (192.168.1.10)

Server Manager → Verwalten → Rollen und Features hinzufügen
→ AD DS installieren → Neustart nicht nötig

Server Manager → Benachrichtigung →
"Server zu einem Domänencontroller heraufstufen"

Bereitstellungskonfiguration:
→ "Domänencontroller vorhandener Domäne hinzufügen"
  Domäne: firma.local
  Anmeldeinformationen: FIRMA\Administrator → Anmeldeinformationen ändern

DC-Optionen:
  ✓ DNS-Server
  ✓ Globaler Katalog (GC)
  DSRM-Kennwort: ************
→ Weiter → Weiter → Installieren → Neustart
```

### PowerShell

```powershell
# Schritt 1: Rolle installieren
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Schritt 2: DNS des neuen Servers auf vorhandenen DC zeigen
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.1.10"

# Schritt 3: Als DC zur Domain hinzufügen
Install-ADDSDomainController `
    -DomainName "firma.local" `
    -Credential (Get-Credential FIRMA\Administrator) `
    -InstallDns:$true `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "DSRM-Pass!2024" -AsPlainText -Force) `
    -Force
```

---

## Child-Domain erstellen

```powershell
# Untergeordnete Domain (z.B. tochter.firma.local)
Install-ADDSDomain `
    -NewDomainName "tochter" `
    -ParentDomainName "firma.local" `
    -DomainType ChildDomain `
    -Credential (Get-Credential FIRMA\Administrator) `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "DSRM-Pass!2024" -AsPlainText -Force) `
    -Force
```

---

## RSAT-Tools installieren (für Remote-Verwaltung)

```powershell
# Auf Windows 11 (Admin-Workstation)
Get-WindowsCapability -Online | Where-Object Name -like "Rsat.ActiveDirectory*"
Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0

# Alle RSAT-Tools auf einmal (Server)
Install-WindowsFeature -Name RSAT-AD-Tools, RSAT-DNS-Server, RSAT-DHCP -IncludeManagementTools

# Auf Windows 10/11 alle RSAT-Features
Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online
```

---

## AD-Benutzerkonten und OUs nach Installation

```powershell
# Basis-OU-Struktur erstellen
$domain = "DC=firma,DC=local"

New-ADOrganizationalUnit -Name "Benutzer" -Path $domain
New-ADOrganizationalUnit -Name "Gruppen" -Path $domain
New-ADOrganizationalUnit -Name "Computer" -Path $domain
New-ADOrganizationalUnit -Name "Server" -Path $domain
New-ADOrganizationalUnit -Name "Service Accounts" -Path $domain

# Abteilungen unter Benutzer
New-ADOrganizationalUnit -Name "IT" -Path "OU=Benutzer,$domain"
New-ADOrganizationalUnit -Name "Buchhaltung" -Path "OU=Benutzer,$domain"
New-ADOrganizationalUnit -Name "Vertrieb" -Path "OU=Benutzer,$domain"

# Standard-Container (Computers, Users) können nicht umbenannt werden
# → Eigene OUs erstellen und Computer/User dort einsortieren

# Default-Container für neue Computer umleiten
redircmp "OU=Computer,DC=firma,DC=local"
# Default-Container für neue Benutzer umleiten
redirusr "OU=Benutzer,DC=firma,DC=local"
```

---

## Passwort-Policy konfigurieren

```powershell
# Standard-Domänenrichtlinie anpassen
Set-ADDefaultDomainPasswordPolicy `
    -Identity firma.local `
    -MinPasswordLength 12 `
    -PasswordHistoryCount 12 `
    -MaxPasswordAge "90.00:00:00" `
    -MinPasswordAge "1.00:00:00" `
    -LockoutThreshold 5 `
    -LockoutDuration "00:30:00" `
    -LockoutObservationWindow "00:30:00" `
    -ComplexityEnabled $true

# Fine-Grained Password Policy (für spezifische Gruppen)
New-ADFineGrainedPasswordPolicy `
    -Name "Admins-Policy" `
    -Precedence 10 `
    -MinPasswordLength 16 `
    -PasswordHistoryCount 24 `
    -MaxPasswordAge "60.00:00:00" `
    -LockoutThreshold 3 `
    -ComplexityEnabled $true `
    -ProtectedFromAccidentalDeletion $true

# Policy einer Gruppe zuweisen
Add-ADFineGrainedPasswordPolicySubject `
    -Identity "Admins-Policy" `
    -Subjects "Domain Admins"
```

---

## Verwandte Notizen

- [[AD Grundlagen]] — Forest, Domain, FSMO-Konzepte
- [[Benutzer und Gruppen]] — Objekte nach der Installation anlegen
- [[Group Policy]] — GPOs nach der Installation konfigurieren
- [[Windows DNS]] — DNS ist Voraussetzung für AD
- [[AD Troubleshooting]] — Probleme nach der Installation beheben
