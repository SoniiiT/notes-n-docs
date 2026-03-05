# Prompt Engineering Techniken

## Überblick

Prompt Engineering bezeichnet die systematische Gestaltung von Eingaben (Prompts) für KI-Sprachmodelle, um qualitativ hochwertige, zuverlässige und nützliche Ausgaben zu erzielen. Die Wahl der richtigen Technik hängt von der Aufgabenart, der gewünschten Ausgabe und den verfügbaren Ressourcen ab.

> [!NOTE]
> Keine Technik ist universell optimal. Die folgenden Techniken sind Werkzeuge – sie müssen situativ eingesetzt werden. Oft ist eine Kombination mehrerer Techniken effektiver als eine einzelne.

---

## Zero-Shot Prompting

### Definition

Zero-Shot Prompting bedeutet, dem Modell eine Aufgabe zu stellen, ohne Beispiele dafür zu liefern. Das Modell verlässt sich vollständig auf sein vortrainiertes Wissen.

### Wann verwenden

- Einfache, klar definierbare Aufgaben
- Wenn keine Beispieldaten verfügbar sind
- Schnelle Tests und Prototypen

### Beispiel

```
Klassifiziere den folgenden Kundenfeedback als "positiv", "negativ" oder "neutral".
Antworte nur mit einem der drei Wörter.

Feedback: "Die Lieferung war schnell, aber die Verpackung war beschädigt."
```

**Ausgabe:**
```
negativ
```

> [!TIP]
> Zero-Shot funktioniert besser, wenn die Aufgabe klar formuliert und die möglichen Ausgaben explizit aufgeführt sind. "Klassifiziere als positiv, negativ oder neutral" ist besser als "Bewerte das Feedback".

---

## Few-Shot Prompting

### Definition

Few-Shot Prompting liefert dem Modell 2–5 Beispiele (Shots) für die gewünschte Eingabe-Ausgabe-Struktur, bevor die eigentliche Aufgabe gestellt wird. Das Modell lernt das Muster aus den Beispielen und wendet es auf die neue Eingabe an.

### Wann verwenden

- Wenn das Ausgabeformat sehr spezifisch ist
- Bei domänenspezifischen Klassifizierungsaufgaben
- Wenn Zero-Shot inkonsistente Ergebnisse liefert
- Bei Transformationsaufgaben mit klarem Muster

### Beispiel: Stimmungsanalyse mit 3 Shots

```
Analysiere den folgenden Text und gib die Stimmung sowie eine Begründung aus.
Format: Stimmung: [POSITIV/NEGATIV/NEUTRAL] | Grund: [kurze Erklärung]

Text: "Endlich hat mein Paket angekommen! Der Support war super hilfsbereit."
Stimmung: POSITIV | Grund: Positive Erfahrung mit Lieferung und Support.

Text: "Dritte Bestellung in Folge falsch geliefert. Totale Katastrophe."
Stimmung: NEGATIV | Grund: Wiederholte Fehler, starke Frustration.

Text: "Artikel angekommen. Entspricht der Beschreibung."
Stimmung: NEUTRAL | Grund: Sachliche Aussage ohne emotionale Wertung.

Text: "Die App läuft gut, aber der Akku meines Handys wird dabei sehr warm."
```

**Ausgabe:**
```
Stimmung: NEUTRAL | Grund: Gemischte Aussage – positive Funktionalität, aber negative Nebenwirkung ohne starke emotionale Färbung.
```

> [!IMPORTANT]
> Die Qualität der Shots ist entscheidend. Wähle Beispiele, die die Grenzfälle deiner Aufgabe repräsentieren, nicht nur die einfachen Fälle. Ein schlechtes Beispiel kann das Modell in die falsche Richtung führen.

### Tipps für gute Shots

- Variiere die Beispiele: Zeige verschiedene Schwierigkeitsgrade
- Halte das Format der Shots exakt konsistent
- Platziere den schwierigsten Fall möglichst nah an der eigentlichen Aufgabe
- 3–5 Shots sind meist optimal; mehr hilft selten

---

## Chain-of-Thought Prompting (CoT)

### Definition

Chain-of-Thought Prompting weist das Modell an, seinen Denkprozess Schritt für Schritt explizit aufzuschreiben, bevor es zur Antwort kommt. Dies verbessert deutlich die Qualität bei Aufgaben, die logisches Schlussfolgern erfordern.

### Wann verwenden

- Mathematische Probleme
- Logische Schlussfolgerungen
- Mehrstufige Entscheidungen
- Analyse komplexer Sachverhalte

### Varianten

**Variante 1: Direkte CoT-Aufforderung**
```
Löse das folgende Problem Schritt für Schritt. Zeige jeden Denkschritt explizit.

Ein Lager hat 240 Artikel. 30% werden verkauft, dann kommen 50 neue Artikel rein.
Wie viele Artikel sind im Lager?
```

**Ausgabe (Modell):**
```
Schritt 1: Ausgangsbestand = 240 Artikel
Schritt 2: 30% von 240 werden verkauft → 240 × 0,30 = 72 Artikel verkauft
Schritt 3: Verbleibend nach Verkauf → 240 - 72 = 168 Artikel
Schritt 4: Neue Artikel hinzugefügt → 168 + 50 = 218 Artikel
Ergebnis: 218 Artikel
```

**Variante 2: Few-Shot CoT (Denkschritte als Beispiel)**
```
Frage: Marie hat 3 Äpfel. Sie gibt 1 ab und kauft 4. Wie viele hat sie?
Denken: Start: 3 Äpfel. Minus 1: 2 Äpfel. Plus 4: 6 Äpfel.
Antwort: 6

Frage: Ein Zug fährt 60 km/h. Nach 2,5 Stunden, wie weit ist er gefahren?
Denken: Distanz = Geschwindigkeit × Zeit = 60 × 2,5 = 150 km.
Antwort: 150 km

Frage: Ein Shop hat 500 Produkte. 20% sind ausverkauft. Wie viele sind verfügbar?
```

> [!TIP]
> Der Trigger-Satz "Denke Schritt für Schritt" (oder "Let's think step by step") allein reicht bei einfacheren Modellen bereits aus, um CoT zu aktivieren. Bei komplexen Aufgaben ist explizites Few-Shot CoT zuverlässiger.

---

## Tree-of-Thought (ToT)

### Definition

Tree-of-Thought erweitert CoT, indem das Modell mehrere verschiedene Lösungspfade parallel entwickelt, diese bewertet und dann den vielversprechendsten weiterverfolgt. Es simuliert ein Entscheidungsbaum-Denken.

### Wann verwenden

- Komplexe Planungsaufgaben
- Kreative Aufgaben mit vielen möglichen Ansätzen
- Strategische Entscheidungen
- Aufgaben, bei denen der erste Ansatz oft suboptimal ist

### Beispiel-Prompt

```
Du sollst eine Marketing-Kampagne für eine neue App planen. Entwickle DREI verschiedene
Strategieansätze. Bewerte jeden Ansatz dann nach Kosten (1-5), Reichweite (1-5) und
Umsetzungsaufwand (1-5). Wähle anschließend den besten Ansatz und entwickle ihn weiter.

App: Eine Habit-Tracking-App für Programmierer.
Budget: 5.000 Euro/Monat.

ANSATZ 1: [dein erster Ansatz]
Bewertung: Kosten: X/5 | Reichweite: X/5 | Aufwand: X/5

ANSATZ 2: [dein zweiter Ansatz]
Bewertung: Kosten: X/5 | Reichweite: X/5 | Aufwand: X/5

ANSATZ 3: [dein dritter Ansatz]
Bewertung: Kosten: X/5 | Reichweite: X/5 | Aufwand: X/5

BESTE WAHL: [begründete Auswahl]
DETAILPLAN: [Ausarbeitung des besten Ansatzes]
```

> [!NOTE]
> ToT ist rechenintensiv und produziert lange Ausgaben. Es eignet sich nicht für einfache Aufgaben, bietet aber bei komplexen Planungsszenarien deutlich bessere Ergebnisse als lineares CoT.

---

## ReAct (Reasoning + Acting)

### Definition

ReAct kombiniert Denken (Reasoning) und Handeln (Acting) in einem iterativen Prozess. Das Modell denkt nach, führt eine Aktion aus (z. B. Tool-Aufruf, Suche), beobachtet das Ergebnis und denkt dann weiter. Dieses Muster bildet die Grundlage für KI-Agenten.

### Ablaufschema

```
Thought: [Was denke ich / Was brauche ich?]
Action: [Welche Aktion führe ich aus?]
Observation: [Was ist das Ergebnis?]
Thought: [Was schlussfolgere ich daraus?]
Action: [Nächste Aktion]
...
Answer: [Finale Antwort]
```

### Beispiel (simuliert ohne echte Tools)

```
Du bist ein Recherche-Assistent. Löse die folgende Aufgabe. Zeige deinen
Denkprozess mit den Labels "Thought:", "Action:", "Observation:".

Aufgabe: Finde heraus, welche Python-Version für ein neues Backend-Projekt
empfohlen wird und warum.
```

**Modell-Ausgabe:**
```
Thought: Ich muss die aktuell empfohlene Python-Version für produktive Backends
         ermitteln. Ich sollte den offiziellen Python-Status und gängige
         Empfehlungen berücksichtigen.

Action: Überprüfe Python-Versionsstand (aus meinem Trainingswissen)

Observation: Python 3.12 ist die aktuelle stabile Version. Python 3.11 hat
             deutliche Performance-Verbesserungen (10-60% schneller als 3.10).
             Python 3.9 ist End-of-Life.

Thought: Für ein neues Projekt sollte man eine aktuelle, gut unterstützte
         Version wählen. 3.12 ist aktuell, 3.11 hat breite Library-Unterstützung.

Answer: Für ein neues Backend-Projekt empfehle ich Python 3.12 als erste Wahl,
        da es die aktuelle stabile Version ist. Wenn Library-Kompatibilität
        kritisch ist, ist 3.11 eine sichere Alternative mit bewährtem Ökosystem.
```

> [!IMPORTANT]
> ReAct entfaltet sein volles Potenzial erst in Verbindung mit echten Tool-Calls (Websuche, Code-Ausführung, Datenbankabfragen). In reinen Text-Szenarien ist es ein nützliches Denk-Framework, aber kein echter Agent.

---

## Self-Consistency

### Definition

Self-Consistency lässt das Modell dieselbe Frage mehrfach mit einer festen, höheren Temperatur beantworten. Die Varianz zwischen den Antworten entsteht dabei durch das stochastische Sampling des Modells, nicht durch unterschiedliche Temperaturwerte. Anschließend wird die häufigste oder konsistenteste Antwort als finale Ausgabe gewählt. Dies erhöht die Zuverlässigkeit bei schwierigen Fragen.

### Wann verwenden

- Faktenbasierte Fragen mit eindeutiger Antwort
- Wenn einzelne Modell-Antworten unzuverlässig sind
- Entscheidungsunterstützung bei kritischen Fragen
- Validierung von komplexen Berechnungen

### Implementierung (Prompt-Seite)

```
Löse das folgende Problem. Generiere DREI unabhängige Lösungswege.
Vergleiche dann die Ergebnisse und gib die finale Antwort an.

Problem: Ein Unternehmen hat 150 Mitarbeiter. 40% arbeiten remote. Von den
Remote-Mitarbeitern sind 25% Teilzeit. Wie viele Vollzeit-Remote-Mitarbeiter gibt es?

Weg 1: [erste Lösung]
Weg 2: [zweite Lösung, anderer Ansatz]
Weg 3: [dritte Lösung]
Konsens: [Finale Antwort mit Begründung]
```

**Ausgabe:**
```
Weg 1: 150 × 0,40 = 60 Remote. 60 × 0,25 = 15 Teilzeit-Remote. 60 - 15 = 45 Vollzeit-Remote.
Weg 2: 40% von 150 = 60 Remote-Mitarbeiter. 75% davon Vollzeit = 60 × 0,75 = 45.
Weg 3: Remote: 150 × 0,4 = 60. Vollzeit-Remote: 60 × (1 - 0,25) = 60 × 0,75 = 45.
Konsens: Alle drei Wege kommen zum gleichen Ergebnis: 45 Vollzeit-Remote-Mitarbeiter.
```

> [!TIP]
> In Produktionssystemen kann Self-Consistency durch mehrere API-Calls mit erhöhter Temperatur implementiert werden. Der API-Aufwand steigt proportional zur Anzahl der Samples, aber die Zuverlässigkeit verbessert sich deutlich.

---

## Meta-Prompting

### Definition

Meta-Prompting bedeutet, das Modell selbst einen Prompt erstellen zu lassen. Anstatt den optimalen Prompt manuell zu schreiben, beschreibt man das Ziel auf hoher Ebene und lässt das Modell den eigentlichen Prompt generieren.

### Wann verwenden

- Wenn man nicht weiß, wie man eine Aufgabe am besten formuliert
- Um eine Sammlung von Prompt-Varianten zu generieren
- Für die Iteration und Verbesserung bestehender Prompts
- Für spezialisierte Domänen, in denen man kein Experte ist

### Beispiel: Prompt-Generierung

```
Du bist ein Experte für Prompt Engineering. Erstelle einen optimalen System Prompt
für folgenden Anwendungsfall:

Anwendungsfall: Ein KI-Assistent soll Datenbank-Abfragen in natürlicher Sprache
entgegennehmen und daraus korrektes PostgreSQL generieren. Die Datenbank ist ein
E-Commerce-System mit Tabellen für Orders, Products, Customers und Inventory.

Anforderungen an den System Prompt:
- Klar definierte Rolle und Aufgabe
- Regeln für die SQL-Generierung (Sicherheit, Performance)
- Ausgabeformat (SQL + Erklärung)
- Umgang mit unklaren Anfragen

Erstelle den fertigen System Prompt:
```

### Beispiel: Prompt-Verbesserung

```
Ich habe folgenden Prompt, der nicht gut funktioniert:
"Schreibe einen Blogpost über Python."

Analysiere, was an diesem Prompt fehlt und erstelle eine deutlich bessere Version.
Erkläre kurz, was du geändert hast und warum.
```

> [!WARNING]
> Von Meta-Prompting generierte Prompts müssen immer von einem Menschen geprüft werden. Das Modell kann Annahmen treffen, die nicht mit deinen tatsächlichen Anforderungen übereinstimmen. Nutze Meta-Prompting als Ausgangspunkt, nicht als fertiges Ergebnis.

---

## Wann welche Technik verwenden?

| Technik | Aufgabentyp | Aufwand | Qualitätssteigerung | Typische Anwendung |
|---|---|---|---|---|
| Zero-Shot | Einfach, klar definiert | Gering | Basis | Klassifizierung, Übersetzung, Zusammenfassung |
| Few-Shot | Format-kritisch, domänenspezifisch | Mittel | Hoch | Datenextraktion, strukturierte Ausgaben |
| Chain-of-Thought | Logik, Mathe, Analyse | Mittel | Sehr hoch | Berechnungen, Diagnosen, Entscheidungsanalysen |
| Tree-of-Thought | Planung, kreative Aufgaben | Hoch | Sehr hoch | Strategieentwicklung, komplexe Planung |
| ReAct | Agentenaufgaben mit Tools | Sehr hoch | Sehr hoch | KI-Agenten, Research-Workflows |
| Self-Consistency | Fakten, kritische Entscheidungen | Hoch | Hoch (Zuverlässigkeit) | Medizin, Finanzen, Faktenprüfung |
| Meta-Prompting | Prompt-Entwicklung | Gering | Mittel | Rapid Prototyping, Domänen-Onboarding |

---

## Kombinationsstrategien

Die stärksten Ergebnisse entstehen oft durch Kombination:

### Few-Shot + CoT

Zeige dem Modell Beispiele, die den Denkprozess explizit enthalten:

```
Frage: Ein Server hat 32 GB RAM. 60% sind belegt. Wie viel GB sind frei?
Denken: 60% von 32 GB = 32 × 0,6 = 19,2 GB belegt. Frei: 32 - 19,2 = 12,8 GB.
Antwort: 12,8 GB

Frage: 5 Server haben je 16 GB RAM. Insgesamt sind 40 GB belegt. Wie viel % sind frei?
Denken: Gesamt: 5 × 16 = 80 GB. Frei: 80 - 40 = 40 GB. Prozent frei: 40/80 × 100 = 50%.
Antwort: 50%

Frage: Eine Datenbank nutzt 75% von 512 GB Storage. Wie viel GB sind noch frei?
```

### System Prompt + Few-Shot + CoT

- System Prompt: Definiert Rolle und Ausgabeformat
- Few-Shot: Zeigt konkrete Beispiele im definierten Format
- CoT: Fordert explizites Denken innerhalb des Formats an

> [!NOTE]
> Bei der Kombination von Techniken gilt: Teste immer die Wirkung jedes zusätzlichen Elements einzeln. Manchmal verschlechtert eine hinzugefügte Technik das Ergebnis, weil sie den Prompt zu komplex macht.
