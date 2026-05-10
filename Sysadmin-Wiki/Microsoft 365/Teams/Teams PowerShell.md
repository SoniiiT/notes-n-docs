---
tags:
  - microsoft365
  - teams
  - powershell
  - administration
  - automatisierung
aliases:
  - Teams PowerShell
  - MicrosoftTeams Modul
  - Teams PS
---

# Teams PowerShell

> [!ABSTRACT] TL;DR
> Das MicrosoftTeams PowerShell-Modul ist das primäre Verwaltungswerkzeug für Microsoft Teams. Es kombiniert Teams-spezifische Cmdlets (Teams, Kanäle, Mitglieder) mit Skype/CS-Cmdlets für Telefonie und Policies. Für komplexe Szenarien kann die Microsoft Graph API direkt über `Invoke-MgGraphRequest` oder das Microsoft.Graph-Modul angesprochen werden.

---

## Via Teams Admin Center (UI)

> [!NOTE] TAC-Dokumentation in den Fachartikeln
> Diese Seite dokumentiert ausschließlich die **PowerShell-Verwaltung** von Microsoft Teams über das MicrosoftTeams-Modul und die Microsoft Graph API. Die Verwaltung über das **Teams Admin Center** ([admin.teams.microsoft.com](https://admin.teams.microsoft.com)) ist in den jeweiligen Fachartikeln mit Schritt-für-Schritt-Anleitungen dokumentiert:
>
> - [[Teams und Kanäle verwalten]] — Teams erstellen, archivieren, wiederherstellen, Mitglieder und Rollen verwalten
> - [[Governance und Policies]] — Meeting-, Messaging- und App-Policies erstellen und zuweisen
> - [[Teams-Telefonie]] — Rufnummern, Auto Attendant, Call Queues, Operator Connect, Notfallstandorte
> - [[Meetings und Live Events]] — Besprechungsrichtlinien und Live-Ereignisrichtlinien konfigurieren
> - [[Teams Apps und Bots]] — App-Berechtigungs- und App-Setuprichtlinien verwalten
> - [[Teams Rooms]] — MTR-Geräte überwachen, Lizenzen zuweisen, Remote-Verwaltung
> - [[Teams Premium]] — Premium-Features, Custom Templates und Meeting Branding konfigurieren
> - [[Teams Phone Mobile]] — Operator Connect Mobile (Carrier autorisieren, Nummern zuweisen)

---

## Modul-Installation und Verbindung

### Installation

```powershell
# MicrosoftTeams-Modul installieren (empfohlen: aus PSGallery)
Install-Module -Name MicrosoftTeams -Scope CurrentUser -Force -AllowClobber

# Version prüfen
Get-Module MicrosoftTeams -ListAvailable | Select-Object Name, Version

# Modul aktualisieren
Update-Module -Name MicrosoftTeams

# Ergänzend für Graph-API-Zugriff
Install-Module -Name Microsoft.Graph -Scope CurrentUser -Force
```

### Verbindung herstellen

```powershell
# Interaktive Anmeldung (mit Browser/MFA)
Connect-MicrosoftTeams

# Mit bestimmtem Tenant
Connect-MicrosoftTeams -TenantId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Mit Service Principal (Certificate Based Auth — CBA für Skripte)
Connect-MicrosoftTeams `
    -CertificateThumbprint "ABCDEF1234567890..." `
    -ApplicationId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
    -TenantId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Verbindung trennen
Disconnect-MicrosoftTeams

# Verbindungsstatus prüfen
Get-CsOnlineSession
```

> [!NOTE] Zwei Cmdlet-Familien
> Das MicrosoftTeams-Modul enthält zwei Cmdlet-Familien:
> - **`*-Team*`-Cmdlets** (z.B. `Get-Team`, `New-Team`): für Teams-Objekte
> - **`*-Cs*`-Cmdlets** (z.B. `Get-CsTeamsMeetingPolicy`): für Policies und Telefonie (aus Skype for Business)

---

## Teams verwalten

```powershell
# Alle Teams im Tenant anzeigen
Get-Team | Select-Object DisplayName, GroupId, Visibility, Archived, MailNickName |
    Sort-Object DisplayName

# Einzelnes Team abrufen (nach Name)
Get-Team -DisplayName "Projektteam Alpha"

# Team nach GroupId abrufen
Get-Team -GroupId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Neues Team erstellen
$newTeam = New-Team `
    -DisplayName "Projektteam Beta" `
    -Description "Projektteam für das Beta-Projekt" `
    -Visibility Private `
    -AllowMemberCreateUpdateChannels $false `
    -AllowMemberDeleteChannels $false

# Team-Einstellungen ändern
Set-Team -GroupId $newTeam.GroupId `
    -Description "Aktualisierte Beschreibung" `
    -AllowGuestCreateUpdateChannels $false

# Team löschen
Remove-Team -GroupId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Alle archivierten Teams anzeigen
Get-Team | Where-Object { $_.Archived -eq $true } |
    Select-Object DisplayName, GroupId

# Team-Report: Alle Teams mit Mitgliederzahl
Get-Team | ForEach-Object {
    $members = Get-TeamUser -GroupId $_.GroupId
    [PSCustomObject]@{
        Team        = $_.DisplayName
        GroupId     = $_.GroupId
        Visibility  = $_.Visibility
        Archived    = $_.Archived
        Members     = ($members | Where-Object { $_.Role -eq "Member" }).Count
        Owners      = ($members | Where-Object { $_.Role -eq "Owner" }).Count
        Guests      = ($members | Where-Object { $_.Role -eq "Guest" }).Count
        Total       = $members.Count
    }
} | Sort-Object Team | Export-Csv "Teams-Report.csv" -NoTypeInformation -Encoding UTF8
```

---

## Kanäle verwalten

```powershell
# Alle Kanäle eines Teams anzeigen
$groupId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
Get-TeamChannel -GroupId $groupId | Select-Object DisplayName, MembershipType, Id

# Kanal erstellen (Standard)
New-TeamChannel -GroupId $groupId -DisplayName "Planung" -MembershipType Standard

# Privaten Kanal erstellen
New-TeamChannel -GroupId $groupId -DisplayName "Vertraulich" -MembershipType Private

# Kanal-Einstellungen ändern
Set-TeamChannel -GroupId $groupId -CurrentDisplayName "Planung" -NewDisplayName "Projektplanung"

# Kanal löschen
Remove-TeamChannel -GroupId $groupId -DisplayName "Alter Kanal"

# Kanäle aus allen Teams exportieren
Get-Team | ForEach-Object {
    $teamName = $_.DisplayName
    Get-TeamChannel -GroupId $_.GroupId | ForEach-Object {
        [PSCustomObject]@{
            Team             = $teamName
            Kanal            = $_.DisplayName
            KanalTyp         = $_.MembershipType
        }
    }
} | Export-Csv "Alle-Kanaele.csv" -NoTypeInformation -Encoding UTF8
```

---

## Mitglieder verwalten

```powershell
# Alle Mitglieder eines Teams anzeigen
Get-TeamUser -GroupId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" |
    Select-Object Name, User, Role

# Mitglied hinzufügen
Add-TeamUser -GroupId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
    -User "user@contoso.com" `
    -Role Member

# Besitzer hinzufügen
Add-TeamUser -GroupId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
    -User "admin@contoso.com" `
    -Role Owner

# Mitglied entfernen
Remove-TeamUser -GroupId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
    -User "user@contoso.com"

# Gastzugang eines Benutzers prüfen (alle Teams wo er Gast ist)
$guestUPN = "gast@extern.com"
Get-Team | ForEach-Object {
    $members = Get-TeamUser -GroupId $_.GroupId
    $guestEntry = $members | Where-Object { $_.User -like "*$guestUPN*" -and $_.Role -eq "Guest" }
    if ($guestEntry) {
        [PSCustomObject]@{
            Team    = $_.DisplayName
            GroupId = $_.GroupId
            Guest   = $guestEntry.User
        }
    }
}
```

---

## Policies verwalten

### Meeting Policies

```powershell
# Alle Meeting Policies anzeigen
Get-CsTeamsMeetingPolicy | Select-Object Identity, AllowCloudRecording, AllowTranscription

# Neue Meeting Policy erstellen
New-CsTeamsMeetingPolicy -Identity "Executives-Meeting" `
    -AllowCloudRecording $true `
    -AllowTranscription $true `
    -AllowMeetingRegistration $false `
    -AutoAdmittedUsers "EveryoneInCompanyExcludingGuests"

# Bestehende Policy ändern
Set-CsTeamsMeetingPolicy -Identity "Executives-Meeting" `
    -AllowRecordingStorageOutsideRegion $false

# Policy einem Benutzer zuweisen
Grant-CsTeamsMeetingPolicy -Identity "user@contoso.com" -PolicyName "Executives-Meeting"

# Policy einer Gruppe zuweisen (Empfehlung für viele Benutzer)
New-CsGroupPolicyAssignment `
    -GroupId "<Entra-ID-Gruppen-ID>" `
    -PolicyType TeamsMeetingPolicy `
    -PolicyName "Executives-Meeting" `
    -Rank 1

# Policy-Zuweisung prüfen
Get-CsUserPolicyAssignment -Identity "user@contoso.com" -PolicyType TeamsMeetingPolicy

# Policy-Zuweisung entfernen (zurück zur Global-Policy)
Grant-CsTeamsMeetingPolicy -Identity "user@contoso.com" -PolicyName $null
```

### Messaging Policies

```powershell
# Alle Messaging Policies anzeigen
Get-CsTeamsMessagingPolicy | Select-Object Identity, AllowGiphy, AllowMemes, AllowUserDeleteMessage

# Neue Policy erstellen
New-CsTeamsMessagingPolicy -Identity "Frontline-Messaging" `
    -AllowGiphy $false `
    -AllowMemes $false `
    -AllowImmersiveReader $true `
    -AllowUserDeleteMessage $false

# Policy zuweisen
Grant-CsTeamsMessagingPolicy -Identity "user@contoso.com" -PolicyName "Frontline-Messaging"
```

---

## Telefonie verwalten

```powershell
# Alle Online-Benutzer mit Telefonnummer
Get-CsOnlineUser -Filter { EnterpriseVoiceEnabled -eq $true } |
    Select-Object UserPrincipalName, LineUri, EnterpriseVoiceEnabled, HostingProvider

# Telefonnummer zuweisen (Direct Routing)
Set-CsPhoneNumberAssignment -Identity "user@contoso.com" `
    -PhoneNumber "+4930123456789" `
    -PhoneNumberType DirectRouting

# Telefonnummer entfernen
Remove-CsPhoneNumberAssignment -Identity "user@contoso.com" `
    -PhoneNumber "+4930123456789" `
    -PhoneNumberType DirectRouting

# Alle zugewiesenen Nummern anzeigen
Get-CsPhoneNumberAssignment -NumberType DirectRouting |
    Select-Object TelephoneNumber, UserPrincipalName, NumberType

# Direct Routing SBC anzeigen
Get-CsOnlinePSTNGateway | Select-Object Fqdn, Enabled, SipSignalingPort, MaxConcurrentSessions

# Voice Routing Policy zuweisen
Grant-CsOnlineVoiceRoutingPolicy -Identity "user@contoso.com" -PolicyName "DE-Standard-Policy"

# Dial Plan zuweisen
Grant-CsTenantDialPlan -Identity "user@contoso.com" -PolicyName "DE-Dialplan"

# EnterpriseVoice für Benutzer aktivieren (bei Direct Routing)
Enable-CsOnlineUser -Identity "user@contoso.com"
Set-CsUser -Identity "user@contoso.com" -EnterpriseVoiceEnabled $true -HostedVoiceMail $true
```

---

## Praxis-Skripte

### Inaktive Teams finden

```powershell
# Teams finden die seit X Tagen keine Aktivität haben
# (Erfordert Microsoft Graph)
Connect-MgGraph -Scopes "Reports.Read.All", "Team.ReadBasic.All"

$inactiveDays = 90
$cutoffDate = (Get-Date).AddDays(-$inactiveDays)

Get-MgGroup -Filter "groupTypes/any(g:g eq 'Unified')" -All |
    Where-Object { $_.ResourceProvisioningOptions -contains "Team" } |
    ForEach-Object {
        $teamId = $_.Id
        try {
            $activity = Get-MgTeamActivity -TeamId $teamId -ErrorAction SilentlyContinue
            if ($activity.LastActivityDateTime -lt $cutoffDate -or $null -eq $activity.LastActivityDateTime) {
                [PSCustomObject]@{
                    Team             = $_.DisplayName
                    GroupId          = $_.Id
                    LetzteAktivität  = $activity.LastActivityDateTime
                    Inaktiv          = $true
                }
            }
        } catch {}
    } | Sort-Object LetzteAktivität | Export-Csv "Inaktive-Teams.csv" -NoTypeInformation
```

### Gastbenutzer-Report: Alle Gäste in allen Teams

```powershell
$report = @()
$allTeams = Get-Team

foreach ($team in $allTeams) {
    try {
        $members = Get-TeamUser -GroupId $team.GroupId -ErrorAction SilentlyContinue
        $guests = $members | Where-Object { $_.Role -eq "Guest" }
        foreach ($guest in $guests) {
            $report += [PSCustomObject]@{
                Team       = $team.DisplayName
                GroupId    = $team.GroupId
                GastName   = $guest.Name
                GastUPN    = $guest.User
                Rolle      = $guest.Role
            }
        }
    } catch {
        Write-Warning "Fehler bei Team: $($team.DisplayName) — $_"
    }
}

$report | Export-Csv "Gaeste-Report.csv" -NoTypeInformation -Encoding UTF8
Write-Host "Gesamt: $($report.Count) Gasteinträge in $($allTeams.Count) Teams"
```

### Gäste aus allen Teams entfernen

```powershell
# VORSICHT: Entfernt alle Gäste aus allen Teams!
# Erst Gastbenutzer-Report erstellen und prüfen!
$allTeams = Get-Team

foreach ($team in $allTeams) {
    $guests = Get-TeamUser -GroupId $team.GroupId | Where-Object { $_.Role -eq "Guest" }
    foreach ($guest in $guests) {
        try {
            Remove-TeamUser -GroupId $team.GroupId -User $guest.User
            Write-Host "Entfernt: $($guest.User) aus $($team.DisplayName)" -ForegroundColor Green
        } catch {
            Write-Warning "Fehler: $($guest.User) aus $($team.DisplayName) — $_"
        }
    }
}
```

### Policy Bulk-Zuweisung

```powershell
# Policy vielen Benutzern aus CSV zuweisen
# CSV-Format: UPN,PolicyName
# user1@contoso.com,Executives-Meeting
# user2@contoso.com,Standard-Meeting

$assignments = Import-Csv "Policy-Assignments.csv"
foreach ($a in $assignments) {
    try {
        Grant-CsTeamsMeetingPolicy -Identity $a.UPN -PolicyName $a.PolicyName
        Write-Host "Policy zugewiesen: $($a.UPN) → $($a.PolicyName)" -ForegroundColor Green
    } catch {
        Write-Warning "Fehler bei $($a.UPN): $_"
    }
}
```

### Teams-Report: Alle Teams mit Mitgliederzahl exportieren

```powershell
$result = Get-Team | ForEach-Object {
    $members = Get-TeamUser -GroupId $_.GroupId
    [PSCustomObject]@{
        Name        = $_.DisplayName
        GroupId     = $_.GroupId
        Sichtbar    = $_.Visibility
        Archiviert  = $_.Archived
        Besitzer    = ($members | Where-Object Role -eq Owner).Count
        Mitglieder  = ($members | Where-Object Role -eq Member).Count
        Gäste       = ($members | Where-Object Role -eq Guest).Count
        Gesamt      = $members.Count
        Nickname    = $_.MailNickName
    }
}

$result | Sort-Object Name | Export-Csv "Teams-Vollreport.csv" -NoTypeInformation -Encoding UTF8
Write-Host "Report erstellt: $($result.Count) Teams"
```

---

## Microsoft Graph für Teams

Die **Microsoft Graph API** ermöglicht erweiterte Operationen, die über PowerShell-Cmdlets hinausgehen:

```powershell
# Mit Microsoft Graph verbinden
Connect-MgGraph -Scopes "Team.ReadBasic.All", "TeamMember.ReadWrite.All", "Channel.ReadBasic.All"

# Alle Teams abrufen (Graph)
Get-MgTeam -All | Select-Object DisplayName, Id

# Team per Graph erstellen
$teamBody = @{
    "template@odata.bind" = "https://graph.microsoft.com/v1.0/teamsTemplates('standard')"
    displayName           = "Neues Graph-Team"
    description           = "Erstellt via Graph API"
    visibility            = "Private"
} | ConvertTo-Json

Invoke-MgGraphRequest -Method POST -Uri "https://graph.microsoft.com/v1.0/teams" `
    -Body $teamBody -ContentType "application/json"

# Teams-Aktivitätsbericht abrufen
Invoke-MgGraphRequest -Method GET `
    -Uri "https://graph.microsoft.com/v1.0/reports/getTeamsUserActivityCounts(period='D30')"
```

---

## Fehlerbehandlung

```powershell
# Robustes Skript-Template mit Fehlerbehandlung
try {
    Connect-MicrosoftTeams -ErrorAction Stop
    
    $teams = Get-Team -ErrorAction Stop
    Write-Host "Gefunden: $($teams.Count) Teams" -ForegroundColor Cyan
    
    foreach ($team in $teams) {
        try {
            # Operationen pro Team
            $members = Get-TeamUser -GroupId $team.GroupId -ErrorAction Stop
        } catch {
            Write-Warning "Fehler bei Team '$($team.DisplayName)': $($_.Exception.Message)"
            continue  # Nächstes Team
        }
    }
} catch {
    Write-Error "Kritischer Fehler: $($_.Exception.Message)"
} finally {
    Disconnect-MicrosoftTeams -ErrorAction SilentlyContinue
}
```

> [!TIP] Throttling vermeiden
> Bei großen Tenants mit vielen Teams kann es zu **API Throttling** kommen. Füge `Start-Sleep -Milliseconds 500` zwischen iterativen API-Aufrufen ein und verwende Fehlerbehandlung mit Retry-Logik.

> [!NOTE] Graph vs. CS-Cmdlets
> Einige Operationen sind nur über **Graph API** möglich (z.B. Teams-Aktivitätsberichte, Teams-Archivierung). Andere sind nur über **CS-Cmdlets** möglich (Telefonie, Policies). Oft braucht man beide.

---

## Verwandte Notizen

- [[Teams Grundlagen]]
- [[Teams und Kanäle verwalten]]
- [[Governance und Policies]]
- [[Teams-Telefonie]]
- [[Teams Apps und Bots]]
