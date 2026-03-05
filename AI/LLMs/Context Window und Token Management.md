# Context Window und Token Management

> **Stand: März 2026**

> [!NOTE]
> Das Verstehen von Tokens und Context Windows ist grundlegend für die effiziente und kostengünstige Nutzung von LLMs. Falsch dimensionierte Prompts können die Kosten verfünffachen und die Qualität verschlechtern.

---

## Was sind Tokens?

Tokens sind die grundlegenden Einheiten, in die ein LLM Text aufteilt. Das Modell sieht weder einzelne Buchstaben noch komplette Wörter – sondern Tokens, die von einem **Tokenizer** erzeugt werden.

### Wie Tokenisierung funktioniert

Der am häufigsten verwendete Ansatz heißt **Byte Pair Encoding (BPE)**. Häufige Wörter und Silben werden zu einzelnen Tokens zusammengefasst, seltene Wörter werden in Teilstücke aufgeteilt.

```
Beispiel (GPT-4-Tokenizer):
"Hello" → [Hello]              = 1 Token
"Tokenisierung" → [Token][is][ierung] = 3 Tokens
"ChatGPT" → [Chat][GPT]       = 2 Tokens
" " (Leerzeichen) → oft Teil des nächsten Tokens
```

### Daumenregeln

| Sprache | Faustregel |
|---------|-----------|
| Englisch | ~4 Zeichen = 1 Token, ~0,75 Wörter = 1 Token |
| Deutsch | ~3,5-4,5 Zeichen = 1 Token (Komposita oft ungünstiger) |
| Code | Variiert stark; Einrückungen und Klammern kosten Tokens |
| CJK (Chinesisch, Japanisch, Koreanisch) | ~1-2 Zeichen = 1 Token |

> [!TIP]
> Als schnelle Schätzung: **1.000 Wörter ≈ 1.300 Tokens** auf Englisch. Auf Deutsch kann es durch lange Komposita (z.B. "Rindfleischetikettierungsüberwachungsaufgabenübertragungsgesetz") zu mehr Tokens pro Wort kommen.

### Token-Zählung in der Praxis

```python
# Mit der tiktoken-Bibliothek (für OpenAI-Modelle)
import tiktoken

encoder = tiktoken.encoding_for_model("gpt-4o")
text = "Dieses ist ein Beispieltext für die Token-Zählung."
tokens = encoder.encode(text)
print(f"Anzahl Tokens: {len(tokens)}")  # ca. 11-13 Tokens

# Für Claude-Modelle (Anthropic API)
import anthropic
client = anthropic.Anthropic()
response = client.messages.count_tokens(
    model="claude-sonnet-4-5",
    messages=[{"role": "user", "content": "Hallo, wie geht es dir?"}]
)
print(f"Input Tokens: {response.input_tokens}")
```

```javascript
// JavaScript mit tiktoken
import { encoding_for_model } from "tiktoken";

const enc = encoding_for_model("gpt-4o");
const tokens = enc.encode("Beispieltext für Token-Zählung");
console.log(`Token-Anzahl: ${tokens.length}`);
enc.free();
```

---

## Was ist das Context Window?

Das **Context Window** (auch: Kontextfenster, Context Length) ist die maximale Anzahl an Tokens, die ein Modell in einem einzigen Schritt verarbeiten kann. Es umfasst:

1. Den **System-Prompt** (Anweisungen, Persona, Kontext)
2. Den **Gesprächsverlauf** (alle bisherigen Nachrichten)
3. Die aktuelle **Nutzer-Anfrage**
4. Die generierte **Antwort** des Modells

```
┌─────────────────────────────────────────────────────────┐
│                    Context Window                        │
│  ┌──────────────┐  ┌─────────────────┐  ┌───────────┐  │
│  │ System Prompt│  │  Conversation   │  │  Output   │  │
│  │  (statisch)  │  │  History        │  │ (generiert│  │
│  └──────────────┘  └─────────────────┘  └───────────┘  │
│  ←──────────────── Input Tokens ──────────────────────→ │
│                                          ←Output Tokens→│
└─────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> Das Context Window ist ein **gleitendes Fenster** – es umfasst alles, was das Modell "sehen" kann. Ältere Teile des Gesprächs fallen heraus, sobald das Maximum erreicht ist (je nach Implementierung).

---

## Input-Tokens vs. Output-Tokens

Dies ist besonders wichtig für die **Kostenberechnung**, da die meisten Anbieter Input und Output unterschiedlich berechnen.

| Typ | Definition | Typischer Preis |
|-----|-----------|----------------|
| **Input Tokens** | Alles, was das Modell liest: System-Prompt, History, Nutzernachricht, Dokumente | Günstiger (z.B. $3/M bei Claude Sonnet 4.6) |
| **Output Tokens** | Das, was das Modell generiert (die Antwort) | Teurer (z.B. $15/M bei Claude Sonnet 4.6) |
| **Cache-Read Tokens** | Input-Tokens, die aus dem Prompt-Cache gelesen werden | Sehr günstig (z.B. $0,30/M bei Anthropic) |
| **Cache-Write Tokens** | Input-Tokens, die in den Cache geschrieben werden | Leicht teurer als normale Input-Tokens |

> [!WARNING]
> Output-Tokens sind typischerweise **3-5x teurer** als Input-Tokens. Sehr lange Antworten treiben die Kosten schnell in die Höhe. Es lohnt sich, Antwortlängen gezielt zu steuern.

---

## Context Window verschiedener Modelle

| Modell | Context Window | Max. Output Tokens | Besonderheit |
|--------|---------------|-------------------|-------------|
| Claude Opus 4.6 | 200.000 (1M Beta) | 32.000 | 1M Beta-Context verfügbar |
| Claude Sonnet 4.6 | 200.000 (1M Beta) | 16.000 | 1M Beta-Context verfügbar |
| Claude Haiku 4.5 | 200.000 | 8.192 | Schnell auch bei vollem Context |
| GPT-5.2 | 400.000 | 32.768 | Großer stabiler Context |
| GPT-5.3 | 400.000 | 32.768 | Neuestes OpenAI-Flaggschiff |
| o1 | 200.000 | 100.000 | Lange Reasoning-Chains |
| o3 | 200.000 | 100.000 | Erweiterte Reasoning-Ausgaben |
| Gemini 2.5 Pro | 1.000.000 | 8.192 | Stabiles 1M Context Window |
| Gemini 3.1 Pro | 1.000.000 | 8.192 | Neueste Gemini-Generation |
| Llama 4 Scout | 10.000.000 | ~4.096 | Branchenrekord: 10M Tokens! |
| Llama 4 Maverick | 1.000.000 | ~4.096 | 1M Context, Open Source |
| DeepSeek V3.2 | 128.000–164.000 | ~8.192 | Starker Open-Source-Konkurrent |
| Mistral Large 3 | 256.000 | ~4.096 | Erweiterter Context |
| Mistral Small 3.2 | 262.000 | ~4.096 | Kompaktes Modell, großer Context |

> [!NOTE]
> Ein großes Context Window bedeutet nicht automatisch gute Context-Nutzung. Das **"Lost in the Middle"**-Problem beschreibt, dass Modelle Informationen in der Mitte sehr langer Prompts schlechter abrufen als am Anfang oder Ende. Claude und Gemini sind hier besonders gut optimiert. Llama 4 Scout hält mit 10M Tokens den aktuellen Branchenrekord beim Context Window.

---

## Was passiert, wenn das Context Window voll ist?

### Szenario 1: Harter Fehler (API)

```json
{
  "error": {
    "type": "invalid_request_error",
    "message": "prompt is too long: 205,432 tokens > 200,000 maximum"
  }
}
```

Die API gibt einen Fehler zurück, wenn die Eingabe das Maximum überschreitet. Der Request schlägt fehl.

### Szenario 2: Automatisches Kürzen (Chat-Interfaces)

ChatGPT und ähnliche Interfaces kürzen automatisch alte Gesprächsteile ("Context Truncation"). Das Modell "vergisst" dann frühe Teile des Gesprächs.

**Symptome:**
- Das Modell wiederholt Fragen, die bereits beantwortet wurden
- Früherer Kontext wird nicht mehr berücksichtigt
- Die Qualität nimmt über lange Gespräche ab

### Best Practices bei vollem Context

1. **Neues Gespräch starten** – für komplett neue Themen
2. **Zusammenfassung erstellen** – das bisherige Gespräch zusammenfassen und als neuen Start verwenden
3. **Relevanten Context extrahieren** – nur die wirklich wichtigen Teile in den neuen Prompt nehmen
4. **[[RAG Grundlagen|RAG]] verwenden** – statt alles im Context zu halten, relevante Informationen zur Laufzeit abrufen

---

## Token-Effizienz: Prompts token-sparsam schreiben

### Was Tokens "frisst"

```markdown
❌ Tokenintensiv:
"Könntest du mir bitte erklären, wie man in Python eine Funktion schreibt,
die zwei Zahlen addiert? Es wäre super nett, wenn du auch ein Beispiel
geben könntest. Vielen Dank im Voraus!"

✅ Token-effizient:
"Python-Funktion: zwei Zahlen addieren. Mit Beispiel."
```

| Ineffizienz | Beispiel | Besser |
|-------------|---------|--------|
| Höflichkeitsfloskeln | "Bitte erkläre mir..." | "Erkläre:" |
| Redundante Einleitungen | "In dieser Antwort werde ich..." | Direkt zur Sache |
| Wiederholungen | Den gleichen Kontext mehrfach beschreiben | Einmal klar definieren |
| Leere Formatierung | Zu viele Leerzeilen | Kompakte Struktur |
| Verbose Anweisungen | Lange Erklärungen was das Modell tun soll | Kurze, präzise Anweisungen |

### System-Prompt Optimierung

```markdown
❌ Ineffizienter System-Prompt (ca. 150 Tokens):
Du bist ein hilfsbereiter Assistent. Deine Aufgabe ist es, dem Nutzer
bei allen seinen Fragen zu helfen. Du solltest immer freundlich und
zuvorkommend sein. Antworte auf Deutsch. Versuche, deine Antworten
klar und verständlich zu gestalten.

✅ Optimierter System-Prompt (ca. 25 Tokens):
Assistent. Antworte auf Deutsch. Klar und präzise.
```

> [!TIP]
> Bei vielen API-Calls mit demselben System-Prompt lohnt sich **Prompt Caching** (Anthropic) oder **Cached System Instructions** (OpenAI) erheblich. Bei täglichen Nutzungsmustern kann das 70-90% der System-Prompt-Kosten sparen.

---

## Prompt Caching (Anthropic-spezifisch)

### Was ist Prompt Caching?

Prompt Caching ist ein Feature der Anthropic API, das es erlaubt, bestimmte Teile eines Prompts zu cachen. Beim nächsten Aufruf mit demselben gecachten Prefix wird dieser Teil **nicht neu verarbeitet**, sondern aus dem Cache gelesen.

### Wie es funktioniert

```python
import anthropic

client = anthropic.Anthropic()

# System-Prompt mit Cache-Markierung
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "Du bist ein Experte für Python-Entwicklung.",
        },
        {
            "type": "text",
            "text": open("large_codebase_context.txt").read(),  # Großes Dokument
            "cache_control": {"type": "ephemeral"}  # Diese Stelle cachen
        }
    ],
    messages=[
        {"role": "user", "content": "Erkläre die Funktion `process_data` in Zeile 142."}
    ]
)

# Erster Aufruf: Cache wird geschrieben (cache_creation_input_tokens)
# Folgeaufrufe: Cache wird gelesen (cache_read_input_tokens) – ~90% günstiger
print(response.usage)
# Usage(cache_creation_input_tokens=8432, cache_read_input_tokens=0, input_tokens=12, output_tokens=234)
```

### Wann sich Caching lohnt

| Szenario | Cache-Kandidat | Ersparnis |
|----------|---------------|----------|
| Code-Review über mehrere Fragen | Gesamte Codebasis | 70-90% |
| Dokumenten-Analyse (Buch, Report) | Das Dokument | 70-90% |
| Multi-Turn-Chatbot | System-Prompt + Persona | 50-80% |
| Few-Shot-Prompting | Alle Beispiele | 60-80% |
| RAG mit großem Context | Relevante Chunks | 40-70% |

**Technische Details:**
- Cache wird für **5 Minuten** gespeichert (TTL: ephemeral)
- Minimum-Größe für Cache: **1.024 Tokens** (muss sich lohnen)
- Bis zu **4 Cache-Breakpoints** pro Request
- Cache-Write kostet 25% mehr als normaler Input
- Cache-Read kostet 90% weniger als normaler Input

> [!IMPORTANT]
> Prompt Caching funktioniert nur, wenn der gecachte Teil am **Anfang des Prompts** steht und bei jedem Aufruf **identisch** ist (byte-for-byte). Auch kleine Änderungen invalidieren den Cache.

---

## Kosten berechnen

### Grundformel

```
Kosten = (Input_Tokens / 1.000.000 × Preis_Input)
       + (Output_Tokens / 1.000.000 × Preis_Output)
```

### Preise (Stand März 2026, in USD pro Million Tokens)

| Modell | Input | Output | Cache Read | Cache Write |
|--------|-------|--------|-----------|------------|
| Claude Opus 4.6 | $15,00 | $75,00 | $1,50 | $18,75 |
| Claude Sonnet 4.6 | $3,00 | $15,00 | $0,30 | $3,75 |
| Claude Haiku 4.5 | $0,25 | $1,25 | $0,03 | $0,30 |
| GPT-5.2 | $5,00 | $15,00 | $1,25 | - |
| GPT-5.3 | $10,00 | $30,00 | $2,50 | - |
| o3 | $10,00 | $40,00 | - | - |
| Gemini 2.5 Pro | $3,50 | $10,50 | $0,875 | - |
| Gemini 3.1 Pro | $3,50 | $10,50 | $0,875 | - |
| DeepSeek V3.2 | $0,27 | $1,10 | $0,07 | - |

### Praktisches Rechenbeispiel

```python
# Kostenkalkulator
def calculate_llm_cost(
    input_tokens: int,
    output_tokens: int,
    input_price_per_million: float,
    output_price_per_million: float,
    cached_input_tokens: int = 0,
    cache_read_price_per_million: float = 0
) -> dict:

    regular_input_tokens = input_tokens - cached_input_tokens

    input_cost = (regular_input_tokens / 1_000_000) * input_price_per_million
    cache_cost = (cached_input_tokens / 1_000_000) * cache_read_price_per_million
    output_cost = (output_tokens / 1_000_000) * output_price_per_million

    total = input_cost + cache_cost + output_cost

    return {
        "input_cost": round(input_cost, 6),
        "cache_cost": round(cache_cost, 6),
        "output_cost": round(output_cost, 6),
        "total_usd": round(total, 6)
    }

# Beispiel: 1000 Anfragen mit Claude Sonnet 4.6
# Je 500 Input-Tokens, 200 Output-Tokens
kosten = calculate_llm_cost(
    input_tokens=500 * 1000,      # 500.000 Input-Tokens
    output_tokens=200 * 1000,     # 200.000 Output-Tokens
    input_price_per_million=3.00,
    output_price_per_million=15.00
)
print(f"Kosten für 1.000 Anfragen: ${kosten['total_usd']:.4f}")
# Ausgabe: Kosten für 1.000 Anfragen: $4.5000
```

---

## Praktische Tipps für lange Dokumente

### Strategie 1: Chunking

Teile große Dokumente in überschneidende Abschnitte auf:

```python
def chunk_document(text: str, chunk_size: int = 2000, overlap: int = 200) -> list[str]:
    """
    Teilt einen Text in Chunks mit Überlappung.
    chunk_size: Größe jedes Chunks in Tokens (geschätzt)
    overlap: Überlappende Tokens zwischen Chunks
    """
    words = text.split()
    chunks = []
    # Grobe Schätzung: 1 Wort ≈ 1,3 Tokens
    words_per_chunk = int(chunk_size / 1.3)
    words_overlap = int(overlap / 1.3)

    start = 0
    while start < len(words):
        end = min(start + words_per_chunk, len(words))
        chunk = " ".join(words[start:end])
        chunks.append(chunk)
        start += words_per_chunk - words_overlap

    return chunks
```

### Strategie 2: Hierarchische Zusammenfassungen

```
Dokument (500 Seiten)
    → Kapitel-Zusammenfassungen (je ~500 Tokens)
        → Gesamtzusammenfassung (~2.000 Tokens)
            → LLM-Verarbeitung mit kompaktem Context
```

### Strategie 3: Map-Reduce-Pattern

```python
# Map: Jedes Chunk einzeln verarbeiten
chunk_summaries = []
for chunk in chunks:
    summary = llm.summarize(chunk)
    chunk_summaries.append(summary)

# Reduce: Alle Zusammenfassungen kombinieren
final_summary = llm.synthesize(chunk_summaries)
```

### Strategie 4: RAG (Retrieval-Augmented Generation)

Statt das gesamte Dokument in den Context zu laden, werden nur die **relevantesten Abschnitte** abgerufen:

```
Nutzer-Frage → Embedding → Vektorsuche → Top-K Chunks → LLM Context
```

Mehr dazu: [[RAG Grundlagen]]

---

## Kontext-Kompression Strategien

### 1. Zusammenfassungen im Gesprächsverlauf

```python
# Wenn Gesprächsverlauf > 80% des Context Windows:
if token_count(conversation_history) > 0.8 * max_context:
    # Ältere Nachrichten durch Zusammenfassung ersetzen
    older_messages = conversation_history[:-10]  # Letzte 10 behalten
    summary = llm.summarize(older_messages)

    conversation_history = [
        {"role": "system", "content": f"Bisheriger Gesprächsverlauf (zusammengefasst): {summary}"},
        *conversation_history[-10:]  # Letzte 10 Nachrichten vollständig
    ]
```

### 2. Selektives Context-Management

Nicht alle Informationen sind immer relevant. Metadaten-basiertes Filtern:

```python
# Nur relevante Gesprächsteile beibehalten
def filter_relevant_context(history: list, current_query: str) -> list:
    # Semantische Ähnlichkeit berechnen
    relevant = []
    for message in history:
        similarity = compute_similarity(message["content"], current_query)
        if similarity > 0.5:  # Schwellenwert
            relevant.append(message)
    return relevant
```

### 3. Strukturierte Kompression

```markdown
Statt vollständiger Nachrichten – strukturierte Zusammenfassung:

Original (ca. 300 Tokens):
"Nutzer fragte nach Python-Variablen, ich erklärte Datentypen int,
float, str, list, dict. Nutzer hatte Frage zu mutability. Ich erklärte
den Unterschied zwischen mutable (list, dict) und immutable (int, str, tuple)."

Komprimiert (ca. 25 Tokens):
[Thema: Python-Grundlagen: Datentypen, Mutability erklärt]
```

---

## Verwandte Notizen

- [[Modell Übersicht]]
- [[Lokale LLMs]]
- [[Modell Vergleiche]]
- [[RAG Grundlagen]]
- [[Prompt Engineering Techniken]]
