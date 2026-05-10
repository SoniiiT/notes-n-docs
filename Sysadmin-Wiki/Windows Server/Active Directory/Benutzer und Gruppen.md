---
tags: [active-directory, benutzer, gruppen, ou, aduc, powershell, bulk, import]
aliases: ["Benutzer und Gruppen", "AD Benutzer", "AD Gruppen", "ADUC"]
---

# Benutzer und Gruppen

> [!ABSTRACT]
> Verwaltung von Benutzern, Gruppen und OUs in Active Directory — über die Konsole Active Directory-Benutzer und -Computer (ADUC) sowie vollständig per PowerShell (ActiveDirectory-Modul). Enthält Bulk-Import aus CSV und Delegierungskonzepte.

---

## Active Directory-Benutzer und -Computer (ADUC)

```
Öffnen:
- Server Manager → Tools → Active Directory-Benutzer und -Computer
- dsa.msc
- Windows Admin Center → AD-Extension
```

### Ansicht konfigurieren

```
Ansicht → Erweiterte Features aktivieren
→ Zeigt zusätzliche Attribute (Objekt-GUID, Security Descriptor, etc.)

Ansicht → Filter → Benutzer, Gruppen, Computer
→ Schneller in großen OUs navigieren
```

---

## Benutzer verwalten

### GUI (ADUC)

```
Active Directory-Benutzer und -Computer (dsa.msc)

Neuen Benutzer erstellen:
→ OU auswählen (z.B. OU=IT) → Rechtsklick → Neu → Benutzer
  Vorname:              Max
  Nachname:             Mustermann
  Anmeldename:          mmustermann
  → Weiter
  Kennwort:             TempPass!2024
  ✓ Benutzer muss Kennwort bei nächster Anmeldung ändern
  → Weiter → Fertigstellen

Benutzer bearbeiten:
→ Benutzer doppelklicken → Registerkarten:
  Allgemein:     Anzeigename, Beschreibung, Telefon
  Konto:         Anmeldename, Kontosperrung, Ablaufdatum
  Profil:        Anmeldeskript, Profilpfad, Basisordner
  Mitglied von:  Gruppenmitgliedschaften
  Einwählen:     VPN-Zugriffsrechte

Passwort zurücksetzen:
→ Benutzer → Rechtsklick → Kennwort zurücksetzen
  Neues Kennwort eingeben + bestätigen
  ✓ Benutzer muss Kennwort bei nächster Anmeldung ändern
  ✓ Konto entsperren (wenn gesperrt)

Konto deaktivieren:
→ Benutzer → Rechtsklick → Konto deaktivieren
```

### PowerShell

```powershell
# Modul laden
Import-Module ActiveDirectory

# --- LESEN ---

# Alle Benutzer
Get-ADUser -Filter *

# Mit Attributen
Get-ADUser -Filter * -Properties DisplayName, EmailAddress, Department, LastLogonDate, Enabled |
    Select-Object SamAccountName, DisplayName, EmailAddress, Department, LastLogonDate, Enabled

# Bestimmten Benutzer suchen
Get-ADUser -Identity "jdoe"
Get-ADUser -Filter {DisplayName -eq "John Doe"}
Get-ADUser -Filter {Department -eq "IT" -and Enabled -eq $true}

# Inaktive Benutzer (90 Tage nicht angemeldet)
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
    -Properties LastLogonDate | Select-Object Name, LastLogonDate

# --- ERSTELLEN ---

New-ADUser `
    -Name "Max Mustermann" `
    -GivenName "Max" `
    -Surname "Mustermann" `
    -SamAccountName "mmustermann" `
    -UserPrincipalName "mmustermann@firma.local" `
    -EmailAddress "m.mustermann@firma.de" `
    -Department "IT" `
    -Title "Systemadministrator" `
    -Path "OU=IT,OU=Benutzer,DC=firma,DC=local" `
    -AccountPassword (ConvertTo-SecureString "TempPass!2024" -AsPlainText -Force) `
    -ChangePasswordAtLogon $true `
    -Enabled $true

# --- ÄNDERN ---

Set-ADUser -Identity "mmustermann" -Department "IT-Infrastruktur"
Set-ADUser -Identity "mmustermann" -Manager (Get-ADUser "itchef")
Set-ADUser -Identity "mmustermann" -Description "Server-Admin, Erstellt 2024"

# Passwort zurücksetzen
Set-ADAccountPassword -Identity "mmustermann" `
    -Reset `
    -NewPassword (ConvertTo-SecureString "NeuesPass!2024" -AsPlainText -Force)
Set-ADUser -Identity "mmustermann" -ChangePasswordAtLogon $true

# --- AKTIVIEREN/SPERREN ---

Disable-ADAccount -Identity "mmustermann"
Enable-ADAccount -Identity "mmustermann"
Unlock-ADAccount -Identity "mmustermann"

# Gesperrte Konten finden
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LockedOut

# --- LÖSCHEN ---

Remove-ADUser -Identity "mmustermann" -Confirm:$false

# Vor Löschen: Aus Gruppen entfernen
(Get-ADUser -Identity "mmustermann" -Properties MemberOf).MemberOf |
    ForEach-Object { Remove-ADGroupMember -Identity $_ -Members "mmustermann" -Confirm:$false }
```

---

## Bulk-Import aus CSV

```powershell
# CSV-Datei: benutzer.csv
# Name,Vorname,Nachname,Login,Email,Abteilung,OU
# "Max Mustermann","Max","Mustermann","mmustermann","m.muster@firma.de","IT","OU=IT,OU=Benutzer,DC=firma,DC=local"

$benutzer = Import-Csv -Path "C:\Scripts\benutzer.csv" -Delimiter ","

foreach ($user in $benutzer) {
    try {
        New-ADUser `
            -Name $user.Name `
            -GivenName $user.Vorname `
            -Surname $user.Nachname `
            -SamAccountName $user.Login `
            -UserPrincipalName "$($user.Login)@firma.local" `
            -EmailAddress $user.Email `
            -Department $user.Abteilung `
            -Path $user.OU `
            -AccountPassword (ConvertTo-SecureString "TempPass!2024" -AsPlainText -Force) `
            -ChangePasswordAtLogon $true `
            -Enabled $true

        Write-Host "Erstellt: $($user.Name)" -ForegroundColor Green
    } catch {
        Write-Warning "Fehler bei $($user.Name): $_"
    }
}
```

---

## Gruppen verwalten

### GUI (ADUC)

```
Active Directory-Benutzer und -Computer:

Neue Gruppe erstellen:
→ Ziel-OU → Rechtsklick → Neu → Gruppe
  Gruppenname:       IT-Administratoren
  Gruppenbereich:    Global  (DomainLocal / Global / Universal)
  Gruppentyp:        Sicherheit  (oder Verteilung für E-Mail)
  → OK

Mitglieder hinzufügen:
→ Gruppe doppelklicken → Mitglieder → Hinzufügen
  → Benutzer/Computer/Gruppen suchen und hinzufügen

Benutzer zu Gruppe hinzufügen (vom Benutzer aus):
→ Benutzer doppelklicken → Mitglied von → Hinzufügen
  → Gruppe suchen und hinzufügen
```

### PowerShell

```powershell
# --- ERSTELLEN ---

New-ADGroup `
    -Name "IT-Administratoren" `
    -SamAccountName "IT-Administratoren" `
    -GroupCategory Security `
    -GroupScope Global `
    -Path "OU=Gruppen,DC=firma,DC=local" `
    -Description "Alle IT-Administratoren"

# GroupScope: DomainLocal, Global, Universal
# GroupCategory: Security, Distribution

# --- MITGLIEDER ---

# Benutzer hinzufügen
Add-ADGroupMember -Identity "IT-Administratoren" -Members "mmustermann", "jdoe"

# Computer hinzufügen (z.B. für GPO)
Add-ADGroupMember -Identity "VPN-Clients" -Members (Get-ADComputer "WRK-IT01")

# Gruppe in Gruppe verschachteln
Add-ADGroupMember -Identity "Alle-IT" -Members "IT-Administratoren", "IT-Helpdesk"

# Mitglieder anzeigen
Get-ADGroupMember -Identity "IT-Administratoren" -Recursive |
    Select-Object Name, objectClass

# Mitgliedschaften eines Benutzers
(Get-ADUser "mmustermann" -Properties MemberOf).MemberOf |
    ForEach-Object { (Get-ADGroup $_).Name }

# Mitglied entfernen
Remove-ADGroupMember -Identity "IT-Administratoren" -Members "mmustermann" -Confirm:$false

# --- LÖSCHEN ---
Remove-ADGroup -Identity "IT-Administratoren" -Confirm:$false
```

---

## OUs (Organizational Units)

```powershell
# OU erstellen
New-ADOrganizationalUnit `
    -Name "IT" `
    -Path "OU=Benutzer,DC=firma,DC=local" `
    -Description "IT-Abteilung" `
    -ProtectedFromAccidentalDeletion $true

# OU umbenennen
Rename-ADObject -Identity "OU=IT,OU=Benutzer,DC=firma,DC=local" -NewName "IT-Infrastruktur"

# OU verschieben
Move-ADObject `
    -Identity "OU=IT,OU=Benutzer,DC=firma,DC=local" `
    -TargetPath "OU=Technik,DC=firma,DC=local"

# Objekte in eine andere OU verschieben
Move-ADObject -Identity "CN=Max Mustermann,OU=Benutzer,DC=firma,DC=local" `
    -TargetPath "OU=IT,OU=Benutzer,DC=firma,DC=local"

# OU löschen (erst Löschschutz entfernen!)
Set-ADOrganizationalUnit -Identity "OU=Test,DC=firma,DC=local" `
    -ProtectedFromAccidentalDeletion $false
Remove-ADOrganizationalUnit -Identity "OU=Test,DC=firma,DC=local" -Confirm:$false
```

---

## Computer-Accounts

```powershell
# Computer-Accounts anzeigen
Get-ADComputer -Filter * -Properties OperatingSystem, LastLogonDate |
    Select-Object Name, OperatingSystem, LastLogonDate, Enabled

# Veraltete Computer (90 Tage nicht angemeldet)
$cutoff = (Get-Date).AddDays(-90)
Get-ADComputer -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
    -Properties LastLogonDate | Select-Object Name, LastLogonDate

# Computer in OU verschieben
Get-ADComputer -Filter {OperatingSystem -like "*Server*"} |
    Move-ADObject -TargetPath "OU=Server,DC=firma,DC=local"
```

---

## Delegierung (Delegation of Control)

```powershell
# Helpdesk darf Passwörter zurücksetzen (auf OU-Ebene)
# Via GUI: OU → Rechtsklick → Steuerelement zuweisen (Delegation of Control Wizard)

# Per PowerShell (ACL auf OU):
$ou = Get-ADOrganizationalUnit "OU=Benutzer,DC=firma,DC=local"
$helpdesk = Get-ADGroup "IT-Helpdesk"

# ACL lesen
$acl = Get-Acl "AD:$($ou.DistinguishedName)"

# Regel für "Passwort zurücksetzen" hinzufügen
$sid = [System.Security.Principal.SecurityIdentifier]$helpdesk.SID
$adRights = [System.DirectoryServices.ActiveDirectoryRights]::WriteProperty
$guid = [GUID]"00299570-246d-11d0-a768-00aa006e0529"  # Reset Password GUID

$ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid, $adRights, "Allow", $guid, "Descendents",
    [GUID]"bf967aba-0de6-11d0-a285-00aa003049e2"  # User object
)
$acl.AddAccessRule($ace)
Set-Acl -Path "AD:$($ou.DistinguishedName)" -AclObject $acl
```

---

## Verwandte Notizen

- [[AD Grundlagen]] — AGDLP-Prinzip, Gruppentypen
- [[Group Policy]] — GPOs auf OUs anwenden
- [[AD Troubleshooting]] — Anmeldeprobleme, gesperrte Konten
