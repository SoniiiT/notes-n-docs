---
tags:
  - dynamics365
  - hr
  - personalwesen
  - human-resources
  - mitarbeiter
  - urlaub
  - vergütung
  - payroll
aliases:
  - D365 HR
  - Dynamics 365 Human Resources
  - D365 Personalwesen
---

# Dynamics 365 HR

> [!ABSTRACT] TL;DR
> Dynamics 365 Human Resources ist die Cloud-HR-Lösung von Microsoft. Es deckt Mitarbeiterverwaltung, Urlaubs- und Abwesenheitsmanagement, Benefits, Vergütungsmanagement, Lernen und Leistungsbeurteilung ab. Über Employee Self-Service und den Teams HR-Bot können Mitarbeiter viele Anfragen selbst erledigen.

---

## Mitarbeiterverwaltung

### Organisationsstruktur

```mermaid
graph TD
    ORG[Organisation] --> DEP1[Abteilung\nVertrieb]
    ORG --> DEP2[Abteilung\nEntwicklung]
    ORG --> DEP3[Abteilung\nHR]
    DEP1 --> POS1[Position:\nVertriebsleiter]
    DEP1 --> POS2[Position:\nAccount Manager]
    POS1 --> EMP1[Mitarbeiter:\nMax Mustermann]
    POS2 --> EMP2[Mitarbeiter:\nAnna Schmidt]
    POS2 --> EMP3[Mitarbeiter:\nPeter Bauer]
```

**Positionsverwaltung:**
- Jede Position gehört zu einer Abteilung und einem Job
- Positionen haben Stellenbeschreibungen und Anforderungsprofile
- Vakante Positionen können im Recruitment-Modul ausgeschrieben werden

### Personalaktionen (Personnel Actions)

Personalaktionen dokumentieren alle mitarbeiterrelevanten Vorgänge mit Genehmigungsworkflow:

| Aktionstyp | Beschreibung | Workflow |
|-----------|-------------|---------|
| **Einstellung (Hire)** | Neuer Mitarbeiter wird eingestellt | HR → Manager → Genehmigung |
| **Versetzung (Transfer)** | Abteilungs-/Standortwechsel | Manager → HR → Genehmigung |
| **Beförderung (Promotion)** | Neue Position / höhere Vergütung | Manager → HR → Geschäftsführung |
| **Kündigung (Terminate)** | Austritt des Mitarbeiters | Manager → HR → Offboarding-Flow |
| **Wiedereintritt (Rehire)** | Ehemaliger Mitarbeiter kehrt zurück | HR-Genehmigung |

### Mitarbeiter-Lebenszyklus

```mermaid
graph LR
    REC[Recruiting\nStellenausschreibung] -->|Eingestellt| ONB[Onboarding\nAufgaben und Docs]
    ONB -->|Aktiv| EMP4[Beschäftigt\nAktiver Mitarbeiter]
    EMP4 -->|Beförderung| EMP4
    EMP4 -->|Versetzung| EMP4
    EMP4 -->|Kündigung| OFF[Offboarding\nAbmeldeprozesse]
    OFF --> EX[Ehemalig\nAlumni]
```

---

## Urlaubs- und Abwesenheitsmanagement

### Abwesenheitstypen

| Typ | Beschreibung | Bezahlt |
|-----|-------------|---------|
| **Jahresurlaub** | Regulärer Erholungsurlaub | Ja |
| **Krankheit** | Krankmeldungen | Ja (nach Lohnfortzahlungsgesetz) |
| **Elternzeit** | Mutterschutz, Elternzeit | Teilweise |
| **Sonderurlaub** | Hochzeit, Beerdigung | Ja |
| **Unbezahlter Urlaub** | Auf Antrag | Nein |
| **Zeitausgleich** | Abbau von Überstunden | Ja |
| **Fortbildung** | Externe Schulungen | Ja |

### Richtlinien und Akkumulation

**Urlaubsanspruch-Konfiguration:**
```json
{
  "Urlaubsplan": "Standard Deutschland",
  "Anspruch pro Jahr": 30,
  "Akkumulations-Typ": "Monatlich",
  "Akkumulation pro Monat": 2.5,
  "Maximaler Übertrag": 10,
  "Verfalldatum": "31.03. des Folgejahres",
  "Wartezeit": "0 Monate (sofort)"
}
```

### Genehmigungsworkflow

```mermaid
graph TD
    EMP5[Mitarbeiter\nbeantragt Urlaub] -->|Antrag| MAN[Manager\nerhält Benachrichtigung]
    MAN -->|Genehmigt| CONF[Bestätigung\nan Mitarbeiter]
    MAN -->|Abgelehnt| NOTIF[Ablehnung\nmit Begründung]
    CONF -->|Synchronisiert| CAL[Teams-Kalender\nOut-of-Office]
    CONF -->|Aktualisiert| BAL[Urlaubssaldo\nreduziert]
```

**Teams HR Bot für Urlaubsanträge:**
```
Mitarbeiter: "@HR Bot Ich möchte vom 23.12. bis 27.12. Urlaub nehmen"
HR Bot: "Urlaubsantrag für 23.12.-27.12.2024 (3 Arbeitstage) 
         erstellt. Ihr aktueller Saldo: 12 Tage.
         Ihr Manager wurde zur Genehmigung benachrichtigt."
```

### Abwesenheitskalender und Teamübersicht

- Manager sehen Abwesenheiten des Teams in einer Kalenderansicht
- Konflikterkennung: Zu viele Mitarbeiter gleichzeitig abwesend?
- Integration in Microsoft Teams Kalender

---

## Benefits-Verwaltung

### Benefits-Typen

| Benefit | Beschreibung |
|---------|-------------|
| **Krankenversicherung** | Gesundheitspläne (Einzel-/Familientarif) |
| **Altersvorsorge** | Betriebliche Altersversorgung (bAV) |
| **Lebensversicherung** | Gruppen-Lebensversicherung |
| **Firmenwagen** | Fahrzeug-Benefit |
| **Jobticket** | ÖPNV-Zuschuss |
| **Kinderbetreuung** | Zuschuss zur Kinderbetreuung |
| **Flexible Spending** | Essensgutscheine, Sachbezüge |

### Open Enrollment (Jahresauswahl)

Mitarbeiter können Benefits jährlich neu auswählen:

```mermaid
graph TD
    OPEN[Enrollment-Periode\nöffnet (z.B. Oktober)] --> NOTI[Benachrichtigung\nan alle Mitarbeiter]
    NOTI --> SEL[Mitarbeiter wählt\nBenefits im Self-Service]
    SEL -->|Bestätigt| CONF2[Bestätigung\nund Wirksamkeitsdatum]
    CONF2 -->|01.01.| EFFECT[Benefits\naktiv ab 01.01.]
```

---

## Vergütungsmanagement (Compensation Management)

### Vergütungsstrukturen

| Typ | Beschreibung | Beispiel |
|-----|-------------|---------|
| **Fixed Plan** | Festes Grundgehalt in Gehaltsband | Tarifgehalt nach Gruppe |
| **Variable Plan** | Bonus und Provision | 10% Jahresbonus |
| **Band** | Gehaltsband je Stufe/Level | Junior, Senior, Lead |

### Gehaltsband-Verwaltung

```
Level Senior Software Engineer:
├── Minimum: €75.000
├── Midpoint: €90.000
└── Maximum: €105.000

Compa-Ratio = (Gehalt / Midpoint) × 100
├── < 85%: Unter Midpoint → Erhöhungsbedarf
├── 85-115%: Im Band → Normal
└── > 115%: Über Band → Risiko (Budget/Compliance)
```

### Gehaltsrunden (Merit Process)

```mermaid
graph LR
    MAN2[Manager\nerhält Budget-Zuteilung] -->|Vorschläge| PROP[Gehaltserhöhungen\nvorschlagen]
    PROP -->|Review| HR2[HR-Prüfung\nBudget-Check]
    HR2 -->|Genehmigung| CFO[Geschäftsführung\nGenehmigung]
    CFO -->|Approved| EFFECT2[Wirksamkeit\nnächste Lohnperiode]
```

---

## Lernen (Learning)

### Learning-Features

| Feature | Beschreibung |
|---------|-------------|
| **Kurse** | Interne und externe Schulungen |
| **Lernpfade** | Strukturierte Entwicklungsprogramme |
| **Zertifizierungen** | Pflicht-Zertifizierungen tracken |
| **Viva Learning** | Integration mit LinkedIn Learning, Microsoft Learn |
| **Compliance-Training** | Pflichtschulungen (DSGVO, IT-Sicherheit) |

### Viva Learning Integration

D365 HR ist mit **Microsoft Viva Learning** integriert:
- LinkedIn Learning-Kurse direkt in Teams
- Microsoft Learn-Inhalte
- Externe Lernplattformen (Coursera, Skillsoft)
- Eigene interne Kurse

---

## Leistungsbeurteilung (Performance Review)

### Beurteilungszyklus

```mermaid
graph LR
    GOAL[Ziele setzen\n(Jahresanfang)] -->|Midyear| MID[Halbjahrsgespräch\nFortschritt prüfen]
    MID -->|Jahresende| SELF[Selbstbeurteilung\nMitarbeiter]
    SELF -->|Manager| MREV[Manager-Beurteilung]
    MREV -->|360°| PEER[Peer-Feedback\noptional]
    PEER -->|Kalibrierung| CAL2[Kalibrier-Meeting\nHR + Manager]
    CAL2 -->|Final| RTNG[Rating\nund Feedback-Gespräch]
```

### Performance-Ratings

| Rating | Bedeutung | Anteil (typisch) |
|--------|----------|-----------------|
| **5 - Exceptional** | Übertrifft alle Erwartungen | 10% |
| **4 - Exceeds** | Übertrifft Erwartungen | 20% |
| **3 - Meets** | Erfüllt alle Erwartungen | 50% |
| **2 - Below** | Teilweise nicht erfüllt | 15% |
| **1 - Unsatisfactory** | Erwartungen nicht erfüllt | 5% |

---

## Payroll-Integration

D365 HR enthält keine native Lohnbuchhaltung — Integration mit externen Lohnanbietern:

### Unterstützte Payroll-Systeme

| Anbieter | Integration | Region |
|---------|-------------|--------|
| **Ceridian Dayforce** | Native API-Integration | USA, Kanada |
| **ADP Workforce Now** | Native API-Integration | USA, global |
| **DATEV** | Export/Import | Deutschland |
| **SAP HCM** | Integration via API | Global |
| **Lohnsteuer-Kompakt** | Partner-Lösung | Deutschland |

### Payroll-Datenexport

```powershell
# Beispiel: Payroll-Daten aus D365 HR exportieren
# Enthaltene Informationen:
# - Mitarbeiter-ID und Stammdaten
# - Gehaltsänderungen im Zeitraum
# - Neue/austretende Mitarbeiter
# - Abwesenheitsdaten (Urlaub, Krankheit)
# - Benefits-Abzüge
# - Überstunden (aus Zeiterfassung)

# Export als CSV oder via API-Integration
```

---

## Employee Self-Service Portal

### Self-Service-Funktionen

| Funktion | Beschreibung |
|---------|-------------|
| **Persönliche Daten** | Adresse, Bankverbindung, Notfallkontakte aktualisieren |
| **Urlaubsanträge** | Anträge stellen, Saldo einsehen |
| **Gehaltsabrechnungen** | Download aktueller und historischer Abrechnungen |
| **Benefits** | Aktuelle Benefits einsehen, open Enrollment |
| **Schulungsanträge** | Weiterbildungen beantragen |
| **Dokumente** | Arbeitsvertrag, Zeugnisse einsehen |

### Teams HR Bot

```
Teams-Konversation mit HR Bot:
├── "Wie viele Urlaubstage habe ich noch?" 
│   → Bot: "12 Tage verbleibend in 2024"
├── "Wer ist heute abwesend?"
│   → Bot: "3 Kollegen aus Ihrem Team sind heute abwesend"
├── "Ich möchte morgen von zuhause arbeiten"
│   → Bot: Homeoffice-Antrag erstellt
└── "Zeige mir meine letzten 3 Gehaltsabrechnungen"
    → Bot: PDF-Links zu Abrechnungen
```

---

## Preise und Lizenzierung

| Lizenz | Preis/User/Monat | Enthält |
|--------|-----------------|---------|
| **Human Resources** | $120 | Vollständiges HR-Modul |
| **HR Attach** | $40 | Mit anderer D365-Lizenz |
| **Team Members** | $8 | Lesezugriff und ESS-Portal |

---

## Verwandte Notizen

- [[Dynamics 365 Überblick]]
- [[Dynamics 365 Finance]]
- [[Dynamics 365 Administration]]
