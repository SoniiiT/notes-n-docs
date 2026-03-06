---
tags: [windows-server, file-server, smb, freigaben, berechtigungen, audit, storage]
aliases: ["SMB Freigaben", "Dateifreigaben", "Windows Freigaben", "File Share"]
---

# SMB Freigaben

> [!ABSTRACT]
> Windows-Dateifreigaben basieren auf dem **SMB-Protokoll (Server Message Block)**. Zugriff wird durch zwei Ebenen gesteuert: **Freigabe-Berechtigungen** (beim Netzwerkzugriff) und **NTFS-Berechtigungen** (auf dem Dateisystem). Es gilt immer die restriktivere Kombination aus beiden.

---

## Was ist SMB?

SMB (Server Message Block) ist ein **Netzwerkprotokoll für gemeinsamen Dateizugriff, Druckerdienste und Inter-Prozess-Kommunikation** in Windows-Netzwerken. Es wurde ursprünglich in den 1980er Jahren von IBM entwickelt und von Microsoft für Windows adaptiert. Microsoft entwickelte das Protokoll unter dem Namen **CIFS** (Common Internet File System) weiter; der aktuelle Name SMB 3.x ist die moderne Implementierung.

SMB arbeitet über **TCP Port 445** (direktes SMB, seit Windows 2000) und ermöglicht den transparenten Zugriff auf Netzwerkressourcen über **UNC-Pfade** (`\\Server\Freigabe`).

### SMB-Versionen

| Version | Windows | Wesentliche Merkmale |
|---------|---------|----------------------|
| **SMB 1.0 / CIFS** | Windows NT/2000/XP | Ursprungsprotokoll, keine Verschlüsselung |
| **SMB 2.0** | Windows Vista / Server 2008 | Reduzierte Befehlsanzahl, bessere Performance, größere Puffergrößen |
| **SMB 2.1** | Windows 7 / Server 2008 R2 | Opportunistic Locking (Oplock-Verbesserungen), BranchCache-Unterstützung |
| **SMB 3.0** | Windows 8 / Server 2012 | Transparent Failover, SMB Multichannel, SMB Direct (RDMA), AES-128-CCM-Verschlüsselung |
| **SMB 3.02** | Windows 8.1 / Server 2012 R2 | Optimierungen für SMB 3.0 |
| **SMB 3.1.1** | Windows 10 / Server 2016+ | AES-128-GCM, Pre-Authentication Integrity (SHA-512), Cluster Dialect Fencing |

> [!DANGER] SMB 1.0 abschalten
> SMB 1.0 enthält kritische Schwachstellen (u.a. EternalBlue / MS17-010), die von WannaCry (2017) und NotPetya für weltweite Angriffe genutzt wurden. In modernen Umgebungen sollte SMB 1.0 vollständig deaktiviert sein.

### UNC-Pfade und Adressierung

```
\\SRV-FS01\Marketing\Projekte\2024\datei.docx

\\          UNC-Präfix (Universal Naming Convention)
SRV-FS01    Hostname des Servers (wird über DNS aufgelöst)
Marketing   Freigabename (im Server-Dienst registriert)
\Projekte\  Unterverzeichnis auf dem Server
2024\       Weiteres Unterverzeichnis
datei.docx  Dateiname
```

### Zwei-Ebenen-Berechtigungsmodell

Der Zugriff auf eine Freigabe wird durch zwei unabhängige Berechtigungsebenen gesteuert:

| Ebene | Wo | Gilt für | Granularität |
|-------|-----|---------|-------------|
| **Freigabe-Berechtigung** | SMB-Dienst | Nur Netzwerkzugriff | Lesen / Ändern / Vollzugriff |
| **NTFS-Berechtigung** | Dateisystem | Netzwerk- UND lokaler Zugriff | Fein granular (Lesen, Schreiben, Ausführen, ...) |

Der **effektive Zugriff** ist der restriktivere Wert beider Ebenen. Best Practice: Freigabe-Berechtigung auf "Vollzugriff für Domänen-Benutzer", alle Zugriffssteuerung über NTFS-Berechtigungen.

---

### GUI

```
Server Manager → Datei- und Speicherdienste → Freigaben → Aufgaben → Neue Freigabe

Wizard — Neue Freigabe:
1. Freigabeprofil:
   → "SMB-Freigabe – Schnell" (Standard, empfohlen)
   → "SMB-Freigabe – Erweitert" (mit ABE, Quoten, BranchCache)

2. Serverpfad:
   → Laufwerk auswählen: D:
   → Pfad eingeben: D:\Freigaben\Marketing
   (oder: "Benutzerdefinierter Pfad" → Durchsuchen)

3. Freigabename:
   Freigabename:     Marketing
   Beschreibung:     Marketing-Abteilung Freigabe
   (Zugriffspfad:    \\SRV-FS01\Marketing — wird angezeigt)

4. Weitere Einstellungen:
   ✓ Zugriffsenumeration aktivieren (ABE — empfohlen)
   ☐ Zwischenspeicherung deaktivieren
   ☐ Datenverschlüsselung (für sensible Daten)

5. Berechtigungen:
   → Berechtigungen anpassen
   Freigabeberechtigung: Alle — Vollzugriff (NTFS steuert)
   NTFS-Berechtigungen:
   → FIRMA\Marketing-Mitarbeiter: Ändern
   → FIRMA\IT-Admins: Vollzugriff
   → Vererbung deaktivieren → Berechtigungen kopieren

6. Bestätigen → Erstellen
```

### PowerShell

```powershell
# Verzeichnis erstellen
New-Item -Path "D:\Freigaben\Marketing" -ItemType Directory

# Einfache Freigabe
New-SmbShare `
    -Name "Marketing" `
    -Path "D:\Freigaben\Marketing" `
    -Description "Marketing-Abteilung Freigabe" `
    -FullAccess "FIRMA\IT-Admins" `
    -ReadAccess "FIRMA\Marketing-Mitarbeiter" `
    -NoAccess "FIRMA\Domain Users"

# Erweiterte Freigabe mit allen Optionen
New-SmbShare `
    -Name "HR-Daten" `
    -Path "D:\Freigaben\HR" `
    -ChangeAccess "FIRMA\HR-Mitarbeiter" `
    -FullAccess "FIRMA\HR-Admins", "FIRMA\IT-Admins" `
    -FolderEnumerationMode AccessBased `    # Zugriffsenumeration (ABE)
    -EncryptData $true `                   # SMB-Verschlüsselung
    -CachingMode None `                    # Offline-Caching deaktiviert
    -Description "HR Vertraulich"

# --- BERECHTIGUNGSSTUFEN ---
# FullAccess    = Vollzugriff (auf Freigabe-Ebene)
# ChangeAccess  = Lesen + Schreiben
# ReadAccess    = Nur lesen
# NoAccess      = Kein Zugriff

# Freigaben anzeigen
Get-SmbShare
Get-SmbShare -Name "Marketing"

# Freigabe ändern
Set-SmbShare -Name "Marketing" -Description "Marketing aktuell" -Force

# Freigabe entfernen
Remove-SmbShare -Name "Marketing" -Force
```

---

## Freigabe-Berechtigungen verwalten

```powershell
# Aktuelle Freigabe-ACL anzeigen
Get-SmbShareAccess -Name "Marketing"

# Berechtigung hinzufügen
Grant-SmbShareAccess `
    -Name "Marketing" `
    -AccountName "FIRMA\Praktikanten" `
    -AccessRight Read `
    -Force

# Berechtigung entfernen
Revoke-SmbShareAccess `
    -Name "Marketing" `
    -AccountName "FIRMA\Praktikanten" `
    -Force

# Berechtigung ändern
Grant-SmbShareAccess `
    -Name "Marketing" `
    -AccountName "FIRMA\Marketing-Mitarbeiter" `
    -AccessRight Change `
    -Force
```

> [!IMPORTANT] Freigabe + NTFS
> Freigabe-Berechtigung `Everyone - Full Control` + NTFS-Berechtigung regeln = Best Practice.
> Granulare Steuerung nur über NTFS-Ebene — keine doppelte Bürokratie.

---

## Hidden Shares (Administrative Freigaben)

```powershell
# Administrative Standard-Freigaben
# C$, D$, E$    → Laufwerke (nur Admins)
# ADMIN$        → %SYSTEMROOT% (Remoteverwaltung)
# IPC$          → Interprozesskommunikation

# Versteckte Freigabe erstellen ($ am Ende)
New-SmbShare -Name "Intern$" -Path "D:\Intern" -FullAccess "FIRMA\IT-Admins"
# Nicht in Netzwerkumgebung sichtbar, direkte Pfadeingabe möglich

# Alle Freigaben (auch $)
Get-SmbShare | Where-Object {$_.Name -like "*$"}
```

---

## SMB-Sitzungen und geöffnete Dateien

```powershell
# Aktive SMB-Sitzungen anzeigen
Get-SmbSession

# Offene Dateien
Get-SmbOpenFile

# Sitzung trennen (nach Meldung an Benutzer!)
Get-SmbSession | Close-SmbSession -Force

# Geöffnete Datei schließen
Close-SmbOpenFile -FileId <ID> -Force
```

---

## Zugriffsenumeration (ABE)

Access-Based Enumeration blendet Ordner/Dateien aus, auf die kein Zugriff besteht:

```powershell
# ABE für Freigabe aktivieren
Set-SmbShare -Name "HR-Daten" -FolderEnumerationMode AccessBased

# Prüfen
Get-SmbShare -Name "HR-Daten" | Select-Object Name, FolderEnumerationMode
```

---

## SMB-Verschlüsselung

```powershell
# Verschlüsselung für einzelne Freigabe
Set-SmbShare -Name "HR-Daten" -EncryptData $true

# Verschlüsselung für gesamten Server erzwingen
Set-SmbServerConfiguration -EncryptData $true -Force

# SMB-Version und Verschlüsselung prüfen
Get-SmbServerConfiguration | Select-Object Smb2, EncryptData, RequireSecuritySignature
Get-SmbConnection   # Sitzungen und verwendete SMB-Version
```

---

## SMB-Versionen

```powershell
# SMB-Versionen auf Server anzeigen
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol

# SMBv1 DEAKTIVIEREN (Sicherheitsrisiko — WannaCry nutzte SMBv1)
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force

# SMBv2 aktivieren (Standard, sollte aktiv sein)
Set-SmbServerConfiguration -EnableSMB2Protocol $true -Force

# SMB-Version auf Client prüfen
Get-SmbConnection | Select-Object ServerName, SmbInstance, Dialect
```

> [!DANGER] SMBv1 deaktivieren!
> SMBv1 ist veraltet und Einfallstor für EternalBlue/WannaCry. In allen Windows Server 2019+ Umgebungen ist es standardmäßig deaktiviert — prüfen und aktivieren wenn noch aktiv!

---

## Freigabe-Mapping per GPO (für Clients)

```
GPO → Benutzerkonfiguration → Einstellungen → Windows-Einstellungen → Laufwerkzuordnungen

Neu → Zugeordnetes Laufwerk:
- Aktion: Erstellen
- Laufwerk: M:
- Pfad: \\SRV-FS01\Marketing
- Label: Marketing
- Verbinden als: (leer = angemeldeter Benutzer)
- Element-Zielgruppenadressierung: Nur für Marketing-Gruppe
```

```powershell
# PowerShell: Laufwerk verbinden
New-PSDrive -Name "M" -PSProvider FileSystem -Root "\\SRV-FS01\Marketing" -Persist

# Net Use (klassisch)
net use M: \\SRV-FS01\Marketing /persistent:yes
```

---

## Audit (Überwachung von Dateizugriffen)

```powershell
# Überwachungsrichtlinie aktivieren
auditpol /set /subcategory:"Objektzugriff" /success:enable /failure:enable

# NTFS-Audit auf Verzeichnis aktivieren
$acl = Get-Acl "D:\Freigaben\HR"
$auditRule = New-Object System.Security.AccessControl.FileSystemAuditRule(
    "Everyone", "Delete,Write", "ContainerInherit,ObjectInherit", "None", "Success,Failure"
)
$acl.AddAuditRule($auditRule)
Set-Acl -Path "D:\Freigaben\HR" -AclObject $acl

# Audit-Ereignisse anzeigen (Event 4663 = Dateizugriff)
Get-WinEvent -LogName Security |
    Where-Object {$_.Id -eq 4663} |
    Select-Object -First 20
```

---

## Verwandte Notizen

- [[NTFS Berechtigungen]] — Dateisystem-Berechtigungen (zweite Ebene)
- [[DFS]] — Verteilte Freigaben mit DFS Namespace
- [[AD Grundlagen]] — AD-Gruppen für Freigabe-Berechtigungen
- [[Group Policy]] — Laufwerkszuordnungen per GPO
