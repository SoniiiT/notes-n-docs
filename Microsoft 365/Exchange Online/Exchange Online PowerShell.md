---
tags:
  - exchange-online
  - powershell
  - automatisierung
  - verwaltung
  - scripting
aliases:
  - EXO PowerShell
  - ExchangeOnlineManagement
  - Exchange PowerShell
---

# Exchange Online PowerShell

> [!ABSTRACT] TL;DR
> Das `ExchangeOnlineManagement`-Modul (V3) verbindet PowerShell mit Exchange Online über moderne REST-basierte APIs. Es unterstützt zertifikatbasierte Authentifizierung für unbeaufsichtigte Skripte, MFA-kompatible interaktive Anmeldung und bietet hunderte Cmdlets für Postfach-, Transport- und Schutzverwaltung.

---

## Modul installieren und verbinden

### Installation

```powershell
# ExchangeOnlineManagement Modul installieren (PowerShell 5.1+ oder PowerShell 7+)
Install-Module -Name ExchangeOnlineManagement -Force -AllowClobber -Scope CurrentUser

# Aktuell installierte Version prüfen
Get-Module ExchangeOnlineManagement -ListAvailable | Select-Object Name, Version

# Modul aktualisieren
Update-Module -Name ExchangeOnlineManagement

# Modul laden
Import-Module ExchangeOnlineManagement
```

> [!IMPORTANT] PowerShell V3 Modul
> Ab Version 3.0 des `ExchangeOnlineManagement`-Moduls werden Cmdlets **REST-basiert** ausgeführt. Das bedeutet schnellere Ausführung, aber einige ältere Parameter sind nicht mehr verfügbar. Für Legacy-Cmdlets kann `-UseRPSSession` genutzt werden.

### Interaktive Verbindung (mit MFA)

```powershell
# Standard-Verbindung (MFA-kompatibel, Browser-Fenster öffnet sich)
Connect-ExchangeOnline -UserPrincipalName admin@contoso.com

# Mit Tenant-ID (empfohlen für Multi-Tenant-Szenarien)
Connect-ExchangeOnline -UserPrincipalName admin@contoso.com `
    -Organization "contoso.onmicrosoft.com"

# Verbindung testen
Get-OrganizationConfig | Select-Object DisplayName, Name
```

### Verbindung trennen

```powershell
# Aktuelle Session beenden
Disconnect-ExchangeOnline -Confirm:$false
```

---

## Zertifikatbasierte Authentifizierung (CBA)

Für Skripte ohne interaktive Anmeldung (Automation, Scheduled Tasks) wird Certificate-Based Authentication empfohlen.

### Zertifikat erstellen und vorbereiten

```powershell
# Selbstsigniertes Zertifikat erstellen
$cert = New-SelfSignedCertificate `
    -Subject "CN=ExchangeOnlineApp" `
    -CertStoreLocation "cert:\CurrentUser\My" `
    -NotAfter (Get-Date).AddYears(2) `
    -KeySpec Signature

# Zertifikat exportieren (öffentlicher Schlüssel für Azure AD App-Registrierung)
Export-Certificate -Cert $cert -FilePath "C:\certs\ExchangeOnlineApp.cer"

# Zertifikat mit privatem Schlüssel exportieren (für Skript-Server)
$password = ConvertTo-SecureString "ZertifikatPasswort!" -AsPlainText -Force
Export-PfxCertificate -Cert $cert `
    -FilePath "C:\certs\ExchangeOnlineApp.pfx" `
    -Password $password
```

### Azure AD App-Registrierung konfigurieren

```powershell
# 1. App in Entra ID registrieren:
#    entra.microsoft.com → App-Registrierungen → Neue Registrierung
#    Name: "Exchange Online Automation"
#    Kontotyp: Nur diese Organisation
#    → Registrieren

# 2. API-Berechtigungen hinzufügen:
#    APIs → Exchange → Anwendungsberechtigungen → Exchange.ManageAsApp

# 3. Admin-Zustimmung erteilen

# 4. Zertifikat hochladen:
#    Zertifikate und Geheimnisse → Zertifikate → Hochladen
#    Die .cer-Datei hochladen

# 5. Exchange Admin-Rolle zuweisen:
#    Entra ID → Rollen → Exchange-Administrator → Mitglieder → App hinzufügen
```

### Skript-Verbindung mit Zertifikat

```powershell
# Variante 1: Zertifikat aus Zertifikatsspeicher
Connect-ExchangeOnline `
    -CertificateThumbprint "ABC123DEF456..." `
    -AppId "12345678-1234-1234-1234-123456789012" `
    -Organization "contoso.onmicrosoft.com"

# Variante 2: Zertifikat aus PFX-Datei
Connect-ExchangeOnline `
    -CertificateFilePath "C:\certs\ExchangeOnlineApp.pfx" `
    -CertificatePassword (ConvertTo-SecureString "ZertifikatPasswort!" -AsPlainText -Force) `
    -AppId "12345678-1234-1234-1234-123456789012" `
    -Organization "contoso.onmicrosoft.com"

# In Scheduled Task / Azure Automation nutzen
# Kein Browser-Fenster, keine interaktive Eingabe nötig
```

---

## Wichtige Cmdlets-Referenz

### Postfach-Verwaltung

```powershell
# Postfach abrufen
Get-Mailbox -Identity "user@contoso.com"
Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails UserMailbox
Get-Mailbox -Filter {CustomAttribute1 -eq "VIP"}

# Postfach konfigurieren
Set-Mailbox -Identity "user@contoso.com" -DisplayName "Neuer Name"

# Postfachstatistiken
Get-MailboxStatistics -Identity "user@contoso.com"
Get-MailboxStatistics -ResultSize Unlimited | Sort-Object TotalItemSize -Descending

# Postfach aktivieren/deaktivieren
Enable-Mailbox -Identity "user@contoso.com"
Disable-Mailbox -Identity "user@contoso.com"

# Ordner-Statistiken
Get-MailboxFolderStatistics -Identity "user@contoso.com" -FolderScope Inbox
```

### Empfänger

```powershell
# Alle Empfängertypen suchen
Get-Recipient -ResultSize Unlimited | Select-Object Name, RecipientTypeDetails, PrimarySmtpAddress
Get-Recipient -Filter {Department -eq "Vertrieb"}

# Kontakte verwalten
New-MailContact -Name "Externer Partner" -ExternalEmailAddress "partner@extern.de"
Set-MailContact -Identity "partner@extern.de" -Company "Partner GmbH"
```

### Gruppen

```powershell
# Verteilergruppen
Get-DistributionGroup -ResultSize Unlimited
New-DistributionGroup -Name "Test" -Alias "test" -PrimarySmtpAddress "test@contoso.com"
Get-DistributionGroupMember -Identity "gruppe@contoso.com"

# M365-Gruppen
Get-UnifiedGroup -ResultSize Unlimited
New-UnifiedGroup -DisplayName "Projekt" -Alias "projekt"
Get-UnifiedGroupLinks -Identity "projekt@contoso.com" -LinkType Members

# Dynamische Gruppen
Get-DynamicDistributionGroup
```

### Transport und Mail Flow

```powershell
# Transport Rules
Get-TransportRule | Sort-Object Priority | Select-Object Priority, Name, State
New-TransportRule -Name "Regel" -Conditions ... -Actions ...
Set-TransportRule -Identity "Regel" -Priority 1
Enable-TransportRule / Disable-TransportRule

# Konnektoren
Get-InboundConnector
Get-OutboundConnector
New-InboundConnector / New-OutboundConnector

# Akzeptierte Domänen
Get-AcceptedDomain
New-AcceptedDomain

# Message Trace
Get-MessageTrace -SenderAddress "x@y.de" -StartDate (Get-Date).AddDays(-1)
Get-MessageTraceDetail -MessageTraceId "ID" -RecipientAddress "emp@contoso.com"
```

### Schutz und Sicherheit

```powershell
# Anti-Spam
Get-HostedContentFilterPolicy
Set-HostedContentFilterPolicy -Identity "Default" -SpamAction MoveToJmf

# Anti-Malware
Get-MalwareFilterPolicy
Set-MalwareFilterPolicy -Identity "Default" -Action DeleteMessage

# Anti-Phishing
Get-AntiPhishPolicy
New-AntiPhishPolicy / Set-AntiPhishPolicy

# Safe Links / Safe Attachments (MDO)
Get-SafeLinksPolicy
Get-SafeAttachmentPolicy

# Quarantäne
Get-QuarantineMessage -ResultSize Unlimited
Release-QuarantineMessage -Identity "ID"

# DKIM
Get-DkimSigningConfig
Set-DkimSigningConfig -Identity contoso.com -Enabled $true
```

### Berechtigungen

```powershell
# Postfachberechtigungen
Add-MailboxPermission / Remove-MailboxPermission
Get-MailboxPermission -Identity "postfach@contoso.com"

# Send As
Add-RecipientPermission / Remove-RecipientPermission
Get-RecipientPermission

# Kalenderberechtigungen
Add-MailboxFolderPermission / Set-MailboxFolderPermission / Remove-MailboxFolderPermission
Get-MailboxFolderPermission -Identity "user@contoso.com:\Kalender"
```

---

## PowerShell-Rezepte (Praxis)

### Rezept 1: Postfachgrößenbericht erstellen

```powershell
Connect-ExchangeOnline -Organization "contoso.onmicrosoft.com" -CertificateThumbprint "THUMBPRINT" -AppId "APPID"

$report = Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails UserMailbox | ForEach-Object {
    $stats = Get-MailboxStatistics -Identity $_.Identity -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Anzeigename    = $_.DisplayName
        Email          = $_.PrimarySmtpAddress
        Abteilung      = $_.Department
        Groesse_GB     = if ($stats) { [math]::Round($stats.TotalItemSize.Value.ToGB(), 2) } else { 0 }
        Elemente       = if ($stats) { $stats.ItemCount } else { 0 }
        Letzter_Login  = if ($stats) { $stats.LastLogonTime } else { "Nie" }
        Archiv_Status  = $_.ArchiveStatus
    }
}

$report | Sort-Object Groesse_GB -Descending | 
    Export-Csv "C:\berichte\postfach-groessen.csv" -Encoding UTF8 -NoTypeInformation

Write-Output "Bericht erstellt: $($report.Count) Postfächer"
Disconnect-ExchangeOnline -Confirm:$false
```

### Rezept 2: Massenänderung — Sprache und Zeitzone für alle Postfächer

```powershell
$mailboxes = Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails UserMailbox

$total = $mailboxes.Count
$counter = 0

foreach ($mb in $mailboxes) {
    $counter++
    Write-Progress -Activity "Sprache setzen" -Status "$counter von $total" -PercentComplete (($counter/$total)*100)
    
    try {
        Set-MailboxRegionalConfiguration -Identity $mb.Identity `
            -Language "de-DE" `
            -TimeZone "W. Europe Standard Time" `
            -DateFormat "dd.MM.yyyy" `
            -ErrorAction Stop
    } catch {
        Write-Warning "Fehler bei $($mb.DisplayName): $($_.Exception.Message)"
    }
}

Write-Output "Fertig: $counter Postfächer aktualisiert"
```

### Rezept 3: Alle Weiterleitungsregeln finden

```powershell
# Weiterleitungen via Set-Mailbox (ForwardingSmtpAddress)
$weiterleitungen = Get-Mailbox -ResultSize Unlimited | 
    Where-Object {$_.ForwardingSmtpAddress -ne $null} |
    Select-Object DisplayName, PrimarySmtpAddress, ForwardingSmtpAddress, DeliverToMailboxAndForward

# Weiterleitungsregeln in Outlook (Inbox Rules)
$alleRegeln = Get-Mailbox -ResultSize Unlimited | ForEach-Object {
    Get-InboxRule -Mailbox $_.Identity -ErrorAction SilentlyContinue | 
        Where-Object {$_.ForwardTo -ne $null -or $_.RedirectTo -ne $null}
}

$weiterleitungen | Export-Csv "C:\berichte\weiterleitungen.csv" -Encoding UTF8
Write-Output "Gefunden: $($weiterleitungen.Count) Postfach-Weiterleitungen"
Write-Output "Postfachregeln mit Weiterleitung: $($alleRegeln.Count)"
```

### Rezept 4: Shared Mailboxes ohne Berechtigung finden

```powershell
$sharedMailboxes = Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails SharedMailbox

$ohneBerechtigung = foreach ($mb in $sharedMailboxes) {
    $berechtigungen = Get-MailboxPermission -Identity $mb.Identity | 
        Where-Object {$_.AccessRights -eq "FullAccess" -and 
                     $_.IsInherited -eq $false -and 
                     $_.User -notlike "NT AUTHORITY*" -and
                     $_.Deny -eq $false}
    
    if ($berechtigungen.Count -eq 0) {
        [PSCustomObject]@{
            Name  = $mb.DisplayName
            Email = $mb.PrimarySmtpAddress
            Groesse = (Get-MailboxStatistics -Identity $mb.Identity -ErrorAction SilentlyContinue).TotalItemSize
        }
    }
}

$ohneBerechtigung | Format-Table -AutoSize
```

### Rezept 5: Message Trace für die letzten 24 Stunden exportieren

```powershell
$startzeit = (Get-Date).AddDays(-1)
$endzeit = Get-Date

$trace = Get-MessageTrace `
    -StartDate $startzeit `
    -EndDate $endzeit `
    -PageSize 5000 |
    Select-Object Received, SenderAddress, RecipientAddress, Subject, Status, Size, MessageTraceId

$trace | Export-Csv "C:\berichte\message-trace-$(Get-Date -Format 'yyyyMMdd').csv" `
    -Encoding UTF8 -NoTypeInformation

Write-Output "Exportiert: $($trace.Count) Nachrichten"
```

### Rezept 6: Inaktive Postfächer inventarisieren

```powershell
$inaktiv = Get-Mailbox -InactiveMailboxOnly -ResultSize Unlimited | ForEach-Object {
    $stats = Get-MailboxStatistics -Identity $_.DistinguishedName -InactiveMailboxOnly -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Name             = $_.DisplayName
        Email            = $_.PrimarySmtpAddress
        GeloeschtAm      = $_.WhenSoftDeleted
        Groesse_GB       = if ($stats) { [math]::Round($stats.TotalItemSize.Value.ToGB(), 2) } else { 0 }
        LitigationHold   = $_.LitigationHoldEnabled
        RetentionPolicy  = $_.RetentionPolicy
    }
}

$inaktiv | Sort-Object GeloeschtAm -Descending | 
    Export-Csv "C:\berichte\inaktive-postfaecher.csv" -Encoding UTF8
```

---

## Microsoft Graph als Alternative

Für moderne Anwendungen und Automatisierungen bietet **Microsoft Graph** eine REST-API-Alternative zu Exchange PowerShell.

```powershell
# Microsoft Graph PowerShell Modul installieren
Install-Module Microsoft.Graph -Scope CurrentUser

# Verbinden
Connect-MgGraph -Scopes "Mail.Read","Mail.Send","User.Read.All","MailboxSettings.ReadWrite"

# Postfacheinstellungen über Graph abrufen
Get-MgUserMailboxSetting -UserId "user@contoso.com"

# E-Mail senden über Graph
$message = @{
    Subject = "Test über Microsoft Graph"
    Body = @{ ContentType = "Text"; Content = "Hallo Welt" }
    ToRecipients = @(@{ EmailAddress = @{ Address = "empfaenger@contoso.com" } })
}
Send-MgUserMail -UserId "absender@contoso.com" -Message $message

# Graph vs. Exchange PowerShell
# Graph: Moderne REST-API, für Anwendungen optimiert, unterstützt alle M365-Dienste
# Exchange PS: Vollständige Verwaltungsfunktionen, besser für Exchange-spezifische Administration
```

---

## Fehlerbehandlung

```powershell
# Robustes Skript-Template mit Fehlerbehandlung
function Invoke-ExchangeTask {
    param(
        [string]$Organization,
        [string]$AppId,
        [string]$CertThumbprint
    )
    
    try {
        # Verbinden
        Connect-ExchangeOnline -Organization $Organization `
            -AppId $AppId `
            -CertificateThumbprint $CertThumbprint `
            -ErrorAction Stop
        
        Write-Output "Verbunden mit $Organization"
        
        # Hauptlogik
        $mailboxes = Get-Mailbox -ResultSize Unlimited -ErrorAction Stop
        Write-Output "Gefunden: $($mailboxes.Count) Postfächer"
        
        # Verarbeitung...
        
    } catch [System.Management.Automation.CommandNotFoundException] {
        Write-Error "Modul nicht geladen. 'Import-Module ExchangeOnlineManagement' ausführen."
    } catch [Microsoft.Exchange.Configuration.Tasks.ManagementObjectNotFoundException] {
        Write-Error "Objekt nicht gefunden: $($_.Exception.Message)"
    } catch {
        Write-Error "Unbekannter Fehler: $($_.Exception.Message)"
        Write-Error $_.ScriptStackTrace
    } finally {
        # Immer trennen, auch bei Fehler
        Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue
        Write-Output "Verbindung getrennt"
    }
}

# Aufruf
Invoke-ExchangeTask -Organization "contoso.onmicrosoft.com" `
    -AppId "12345678-..." `
    -CertThumbprint "ABC123..."
```

---

## Nützliche Aliase und Tipps

```powershell
# Cmdlets nach Kategorie suchen
Get-Command -Module ExchangeOnlineManagement | Group-Object Verb | Sort-Object Count -Descending

# Hilfe zu einem Cmdlet
Get-Help Get-Mailbox -Detailed
Get-Help Get-Mailbox -Examples
Get-Help New-TransportRule -Parameter Conditions

# Ausgabe formatieren
Get-Mailbox | Select-Object Name, PrimarySmtpAddress | Format-Table -AutoSize
Get-Mailbox | Select-Object Name, PrimarySmtpAddress | Format-List
Get-Mailbox | Select-Object Name, PrimarySmtpAddress | Out-GridView  # Interaktives Fenster

# In Variable speichern und filtern
$alle = Get-Mailbox -ResultSize Unlimited
$alle | Where-Object {$_.Department -eq "IT"} | Select-Object Name

# Für große Tenants: Paginierung
$seite = 1
do {
    $batch = Get-Mailbox -ResultSize 1000 -Skip (($seite-1)*1000)
    # Verarbeiten...
    $seite++
} while ($batch.Count -eq 1000)
```

---

## Verwandte Notizen

- [[Exchange Online Grundlagen]]
- [[Postfächer verwalten]]
- [[Mail Flow und Transport Rules]]
- [[Anti-Spam und Anti-Phishing]]
- [[Postfach-Berechtigungen]]
- [[E-Mail-Migration]]
