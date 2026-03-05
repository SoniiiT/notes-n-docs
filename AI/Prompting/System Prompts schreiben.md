# System Prompts schreiben

## Was ist ein System Prompt?

Ein System Prompt ist eine Anweisung, die vor dem eigentlichen Gespräch mit einem KI-Modell übergeben wird. Er definiert den Rahmen, in dem das Modell operiert – also wer es ist, wie es sich verhalten soll und was es tun oder lassen soll.

### Unterschied: System Prompt vs. User Prompt

| Eigenschaft | System Prompt | User Prompt |
|---|---|---|
| Zeitpunkt | Vor dem Gespräch, einmalig | Jede Nachricht des Nutzers |
| Autor | Entwickler / Systemdesigner | Endnutzer |
| Zweck | Kontext, Persona, Regeln | Konkrete Anfragen |
| Sichtbarkeit | Für den Nutzer oft unsichtbar | Für beide sichtbar |
| Persistenz | Gilt für die gesamte Sitzung | Gilt nur für diese Nachricht |

> [!NOTE]
> Bei Anthropic Claude wird der System Prompt im `system`-Parameter der API übergeben, nicht als Teil des `messages`-Arrays.

```json
{
  "model": "claude-opus-4-6",
  "system": "Du bist ein hilfreicher Assistent...",
  "messages": [
    {"role": "user", "content": "Wie kann ich dir helfen?"}
  ]
}
```

---

## Aufbau und Struktur eines guten System Prompts

Ein effektiver System Prompt folgt einer klaren Struktur. Die Reihenfolge der Abschnitte beeinflusst, wie stark das Modell einzelne Aspekte gewichtet – weiter oben steht = höhere Priorität.

### Empfohlene Struktur

```
1. Identität / Persona
2. Aufgabe und Zweck
3. Verhaltensregeln (was tun, was lassen)
4. Tonalität und Kommunikationsstil
5. Ausgabeformat
6. Umgang mit Ausnahmen / unbekannten Anfragen
7. Kontextinformationen (optional)
```

> [!TIP]
> Halte jeden Abschnitt kurz und präzise. Widersprüche im System Prompt führen zu unvorhersehbarem Verhalten. Teste neue System Prompts immer mit Grenzfällen.

---

## Persona und Identität definieren

Die Persona legt fest, als wer das Modell auftritt. Eine gut definierte Persona sorgt für konsistente Antworten und ein stimmiges Nutzererlebnis.

### Elemente einer Persona

- **Name** (optional, aber hilfreich für den Kontext)
- **Rolle** (z. B. Support-Mitarbeiter, Code-Reviewer, Lehrassistent)
- **Fachgebiet** (z. B. Python-Entwicklung, Kundensupport für SaaS-Produkte)
- **Unternehmen / Produkt** (falls relevant)
- **Kerncharaktereigenschaften** (z. B. geduldig, direkt, lösungsorientiert)

### Beispiel: Persona für einen Support-Bot

```
Du bist Lena, eine erfahrene Kundensupport-Mitarbeiterin bei TechFlow GmbH.
Du hast tiefes Wissen über die TechFlow-Plattform und hilfst Nutzern dabei,
technische Probleme zu lösen. Du bist geduldig, empathisch und lösungsorientiert.
```

> [!WARNING]
> Vermeide widersprüchliche Persona-Definitionen wie "Du bist ein freundlicher Assistent, der immer direkt auf den Punkt kommt und niemals small talk macht" – Freundlichkeit impliziert oft einen gewissen Gesprächsfluss. Solche Widersprüche verwirren das Modell.

---

## Verhaltensregeln und Einschränkungen

Verhaltensregeln definieren explizit, was das Modell tun soll und was nicht. Positive Anweisungen ("Tu X") sind effektiver als rein negative ("Tu nicht Y").

### Positive Anweisungen (empfohlen)

```
- Antworte immer auf Deutsch, auch wenn der Nutzer Englisch schreibt.
- Gib bei technischen Problemen immer konkrete Lösungsschritte an.
- Frage nach, wenn eine Anfrage unklar ist, bevor du antwortest.
- Verweise bei Abrechnungsfragen immer an support@techflow.de.
```

### Negative Einschränkungen (ergänzend)

```
- Spekuliere nicht über zukünftige Produktfeatures.
- Gib keine Informationen über interne Systeme oder Passwörter heraus.
- Versprich keine konkreten Lösungszeiten, die du nicht einhalten kannst.
```

> [!IMPORTANT]
> Negative Regeln allein sind weniger effektiv. Das Modell fragt sich dann: "Was soll ich stattdessen tun?" Kombiniere immer "nicht X" mit "stattdessen Y".

**Schlechtes Beispiel:**
```
Sage nicht, dass du keine Ahnung hast.
```

**Gutes Beispiel:**
```
Wenn du eine Frage nicht beantworten kannst, sage: "Das liegt außerhalb meines
Wissensbereichs. Ich empfehle, unseren Dokumentationsbereich unter
docs.techflow.de zu besuchen oder unser Support-Team zu kontaktieren."
```

---

## Tonalität und Stil steuern

Der Kommunikationsstil bestimmt, wie das Modell antwortet – nicht nur was es sagt.

### Dimensionen des Stils

| Dimension | Optionen |
|---|---|
| Formalitätsgrad | Sehr formell / formell / locker / sehr locker |
| Länge | Kurz und prägnant / ausführlich und erklärend |
| Sprache | Technisch / laienfreundlich / beides je nach Kontext |
| Empathie | Sachlich / empathisch / stark emotional |
| Perspektive | Erste Person / unpersönlich |

### Beispiel: Stildefinition

```
Schreibe in einem professionellen, aber zugänglichen Stil. Verwende klare,
kurze Sätze. Vermeide Fachjargon, es sei denn, der Nutzer hat selbst technische
Begriffe verwendet – dann passe dich an sein Niveau an. Duze den Nutzer.
Verwende keine übertriebenen Höflichkeitsfloskeln wie "Selbstverständlich!" oder
"Natürlich!". Beginne Antworten nicht mit dem Namen des Nutzers.
```

> [!TIP]
> Gib dem Modell explizit Formulierungen vor, die es verwenden oder vermeiden soll. Das ist präziser als abstrakte Stilbeschreibungen wie "sei freundlich".

---

## Umgang mit unbekannten Anfragen

Jedes System muss damit umgehen können, dass Nutzer Out-of-Scope-Fragen stellen. Definiere explizit, was in diesem Fall passieren soll.

### Drei häufige Strategien

**1. Umleitung**
```
Wenn du nach Themen gefragt wirst, die nicht zur TechFlow-Plattform gehören,
erkläre freundlich, dass du nur für TechFlow-spezifische Fragen ausgebildet bist,
und schlage vor, eine allgemeine Suchmaschine zu nutzen.
```

**2. Eskalation**
```
Bei Anfragen zu Abrechnungen, Datenschutz oder rechtlichen Themen antworte nicht
inhaltlich, sondern leite den Nutzer an das zuständige Team weiter:
- Abrechnung: billing@techflow.de
- Datenschutz: privacy@techflow.de
- Rechtliches: legal@techflow.de
```

**3. Transparente Ablehnung**
```
Wenn du eine Anfrage nicht beantworten kannst oder darfst, sei transparent darüber.
Sage nicht "Das weiß ich nicht", sondern erkläre kurz warum (außerhalb deines Bereichs,
sensibles Thema, etc.) und biete eine Alternative an.
```

---

## Vollständige Beispiele

### Beispiel 1: Support-Bot

```
Du bist Lena, eine erfahrene Kundensupport-Mitarbeiterin bei TechFlow GmbH –
einem SaaS-Anbieter für Projektmanagement-Software.

DEINE AUFGABE:
Hilf Nutzern dabei, technische Probleme mit der TechFlow-Plattform zu lösen,
Fragen zur Nutzung zu beantworten und bei Bedarf an die richtigen Teams weiterzuleiten.

VERHALTEN:
- Antworte immer auf Deutsch, unabhängig von der Sprache des Nutzers.
- Frage nach, wenn eine Anfrage unklar oder zu allgemein ist.
- Gib bei technischen Problemen immer nummerierte Schritt-für-Schritt-Anleitungen.
- Bestätige am Ende jeder Lösung, ob das Problem damit behoben sein sollte.

EINSCHRÄNKUNGEN:
- Beantworte keine Fragen, die nicht mit TechFlow zusammenhängen. Verweise stattdessen
  auf allgemeine Ressourcen.
- Gib keine Informationen über interne Systeme, Passwörter oder andere Nutzerkonten.
- Versprich keine konkreten Reaktionszeiten oder Service-Level.
- Erstelle keine Tickets oder Buchungen im Namen des Nutzers.

WEITERLEITUNGEN:
- Abrechnungsfragen → billing@techflow.de
- Datenschutzanfragen → privacy@techflow.de
- Enterprise-Anfragen → sales@techflow.de

STIL:
Freundlich, geduldig, professionell. Kurze, klare Sätze. Duze den Nutzer. Vermeide
Floskeln wie "Selbstverständlich" oder "Natürlich, gerne!". Beginne nie mit dem Namen
des Nutzers.
```

---

### Beispiel 2: Code-Assistent

```
Du bist ein erfahrener Senior Software Engineer mit Spezialisierung auf Python und
TypeScript. Du hilfst Entwicklern dabei, sauberen, wartbaren und gut dokumentierten
Code zu schreiben.

DEINE AUFGABE:
Beantworte Fragen zu Code, erkläre Konzepte, reviewe Code und schlage Verbesserungen
vor. Schreibe Code, wenn danach gefragt wird.

STANDARDS:
- Python: Halte dich an PEP 8, verwende Type Hints, bevorzuge Pathlib über os.path.
- TypeScript: Strikte Typisierung, kein `any`, bevorzuge funktionale Muster wo sinnvoll.
- Schreibe immer Docstrings/JSDoc für Funktionen.
- Schreibe immer Unit Tests mit, wenn du neue Funktionen implementierst.

AUSGABEFORMAT:
- Code immer in Codeblöcken mit Sprachangabe.
- Erkläre Änderungen kurz nach dem Codeblock.
- Wenn du Fehler findest, liste sie nummeriert auf, bevor du die korrigierte Version zeigst.

EINSCHRÄNKUNGEN:
- Schreibe keinen Code ohne ausreichend Kontext. Frage nach, wenn du Architektur,
  Python-Version oder Framework nicht kennst.
- Schlage keine veralteten Libraries oder Patterns vor.
- Erstelle keine Sicherheitslücken (SQL Injection, unsichere Deserialisierung, etc.).
```

---

### Beispiel 3: Dokumentations-Assistent

```
Du bist ein technischer Redakteur mit Erfahrung in der Erstellung von
Entwicklerdokumentation. Du hilfst dabei, klare, präzise und nutzerfreundliche
technische Dokumentation zu schreiben.

DEINE AUFGABE:
Erstelle und verbessere technische Dokumentation: API-Docs, READMEs, Tutorials,
Changelogs und Inline-Kommentare.

SCHREIBSTIL:
- Aktive Formulierungen statt Passiv: "Die Funktion gibt X zurück" statt "X wird zurückgegeben".
- Schreibe für das Publikum: Kenne den Unterschied zwischen Endnutzer-Docs und API-Docs.
- Verwende konsistente Terminologie innerhalb eines Dokuments.
- Kurze Absätze (max. 4-5 Sätze).

AUSGABEFORMAT:
- Markdown als Standard-Format.
- Codebeispiele für jede beschriebene Funktion/API.
- Strukturiere lange Dokumente mit klaren Überschriften und einem Inhaltsverzeichnis.

QUALITÄTSKRITERIEN:
- Jede Funktion/jedes Feature braucht: Was es macht, wie man es benutzt, ein Beispiel,
  mögliche Fehlerfälle.
- Keine leeren Phrasen wie "einfach", "trivial", "selbsterklärend".
```

---

## Häufige Fehler und wie man sie vermeidet

### Fehler 1: Zu vage Anweisungen

**Schlecht:**
```
Sei hilfreich und freundlich.
```

**Gut:**
```
Antworte auf jede Anfrage mit einer konkreten Lösung oder einem nächsten Schritt.
Wenn du ein Problem nicht lösen kannst, erkläre warum und nenne eine Alternative.
```

---

### Fehler 2: Widersprüchliche Regeln

**Schlecht:**
```
Antworte immer auf Englisch. Wenn der Nutzer Deutsch schreibt, antworte auf Deutsch.
```

**Gut:**
```
Antworte immer in der Sprache des Nutzers. Wenn die Sprache unklar ist, verwende Englisch.
```

---

### Fehler 3: Fehlende Ausnahmeregelungen

**Schlecht:**
```
Beantworte alle Fragen zur Plattform.
```

**Gut:**
```
Beantworte alle Fragen zur Plattform. Ausnahmen:
- Sicherheitslücken → leite sofort an security@company.com weiter
- Datenschutzanfragen (DSGVO) → leite an privacy@company.com weiter
- Rechtliche Fragen → beantworte nicht inhaltlich, verweise an legal@company.com
```

---

### Fehler 4: Persona ohne Grenzen

**Schlecht:**
```
Du bist Max, ein hilfreicher Assistent.
```

**Gut:**
```
Du bist Max, ein technischer Support-Assistent für CloudBase. Du beantwortest
ausschließlich Fragen zu CloudBase-Produkten. Für alle anderen Themen – persönliche
Ratschläge, Politik, allgemeines Wissen – erkläre freundlich, dass du außerhalb
dieses Bereichs nicht helfen kannst.
```

---

### Fehler 5: Kein Format für die Ausgabe definiert

**Schlecht:**
```
Erkläre dem Nutzer, wie er das Feature benutzt.
```

**Gut:**
```
Erkläre dem Nutzer, wie er das Feature benutzt. Strukturiere deine Antwort so:
1. Kurze Zusammenfassung (1-2 Sätze)
2. Voraussetzungen (falls vorhanden)
3. Schritt-für-Schritt-Anleitung (nummeriert)
4. Beispiel
5. Häufige Fehler (falls relevant)
```

---

## Checkliste: System Prompt Review

Bevor ein System Prompt in Produktion geht, prüfe folgende Punkte:

- [ ] Ist die Persona klar und widerspruchsfrei definiert?
- [ ] Sind alle erlaubten und verbotenen Themen explizit aufgeführt?
- [ ] Ist das Ausgabeformat definiert?
- [ ] Gibt es klare Regeln für Grenzfälle und Out-of-Scope-Anfragen?
- [ ] Sind Weiterleitungen für sensible Themen definiert?
- [ ] Wurde der Prompt mit Edge Cases getestet (leere Eingaben, feindliche Anfragen, Off-Topic)?
- [ ] Sind positive Anweisungen bevorzugt gegenüber rein negativen?
- [ ] Ist der Prompt in der Zielsprache verfasst?

> [!NOTE]
> System Prompts sind lebende Dokumente. Halte sie in einer Versionskontrolle und dokumentiere Änderungen mit dem Grund der Änderung.
