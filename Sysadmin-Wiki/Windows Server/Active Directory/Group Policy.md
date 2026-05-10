---
tags: [active-directory, gpo, group-policy, gruppenrichtlinien, softwareverteilung, sicherheit]
aliases: ["Group Policy", "GPO", "Gruppenrichtlinien", "Group Policy Objects"]
---

# Group Policy

> [!ABSTRACT]
> **Group Policy Objects (GPOs)** sind das zentrale Werkzeug für die Konfigurationsverwaltung in AD-Umgebungen. Sie steuern Sicherheitseinstellungen, Softwareinstallationen, Desktop-Konfigurationen und Skripte für Benutzer und Computer in bestimmten OUs. Auswertungsreihenfolge: Local → Site → Domain → OU (LSDOU).

---

## Konzept: Was ist Group Policy?

Group Policy ist der **zentrale Konfigurationsmanagement-Mechanismus** für Windows-Domänenumgebungen. Über GPOs (Group Policy Objects) legen Administratoren Einstellungen für Tausende von Computern und Benutzern zentral fest — von Sicherheitsrichtlinien über Softwareinstallationen bis hin zu Desktop-Konfigurationen.

> [!NOTE] GPO vs. manuelle Konfiguration
> Ohne Group Policy müsste jeder Server und jeder Client einzeln konfiguriert werden. GPOs ermöglichen es, eine Einstellung **einmal** zu definieren und sie automatisch auf alle betroffenen Objekte anzuwenden — und bei Änderungen zieht sich die neue Einstellung innerhalb von 90 Minuten auf alle Geräte.

### Wo werden GPOs gespeichert?

GPOs bestehen aus **zwei Teilen** — beide müssen synchron sein:

| Komponente | Speicherort | Inhalt |
|------------|------------|--------|
| **GPC** (Group Policy Container) | AD-Datenbank (NTDS.DIT) | Metadaten, Versionsnummer, Links |
| **GPT** (Group Policy Template) | SYSVOL (`\\firma.local\SYSVOL\...`) | Einstellungen, Scripts, ADMX-Dateien |

> [!WARNING] GPO-Versionsnummer
> Wenn GPC und GPT nicht synchron sind (Replikationsproblem), entstehen "broken GPOs" — Clients erhalten veraltete oder keine Einstellungen. `gpresult /h` zeigt Versionskonflikte.

---

## GPO-Grundkonzepte

### Auswertungsreihenfolge (LSDOU) — Das wichtigste GPO-Konzept

LSDOU ist die Grundregel: Jede spätere Ebene **überschreibt** die frühere. Die zuletzt angewendete Einstellung gewinnt.

```mermaid
graph LR
    L[Local] --> S[Site]
    S --> D[Domain]
    D --> OU1[OU Parent]
    OU1 --> OU2[OU Child gewinnt]
```

```
Auswertungsreihenfolge auf Gerät WRK-MUC01 (in OU: München, Domain: firma.local):

1. LOCAL    Lokale Gruppenrichtlinie des Computers (gpedit.msc)
            → Hintergrund = kein (keine Einstellung)

2. SITE     GPO "Netzwerk-Basis" verknüpft mit AD-Site "München"
            → Hintergrund = blau

3. DOMAIN   GPO "Corporate Standards" verknüpft mit Domain
            → Hintergrund = grau  (überschreibt blau)

4. OU       GPO "Deutschland-IT" verknüpft mit OU Deutschland
            → kein Hintergrund gesetzt → grau bleibt

4a. OU      GPO "München-Workstations" verknüpft mit OU München
            → Hintergrund = grün  (überschreibt grau)

ERGEBNIS: grün
```

**Ausnahmen und Sonderfälle:**

| Mechanismus | Effekt | Wann einsetzen |
|-------------|--------|----------------|
| **Erzwungen** (Enforced) | GPO kann von keiner untergeordneten OU überschrieben werden | Unternehmensweite Sicherheitsrichtlinien |
| **Vererbung deaktivieren** | OU blockiert alle übergeordneten GPOs | Spezielle Abteilungen (z.B. IT-Testumgebung) |
| **Sicherheitsfilterung** | GPO gilt nur für bestimmte Gruppen | Selective Deployment (z.B. nur für Admins) |
| **WMI-Filter** | GPO nur auf Geräten mit bestimmten Eigenschaften | Hardware/OS-abhängige Einstellungen |
| **Loopback Processing** | Benutzerkonfiguration wird durch Computer-GPO erzwungen | Kiosk-Systeme, RDS-Server |

> [!TIP] Loopback Processing — Wann braucht man das?
> **Szenario:** RDS-Server soll alle Benutzer in einer eingeschränkten Umgebung halten (kein Taskmanager, kein Kontextmenü). Ohne Loopback gilt die Benutzerkonfiguration aus der Benutzer-OU, nicht aus der Server-OU. Mit Loopback "überschreibt" der Computer-GPO die Benutzereinstellungen.

### Computerkonfiguration vs. Benutzerkonfiguration

```
GPO
├── Computerkonfiguration
│   ├── Software-Einstellungen (MSI-Deployment)
│   ├── Windows-Einstellungen (Startskripte, Sicherheitsrichtlinien)
│   └── Administrative Vorlagen (Registry-basiert)
│       → Angewendet beim Computerstart (danach alle 90 Min.)
│
└── Benutzerkonfiguration
    ├── Software-Einstellungen (MSI-Deployment für User)
    ├── Windows-Einstellungen (Anmeldeskripte, Ordnerumleitung)
    └── Administrative Vorlagen
        → Angewendet bei Benutzeranmeldung (danach alle 90 Min.)
```

> [!IMPORTANT] Computer- vs. Benutzer-GPO
> **Computerkonfiguration** gilt für das Gerät — unabhängig davon, wer angemeldet ist.
> **Benutzerkonfiguration** gilt für den Benutzer — unabhängig davon, an welchem Gerät er sitzt.
> Das Gerät muss in einer OU sein, auf die der Computer-GPO verknüpft ist. Der Benutzer muss in einer OU sein, auf die der Benutzer-GPO verknüpft ist!

---

## Group Policy Management Console (GPMC)

```
Öffnen:
- Server Manager → Tools → Gruppenrichtlinienverwaltung
- gpmc.msc
```

### Neue GPO erstellen und verknüpfen

```
GPMC Übersicht:
Forest → Domains → firma.local
├── Gruppenrichtlinienobjekte/  ← Alle GPOs werden hier gespeichert
├── WMI-Filter/
└── OUs (Sites, Abteilungen...)  ← Hier werden GPOs verknüpft

GPO erstellen:
1. "Gruppenrichtlinienobjekte" → Rechtsklick → "Neu"
   Name: "Sicherheits-Baseline-Computer"
   Starter-GPO: (kein)
   → OK

2. GPO bearbeiten (Rechtsklick → Bearbeiten):
   → Gruppenrichtlinienverwaltungs-Editor öffnet sich
   → Einstellungen konfigurieren

3. GPO mit OU verknüpfen:
   Ziel-OU (z.B. "Server") → Rechtsklick → "Vorhandenes GPO verknüpfen"
   → GPO auswählen → OK
   → Verknüpfung erscheint unter der OU

GPO-Status und Erzwingen:
→ Verknüpfte GPO → Rechtsklick:
  "Verknüpfung aktiviert"    → An/Aus schalten
  "Erzwungen"                → Kann nicht durch untergeordnete OU überschrieben werden
  "Vererbung deaktivieren"   → Auf der OU selbst: blockiert übergeordnete GPOs
```

---

## GPO-Bearbeitung: Typische Workflows

### Sicherheitseinstellungen

```
Computer → Windows-Einstellungen → Sicherheitseinstellungen

Kontosperrungsrichtlinie:
├── Schwellenwert für Kontosperrung: 5 Fehlversuche
├── Kontosperrdauer: 30 Minuten
└── Beobachtungsfenster: 30 Minuten

Kennwortrichtlinie:
├── Minimale Kennwortlänge: 12 Zeichen
├── Kennwortchronik: 12 Kennwörter
├── Maximales Kennwortalter: 90 Tage
└── Komplexitätsanforderungen: Aktiviert

Lokale Richtlinien → Benutzerrechte:
├── "Lokal anmelden" → nur Administratoren
├── "Als Dienst anmelden" → Service Accounts
└── "Netzwerk: Zugriff auf diesen Computer..." → Domänenbenutzer
```

### Administrative Vorlagen (Registry)

```
Populäre Einstellungen:

Computer → Administrative Vorlagen:
├── System → Anmeldung: "Immer warten, bis Netzwerk steht"
├── Windows-Komponenten → Windows Update: WSUS-Server, Update-Zeitplan
├── Netzwerk → Netzwerkverbindungen: Firewall-Einstellungen
└── Windows-Komponenten → BitLocker: Verschlüsselung erzwingen

Benutzer → Administrative Vorlagen:
├── Start-Menü und Taskleiste: Optionen ausblenden
├── Systemsteuerung: Zugriff einschränken
└── Windows-Komponenten → Internet Explorer: Proxy-Einstellungen
```

### Ordnerumleitung

```
Benutzer → Windows-Einstellungen → Ordnerumleitung → Dokumente
├── Einstellung: Einfach
├── Stammpfad: \\SRV-FS01\Home\%username%\Dokumente
└── Richtlinienentfernung: Ordner an lokalen Speicherort zurückleiten

Weitere: Desktop, AppData\Roaming, Bilder, Downloads
```

---

## GPO per PowerShell

```powershell
# GPMC-Modul laden
Import-Module GroupPolicy

# --- LESEN ---

# Alle GPOs anzeigen
Get-GPO -All | Select-Object DisplayName, GpoStatus, CreationTime

# GPO-Details
Get-GPO -Name "Sicherheits-Baseline"

# GPO-Verknüpfungen einer OU
(Get-GPInheritance -Target "OU=Server,DC=firma,DC=local").GpoLinks

# --- ERSTELLEN ---

$gpo = New-GPO -Name "IE-Deaktivierung" -Comment "IE11 für alle deaktivieren"

# GPO verknüpfen
New-GPLink `
    -Name "IE-Deaktivierung" `
    -Target "OU=Benutzer,DC=firma,DC=local" `
    -Enforced No `
    -LinkEnabled Yes

# --- EINSTELLUNGEN KONFIGURIEREN ---

# Registry-Wert in GPO setzen
Set-GPRegistryValue `
    -Name "IE-Deaktivierung" `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Internet Explorer" `
    -ValueName "DisableIE" `
    -Type DWord `
    -Value 1

# --- SICHERN UND WIEDERHERSTELLEN ---

# GPO sichern
Backup-GPO -Name "Sicherheits-Baseline" -Path "C:\GPO-Backup"
Backup-GPO -All -Path "C:\GPO-Backup-Komplett"

# GPO wiederherstellen
Restore-GPO -Name "Sicherheits-Baseline" -Path "C:\GPO-Backup"

# GPO in andere Domain importieren
Import-GPO -BackupId <GUID> -Path "C:\GPO-Backup" -TargetName "Sicherheits-Baseline"
```

---

## GPO debuggen

### gpresult — Auswertungsbericht

```powershell
# Für aktuellen Benutzer und Computer
gpresult /r

# Detaillierten HTML-Report erstellen
gpresult /h C:\Temp\gpo-report.html /f
# Dann: start C:\Temp\gpo-report.html

# Für anderen Benutzer/Computer
gpresult /user FIRMA\jdoe /r
gpresult /s SRV-FS01 /r
```

### gpupdate — Sofort aktualisieren

```powershell
# GPOs sofort anwenden (ohne Warten auf 90-Min.-Zyklus)
gpupdate /force

# Nur Computerkonfiguration
gpupdate /target:computer /force

# Nur Benutzerkonfiguration
gpupdate /target:user /force

# Auf Remote-Computer
Invoke-GPUpdate -Computer "SRV-FS01" -Force
```

---

## GPO-Filter

### Sicherheitsfilterung

```
Standard: "Authenticated Users" → alle angemeldeten User/Computer
Einschränken: Authenticated Users entfernen → Gruppe hinzufügen

Beispiel: GPO nur für "IT-Admins"-Gruppe:
1. GPO auswählen → Sicherheitsfilterung
2. "Authentifizierte Benutzer" entfernen
3. "IT-Admins" hinzufügen
```

### WMI-Filter

```powershell
# GPO nur auf Windows 11 anwenden (WMI-Filter)
# WMI-Abfrage: SELECT * FROM Win32_OperatingSystem WHERE Caption LIKE "%Windows 11%"

# In GPMC: WMI-Filter → Neu
# Name: "Nur Windows 11"
# Abfrage: SELECT * FROM Win32_OperatingSystem WHERE Caption LIKE "%Windows 11%"
# GPO → Registerkarte "Details" → WMI-Filter → auswählen
```

---

## Starter-GPOs

```powershell
# Starter-GPO erstellen (Template für neue GPOs)
New-GPStarterGPO -Name "Basis-Sicherheit"

# GPO aus Starter-GPO erstellen
New-GPO -Name "Server-Sicherheit" -StarterGPOName "Basis-Sicherheit"
```

---

## Verwandte Notizen

- [[AD Grundlagen]] — OU-Struktur für GPO-Delegation
- [[Benutzer und Gruppen]] — Sicherheitsgruppen für GPO-Filterung
- [[AD Troubleshooting]] — GPO wird nicht angewendet
- [[Windows Server Installation]] — Initiale GPO-Konfiguration
