# Output-Formatierung steuern

## Warum Output-Formatierung wichtig ist

Ein KI-Modell kann dieselbe Information in dutzenden verschiedenen Formaten ausgeben. Ohne explizite Formatierungsanweisungen wählt das Modell ein Format, das es für passend hält – was oft nicht mit dem übereinstimmt, was die Anwendung erwartet.

Präzise Formatierungsanweisungen sind entscheidend für:

- **Maschinelle Weiterverarbeitung**: JSON, CSV oder XML muss exakt strukturiert sein
- **Konsistenz**: Gleiche Prompts sollen gleich aussehende Ausgaben produzieren
- **Downstream-Verarbeitung**: Parser, Datenbanken und APIs erwarten bestimmte Formate
- **Nutzererfahrung**: Lesbarkeit und Übersichtlichkeit in der Darstellung

> [!IMPORTANT]
> Formatierungsanweisungen gehören immer in den System Prompt oder an den Anfang des User Prompts – nicht ans Ende. Spätere Anweisungen können frühere überschreiben, aber frühe Formatierungsanweisungen setzen den Rahmen für die gesamte Ausgabe.

---

## JSON Output erzwingen

JSON ist das häufigste Format für maschinell verarbeitete KI-Ausgaben. Die Herausforderung: Modelle neigen dazu, JSON in Markdown-Codeblöcke zu verpacken oder erläuternden Text hinzuzufügen.

### Prompt-Strategie: Reines JSON

```
Analysiere den folgenden Produkttext und extrahiere die Informationen.
Antworte AUSSCHLIESSLICH mit validem JSON. Kein erklärender Text, keine Codeblöcke,
kein Markdown. Nur das JSON-Objekt.

Schema:
{
  "name": "string",
  "price_eur": number,
  "category": "string",
  "in_stock": boolean,
  "features": ["string"]
}

Produkttext:
"Das Sony WH-1000XM5 ist ein Premium Over-Ear Kopfhörer mit aktiver
Geräuschunterdrückung für 349 Euro. Aktuell auf Lager. Features:
30h Akku, Multipoint-Verbindung, LDAC-Unterstützung, faltbares Design."
```

**Erwartete Ausgabe:**
```json
{
  "name": "Sony WH-1000XM5",
  "price_eur": 349,
  "category": "Kopfhörer",
  "in_stock": true,
  "features": [
    "30h Akku",
    "Multipoint-Verbindung",
    "LDAC-Unterstützung",
    "faltbares Design",
    "aktive Geräuschunterdrückung"
  ]
}
```

### Prompt-Strategie: JSON-Array für Mehrfach-Extraktion

```
Extrahiere alle Personen aus dem folgenden Text. Gib ein JSON-Array zurück.
Antworte nur mit dem Array, ohne weiteren Text.

Format pro Person:
{"name": "string", "role": "string", "mentioned_action": "string"}

Text:
"CEO Thomas Müller kündigte heute an, die Expansion nach Asien voranzutreiben.
CTO Anna Weber leitet das neue Technologie-Team. Vertriebsleiter Klaus Braun
soll bis Q2 drei neue Partnerschaften abschließen."
```

**Ausgabe:**
```json
[
  {"name": "Thomas Müller", "role": "CEO", "mentioned_action": "Expansion nach Asien ankündigen"},
  {"name": "Anna Weber", "role": "CTO", "mentioned_action": "Technologie-Team leiten"},
  {"name": "Klaus Braun", "role": "Vertriebsleiter", "mentioned_action": "drei neue Partnerschaften abschließen"}
]
```

> [!TIP]
> Viele Modelle (inkl. Claude und GPT-4) unterstützen JSON Mode / Structured Outputs auf API-Ebene. Nutze diese Funktion, wenn verfügbar – sie ist zuverlässiger als rein prompt-basierte JSON-Anweisungen.

```python
# Beispiel: Claude API mit strukturierter Ausgabe
import anthropic
import json

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system="Antworte ausschließlich mit validem JSON. Kein weiterer Text.",
    messages=[{
        "role": "user",
        "content": "Extrahiere Name und Alter aus: 'Max ist 28 Jahre alt.'"
    }]
)

data = json.loads(response.content[0].text)
```

---

## Markdown-Formatierung steuern

### Markdown aktivieren

```
Erstelle eine strukturierte Anleitung zur Installation von Docker unter Ubuntu.
Verwende Markdown-Formatierung:
- H2 für Hauptabschnitte
- Nummerierte Listen für Schritt-für-Schritt-Anleitungen
- Code-Blöcke (mit bash als Sprachangabe) für alle Befehle
- Hinweisboxen für Warnungen (mit > [!WARNING])
```

### Markdown deaktivieren (für plain text)

```
Erkläre den Unterschied zwischen TCP und UDP. Antworte in reinem Fließtext
ohne Markdown-Formatierung. Keine Sterne, keine Hashtags, keine Codeblöcke,
keine Aufzählungszeichen. Nur normaler Text mit Absätzen.
```

### Markdown-Grad kontrollieren

```
Beantworte die Frage kurz und prägnant. Verwende maximal eine Aufzählungsliste
wenn nötig. Keine Überschriften, keine Code-Blöcke, keine Tabellen.
Maximal 100 Wörter.

Frage: Was sind die drei wichtigsten Vorteile von TypeScript gegenüber JavaScript?
```

> [!NOTE]
> Wenn deine Anwendung Markdown nicht rendert (z.B. Terminal, plain-text E-Mail, SMS), schreibe explizit "Verwende kein Markdown" in den System Prompt. Modelle verwenden standardmäßig Markdown bei Fragen, die strukturierte Antworten nahelegen.

---

## XML-Tags für strukturierte Ausgaben

XML-Tags sind besonders bei Anthropic-Modellen (Claude) effektiv, da sie im Training stark mit strukturierten Antworten assoziiert sind. Sie ermöglichen es, mehrere verschiedene Ausgaben sauber zu trennen.

### Einfache XML-Struktur

```
Analysiere den folgenden Code-Review-Kommentar. Gib deine Ausgabe in XML-Tags:
- <summary>: Ein-Satz-Zusammenfassung
- <severity>: critical / major / minor / suggestion
- <affected_code>: Welcher Code-Teil ist betroffen
- <recommendation>: Konkrete Empfehlung

Kommentar:
"Diese Funktion hat O(n²) Komplexität durch den verschachtelten Loop. Bei großen
Datensätzen wird das zum Bottleneck."
```

**Ausgabe:**
```xml
<summary>Verschachtelter Loop erzeugt O(n²)-Komplexität mit Performance-Risiko bei großen Datensätzen.</summary>
<severity>major</severity>
<affected_code>Verschachtelter Loop innerhalb der Funktion</affected_code>
<recommendation>
Refaktoriere auf O(n) durch Verwendung eines Dictionary/HashMap für den inneren Loop.
Alternativ: numpy-Vektorisierung prüfen wenn numerische Daten verarbeitet werden.
</recommendation>
```

### XML für mehrteilige Ausgaben

```
Übersetze den folgenden Text ins Englische und ins Französische.
Verwende <original>, <english> und <french> Tags.

Text: "Das Deployment war erfolgreich. Alle Services laufen stabil."
```

**Ausgabe:**
```xml
<original>Das Deployment war erfolgreich. Alle Services laufen stabil.</original>
<english>The deployment was successful. All services are running stably.</english>
<french>Le déploiement a été effectué avec succès. Tous les services fonctionnent de manière stable.</french>
```

> [!TIP]
> XML-Tags lassen sich in Python einfach mit `re` oder `xml.etree.ElementTree` parsen. Bei Claude-Modellen können XML-Tags in System Prompts auch genutzt werden, um den Prompt selbst zu strukturieren (z.B. `<instructions>`, `<context>`, `<examples>`).

---

## Tabellen anfordern

### Einfache Tabelle

```
Vergleiche PostgreSQL, MySQL und SQLite nach folgenden Kriterien in einer
Markdown-Tabelle: Einsatzbereich, Skalierbarkeit, Lizenz, ACID-Konformität,
und beste Anwendungsfälle.
```

### Tabelle mit spezifischen Spalten und Formatierung

```
Erstelle eine Vergleichstabelle für Python Web-Frameworks. Die Tabelle soll
genau diese Spalten haben (in dieser Reihenfolge):
| Framework | Version | Typ | Performance | Lernkurve | Empfohlen für |

Fülle die Tabelle für: Django, FastAPI, Flask, Starlette.
Verwende für Performance und Lernkurve: Hoch / Mittel / Niedrig.
```

**Erwartete Ausgabe:**

| Framework | Version | Typ | Performance | Lernkurve | Empfohlen für |
|---|---|---|---|---|---|
| Django | 5.x | Full-Stack | Mittel | Hoch | Große Webanwendungen, CMS |
| FastAPI | 0.x | API | Hoch | Mittel | REST APIs, Microservices |
| Flask | 3.x | Micro | Mittel | Niedrig | Kleine APIs, Prototypen |
| Starlette | 0.x | ASGI | Hoch | Hoch | Performance-kritische APIs |

> [!WARNING]
> Modelle halluzinieren manchmal bei Versionsnummern oder spezifischen Benchmarks in Tabellen. Verifiziere numerische Angaben immer aus offiziellen Quellen, bevor du sie in Dokumentation übernimmst.

---

## Listen und nummerierte Ausgaben

### Nummerierte Schritt-für-Schritt-Ausgabe

```
Erkläre, wie man einen Git Feature Branch erstellt und einen Pull Request öffnet.
Gib die Anleitung als nummerierte Liste mit maximal 8 Schritten.
Jeden Schritt beginne mit einem Verb im Imperativ.
Code-Befehle in Backtick-Code-Spans.
```

### Bullet-Point-Liste mit Unterstruktur

```
Liste die wichtigsten HTTP-Statuscodes auf. Strukturiere sie in drei Gruppen:
- 2xx (Erfolg): mindestens 3 Codes
- 4xx (Client-Fehler): mindestens 5 Codes
- 5xx (Server-Fehler): mindestens 3 Codes

Format pro Eintrag: **CODE** – Name – kurze Erklärung (1 Satz)
```

### Sortierte Ausgaben erzwingen

```
Nenne die 5 meistgenutzten Python-Bibliotheken für Data Science.
Sortiere nach Wichtigkeit (wichtigste zuerst).
Format: Rang. **Name** – ein Satz Beschreibung – typischer Use Case
```

---

## Ausgabelänge kontrollieren

### Maximale Wortanzahl

```
Fasse den folgenden Artikel zusammen. Maximale Länge: 150 Wörter.
Priorisiere: Kernaussage, wichtigste Fakten, Schlussfolgerung.
```

### Minimale Tiefe erzwingen

```
Erkläre das CAP-Theorem für Datenbanken. Deine Antwort soll:
- Mindestens 300 Wörter lang sein
- Alle drei Buchstaben (C, A, P) einzeln erklären
- Ein konkretes Praxisbeispiel für jedes Prinzip enthalten
- Eine Tabelle mit bekannten Datenbanken und ihrer CAP-Einordnung enthalten
```

### Stufenweise Ausgabe

```
Erkläre Machine Learning in drei Versionen:
1. Für einen 10-Jährigen (2-3 Sätze)
2. Für einen Nicht-Techniker (1 Absatz)
3. Für einen Softwareentwickler ohne ML-Erfahrung (2-3 Absätze mit technischen Details)

Trenne die Versionen mit einer horizontalen Linie (---).
```

> [!TIP]
> "Maximal X Wörter" ist zuverlässiger als "kurz" oder "knapp". Für kritische Längenkontrollen: Zähle die Wörter in der Ausgabe programmatisch und sende den Prompt erneut wenn nötig.

---

## Mehrsprachige Ausgaben

### Immer in einer bestimmten Sprache antworten

```
[Im System Prompt]
Antworte immer auf Deutsch, unabhängig von der Sprache der Nutzereingabe.
```

### Sprache der Eingabe erkennen und spiegeln

```
[Im System Prompt]
Erkenne die Sprache der Nutzereingabe und antworte in derselben Sprache.
Unterstützte Sprachen: Deutsch, Englisch, Französisch, Spanisch.
Bei anderen Sprachen: Antworte auf Englisch und weise auf die unterstützten
Sprachen hin.
```

### Mehrsprachige Ausgabe gleichzeitig

```
Übersetze den folgenden Text in alle genannten Sprachen.
Verwende folgendes Format:

🇩🇪 **Deutsch:** [Text]
🇬🇧 **Englisch:** [Text]
🇫🇷 **Französisch:** [Text]
🇪🇸 **Spanisch:** [Text]

Text: "The service is currently under maintenance. Expected downtime: 2 hours."
```

---

## Code-Ausgaben mit spezifischen Anforderungen

### Sprache und Version erzwingen

```
Schreibe den folgenden Code in Python 3.12 um. Anforderungen:
- Verwende f-Strings (kein .format() oder %)
- Verwende Type Hints für alle Parameter und Rückgabewerte
- Keine deprecated APIs
- Code-Block mit python als Sprachangabe
- Kurzer Kommentar über der Funktion (kein Docstring)
```

### Code + Erklärung Struktur

```
Implementiere eine Funktion zum Validieren von E-Mail-Adressen in TypeScript.

Ausgabe-Struktur:
1. Code-Block (typescript) mit der Implementierung
2. Kurze Erklärung (max. 5 Sätze) was der Code macht
3. Beispiel-Aufruf mit 3 verschiedenen Eingaben (valide, invalide, Grenzfall)
```

### Mehrere Implementierungen vergleichen

```
Zeige zwei verschiedene Implementierungen für folgende Aufgabe:
"Alle doppelten Einträge aus einer Liste entfernen"

Implementierung 1: Klassischer Ansatz (for-loop)
Implementierung 2: Pythonischer Ansatz (Set/Dict)

Für jede Implementierung:
- Code-Block (python)
- Zeit-Komplexität: O(?)
- Speicher-Komplexität: O(?)
- Wann bevorzugen: [eine Zeile]

Dann eine abschließende Empfehlung.
```

---

## Praktische Referenz: Formatierungs-Prompts

### Schnell-Referenz für häufige Formate

```
# Reines JSON
"Antworte ausschließlich mit validem JSON. Kein weiterer Text, kein Markdown."

# Markdown mit Struktur
"Verwende H2-Überschriften für Abschnitte, Code-Blöcke für Code,
und Fett-Formatierung für wichtige Begriffe."

# Plain Text
"Antworte in reinem Fließtext. Kein Markdown, keine Formatierungszeichen."

# Nummerierte Liste
"Gib deine Antwort als nummerierte Liste. Maximal [N] Punkte."

# Tabelle
"Fasse die Informationen in einer Markdown-Tabelle zusammen.
Spalten: [Spalte1] | [Spalte2] | [Spalte3]"

# XML-Struktur
"Verwende XML-Tags für die Ausgabe: <tag1>, <tag2>, <tag3>"

# Länge
"Maximal [N] Wörter / Sätze / Zeichen."
```

> [!NOTE]
> Teste Formatierungsanweisungen immer mit Edge Cases: Was passiert, wenn die Anfrage unvollständige Daten enthält? Hält das Modell das Format auch bei kurzen/langen Eingaben ein? Passt die Ausgabe zum Schema auch wenn keine passenden Daten gefunden werden?
