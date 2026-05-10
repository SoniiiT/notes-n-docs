---
tags: [ai, llm, modell-vergleich]
---

# Modell Vergleiche

> [!NOTE]
> **Stand: März 2026.** Dieser Vergleich basiert auf dem Stand **März 2026**. LLM-Benchmarks und -Preise ändern sich häufig. Die Empfehlungen hier sind praxisorientiert und nicht rein benchmark-basiert – da Benchmarks oft nicht die reale Nutzung abbilden.

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

## Claude vs. GPT-5: Detaillierter Vergleich

### Stärken von Claude (Anthropic)

**Instruction Following:**
Claude hält sich sehr präzise an komplexe, mehrstufige Anweisungen. Bei strukturierten Ausgaben (JSON-Schemas, spezifische Formate) ist Claude konsistenter.

```
Aufgabe: "Antworte ausschließlich in gültigem JSON ohne Markdown-Codeblöcke.
Schema: { 'name': string, 'score': number, 'reasons': string[] }"

Claude: Gibt konsistent valides JSON zurück, auch über viele Turns
GPT-5.2: Fügt manchmal trotzdem ```json-Blöcke hinzu oder weicht vom Schema ab
```

**Längere Dokumente und Context-Nutzung:**
Claude nutzt den 200K-Token-Context (1M in Beta) sehr zuverlässig. Die Performance nimmt auch bei fast vollem Context weniger stark ab.

**Schreiben und Nuancierung:**
Claude ist bekannt für präziseres, nuancierteres Schreiben. Weniger Neigung zu übertriebener Positivität ("Certainly! Great question!").

**Sicherheit und Ablehnung:**
Claude lehnt schädliche Anfragen ab, aber die Ablehnung ist präziser kalibriert – weniger False Positives bei legitimen Anfragen in Graubereichen.

### Stärken von GPT-5.2 (OpenAI)

**Multimodalität:**
GPT-5.2 verarbeitet Text, Bilder und Audio nativ in einem Modell. Besonders bei Audio-Verarbeitung und Echtzeit-Sprachinteraktion ist GPT-5.2 führend.

**Breites Ökosystem:**
OpenAI hat das größte Tooling-Ökosystem: Assistants API, Fine-Tuning, DALL-E-Integration, Code Interpreter, umfangreiche SDKs.

**Großes Context Window (400K):**
GPT-5.2 bietet mit 400K Tokens eines der größten Context Windows unter den proprietären Modellen.

**Web-Browsing (ChatGPT Plus):**
Bessere Integration von Echtzeit-Websuche in ChatGPT.

### Direkter Vergleich

| Kriterium | Claude Sonnet 4.6 | GPT-5.2 | Vorteil |
|-----------|------------------|--------|---------|
| Instruction Following | Hervorragend | Sehr gut | Claude |
| Code-Generierung | Hervorragend | Sehr gut | Claude |
| Reasoning | Sehr gut | Sehr gut | Gleich (ohne Extended Thinking) |
| Langer Context | Hervorragend (200K, 1M Beta) | Hervorragend (400K) | GPT-5.2 (Größe), Claude (Recall) |
| Multimodalität | Gut (Text+Bild) | Hervorragend (Text+Bild+Audio) | GPT-5.2 |
| Schreiben/Nuancierung | Hervorragend | Sehr gut | Claude |
| Deutsch-Qualität | Sehr gut | Sehr gut | Gleich |
| API-Ökosystem | Gut | Hervorragend | GPT-5.2 |
| Kosten (Input) | $3/M | $1,75/M | GPT-5.2 |
| Verfügbarkeit/Uptime | Sehr gut | Sehr gut | Gleich |
| JSON/Tool Reliability | Hervorragend | Sehr gut | Claude |

> [!TIP]
> **Praxisempfehlung:** Für die meisten Produktionsanwendungen (Agents, RAG, strukturierte Extraktion) bietet Claude Sonnet 4.6 das beste Preis-Leistungs-Verhältnis. GPT-5.2 ist preislich günstiger und bietet ein 400K Context Window. Für multimodale Anwendungen mit Audio ist GPT-5.2 besser.

---

## Reasoning-Modelle: Extended Thinking vs. o1/o3 vs. Gemini Thinking

### Was sind Reasoning-Modelle?

Reasoning-Modelle führen vor der Antwort einen sichtbaren oder versteckten Denkprozess durch. Dieser "Chain of Thought" verbessert die Leistung bei:
- Mathematik (von AMC bis Olympiade-Niveau)
- Logik-Puzzles
- Komplexem Code-Debugging
- Mehrstufigen Planungsaufgaben

**Kosten:** Reasoning-Modelle sind deutlich teurer als Standard-Modelle, da auch die Denk-Tokens berechnet werden.

### Claude Extended Thinking (Opus 4.6)

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
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

| Kriterium | Claude Opus 4.6 (Thinking) | o3 | GPT-5.3 Codex | Gemini 2.5 Pro (Thinking) |
|-----------|---------------------------|----|----|----------------------|
| Mathematik (AIME) | Sehr stark | Stark | Sehr stark | Stark |
| Code-Reasoning | Sehr stark | Sehr stark | State-of-the-Art | Gut |
| Denkprozess sichtbar | Ja | Nein | Nein | Ja |
| Reasoning-Budget einstellbar | Ja (budget_tokens) | Ja (reasoning_effort) | Ja (reasoning_effort) | Begrenzt |
| Kosten | Hoch ($5/$25 Input/Output) | Sehr hoch ($10/$40) | ~$1,75/$14 | Moderat (~$1,25/$5) |
| Geschwindigkeit | Mittel | Mittel | Mittel | Mittel |
| Context Window | 200K (1M Beta) | 200K | 400K | 1M |

> [!WARNING]
> Reasoning-Modelle sind für die meisten alltäglichen Aufgaben **überdimensioniert**. Nur einsetzen bei wirklich komplexen Problemen: Olympiade-Mathematik, komplexe Beweise, sehr schwieriges Code-Debugging. Für normale Programmierung ist Claude Sonnet 4.6 oder GPT-5.2 besser geeignet (schneller und günstiger).

---

## Code-Generierung: Welches Modell ist am besten?

### Benchmark: SWE-Bench Verified

SWE-Bench misst die Fähigkeit, echte GitHub-Issues zu lösen – ein praxisnahes Maß für Code-Qualität.

| Modell | SWE-Bench Score (ca.) | Stärke |
|--------|----------------------|--------|
| Claude Opus 4.6 | ~75%+ | Architektur, komplexe Refactorings |
| GPT-5.3 Codex | ~75% | Spezialisiert auf Code-Aufgaben |
| Claude Sonnet 4.6 | ~72% | Beste Balance Qualität/Kosten |
| o3 | ~72% | Logische Code-Probleme |
| Gemini 2.5 Pro | ~65% | Google-Stack-Integration |
| DeepSeek V3.2 | ~55% | Open-Weight, sehr günstig |
| Llama 4 Maverick | ~50% | Open-Weight, großer Context |

### Praktische Code-Fähigkeiten

**Code-Generierung (neuer Code):**
```
Aufgabe: "Schreibe eine Python-Klasse für einen LRU-Cache mit Thread-Sicherheit"

Top-Performer: Claude Sonnet 4.6, GPT-5.3 Codex, DeepSeek V3.2
→ Alle produzieren funktionierenden Code
→ Claude und GPT-5.3 tendieren zu besserem Docstring/Typ-Annotation-Stil
→ DeepSeek V3.2 oft knapperer, idiomatischerer Code bei extrem niedrigen Kosten
```

**Code-Debugging:**
```
Aufgabe: "Finde den Fehler in diesem 200-Zeilen Python-Script mit Off-by-One-Error"

Top-Performer: Claude Opus 4.6 (Extended Thinking), GPT-5.3 Codex
→ Extended Thinking hilft bei subtilen Bugs
→ Claude erklärt den Fehler besonders präzise
```

**Multi-File-Projekte:**
```
Aufgabe: "Verstehe diese 3.000-Zeilen-Codebase und füge eine neue Feature hinzu"

Top-Performer: Claude (großer Context, gutes File-Tracking), GPT-5.2/5.3 (400K Context)
→ Claude Sonnet/Opus 4.6 behalten den Überblick über große Codebases
→ GPT-5.2 kann mit 400K Context noch größere Codebases in einem Prompt verarbeiten
→ Llama 4 Scout bietet mit 10M Context theoretisch den größten Überblick
```

**Empfehlungen nach Code-Aufgabe:**

| Aufgabe | Empfohlenes Modell | Begründung |
|---------|-------------------|-----------|
| Neuen Code schreiben | Claude Sonnet 4.6 oder GPT-5.3 Codex | Beste Balance |
| Debugging (einfach) | Claude Haiku 4.5 oder DeepSeek V3.2 | Schnell und günstig |
| Debugging (komplex) | Claude Opus 4.6 mit Extended Thinking | Sichtbares Reasoning |
| Code-Review | Claude Sonnet 4.6 | Präzises Instruction Following |
| Speziell für Code | GPT-5.3 Codex | Code-spezialisiert |
| Lokal, Code-Focus | DeepSeek V3.2 oder Llama 4 Maverick | Beste Open-Weight-Optionen |
| Erklärungen schreiben | Claude Sonnet 4.6 | Besonders klare Erklärungen |

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
| Claude Sonnet/Opus 4.6 | 200K (1M Beta) | Sehr zuverlässig bis ~150K | Codebase-Analyse, lange Reports |
| GPT-5.2/5.3 | 400K | Sehr gut bis ~300K | Große Codebases, lange Dokumente |
| Gemini 2.5 Pro | 1M | Gut, leichte Degradation bei 500K+ | Sehr lange Dokumente, Bücher |
| Gemini 3.1 Pro | 1M | In Erprobung (Preview) | Experimentelle Nutzung |
| Llama 4 Scout | 10M | Noch in Erprobung | Extrem große Kontexte |
| Llama 4 Maverick | 1M | Gut bis ~500K | Wenn lokal/Open-Weight nötig |
| DeepSeek V3.2 | 128-164K | Mittel (Context-Recall schwächer) | Günstige Analyse mittlerer Dokumente |

> [!IMPORTANT]
> Ein Context Window von 10M Tokens (Llama 4 Scout) klingt beeindruckend, aber Qualität und Kosten sind entscheidend. **Für zuverlässige Analyse langer Dokumente ist Claude mit 200K Context oft besser als Modelle mit größerem Context**, weil die Recall-Qualität höher ist. GPT-5.2 bietet mit 400K einen guten Mittelweg. Gemini 2.5 Pro ist sinnvoll, wenn das Dokument wirklich >400K Tokens hat.

**Praktische Grenzen für zuverlässige Nutzung:**

| Modell | Theoretisches Max | Zuverlässiges Praktik-Limit |
|--------|-----------------|----------------------------|
| Claude 4.6 | 200K (1M Beta) | ~150K |
| GPT-5.2/5.3 | 400K | ~300K |
| Gemini 2.5 Pro | 1M | ~400-500K |
| Gemini 3.1 Pro | 1M | TBD (Preview) |
| Llama 4 Scout | 10M | ~1-2M (experimentell) |
| DeepSeek V3.2 | 128-164K | ~80-100K |
| Mistral Large 3 | 256K | ~150K |

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
| Claude Sonnet/Opus 4.6 | Hervorragend | Sehr gut | Sehr gut | ★★★★★ |
| GPT-5.2 | Hervorragend | Sehr gut | Gut | ★★★★★ |
| Gemini 2.5 Pro | Sehr gut | Gut | Gut | ★★★★ |
| Mistral Large 3 | Sehr gut | Gut | Gut | ★★★★ |
| Llama 4 Maverick | Gut | Gut | Gut | ★★★★ |
| DeepSeek V3.2 | Gut | Gut | Mittel | ★★★ |

**Deutsch-spezifische Stärken:**

```
Aufgabe: "Schreibe einen formellen Brief auf Deutsch, der eine Kündigung
des Mietvertrags nach deutschem Recht korrekt formuliert"

Claude Sonnet 4.6:
→ Korrekte Anrede, rechtlich präzise Formulierungen, korrekter Konjunktiv
→ Kennt deutschrechtliche Spezifika (Kündigungsfristen, Schriftform)

GPT-5.2:
→ Ebenfalls sehr gut, manchmal etwas "amerikanischer" Stil

DeepSeek V3.2:
→ Grammatisch korrekt, aber weniger nuanciert, manchmal falsche Formalitätsstufe
```

**Empfehlungen für deutschsprachige Anwendungen:**

| Anwendungsfall | Empfehlung | Begründung |
|----------------|-----------|-----------|
| Professionelle Texte (Briefe, Reports) | Claude Sonnet 4.6 | Höchste Stilqualität |
| Kundensupport (Chatbot) | Claude Haiku 4.5 oder DeepSeek V3.2 | Schnell, günstig, gute Qualität |
| Rechtliche/behördliche Texte | Claude Sonnet 4.6 | Präzisere Formulierungen |
| Technische Dokumentation | Claude Sonnet 4.6 oder GPT-5.2 | Gleich stark |
| Lokale Verarbeitung, Datenschutz | Mistral Large 3 (API, EU) oder Llama 4 (lokal) | Beste europäische Alternative |
| Kreatives Schreiben | Claude Sonnet/Opus 4.6 | Stilistisch stärkster |
| Einfache Klassifizierung | Beliebiges Modell | Alle ausreichend |

> [!NOTE]
> **Mistral AI** (französisches Unternehmen) bietet einen **EU-Datenschutz-Vorteil**: Server in Europa, DSGVO-konformer Vertrag einfacher zu erhalten als bei US-Unternehmen. Für sensible deutsche/europäische Daten eine wichtige Alternative. **Llama 4** (Meta) und **DeepSeek** (China) sind als Open-Weight-Modelle ebenfalls lokal betreibbar.

---

## Kostenvergleich (Stand: März 2026)

### Preise pro Million Tokens (USD)

| Modell | Anbieter | Input ($/M) | Output ($/M) | Cache Read ($/M) | Gesamt (50/50) |
|--------|---------|-------------|-------------|-----------------|---------------|
| Claude Opus 4.6 | Anthropic | $5,00 | $25,00 | $0,50 | $15,00 |
| Claude Sonnet 4.6 | Anthropic | $3,00 | $15,00 | $0,30 | $9,00 |
| Claude Haiku 4.5 | Anthropic | $0,80 | $4,00 | $0,08 | $2,40 |
| GPT-5.2 | OpenAI | $1,75 | $14,00 | - | $7,875 |
| GPT-5.3 Codex | OpenAI | ~$1,75 | ~$14,00 | - | ~$7,875 |
| o3 | OpenAI | $10,00 | $40,00 | - | $25,00 |
| Gemini 2.5 Pro | Google | $1,25 | $5,00 | $0,31 | $3,125 |
| Gemini 2.5 Flash | Google | $0,15 | $0,60 | $0,04 | $0,375 |
| Gemini 3.1 Pro | Google | TBD | TBD | - | TBD (Preview) |
| Mistral Large 3 | Mistral | $2,00 | $6,00 | - | $4,00 |
| DeepSeek V3.2 | DeepSeek | $0,25 | $0,38 | - | $0,315 |
| DeepSeek R1 | DeepSeek | $0,70 | $2,50 | - | $1,60 |
| Llama 4 Scout | Meta (Open) | Kostenlos* | Kostenlos* | - | Self-Host / Provider |
| Llama 4 Maverick | Meta (Open) | Kostenlos* | Kostenlos* | - | Self-Host / Provider |

\* Open-Weight-Modelle: Kostenlos zum Selbsthosten, oder über Provider wie Together AI, Fireworks AI etc. zu variablen Preisen.

> [!NOTE]
> "Gesamt (50/50)" ist die theoretische Durchschnittskosten wenn 50% Input und 50% Output. In der Praxis überwiegen oft die Input-Tokens (besonders bei langen Dokumenten oder Gesprächsverläufen).

### Kostenvergleich für typische Anwendungsfälle

**Szenario 1: Einfacher Chatbot (1.000 Nachrichten/Tag)**
```
Annahmen: 300 Input-Tokens, 200 Output-Tokens pro Nachricht

Modell            | Kosten/Tag | Kosten/Monat
------------------|------------|-------------
DeepSeek V3.2     | $0,15      | ~$4,50
Gemini 2.5 Flash  | $0,17      | ~$5,10
Claude Haiku 4.5  | $1,04      | ~$31,20
GPT-5.2           | $3,33      | ~$99,75
Claude Sonnet 4.6 | $3,90      | ~$117,00
```

**Szenario 2: Dokumenten-Analyse (100 Reports/Tag, je 50K Tokens)**
```
Annahmen: 50.000 Input-Tokens, 1.000 Output-Tokens pro Analyse

Modell             | Kosten/Tag | Kosten/Monat
-------------------|------------|-------------
DeepSeek V3.2      | $1,29      | ~$38,70
Gemini 2.5 Flash   | $0,81      | ~$24,30
Gemini 2.5 Pro     | $6,75      | ~$202,50
Claude Haiku 4.5   | $4,40      | ~$132,00
Claude Sonnet 4.6  | $16,50     | ~$495
GPT-5.2            | $10,15     | ~$304,50
```

> [!TIP]
> Für tokenintensive Anwendungen (lange Dokumente) ist **Gemini 2.5 Flash** oft die günstigste Option bei guter Qualität. **DeepSeek V3.2** ist das günstigste Major-Modell insgesamt. Für absolute Kostenkontrolle: Llama 4 Scout/Maverick selbst hosten.

---

## Empfehlungen nach Anwendungsfall

### Schnelle Entscheidungshilfe

```
Brauche ich das Beste, Kosten egal?
    → Claude Opus 4.6 (mit Extended Thinking für Reasoning)

Beste Balance Qualität/Kosten für Produktion?
    → Claude Sonnet 4.6

Günstigste Option mit guter Qualität?
    → DeepSeek V3.2, Gemini 2.5 Flash oder Claude Haiku 4.5

Langer Context (>200K Tokens)?
    → GPT-5.2 (400K), Gemini 2.5 Pro (1M) oder Llama 4 Scout (10M)

Komplexes Reasoning / Mathematik?
    → Claude Opus 4.6 (Thinking) oder o3

Datenschutz (Daten lokal halten)?
    → Llama 4 Maverick (lokal) oder Mistral Large 3 (EU-Server)

Speziell Code?
    → Claude Sonnet 4.6 oder GPT-5.3 Codex

Multimodal (Audio + Bild)?
    → GPT-5.2

Günstigste Option für Massenverarbeitung?
    → DeepSeek V3.2 oder Gemini 2.5 Flash

Open-Weight für fine-tuning?
    → Llama 4 Maverick oder DeepSeek V3.2
```

### Detaillierte Empfehlungsmatrix

| Anwendungsfall | Primärempfehlung | Alternative | Kostengünstige Option |
|----------------|-----------------|-------------|----------------------|
| Allgemeiner Assistent | Claude Sonnet 4.6 | GPT-5.2 | Claude Haiku 4.5 |
| Code-Assistent (IDE) | Claude Sonnet 4.6 | GPT-5.3 Codex | DeepSeek V3.2 |
| Code-Assistent (lokal) | Llama 4 Maverick | DeepSeek V3.2 | Llama 4 Scout |
| Dokumenten-Analyse (kurz, <50K) | Claude Sonnet 4.6 | GPT-5.2 | Claude Haiku 4.5 |
| Dokumenten-Analyse (lang, >200K) | Gemini 2.5 Pro | GPT-5.2 (400K) | Gemini 2.5 Flash |
| Kundenservice-Chatbot | Claude Haiku 4.5 | DeepSeek V3.2 | Gemini 2.5 Flash |
| Komplexes Reasoning | Claude Opus 4.6 (Thinking) | o3 | DeepSeek R1 |
| Übersetzung (DE/EN) | Claude Sonnet 4.6 | GPT-5.2 | DeepL + LLM-Nachbearbeitung |
| Zusammenfassungen | Claude Haiku 4.5 | DeepSeek V3.2 | Gemini 2.5 Flash |
| RAG-System | Claude Sonnet 4.6 | GPT-5.2 | Claude Haiku 4.5 |
| Datenextraktion (JSON) | Claude Sonnet 4.6 | GPT-5.2 | Claude Haiku 4.5 |
| Lokale, sensible Daten | Llama 4 Maverick | Mistral Large 3 (EU) | DeepSeek V3.2 (self-host) |
| Kreatives Schreiben | Claude Opus 4.6 | Claude Sonnet 4.6 | GPT-5.2 |
| Bildanalyse | GPT-5.2 | Claude Sonnet 4.6 | Gemini 2.5 Flash |
| Multimodale Anwendung | GPT-5.2 | Gemini 2.5 Pro | Gemini 2.5 Flash |
| Agentic Workflows | Claude Sonnet 4.6 | GPT-5.3 Codex | Claude Haiku 4.5 |

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
    judge_model: str = "claude-opus-4-6"
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
- [[Prompt Engineering Techniken]]
- [[Was ist ein AI Agent]]
