---
tags:
  - exchange-online
  - postfächer
  - verwaltung
  - powershell
aliases:
  - Mailbox Management
  - Postfachverwaltung
---

# Postfächer verwalten

> [!ABSTRACT] TL;DR
> Exchange Online bietet verschiedene Postfachtypen: User Mailboxes (persönlich, lizenzpflichtig), Shared Mailboxes (gemeinsam genutzt, bis 50 GB ohne Lizenz), Resource Mailboxes (Räume/Geräte) und Archive Mailboxes. Die Verwaltung erfolgt über das Exchange Admin Center (EAC) oder Exchange Online PowerShell.

---

## Postfachtypen im Überblick

| Typ | Zweck | Lizenz | Max. Größe |
|-----|-------|--------|------------|
| User Mailbox | Persönliches Postfach | Ja | 50/100 GB |
| Shared Mailbox | Gemeinsam genutzt | Nein (bis 50 GB) | 50 GB (ohne Lizenz) |
| Room Mailbox | Besprechungsraum | Nein | 50 GB |
| Equipment Mailbox | Ressourcen (Geräte) | Nein | 50 GB |
| Archive Mailbox | Erweitertes Archiv | Plan 2 oder Add-on | Unbegrenzt |
| Inactive Mailbox | Aufbewahrung nach Löschung | Nein | Wie zuletzt konfiguriert |

---

## Via Exchange Admin Center (UI)

### User Mailbox erstellen

1. Öffne das **Exchange Admin Center** unter [admin.exchange.microsoft.com](https://admin.exchange.microsoft.com)
2. Navigiere zu **Empfänger → Postfächer → + Postfach hinzufügen**
3. Wähle **Benutzerpostfach**
4. Wähle **Neuen Benutzer erstellen** oder **Vorhandenen Benutzer auswählen**
5. Konfiguriere die Einstellungen:
   - **Anzeigename:** Vollständiger Name
   - **Benutzername:** E-Mail-Adresse (z.B. `max.mustermann@contoso.com`)
   - **Kennwort:** Sicheres Anfangskennwort setzen
   - **Postfachalias:** Kurzname
6. Klicke auf **Speichern**
7. Weise anschließend im **Microsoft 365 Admin Center → Benutzer → Aktive Benutzer** eine Exchange-Lizenz zu

### Shared Mailbox erstellen

1. EAC → **Empfänger → Postfächer → + Postfach hinzufügen**
2. Wähle **Freigegebenes Postfach**
3. Konfiguriere:
   - **Anzeigename:** (z.B. „Support Team")
   - **E-Mail-Adresse:** (z.B. `support@contoso.com`)
4. Klicke auf **Speichern**
5. Mitglieder hinzufügen: Postfach auswählen → **Delegierung verwalten** → **Full Access** → Benutzer hinzufügen

### Raum- oder Geräte-Postfach (Resource Mailbox) erstellen

1. EAC → **Empfänger → Ressourcen → + Ressource hinzufügen**
2. Wähle **Raum** oder **Gerät**
3. Konfiguriere:
   - **Name:** (z.B. „Besprechungsraum A101")
   - **E-Mail-Adresse:** (z.B. `room-a101@contoso.com`)
   - **Kapazität:** Anzahl der Personen (nur Räume)
4. Unter **Buchungsoptionen** konfigurieren:
   - **Buchungsanfragen automatisch annehmen:** aktivieren
   - **Konflikte zulassen:** deaktivieren (empfohlen)
   - **Buchungsfenster:** maximale Vorausbuchungsdauer
5. Klicke auf **Speichern**

### Archive Mailbox aktivieren

1. EAC → **Empfänger → Postfächer** → Postfach auswählen
2. Im Detailbereich rechts auf **Weitere Einstellungen** klicken
3. Unter **Archivpostfach** auf **Verwalten** klicken
4. Klicke auf **Aktivieren** → Bestätigen
5. Status wechselt auf **Aktiv** (Provisionierung dauert einige Minuten)

> [!TIP]
> Alternativ über: **Microsoft 365 Admin Center → Aktive Benutzer → Benutzer auswählen → E-Mail-Tab → Archiv verwalten**

### Postfach-Einstellungen bearbeiten

1. EAC → **Empfänger → Postfächer** → Postfach auswählen
2. Klicke auf den **Postfachnamen** oder auf **Bearbeiten** (Stift-Symbol)
3. Verfügbare Einstellungen:
   - **Allgemein:** Anzeigename, Alias, E-Mail-Adressen
   - **E-Mail-Adressen:** Zusätzliche SMTP-Adressen (Aliases)
   - **Postfachfeatures:** OWA, ActiveSync, IMAP, POP aktivieren/deaktivieren
   - **E-Mail-Weiterleitung:** Weiterleitung zu anderer Adresse konfigurieren
   - **Postfachkontingent:** Speicherlimits einstellen
4. Klicke auf **Speichern**

> [!NOTE]
> **Inaktive Postfächer** (nach Benutzerlöschung mit aktivem Litigation Hold) können nur über PowerShell wiederhergestellt werden — das EAC zeigt sie nicht an.

---

## User Mailbox erstellen und konfigurieren

### Über das Exchange Admin Center (EAC)

1. `https://admin.exchange.microsoft.com` öffnen
2. **Empfänger → Postfächer → Postfach hinzufügen**
3. Neuen Benutzer anlegen oder vorhandenen auswählen
4. Anzeigename, primäre E-Mail-Adresse, Alias konfigurieren
5. Speichern → Exchange-Lizenz im Microsoft 365 Admin Center zuweisen

### Über PowerShell

```powershell
# Neues Benutzerpostfach erstellen (Benutzer muss bereits in Entra ID existieren)
Enable-Mailbox -Identity "max.mustermann@contoso.com"

# Alternativ: Neuen Benutzer und Postfach gleichzeitig anlegen
New-Mailbox -Name "Max Mustermann" `
            -Alias "max.mustermann" `
            -MicrosoftOnlineServicesID "max.mustermann@contoso.com" `
            -Password (ConvertTo-SecureString "Passwort123!" -AsPlainText -Force) `
            -ResetPasswordOnNextLogon $true

# Postfach-Details abrufen
Get-Mailbox -Identity "max.mustermann@contoso.com" | Format-List

# Alle Postfächer auflisten
Get-Mailbox -ResultSize Unlimited | Select-Object DisplayName, PrimarySmtpAddress, RecipientTypeDetails
```

### Postfach-Einstellungen konfigurieren

```powershell
# Sprache und Zeitzone setzen
Set-MailboxRegionalConfiguration -Identity "max.mustermann@contoso.com" `
    -Language "de-DE" `
    -TimeZone "W. Europe Standard Time" `
    -DateFormat "dd.MM.yyyy" `
    -TimeFormat "HH:mm"

# OWA-Optionen konfigurieren
Set-CasMailbox -Identity "max.mustermann@contoso.com" `
    -OWAEnabled $true `
    -ActiveSyncEnabled $true `
    -ImapEnabled $false `
    -PopEnabled $false

# E-Mail-Weiterleitung einrichten
Set-Mailbox -Identity "max.mustermann@contoso.com" `
    -ForwardingSmtpAddress "extern@beispiel.de" `
    -DeliverToMailboxAndForward $true
```

> [!WARNING] Externe Weiterleitung
> Microsoft blockiert standardmäßig automatische externe Weiterleitungen per Anti-Spam-Outbound-Richtlinie. Diese muss explizit für bestimmte Domänen freigegeben werden.

---

## Shared Mailbox

Eine Shared Mailbox wird gemeinsam von mehreren Benutzern genutzt (z. B. `info@contoso.com`, `support@contoso.com`). Sie benötigt keine eigene Lizenz, solange sie unter 50 GB bleibt.

### Shared Mailbox erstellen

```powershell
# Neue Shared Mailbox erstellen
New-Mailbox -Shared `
            -Name "Support" `
            -DisplayName "Support Team" `
            -Alias "support" `
            -PrimarySmtpAddress "support@contoso.com"

# Benutzer Zugriff geben (Full Access)
Add-MailboxPermission -Identity "support@contoso.com" `
    -User "max.mustermann@contoso.com" `
    -AccessRights FullAccess `
    -InheritanceType All

# Send As-Berechtigung vergeben
Add-RecipientPermission -Identity "support@contoso.com" `
    -Trustee "max.mustermann@contoso.com" `
    -AccessRights SendAs
```

### Shared Mailbox in User Mailbox konvertieren

```powershell
# Shared → User
Set-Mailbox -Identity "support@contoso.com" -Type Regular

# User → Shared
Set-Mailbox -Identity "support@contoso.com" -Type Shared
```

> [!IMPORTANT] 50 GB Limit
> Überschreitet eine Shared Mailbox 50 GB, muss eine **Exchange Online Plan 2**-Lizenz zugewiesen werden, um die automatische Archivierung zu aktivieren und Datenverlust zu vermeiden.

---

## Resource Mailbox (Raum und Gerät)

Resource Mailboxes repräsentieren buchbare Ressourcen (Besprechungsräume, Projektoren, Fahrzeuge).

### Room Mailbox erstellen

```powershell
# Raum erstellen
New-Mailbox -Room `
            -Name "Besprechungsraum A101" `
            -Alias "room-a101" `
            -PrimarySmtpAddress "room-a101@contoso.com"

# Raumeinstellungen konfigurieren
Set-CalendarProcessing -Identity "room-a101@contoso.com" `
    -AutomateProcessing AutoAccept `
    -AllowConflicts $false `
    -BookingWindowInDays 180 `
    -MaximumDurationInMinutes 480 `
    -AddOrganizerToSubject $true `
    -DeleteComments $false `
    -DeleteSubject $false

# Kapazität und Standort setzen
Set-Mailbox -Identity "room-a101@contoso.com" `
    -ResourceCapacity 20 `
    -Office "Gebäude A, 1. OG"
```

### Equipment Mailbox erstellen

```powershell
# Gerät erstellen
New-Mailbox -Equipment `
            -Name "Beamer HD-01" `
            -Alias "beamer-hd01" `
            -PrimarySmtpAddress "beamer-hd01@contoso.com"

Set-CalendarProcessing -Identity "beamer-hd01@contoso.com" `
    -AutomateProcessing AutoAccept `
    -AllowConflicts $false
```

---

## Archive Mailbox aktivieren

Das Archivpostfach erweitert das primäre Postfach um einen separaten Archivbereich, der in Outlook als „Online-Archiv" erscheint.

```powershell
# Archivpostfach aktivieren
Enable-Mailbox -Identity "max.mustermann@contoso.com" -Archive

# Archivstatus prüfen
Get-Mailbox -Identity "max.mustermann@contoso.com" | Select-Object DisplayName, ArchiveStatus, ArchiveGuid

# Auto-Expanding Archive aktivieren (unbegrenzte Archivgröße, Plan 2)
Enable-Mailbox -Identity "max.mustermann@contoso.com" -AutoExpandingArchive

# Für alle Postfächer gleichzeitig aktivieren
Get-Mailbox -ResultSize Unlimited -Filter {ArchiveStatus -eq "None" -AND RecipientTypeDetails -eq "UserMailbox"} |
    Enable-Mailbox -Archive
```

> [!NOTE] Auto-Expanding Archive
> Das Auto-Expanding Archive für Exchange Online Plan 2 ermöglicht theoretisch unbegrenzte Archivgröße. Jede Erweiterung wird automatisch von Exchange Online provisioniert, wenn das aktuelle Archiv 90 % gefüllt ist.

---

## Inactive Mailbox

Wenn ein Benutzer gelöscht wird, bleibt das Postfach als **Inactive Mailbox** erhalten, sofern eine Aufbewahrungsrichtlinie oder ein Litigation Hold aktiv war.

```powershell
# Litigation Hold aktivieren (bevor Benutzer gelöscht wird)
Set-Mailbox -Identity "max.mustermann@contoso.com" `
    -LitigationHoldEnabled $true `
    -LitigationHoldDuration 2555  # 7 Jahre in Tagen

# Inactive Mailboxes anzeigen
Get-Mailbox -InactiveMailboxOnly | Select-Object DisplayName, PrimarySmtpAddress, WhenSoftDeleted

# Inactive Mailbox wiederherstellen
New-Mailbox -InactiveMailbox "max.mustermann@contoso.com" `
            -Name "Max Mustermann Wiederhergestellt" `
            -MicrosoftOnlineServicesID "max.mustermann.neu@contoso.com" `
            -Password (ConvertTo-SecureString "NeuesPasswort123!" -AsPlainText -Force)

# Inactive Mailbox dauerhaft löschen
Remove-Mailbox -InactiveMailbox "max.mustermann@contoso.com" -Confirm:$false
```

---

## Postfachquota-Verwaltung

### Quota-Werte verstehen

| Quota-Typ | Beschreibung |
|-----------|-------------|
| `IssueWarningQuota` | Warnung an Benutzer bei Überschreitung |
| `ProhibitSendQuota` | Senden wird gesperrt |
| `ProhibitSendReceiveQuota` | Senden und Empfangen gesperrt |
| `RecoverableItemsQuota` | Limit für gelöschte Elemente |

```powershell
# Aktuelle Quota anzeigen
Get-Mailbox -Identity "max.mustermann@contoso.com" | 
    Select-Object DisplayName, IssueWarningQuota, ProhibitSendQuota, ProhibitSendReceiveQuota

# Postfachgröße aktuell prüfen
Get-MailboxStatistics -Identity "max.mustermann@contoso.com" | 
    Select-Object DisplayName, TotalItemSize, ItemCount, LastLogonTime

# Benutzerdefinierte Quota setzen
Set-Mailbox -Identity "max.mustermann@contoso.com" `
    -UseDatabaseQuotaDefaults $false `
    -IssueWarningQuota 45GB `
    -ProhibitSendQuota 49GB `
    -ProhibitSendReceiveQuota 50GB

# Alle Postfächer über 40 GB anzeigen
Get-MailboxStatistics -ResultSize Unlimited | 
    Where-Object {$_.TotalItemSize -gt "40 GB"} | 
    Sort-Object TotalItemSize -Descending | 
    Select-Object DisplayName, TotalItemSize
```

---

## Troubleshooting

> [!WARNING] Häufige Fehler

**Problem:** Benutzer kann keine E-Mails senden (ProhibitSendQuota erreicht)
```powershell
# Quota-Status prüfen
Get-MailboxStatistics -Identity "user@contoso.com" | 
    Select-Object TotalItemSize, ProhibitSendQuota
# Lösung: Archivpostfach aktivieren oder alte E-Mails löschen/verschieben
```

**Problem:** Shared Mailbox erscheint nicht automatisch in Outlook
```powershell
# Automapping prüfen und neu setzen
Add-MailboxPermission -Identity "shared@contoso.com" `
    -User "user@contoso.com" `
    -AccessRights FullAccess `
    -AutoMapping $true
```

**Problem:** Archivpostfach wird nicht angezeigt
```powershell
# Archivstatus prüfen
Get-Mailbox -Identity "user@contoso.com" | Select-Object ArchiveStatus, ArchiveDatabase
# Outlook-Cache leeren und Profil neu erstellen
```

**Problem:** Kalendereinladungen für Ressourcen werden nicht automatisch angenommen
```powershell
# CalendarProcessing prüfen
Get-CalendarProcessing -Identity "room@contoso.com" | Select-Object AutomateProcessing
# Korrektur:
Set-CalendarProcessing -Identity "room@contoso.com" -AutomateProcessing AutoAccept
```

---

## Verwandte Notizen

- [[Exchange Online Grundlagen]]
- [[Postfach-Berechtigungen]]
- [[Verteilergruppen und M365-Gruppen]]
- [[Exchange Online PowerShell]]
- [[E-Mail-Migration]]
