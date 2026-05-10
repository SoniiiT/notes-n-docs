---
tags: [ai, llm, tokenization]
---

# Tokenization

> [!ABSTRACT]
> **Tokenization** ist der Prozess, bei dem Text in kleinere Einheiten (**Tokens**) zerlegt wird, die ein LLM verarbeiten kann. Ein Token ist typischerweise ein Sub-Wort — weder ein ganzes Wort noch ein einzelner Buchstabe. Die Wahl des Tokenizers beeinflusst direkt die Kosten (API-Pricing), die Qualität (mehrsprachig) und die Effizienz (Context Window) eines LLMs.

---

## Übersicht

| Aspekt | Details |
|---|---|
| **Was ist ein Token?** | Sub-Wort-Einheit (z.B. "Programm" + "ierung") |
| **Hauptalgorithmus** | Byte Pair Encoding (BPE) |
| **Implementierungen** | tiktoken (OpenAI), SentencePiece (Google), HuggingFace Tokenizers |
| **Typische Vokabulargröße** | 32K–128K Tokens |
| **Faustregel** | 1 Token ≈ ¾ englisches Wort ≈ ½ deutsches Wort |
| **Kosten-Relevanz** | API-Pricing ist pro Token (Input + Output) |

---

## Warum Sub-Wörter?

Es gibt drei Ansätze für Tokenization — Sub-Wörter sind der beste Kompromiss:

| Ansatz | Beispiel "Programmierung" | Vokabular | Problem |
|---|---|---|---|
| **Wort-Level** | ["Programmierung"] | Riesig (>500K) | Unbekannte Wörter → `[UNK]` |
| **Zeichen-Level** | ["P","r","o","g",...] | Klein (~256) | Sehr lange Sequenzen, kein Kontext |
| **Sub-Wort (BPE)** | ["Programm", "ierung"] | Mittel (~50K) | ✅ Bester Kompromiss |

---

## Byte Pair Encoding (BPE)

BPE ist der Standard-Algorithmus für moderne LLMs (GPT, Claude, Llama).

### Wie BPE funktioniert

BPE startet auf Zeichen-Level und mergt iterativ die häufigsten Paare:

```
Trainingstext: "hallo hallo welt hallo welt"

Schritt 1: Starte mit Zeichen
  → ["h","a","l","l","o"," ","h","a","l","l","o"," ","w","e","l","t",...]

Schritt 2: Finde häufigstes Paar → ("l","l") → Merge zu "ll"
  → ["h","a","ll","o"," ","h","a","ll","o"," ","w","e","l","t",...]

Schritt 3: Finde nächstes häufigstes Paar → ("h","a") → Merge zu "ha"
  → ["ha","ll","o"," ","ha","ll","o"," ","w","e","l","t",...]

Schritt 4: ("ha","ll") → "hall"
  → ["hall","o"," ","hall","o"," ","w","e","l","t",...]

Schritt 5: ("hall","o") → "hallo"
  → ["hallo"," ","hallo"," ","w","e","l","t",...]

... bis Vokabulargröße erreicht
```

> [!NOTE]
> BPE-Merges werden auf dem **Trainingskorpus** gelernt, nicht zur Laufzeit. Der resultierende Tokenizer wird dann für alle Eingaben verwendet.

### Ergebnis

Häufige Wörter werden zu einzelnen Tokens:
- "the" → `[1]` (1 Token)
- "hello" → `[15339]` (1 Token)

Seltene Wörter werden in Sub-Wörter zerlegt:
- "Kubernetes" → `["Kub", "ern", "etes"]` (3 Tokens)
- "Observability" → `["Observ", "ability"]` (2 Tokens)

---

## Tokenizer-Implementierungen

### tiktoken (OpenAI)

Wird von GPT-3.5, GPT-4, GPT-4o und Llama 3 verwendet.

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

# Text tokenisieren
tokens = enc.encode("Hallo, wie geht es dir?")
print(tokens)        # [39, 1236, 11, 37074, ...]
print(len(tokens))   # 7 Tokens

# Tokens zurück zu Text
text = enc.decode(tokens)
print(text)          # "Hallo, wie geht es dir?"
```

```bash
pip install tiktoken
```

### SentencePiece (Google)

Wird von T5, BERT, Gemma und vielen Open-Source-Modellen verwendet.

- **Sprachunabhängig** — braucht keine vorherige Wort-Segmentierung
- Unterstützt **BPE** und **Unigram**-Algorithmus
- Robuster für nicht-lateinische Schriften und Code

### HuggingFace Tokenizers

Einheitliche API für alle Tokenizer-Typen:

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3-8B")
tokens = tokenizer.encode("Hallo Welt!")
print(tokenizer.convert_ids_to_tokens(tokens))
# ['▁H', 'allo', '▁Welt', '!']
```

---

## Tokens und Kosten

API-Provider berechnen pro Token (Input + Output):

| Provider | Modell | Input | Output |
|---|---|---|---|
| OpenAI | GPT-4o | $2.50/1M Tokens | $10.00/1M Tokens |
| Anthropic | Claude Sonnet 4 | $3.00/1M Tokens | $15.00/1M Tokens |
| Google | Gemini 2.5 Pro | $1.25/1M Tokens | $10.00/1M Tokens |

### Sprache beeinflusst Kosten

Deutsche Texte verbrauchen **mehr Tokens** als englische, weil der Tokenizer primär auf englischen Daten trainiert wurde:

```
Englisch: "The cat sat on the mat"     → 6 Tokens
Deutsch:  "Die Katze saß auf der Matte" → 9 Tokens (+50%!)

Englisch: "programming"               → 1 Token
Deutsch:  "Programmierung"             → 2-3 Tokens
```

> [!WARNING]
> Nicht-lateinische Schriften (Chinesisch, Arabisch, Japanisch) können bis zu **3-4x** mehr Tokens verbrauchen als Englisch für den gleichen Inhalt.

---

## Tokens und Context Window

Das **[[Context Window und Token Management|Context Window]]** ist in Tokens gemessen, nicht in Wörtern:

| Modell | Context Window | ≈ Englische Wörter | ≈ Deutsche Wörter |
|---|---|---|---|
| GPT-3.5 | 16K Tokens | ~12.000 | ~8.000 |
| GPT-4o | 128K Tokens | ~96.000 | ~64.000 |
| Claude Sonnet 4 | 200K Tokens | ~150.000 | ~100.000 |

### Faustregel zur Schätzung

```
Englisch: 1 Token ≈ 4 Zeichen ≈ ¾ Wort
Deutsch:  1 Token ≈ 3 Zeichen ≈ ½ Wort
Code:     1 Token ≈ 3-4 Zeichen (stark variabel)
```

---

## Spezielle Tokens

Jeder Tokenizer hat **besondere Tokens** mit Steuerfunktionen:

| Token | Bedeutung | Beispiel |
|---|---|---|
| `<\|endoftext\|>` | Ende des Textes | Stoppt die Generierung |
| `<\|im_start\|>` | Beginn einer Nachricht | Chat-Format |
| `<\|im_end\|>` | Ende einer Nachricht | Chat-Format |
| `[PAD]` | Padding (Auffüllung) | Sequenzen gleich lang machen |
| `[UNK]` | Unbekanntes Token | Fallback (bei BPE selten) |

---

## Tokenization testen

### OpenAI Tokenizer (Web)

https://platform.openai.com/tokenizer — Interaktives Tool zum Testen

### Programmatisch

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

text = "Kubernetes Observability mit Prometheus und Grafana"
tokens = enc.encode(text)

print(f"Text: {text}")
print(f"Tokens: {len(tokens)}")
print(f"Token-IDs: {tokens}")

# Jedes Token einzeln anzeigen
for t in tokens:
    print(f"  {t} → '{enc.decode([t])}'")
```

---

## Best Practices

1. **Token-Verbrauch schätzen** — Vor API-Calls die Token-Anzahl prüfen, um Kosten zu kontrollieren
2. **Prompt-Effizienz** — Unnötigen Text vermeiden, auf Englisch prompten wenn möglich (weniger Tokens)
3. **Context Window beachten** — Input + Output zusammen dürfen das Limit nicht überschreiten
4. **Tokenizer zum Modell passend** — Immer den gleichen Tokenizer verwenden, mit dem das Modell trainiert wurde
5. **Multilingual testen** — Bei nicht-englischen Sprachen Token-Verbrauch explizit prüfen

---

## Verwandte Notizen

- [[Wie funktionieren LLMs]] — Transformer-Architektur und Grundlagen
- [[Context Window und Token Management]] — Token-Limits und Strategien
- [[Sampling-Parameter]] — Wie Token-Auswahl gesteuert wird
- [[LLM API Provider und Router]] — API-Pricing pro Token
- [[Lokale LLMs]] — Tokenizer bei lokalen Modellen

### Externe Ressourcen

- [OpenAI Tokenizer](https://platform.openai.com/tokenizer) — Interaktives Tokenization-Tool
- [Hugging Face](https://huggingface.co) — Tokenizer-Bibliothek und Modelle
