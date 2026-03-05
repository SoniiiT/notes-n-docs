# Modell Vergleiche

> [!NOTE]
> Dieser Vergleich basiert auf dem Stand **August 2025**. LLM-Benchmarks und -Preise ändern sich häufig. Die Empfehlungen hier sind praxisorientiert und nicht rein benchmark-basiert – da Benchmarks oft nicht die reale Nutzung abbilden.

---

## Vergleichskriterien

Um LLMs sinnvoll zu vergleichen, braucht es klare Kriterien. Diese sind je nach Anwendungsfall unterschiedlich gewichtet.

| Kriterium | Beschreibung | Messung |
|-----------|-------------|---------|
| **Qualität / Intelligenz** | Genauigkeit, Reasoning, Korrektheit | Benchmarks (MMLU, HumanEval, etc.) + menschliche Bewertung |
| **Geschwindigkeit** | Tokens pro Sekunde (Output), Time-to-First-Token | API-Latenz-Messungen |
| **Kosten** | USD pro Million Input/Output-Tokens | Offizielle Preislisten |
| **Context Window** | Maximale Eingabelänge in Tokens | Dokumentation des Anbieters |
| **Sprachen** | Qualität in nicht-englischen Sprachen | Mehrsprachige Benchmarks |
| **Code-Fähigkeit** | Code-Generierung, Debugging, Erklärung | HumanEval, SWE-Bench |
| **Instruction Following** | Befolgen komplexer Anweisungen | IFEval, manuelle Tests |
| **Tool Use** | Zuverlässigkeit bei Function Calling | Eigene Benchmark-Suites |
| **Multimodalität** | Bild-, Audio-, Video-Verständnis | Visuelle Benchmarks |
| **Datenschutz** | Lokale Ausführung möglich? | Open-Weight vs. Proprietär |

---

## Claude vs. GPT-4o: Detaillierter Vergleich

### Stärken von Claude (Anthropic)

**Instruction Following:**
Claude hält sich sehr präzise an komplexe, mehrstufige Anweisungen. Bei strukturierten Ausgaben (JSON-Schemas, spezifische Formate) ist Claude konsistenter.

```
Aufgabe: "Antworte ausschließlich in gültigem JSON ohne Markdown-Codeblöcke.
Schema: { 'name': string, 'score': number, 'reasons': string[] }"

Claude: Gibt konsistent valides JSON zurück, auch über viele Turns
GPT-4o: Fügt manchmal trotzdem ```json-Blöcke hinzu oder weicht vom Schema ab
```

**Längere Dokumente und Context-Nutzung:**
Claude nutzt den 200K-Token-Context sehr zuverlässig. Die Performance nimmt auch bei fast vollem Context weniger stark ab.

**Schreiben und Nuancierung:**
Claude ist bekannt für präziseres, nuancierteres Schreiben. Weniger Neigung zu übertriebener Positivität ("Certainly! Great question!").

**Sicherheit und Ablehnung:**
Claude lehnt schädliche Anfragen ab, aber die Ablehnung ist präziser kalibriert – weniger False Positives bei legitimen Anfragen in Graubereichen.

### Stärken von GPT-4o (OpenAI)

**Multimodalität:**
GPT-4o verarbeitet Text, Bilder und Audio nativ in einem Modell. Besonders bei Audio-Verarbeitung und Echtzeit-Sprachinteraktion ist GPT-4o führend.

**Breites Ökosystem:**
OpenAI hat das größte Tooling-Ökosystem: Assistants API, Fine-Tuning, DALL-E-Integration, Code Interpreter, umfangreiche SDKs.

**DALL-E und Bildgenerierung:**
Direktintegration von Bildgenerierung in die API.

**Web-Browsing (ChatGPT Plus):**
Bessere Integration von Echtzeit-Websuche in ChatGPT.

### Direkter Vergleich

| Kriterium | Claude Sonnet 4.5 | GPT-4o | Vorteil |
|-----------|------------------|--------|---------|
| Instruction Following | Hervorragend | Sehr gut | Claude |
| Code-Generierung | Sehr gut | Sehr gut | Gleich |
| Reasoning | Sehr gut | Sehr gut | Gleich (ohne Extended Thinking) |
| Langer Context | Hervorragend (200K) | Gut (128K) | Claude |
| Multimodalität | Gut (Text+Bild) | Hervorragend (Text+Bild+Audio) | GPT-4o |
| Schreiben/Nuancierung | Hervorragend | Sehr gut | Claude |
| Deutsch-Qualität | Sehr gut | Sehr gut | Gleich |
| API-Ökosystem | Gut | Hervorragend | GPT-4o |
| Kosten (Input) | $3/M | $5/M | Claude |
| Verfügbarkeit/Uptime | Sehr gut | Sehr gut | Gleich |
| JSON/Tool Reliability | Hervorragend | Sehr gut | Claude |

> [!TIP]
> **Praxisempfehlung:** Für die meisten Produktionsanwendungen (Agents, RAG, strukturierte Extraktion) bietet Claude Sonnet 4.5 das beste Preis-Leistungs-Verhältnis. Für multimodale Anwendungen mit Audio oder intensive DALL-E-Integration ist GPT-4o besser.

---

## Reasoning-Modelle: Extended Thinking vs. o1/o3 vs. Gemini Thinking

### Was sind Reasoning-Modelle?

Reasoning-Modelle führen vor der Antwort einen sichtbaren oder versteckten Denkprozess durch. Dieser "Chain of Thought" verbessert die Leistung bei:
- Mathematik (von AMC bis Olympiade-Niveau)
- Logik-Puzzles
- Komplexem Code-Debugging
- Mehrstufigen Planungsaufgaben

**Kosten:** Reasoning-Modelle sind deutlich teurer als Standard-Modelle, da auch die Denk-Tokens berechnet werden.

### Claude Extended Thinking (Opus 4.5)

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # Max. Tokens für Denkprozess
    },
    messages=[{
        "role": "user",
        "content": "Löse: Wenn f(x) = x³ - 6x² + 11x - 6, finde alle Nullstellen."
    }]
)

for block in response.content:
    if block.type == "thinking":
        print("Denkprozess:", block.thinking)  # Sichtbar!
    elif block.type == "text":
        print("Antwort:", block.text)
```

**Besonderheit:** Der Denkprozess ist **sichtbar** (transparent), was Debugging und Nachvollziehbarkeit verbessert.

### OpenAI o1 und o3

```python
from openai import OpenAI

client = OpenAI()

# o1/o3 haben kein "system"-Feld, Reasoning-Budget wird automatisch gesetzt
response = client.chat.completions.create(
    model="o3",
    messages=[
        {"role": "user", "content": "Erkläre den Beweis des Satzes von Pythagoras."}
    ],
    # reasoning_effort: "low" | "medium" | "high" (o3 unterstützt dies)
    reasoning_effort="high"
)
print(response.choices[0].message.content)
```

**Unterschied:** Das Reasoning bei o1/o3 ist **versteckt** (nicht sichtbar in der API-Antwort). Man sieht nur die finale Antwort.

### Vergleich der Reasoning-Modelle

| Kriterium | Claude Opus 4.5 (Thinking) | o1 | o3 | Gemini 2.0 (Thinking) |
|-----------|---------------------------|----|----|----------------------|
| Mathematik (AIME) | Sehr stark | Stark | State-of-the-Art | Stark |
| Code-Reasoning | Sehr stark | Stark | Sehr stark | Gut |
| Denkprozess sichtbar | Ja | Nein | Nein | Ja (Flash Thinking) |
| Reasoning-Budget einstellbar | Ja (budget_tokens) | Nein (auto) | Ja (reasoning_effort) | Begrenzt |
| Kosten | Hoch ($15/$75 Input/Output) | Sehr hoch | Sehr hoch ($10/$40) | Moderat |
| Geschwindigkeit | Mittel | Langsam | Mittel | Mittel |
| Context Window | 200K | 200K | 200K | 1M |

> [!WARNING]
> Reasoning-Modelle sind für die meisten alltäglichen Aufgaben **überdimensioniert**. Nur einsetzen bei wirklich komplexen Problemen: Olympiade-Mathematik, komplexe Beweise, sehr schwieriges Code-Debugging. Für normale Programmierung ist Claude Sonnet 4.5 oder GPT-4o besser geeignet (schneller und günstiger).

---

## Code-Generierung: Welches Modell ist am besten?

### Benchmark: SWE-Bench Verified

SWE-Bench misst die Fähigkeit, echte GitHub-Issues zu lösen – ein praxisnahes Maß für Code-Qualität.

| Modell | SWE-Bench Score (ca.) | Stärke |
|--------|----------------------|--------|
| Claude Opus 4.5 | ~70%+ | Architektur, komplexe Refactorings |
| GPT-4o | ~48% | Breite Sprachunterstützung |
| o3 | ~72% | Logische Code-Probleme |
| Gemini 2.0 Pro | ~55% | Google-Stack-Integration |
| Qwen2.5-Coder 32B | ~45% | Open-Weight-Spitze |

### Praktische Code-Fähigkeiten

**Code-Generierung (neuer Code):**
```
Aufgabe: "Schreibe eine Python-Klasse für einen LRU-Cache mit Thread-Sicherheit"

Top-Performer: Claude Sonnet 4.5, GPT-4o, Codestral
→ Alle produzieren funktionierenden Code
→ Claude und GPT-4o tendieren zu besserem Docstring/Typ-Annotation-Stil
→ Codestral (Mistral) oft knapperer, idiomatischerer Code
```

**Code-Debugging:**
```
Aufgabe: "Finde den Fehler in diesem 200-Zeilen Python-Script mit Off-by-One-Error"

Top-Performer: Claude Opus 4.5 (Extended Thinking), o3
→ Extended Thinking hilft bei subtilen Bugs
→ Claude erklärt den Fehler besonders präzise
```

**Multi-File-Projekte:**
```
Aufgabe: "Verstehe diese 3.000-Zeilen-Codebase und füge eine neue Feature hinzu"

Top-Performer: Claude (großer Context, gutes File-Tracking)
→ Claude Sonnet/Opus 4.5 behalten den Überblick über große Codebases
→ Konsistentere Variablennamen und Architektur über Dateien hinweg
```

**Empfehlungen nach Code-Aufgabe:**

| Aufgabe | Empfohlenes Modell | Begründung |
|---------|-------------------|-----------|
| Neuen Code schreiben | Claude Sonnet 4.5 oder GPT-4o | Beste Balance |
| Debugging (einfach) | Claude Haiku 4.5 oder GPT-4o-mini | Schnell und günstig |
| Debugging (komplex) | Claude Opus 4.5 mit Extended Thinking | Sichtbares Reasoning |
| Code-Review | Claude Sonnet 4.5 | Präzises Instruction Following |
| Speziell für Code | Codestral (Mistral) | Code-spezialisiert, 80+ Sprachen |
| Lokal, Code-Focus | Qwen2.5-Coder 32B | Beste Open-Weight-Option |
| Erklärungen schreiben | Claude Sonnet 4.5 | Besonders klare Erklärungen |

---

## Lange Dokumente: Context-Vergleich

### Das "Lost in the Middle"-Problem

Modelle neigen dazu, Informationen am **Anfang und Ende** des Contexts besser abzurufen als in der Mitte. Dies ist bei sehr langen Prompts relevant.

```
Context-Struktur:
[Anfang – gut erinnert] ... [Mitte – schlechter erinnert] ... [Ende – gut erinnert]
```

**Messung:** "Needle in a Haystack"-Tests – eine spezifische Information wird an verschiedenen Stellen im Context versteckt, dann wird abgefragt.

| Modell | Context-Größe | "Needle in a Haystack" | Empfehlung für |
|--------|--------------|----------------------|----------------|
| Claude Sonnet/Opus 4.5 | 200K | Sehr zuverlässig bis ~150K | Codebase-Analyse, lange Reports |
| GPT-4o | 128K | Gut bis ~100K | Mittellange Dokumente |
| Gemini 1.5 Pro | 1M | Gut, leichte Degradation bei 500K+ | Sehr lange Dokumente, Bücher |
| Gemini 2.0 Pro | 2M | Noch in Erprobung | Experimentelle Nutzung |
| Llama 3.1 70B | 128K | Mittel (Context-Recall schwächer) | Wenn lokal nötig |

> [!IMPORTANT]
> Ein Context Window von 1M Tokens klingt beeindruckend, aber Qualität und Kosten sind entscheidend. **Für zuverlässige Analyse langer Dokumente ist Claude mit 200K Context oft besser als Gemini mit 1M**, weil die Recall-Qualität höher ist. Gemini 1.5 Pro ist sinnvoll, wenn das Dokument wirklich >200K Tokens hat.

**Praktische Grenzen für zuverlässige Nutzung:**

| Modell | Theoretisches Max | Zuverlässiges Praktik-Limit |
|--------|-----------------|----------------------------|
| Claude 4.5 | 200K | ~150K |
| GPT-4o | 128K | ~80K |
| Gemini 1.5 Pro | 1M | ~300-400K |
| Gemini 2.0 Pro | 2M | ~500K (experimentell) |

---

## Deutsch-Kenntnisse: Welche Modelle sind am besten?

### Warum ist Deutsch herausfordernd für LLMs?

Deutsch ist durch folgende Eigenschaften anspruchsvoller als Englisch:
- Komplexe Grammatik (4 Fälle, Genus-System, Konjunktivformen)
- Lange Komposita (Herausforderung für Tokenizer)
- Unterdurchschnittliche Repräsentation in Trainingsdaten (vs. Englisch)
- Umgangssprache vs. Hochdeutsch vs. regionale Varianten

### Bewertung nach Kategorien

**Allgemeine Textqualität auf Deutsch:**

| Modell | Grammatik | Stil | Nuancen | Gesamt |
|--------|-----------|------|---------|--------|
| Claude Sonnet/Opus 4.5 | Hervorragend | Sehr gut | Sehr gut | ★★★★★ |
| GPT-4o | Hervorragend | Sehr gut | Gut | ★★★★★ |
| Gemini 1.5 Pro | Sehr gut | Gut | Gut | ★★★★ |
| Mistral Large | Sehr gut | Gut | Gut | ★★★★ |
| Llama 3.1 70B | Gut | Gut | Mittel | ★★★ |
| Llama 3.1 8B | Mittel | Mittel | Begrenzt | ★★ |

**Deutsch-spezifische Stärken:**

```
Aufgabe: "Schreibe einen formellen Brief auf Deutsch, der eine Kündigung
des Mietvertrags nach deutschem Recht korrekt formuliert"

Claude Sonnet 4.5:
→ Korrekte Anrede, rechtlich präzise Formulierungen, korrekter Konjunktiv
→ Kennt deutschrechtliche Spezifika (Kündigungsfristen, Schriftform)

GPT-4o:
→ Ebenfalls sehr gut, manchmal etwas "amerikanischer" Stil

Llama 3.1 8B:
→ Grammatisch korrekt, aber weniger nuanciert, manchmal falsche Formalitätsstufe
```

**Empfehlungen für deutschsprachige Anwendungen:**

| Anwendungsfall | Empfehlung | Begründung |
|----------------|-----------|-----------|
| Professionelle Texte (Briefe, Reports) | Claude Sonnet 4.5 | Höchste Stilqualität |
| Kundensupport (Chatbot) | Claude Haiku 4.5 oder GPT-4o-mini | Schnell, günstig, gute Qualität |
| Rechtliche/behördliche Texte | Claude Sonnet 4.5 | Präzisere Formulierungen |
| Technische Dokumentation | Claude Sonnet 4.5 oder GPT-4o | Gleich stark |
| Lokale Verarbeitung, Datenschutz | Mistral Large (API) oder Mistral Nemo (lokal) | Beste europäische Alternative |
| Kreatives Schreiben | Claude Sonnet/Opus 4.5 | Stilistisch stärkster |
| Einfache Klassifizierung | Beliebiges Modell | Alle ausreichend |

> [!NOTE]
> **Mistral AI** (französisches Unternehmen) bietet einen **EU-Datenschutz-Vorteil**: Server in Europa, DSGVO-konformer Vertrag einfacher zu erhalten als bei US-Unternehmen. Für sensible deutsche/europäische Daten eine wichtige Alternative.

---

## Kostenvergleich (Stand: August 2025)

### Preise pro Million Tokens (USD)

| Modell | Anbieter | Input ($/M) | Output ($/M) | Cache Read ($/M) | Gesamt (50/50) |
|--------|---------|-------------|-------------|-----------------|---------------|
| Claude Opus 4.5 | Anthropic | $15,00 | $75,00 | $1,50 | $45,00 |
| Claude Sonnet 4.5 | Anthropic | $3,00 | $15,00 | $0,30 | $9,00 |
| Claude Haiku 4.5 | Anthropic | $0,25 | $1,25 | $0,03 | $0,75 |
| GPT-4o | OpenAI | $5,00 | $15,00 | $1,25 | $10,00 |
| GPT-4o-mini | OpenAI | $0,15 | $0,60 | $0,075 | $0,375 |
| o1 | OpenAI | $15,00 | $60,00 | - | $37,50 |
| o3 | OpenAI | $10,00 | $40,00 | - | $25,00 |
| o3-mini | OpenAI | $1,10 | $4,40 | $0,55 | $2,75 |
| Gemini 1.5 Pro | Google | $3,50 | $10,50 | $0,875 | $7,00 |
| Gemini 1.5 Flash | Google | $0,075 | $0,30 | $0,019 | $0,1875 |
| Gemini 2.0 Pro | Google | $3,50 | $10,50 | - | $7,00 |
| Gemini 2.0 Flash | Google | $0,10 | $0,40 | $0,025 | $0,25 |
| Mistral Large | Mistral | $3,00 | $9,00 | - | $6,00 |
| Mistral Small | Mistral | $0,20 | $0,60 | - | $0,40 |
| Mistral Nemo 12B | Mistral | $0,15 | $0,15 | - | $0,15 |
| Codestral | Mistral | $1,00 | $3,00 | - | $2,00 |

> [!NOTE]
> "Gesamt (50/50)" ist die theoretische Durchschnittskosten wenn 50% Input und 50% Output. In der Praxis überwiegen oft die Input-Tokens (besonders bei langen Dokumenten oder Gesprächsverläufen).

### Kostenvergleich für typische Anwendungsfälle

**Szenario 1: Einfacher Chatbot (1.000 Nachrichten/Tag)**
```
Annahmen: 300 Input-Tokens, 200 Output-Tokens pro Nachricht

Modell            | Kosten/Tag | Kosten/Monat
------------------|------------|-------------
Claude Haiku 4.5  | $0,33      | ~$10,00
GPT-4o-mini       | $0,21      | ~$6,30
Gemini 2.0 Flash  | $0,38      | ~$11,40
Claude Sonnet 4.5 | $3,90      | ~$117,00
GPT-4o            | $4,50      | ~$135,00
```

**Szenario 2: Dokumenten-Analyse (100 Reports/Tag, je 50K Tokens)**
```
Annahmen: 50.000 Input-Tokens, 1.000 Output-Tokens pro Analyse

Modell             | Kosten/Tag | Kosten/Monat
-------------------|------------|-------------
Gemini 1.5 Flash   | $0,405     | ~$12,15
Gemini 2.0 Flash   | $0,54      | ~$16,20
Claude Haiku 4.5   | $1,375     | ~$41,25
Claude Sonnet 4.5  | $16,50     | ~$495
Gemini 1.5 Pro     | $18,55     | ~$556,50
```

> [!TIP]
> Für tokenintensive Anwendungen (lange Dokumente) ist **Gemini 1.5 Flash** oder **Gemini 2.0 Flash** oft die günstigste Option – besonders durch den 1M Token Context und die sehr niedrigen Input-Preise.

---

## Empfehlungen nach Anwendungsfall

### Schnelle Entscheidungshilfe

```
Brauche ich das Beste, Kosten egal?
    → Claude Opus 4.5 (mit Extended Thinking für Reasoning)

Beste Balance Qualität/Kosten für Produktion?
    → Claude Sonnet 4.5

Günstigste Option mit guter Qualität?
    → Claude Haiku 4.5 oder GPT-4o-mini oder Gemini 2.0 Flash

Langer Context (>200K Tokens)?
    → Gemini 1.5 Pro oder Gemini 2.0 Pro

Komplexes Reasoning / Mathematik?
    → Claude Opus 4.5 (Thinking) oder o3

Datenschutz (Daten lokal halten)?
    → Llama 3.1 70B (lokal) oder Mistral Large (EU-Server)

Speziell Code?
    → Claude Sonnet 4.5 oder Codestral (Mistral)

Multimodal (Audio + Bild)?
    → GPT-4o

Günstigste Option für Massenverarbeitung?
    → Gemini 1.5/2.0 Flash

Open-Weight für fine-tuning?
    → Llama 3.1 70B oder Mistral Nemo 12B
```

### Detaillierte Empfehlungsmatrix

| Anwendungsfall | Primärempfehlung | Alternative | Kostengünstige Option |
|----------------|-----------------|-------------|----------------------|
| Allgemeiner Assistent | Claude Sonnet 4.5 | GPT-4o | Claude Haiku 4.5 |
| Code-Assistent (IDE) | Claude Sonnet 4.5 | GPT-4o | Codestral |
| Code-Assistent (lokal) | Qwen2.5-Coder 32B | Llama 3.1 8B | Phi-3.5 Mini |
| Dokumenten-Analyse (kurz, <50K) | Claude Sonnet 4.5 | GPT-4o | Claude Haiku 4.5 |
| Dokumenten-Analyse (lang, >200K) | Gemini 1.5 Pro | Gemini 2.0 Pro | Gemini 1.5 Flash |
| Kundenservice-Chatbot | Claude Haiku 4.5 | GPT-4o-mini | Gemini 2.0 Flash |
| Komplexes Reasoning | Claude Opus 4.5 (Thinking) | o3 | o3-mini |
| Übersetzung (DE/EN) | Claude Sonnet 4.5 | GPT-4o | DeepL + LLM-Nachbearbeitung |
| Zusammenfassungen | Claude Haiku 4.5 | GPT-4o-mini | Gemini 1.5 Flash |
| RAG-System | Claude Sonnet 4.5 | GPT-4o | Claude Haiku 4.5 |
| Datenextraktion (JSON) | Claude Sonnet 4.5 | GPT-4o | Claude Haiku 4.5 |
| Lokale, sensible Daten | Llama 3.1 70B | Mistral Large (EU) | Llama 3.1 8B |
| Kreatives Schreiben | Claude Opus 4.5 | GPT-4o | Claude Sonnet 4.5 |
| Bildanalyse | GPT-4o | Claude Sonnet 4.5 | Gemini 2.0 Flash |
| Multimodale Anwendung | GPT-4o | Gemini 2.0 Pro | Gemini 2.0 Flash |
| Agentic Workflows | Claude Sonnet 4.5 | GPT-4o | Claude Haiku 4.5 |

---

## Kritische Betrachtung von Benchmarks

> [!WARNING]
> Benchmarks sind kein zuverlässiger Ersatz für eigene Tests mit realen Daten. Häufige Probleme mit LLM-Benchmarks:

1. **Data Contamination:** Testdaten können im Trainingsset enthalten sein
2. **Benchmark-Overfitting:** Modelle werden explizit auf Benchmark-Aufgaben trainiert
3. **Kein realer Usecase:** MMLU-Fragen entsprechen oft nicht dem produktiven Einsatz
4. **Schnelle Veraltung:** Ein neues Modell überholt Benchmarks innerhalb von Wochen

**Besser als Benchmarks:**
- Eigene Evaluation-Sets aus realen Anfragen aufbauen
- A/B-Tests in Produktion
- Menschliche Bewertung auf eigenen Daten
- Automatisierte Evaluation mit einem starken Modell als Judge ("LLM as Judge")

```python
# Beispiel: LLM-as-Judge für automatisierte Evaluation
def evaluate_response(
    question: str,
    model_a_response: str,
    model_b_response: str,
    judge_model: str = "claude-opus-4-5"
) -> dict:
    """Nutzt ein starkes LLM als objektiven Bewerter."""

    prompt = f"""Bewerte diese zwei Antworten auf die Frage objektiv.

Frage: {question}

Antwort A: {model_a_response}

Antwort B: {model_b_response}

Bewerte nach: Korrektheit, Vollständigkeit, Klarheit, Prägnanz.
Antworte im JSON-Format: {{"winner": "A" oder "B" oder "Tie", "reasoning": "..."}}"""

    # Aufruf des Judge-Modells
    response = call_llm(judge_model, prompt)
    return parse_json(response)
```

---

## Verwandte Notizen

- [[Modell Übersicht]]
- [[Context Window und Token Management]]
- [[Lokale LLMs]]
- [[Prompting/Grundlagen]]
- [[Agents/Überblick]]
