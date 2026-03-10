---
tags:
  - dynamics365
  - customer-service
  - crm
  - omnichannel
  - support
  - copilot
  - case-management
aliases:
  - D365 Customer Service
  - Dynamics 365 Support
  - Omnichannel Customer Service
---

# Dynamics 365 Customer Service

> [!ABSTRACT] TL;DR
> Dynamics 365 Customer Service bietet umfassendes Case Management, Omnichannel-Support (Chat, Voice, SMS, E-Mail, WhatsApp) und KI-gestützte Agenten-Unterstützung durch Copilot. SLAs, Entitlements und Knowledge Management bilden das Fundament für erstklassigen Kundendienst.

---

## Case Management

### Case-Lebenszyklus

```mermaid
graph LR
    CR[Case erstellt] -->|Routing| QU[Queue eingereiht]
    QU -->|Zugewiesen| AG[Agent übernimmt]
    AG -->|Bearbeitung| IP[In Progress]
    IP -->|Eskalation| ES[Eskaliert an Senior/Manager]
    IP -->|Gelöst| RE[Resolved]
    RE -->|Bestätigt| CL[Closed]
    CL -->|Wiedereröffnet| AG
```

### Case-Felder und Konfiguration

| Feld | Beschreibung |
|------|-------------|
| **Case Title** | Kurze Beschreibung des Problems |
| **Priority** | Normal, Hoch, Kritisch |
| **Case Type** | Frage, Problem, Anfrage |
| **Origin** | E-Mail, Telefon, Chat, Web, Social |
| **Customer** | Verknüpfter Account/Contact |
| **Entitlement** | Verfügbare Supportleistungen |
| **SLA** | Zugewiesene Service Level Agreement |

---

## Queues und Routing

### Queue-Typen

| Queue-Typ | Beschreibung | Nutzung |
|-----------|-------------|---------|
| **Basic Queue** | Manuelle Zuweisung | Kleine Teams |
| **Priority Queue** | Reihenfolge nach Priorität | Standard-Support |
| **Round Robin** | Gleichmäßige Verteilung | Großteams |
| **Skill-basiert** | Nach Agenten-Kompetenz | Spezialisierter Support |

### Unified Routing (Intelligentes Routing)

Unified Routing nutzt KI für die automatische Weiterleitung:

```mermaid
graph TD
    EIN[Eingehender Kontakt] --> CL[Klassifizierung KI-Modell]
    CL --> PRIO[Priorität ermitteln]
    CL --> SKILL[Benötigte Skills ermitteln]
    PRIO --> MAT[Matching Agent + Queue]
    SKILL --> MAT
    MAT --> AGE[Zuweisung an Agent]
```

**Routing-Regeln basieren auf:**
- Sprache des Kunden
- Produkt/Thema der Anfrage
- Kundensegment (VIP, Standard)
- Agenten-Skills und Verfügbarkeit
- Aktuelle Warteschlangenlänge

---

## SLAs (Service Level Agreements)

SLAs definieren, innerhalb welcher Zeit ein Agent auf einen Case reagieren und ihn lösen muss.

### SLA-Typen

| SLA-Typ | Beschreibung |
|---------|-------------|
| **Standard SLA** | Grundlegende Zeit-Tracking |
| **Enhanced SLA** | Mit Pause/Fortsetzen-Funktion, KPIs, Eskalation |

### SLA-Konfiguration

```json
{
  "SLA-Name": "Premium Support SLA",
  "Antwortzeit": "2 Stunden",
  "Lösungszeit": "8 Stunden",
  "Eskalation-bei": "80% der Zeit verbraucht",
  "Gilt-für": "Priority = Hoch oder Kritisch",
  "Erfolg": "Case-Status = Gelöst",
  "Warnung-bei": "1,5 Stunden (Antwort)"
}
```

> [!WARNING] SLA-Uhren und Geschäftszeiten
> SLAs können an **Customer Service Schedule** (Geschäftszeiten) gebunden werden. Außerhalb der Geschäftszeiten wird die SLA-Uhr angehalten. Für 24/7-Support müssen entsprechende Zeitpläne konfiguriert werden.

---

## Entitlements

Entitlements definieren, welche Supportleistungen ein Kunde in Anspruch nehmen darf:

| Entitlement-Typ | Beispiel |
|-----------------|---------|
| **Anzahl Cases** | 10 Support-Cases pro Jahr |
| **Stunden** | 40 Supportstunden pro Quartal |
| **Unbegrenzt** | Unlimitierter Support für Premium-Kunden |

**Entitlement-Kanäle:** Können auf bestimmte Kanäle beschränkt werden (z.B. nur E-Mail-Support für Standard, Telefon nur für Premium).

---

## Omnichannel for Customer Service

Omnichannel ermöglicht einheitliche Kundenkommunikation über alle Kanäle in einer einzigen Agenten-Oberfläche.

### Unterstützte Kanäle

| Kanal | Beschreibung |
|-------|-------------|
| **Chat** | Live-Chat auf Website/Portal |
| **Voice** | Telefonintegration (Azure Communication Services) |
| **SMS** | Textnachrichten via Azure Communication Services |
| **E-Mail** | E-Mail-Handling direkt in D365 |
| **WhatsApp** | WhatsApp Business API Integration |
| **Facebook Messenger** | Social-Media-Support |
| **LINE** | Asiatischer Markt |
| **Twitter/X DMs** | Direkte Nachrichten |
| **WeChat** | Chinesischer Markt |
| **Custom Channel** | Eigene Kanäle via Direct Line API |

### Agenten-Oberfläche (Customer Service Workspace)

```mermaid
graph TD
    CSW[Customer Service Workspace] --> OMNI[Omnichannel-Seitenleiste Aktive Konversationen]
    CSW --> CASE[Case-Formular Details und Aktionen]
    CSW --> KB[Knowledge Base Suchleiste]
    CSW --> COP[Copilot-Seitenleiste KI-Vorschläge]
    OMNI --> CHAT[Chat-Konversation]
    OMNI --> VOICE[Voice-Anruf]
    OMNI --> EMAIL[E-Mail-Thread]
```

**Agenten-Kapazität:**
- Jeder Agent hat eine **Kapazitätseinheit**
- Chat = 1 Einheit, Voice = 5 Einheiten
- Agenten können mehrere Chats gleichzeitig führen (konfigurierbar)

### Voice-Kanal

> [!NOTE] Azure Communication Services
> Der Voice-Kanal basiert auf Azure Communication Services und erfordert keine separate Telefonanlage. Rufnummern werden direkt in Azure bereitgestellt.

**Voice-Features:**
- Inbound/Outbound Calls
- IVR (Interactive Voice Response) via Power Virtual Agents
- Call Recording für Compliance und Coaching
- Echtzeit-Transkription durch Azure Cognitive Services
- Supervisor-Dashboard für Live-Queue-Überwachung

---

## Knowledge Management

### Knowledge-Artikel-Lifecycle

```mermaid
graph LR
    DR[Draft Entwurf] -->|Review| RE[Review Genehmigung]
    RE -->|Genehmigt| PU[Published Veröffentlicht]
    PU -->|Update nötig| DR
    PU -->|Veraltet| EX[Expired Archiviert]
```

### Knowledge-Konfiguration

| Feature | Beschreibung |
|---------|-------------|
| **Kategorien** | Hierarchische Strukturierung der Artikel |
| **Schlüsselwörter** | Tags für bessere Auffindbarkeit |
| **Bewertung** | Agenten und Kunden können Artikel bewerten |
| **Feedback** | Kommentare und Verbesserungsvorschläge |
| **Externe Portale** | Artikel in Customer Self-Service Portal veröffentlichen |

### KI-gestützte Knowledge-Vorschläge

Während ein Agent einen Case bearbeitet, schlägt D365 automatisch relevante Knowledge-Artikel vor:

```
Case: "Drucker druckt nicht nach Windows-Update"
→ Copilot schlägt vor:
  1. KB0001234: "Druckertreiber nach Windows-Update aktualisieren"
  2. KB0001456: "Drucker-Spooler-Dienst neu starten"
  3. KB0001789: "USB-Drucker nach Update nicht erkannt"
```

---

## Customer Service Insights und Dashboards

### Omnichannel Historical Analytics

| Dashboard | Enthält |
|-----------|---------|
| **Summary** | Case-Volumen, Lösungsrate, SLA-Compliance |
| **Agent** | Agent-Performance, Handhabungszeit, CSAT |
| **Bot** | Bot-Eskalationsrate, selbst gelöste Cases |
| **Queue** | Warteschlangenlänge, Wartezeiten, Overflow |

### Customer Service Insights (KI-Analysen)

- **Trendanalyse:** Automatische Erkennung von aufkommenden Case-Themen
- **Root Cause Analysis:** KI identifiziert häufigste Problemursachen
- **Topic Clustering:** Ähnliche Cases werden automatisch gruppiert
- **Resolution Time Prediction:** Vorhergesagte Lösungszeit pro Case-Typ

---

## Copilot in Customer Service

### Copilot-Features für Agenten

**Case-Zusammenfassung:**
```
Case #00012345 geöffnet → Copilot zeigt:
  ├── Problembeschreibung: "Benutzer kann sich nicht anmelden"
  ├── Betroffenes System: Portal-Login
  ├── Bereits versucht: Passwort-Reset (erfolglos)
  ├── Kundenhistorie: 3 ähnliche Cases in letzten 6 Monaten
  └── Empfohlener nächster Schritt: MFA-Status prüfen
```

**Draft Emails:**
- Copilot erstellt E-Mail-Antworten basierend auf Case-Kontext
- Bezieht Knowledge-Artikel automatisch ein
- Tonalität anpassbar (formell, empathisch, technisch)

**Knowledge Search:**
```powershell
# Copilot-Suche: Natürliche Sprache statt Keywords
"Warum kann der Kunde sein Passwort nicht zurücksetzen?"
→ D365 Copilot durchsucht Knowledge Base und gibt strukturierte Antwort
```

**Konversations-Zusammenfassungen:**
- Am Ende eines Chats/Calls erstellt Copilot eine Zusammenfassung
- Wird automatisch in den Case-Timeline geschrieben
- Reduziert After-Call-Work-Zeit erheblich

> [!TIP] Copilot-Akzeptanzrate messen
> Im Omnichannel Analytics Dashboard kann verfolgt werden, wie oft Agenten Copilot-Vorschläge akzeptieren. Hilft beim ROI-Nachweis.

---

## Supervisor und Manager Tools

### Live-Überwachung

- **Supervisor Dashboard:** Echtzeitübersicht über alle Queues und Agenten
- **Agent Monitoring:** Supervisor kann Konversationen in Echtzeit beobachten
- **Whisper/Barge-in:** Bei Voice-Kanal: Supervisor kann Hinweise geben oder übernehmen

### Workforce Intelligence

| Metrik | Beschreibung |
|--------|-------------|
| **AHT** | Average Handle Time (Durchschnittliche Bearbeitungszeit) |
| **FCR** | First Contact Resolution Rate |
| **CSAT** | Customer Satisfaction Score |
| **NPS** | Net Promoter Score |
| **SLA Compliance** | % der Cases innerhalb SLA-Grenzen |

---

## Preise und Lizenzierung

| Lizenz | Preis/User/Monat | Enthält |
|--------|-----------------|---------|
| **Customer Service Professional** | $50 | Basis Case Management |
| **Customer Service Enterprise** | $105 | + Omnichannel, Unified Routing, Copilot |
| **Customer Service Attach** | $40 | Mit anderer D365-Lizenz |
| **Chat Add-on** | $90/Monat/Kanal | Für Professional-Kunden |
| **Voice Add-on** | Extra | Azure Communication Services |

---

## Verwandte Notizen

- [[Dynamics 365 Überblick]]
- [[Dynamics 365 Sales]]
- [[Dynamics 365 Field Service]]
- [[Dynamics 365 Administration]]
