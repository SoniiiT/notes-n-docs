---
tags: [ai, training, datasets]
---

# Datasets vorbereiten

> [!IMPORTANT]
> "Garbage in, garbage out" gilt beim Fine-Tuning absolut. Ein kleineres, hochqualitatives Dataset schlägt ein großes, schmutziges Dataset fast immer. Die Datenaufbereitung ist oft wichtiger als die Wahl der Hyperparameter.

## Warum ist Datenqualität entscheidend?

Das Fine-tuned Modell kann nicht besser sein als seine Trainingsdaten. Probleme im Dataset führen direkt zu Problemen im Modell:

| Problem im Dataset | Auswirkung auf das Modell |
|---|---|
| Widersprüchliche Antworten | Inkonsistentes, unzuverlässiges Verhalten |
| Duplikate | Overfitting auf häufige Beispiele |
| Falsche Informationen | Modell generiert Falschinformationen |
| Schlechte Grammatik / Tippfehler | Modell übernimmt schlechten Stil |
| Falsches Format | Modell folgt Instruktionen nicht |
| Biased Daten | Modell verstärkt Bias |

---

## Dataset-Formate

### Alpaca-Format

Das Alpaca-Format wurde von Stanford popularisiert und ist für Instruction-Following-Training ausgelegt.

```json
{
  "instruction": "Übersetze den folgenden Satz ins Englische.",
  "input": "Das Wetter ist heute schön.",
  "output": "The weather is nice today."
}
```

```json
{
  "instruction": "Erkläre den Begriff 'Overfitting' in der Machine Learning.",
  "input": "",
  "output": "Overfitting bezeichnet das Phänomen, bei dem ein Modell die Trainingsdaten so gut auswendig lernt, dass es auf neuen, unbekannten Daten schlechte Leistungen zeigt. Das Modell hat sich zu stark an die Trainingsbeispiele angepasst und kann nicht mehr generalisieren."
}
```

**Felder:**
- `instruction`: Die Aufgabe / der Befehl (immer vorhanden)
- `input`: Optionaler Kontext oder Eingabe (kann leer sein)
- `output`: Die gewünschte Ausgabe des Modells

**Geeignet für:** Einfache Instruction-Following-Aufgaben, Übersetzung, Klassifikation, Zusammenfassung.

---

### ShareGPT-Format

Das ShareGPT-Format modelliert Mehrfach-Gespräche (Multi-Turn Conversations) und ist flexibler als Alpaca.

```json
{
  "conversations": [
    {
      "from": "human",
      "value": "Was ist der Unterschied zwischen RAM und ROM?"
    },
    {
      "from": "gpt",
      "value": "RAM (Random Access Memory) ist flüchtiger Speicher, der Daten nur solange speichert, wie das Gerät eingeschaltet ist. Er dient als Arbeitsspeicher für laufende Programme.\n\nROM (Read-Only Memory) ist dauerhafter Speicher, dessen Inhalt auch ohne Stromversorgung erhalten bleibt. Er wird typischerweise für Firmware und Betriebssystem-Grundlagen verwendet."
    },
    {
      "from": "human",
      "value": "Welcher ist schneller?"
    },
    {
      "from": "gpt",
      "value": "RAM ist deutlich schneller als ROM. Typische RAM-Zugriffszeiten liegen im Nanosekundenbereich (z.B. 10-100 ns für DDR4), während ROM (z.B. Flash-Speicher) Zugriffszeiten im Mikrosekundenbereich hat und auch beim Schreiben langsamer ist."
    }
  ]
}
```

> [!NOTE]
> Das Feld `from` kann je nach Framework variieren: `human`/`gpt`, `user`/`assistant`, oder `human`/`assistant`. Am verbreitetsten ist `human`/`gpt` im ShareGPT-Format.

**Geeignet für:** Chatbots, Multi-Turn Dialoge, Konversationsagenten.

---

### JSONL-Format

JSONL (JSON Lines) speichert jeden Datensatz als separate JSON-Zeile in einer Datei. Dies ist das effizienteste Format für große Datasets, da es zeilenweise gelesen werden kann.

```jsonl
{"instruction": "Schreibe eine Python-Funktion, die eine Liste sortiert.", "input": "", "output": "def sort_list(lst):\n    return sorted(lst)\n\n# Beispiel:\nnumbers = [3, 1, 4, 1, 5, 9]\nprint(sort_list(numbers))  # [1, 1, 3, 4, 5, 9]"}
{"instruction": "Was ist die Hauptstadt von Frankreich?", "input": "", "output": "Die Hauptstadt von Frankreich ist Paris."}
{"instruction": "Fasse den folgenden Text zusammen.", "input": "Machine Learning ist ein Teilbereich der künstlichen Intelligenz...", "output": "Machine Learning ist ein KI-Teilbereich, bei dem Systeme aus Daten lernen."}
```

**Datei-Endung:** `.jsonl`

**Einlesen mit Python:**
```python
import json

def load_jsonl(filepath):
    data = []
    with open(filepath, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                data.append(json.loads(line))
    return data

dataset = load_jsonl("training_data.jsonl")
print(f"Anzahl Beispiele: {len(dataset)}")
```

---

## Datenquellen

### Eigene Daten

Die wertvollsten Daten für domänenspezifisches Fine-Tuning:

- **Kundensupport-Logs:** Häufige Fragen und gute Antworten extrahieren
- **Interne Dokumentation:** FAQ-Paare generieren (auch LLM-assistiert)
- **Experten-Annotationen:** Manuell erstellte Beispiele von Fachexperten
- **Bestehende Software-Outputs:** Logs, generierte Texte mit Qualitätsfilterung

> [!TIP]
> LLM-assistierte Datengenerierung: Nutze GPT-4 oder Claude, um aus bestehenden Dokumenten Instruction-Output-Paare zu generieren. Anschließend manuell reviewen. Dies skaliert die Datenerstellung erheblich.

```python
# Beispiel: LLM-assistierte Datengenerierung
import anthropic

client = anthropic.Anthropic()

def generate_training_examples(document_text, num_examples=5):
    prompt = f"""
Analysiere den folgenden Text und erstelle {num_examples} Trainingsbeispiele
im Alpaca-Format (instruction, input, output) als JSON-Array.
Die Beispiele sollen verschiedene Aspekte des Textes abdecken.

Text:
{document_text}

Gib nur das JSON-Array zurück, ohne zusätzliche Erklärungen.
"""
    message = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text
```

### Öffentliche Datasets (Hugging Face)

```python
from datasets import load_dataset

# Bekannte Instruction-Datasets
dataset = load_dataset("tatsu-lab/alpaca")           # Stanford Alpaca (52k)
dataset = load_dataset("Open-Orca/OpenOrca")         # OpenOrca (1M+)
dataset = load_dataset("teknium/OpenHermes-2.5")     # OpenHermes (1M)
dataset = load_dataset("HuggingFaceH4/ultrachat_200k") # UltraChat (200k)
dataset = load_dataset("WizardLM/WizardLM_evol_instruct_V2_196k") # WizardLM

# Dataset erkunden
print(dataset["train"][0])
print(f"Trainingsdaten: {len(dataset['train'])} Beispiele")
```

**Empfehlenswerte öffentliche Datasets:**

| Dataset | Größe | Beschreibung |
|---|---|---|
| tatsu-lab/alpaca | 52k | Original Stanford Alpaca |
| teknium/OpenHermes-2.5 | 1M | Hochqualitative Instruction-Daten |
| Open-Orca/OpenOrca | 1M+ | GPT-4 annotierte Daten |
| HuggingFaceH4/ultrachat_200k | 200k | Multi-Turn Chats |
| codeparrot/github-code | 115B tokens | Code-Training |
| iamtarun/python_code_instructions_18k_alpaca | 18k | Python-Code Alpaca |

---

## Datenqualität prüfen

```python
import pandas as pd
from datasets import load_dataset

# Dataset laden
dataset = load_dataset("json", data_files="training_data.jsonl")
df = pd.DataFrame(dataset["train"])

print("=== Dataset Übersicht ===")
print(f"Anzahl Beispiele: {len(df)}")
print(f"Spalten: {df.columns.tolist()}")
print(f"\nFehlende Werte:\n{df.isnull().sum()}")

# Duplikate prüfen
print(f"\nDuplikate (instruction): {df['instruction'].duplicated().sum()}")
print(f"Duplikate (output): {df['output'].duplicated().sum()}")

# Längenverteilung analysieren
df["instruction_len"] = df["instruction"].str.len()
df["output_len"] = df["output"].str.len()

print(f"\n=== Längenstatistiken ===")
print(df[["instruction_len", "output_len"]].describe())

# Sehr kurze oder sehr lange Outputs identifizieren
short_outputs = df[df["output_len"] < 10]
long_outputs = df[df["output_len"] > 2000]
print(f"\nSehr kurze Outputs (<10 Zeichen): {len(short_outputs)}")
print(f"Sehr lange Outputs (>2000 Zeichen): {len(long_outputs)}")
```

---

## Data Cleaning: Gängige Probleme und Lösungen

### Problem 1: Duplikate entfernen

```python
import pandas as pd

df = pd.read_json("training_data.jsonl", lines=True)

# Exakte Duplikate entfernen
before = len(df)
df = df.drop_duplicates(subset=["instruction", "output"])
print(f"Entfernt: {before - len(df)} exakte Duplikate")

# Nahezu gleiche Instructions (Fuzzy Deduplication)
# Empfehlung: datasketch MinHash für große Datasets
from datasketch import MinHash, MinHashLSH

def get_minhash(text, num_perm=128):
    m = MinHash(num_perm=num_perm)
    for word in text.lower().split():
        m.update(word.encode('utf8'))
    return m
```

### Problem 2: Leere oder zu kurze Outputs filtern

```python
def is_valid_example(row):
    # Mindestlänge für Instruction und Output
    if len(row.get("instruction", "")) < 5:
        return False
    if len(row.get("output", "")) < 10:
        return False
    # Prüfe auf Platzhalter-Texte
    placeholder_patterns = ["TODO", "FIXME", "lorem ipsum", "[PLACEHOLDER]"]
    output = row.get("output", "").lower()
    if any(p.lower() in output for p in placeholder_patterns):
        return False
    return True

df = df[df.apply(is_valid_example, axis=1)]
print(f"Nach Qualitätsfilterung: {len(df)} Beispiele")
```

### Problem 3: Zeichensatz-Probleme und Encoding

```python
import ftfy  # pip install ftfy

def fix_text_encoding(text):
    """Behebt häufige Unicode-Encoding-Fehler."""
    if not isinstance(text, str):
        return ""
    return ftfy.fix_text(text)

df["instruction"] = df["instruction"].apply(fix_text_encoding)
df["output"] = df["output"].apply(fix_text_encoding)
```

### Problem 4: Formatierungsprobleme

```python
import re

def clean_output(text):
    # Mehrfache Leerzeilen auf eine reduzieren
    text = re.sub(r'\n{3,}', '\n\n', text)
    # Führende/nachfolgende Whitespace entfernen
    text = text.strip()
    # Überflüssige Spaces vor Satzzeichen entfernen
    text = re.sub(r'\s+([.,!?;:])', r'\1', text)
    return text

df["output"] = df["output"].apply(clean_output)
```

### Problem 5: PII (Persönliche Daten) entfernen

```python
import re

def remove_pii(text):
    # E-Mail-Adressen
    text = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
                  '[EMAIL]', text)
    # Telefonnummern (deutsches Format)
    text = re.sub(r'\b(\+49|0)[0-9\s\-\/]{7,15}\b', '[TELEFON]', text)
    # IP-Adressen
    text = re.sub(r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b', '[IP]', text)
    return text

df["instruction"] = df["instruction"].apply(remove_pii)
df["output"] = df["output"].apply(remove_pii)
```

---

## Dataset-Größe: Wieviele Beispiele braucht man?

> [!NOTE]
> Es gibt keine universelle Antwort, aber die folgende Tabelle gibt Orientierungswerte:

| Anwendungsfall | Minimale Beispiele | Empfohlen | Bemerkung |
|---|---|---|---|
| Stil-Anpassung | 100–200 | 500+ | Einfachster Fall |
| Einfaches Task-Tuning | 500–1.000 | 2.000+ | Z.B. spezifisches Ausgabeformat |
| Domain-Adaption | 1.000–5.000 | 10.000+ | Fachvokabular, Domain-Wissen |
| Allgemeines Instruction Tuning | 10.000+ | 50.000+ | Breite Fähigkeiten |
| Vollständige Assistant-Tuning | 100.000+ | 500.000+ | ChatGPT-ähnliche Systeme |

> [!TIP]
> **Qualität vor Quantität:** 1.000 sorgfältig kuratierte Beispiele schlagen oft 10.000 automatisch generierte. Beginne mit einem kleinen, hochwertigen Dataset und skaliere dann.

**Bekanntes Beispiel:** Das ursprüngliche Stanford Alpaca Modell wurde mit nur 52.000 Beispielen auf Llama-7B fine-getuned und zeigte beeindruckende Instruction-Following-Fähigkeiten.

---

## Train/Validation Split

```python
from sklearn.model_selection import train_test_split
import json

# Dataset laden
with open("training_data.jsonl", "r") as f:
    data = [json.loads(line) for line in f if line.strip()]

# 95% Training, 5% Validation
train_data, val_data = train_test_split(
    data,
    test_size=0.05,
    random_state=42  # Reproduzierbarkeit
)

print(f"Training:   {len(train_data)} Beispiele")
print(f"Validation: {len(val_data)} Beispiele")

# Speichern
def save_jsonl(data, filepath):
    with open(filepath, "w", encoding="utf-8") as f:
        for item in data:
            f.write(json.dumps(item, ensure_ascii=False) + "\n")

save_jsonl(train_data, "train.jsonl")
save_jsonl(val_data, "validation.jsonl")
```

> [!WARNING]
> Bei sehr kleinen Datasets (<500 Beispiele) kann ein 5%-Split zu einem zu kleinen Validation-Set führen. In diesem Fall mindestens 50–100 Validation-Beispiele verwenden (auch wenn das mehr als 5% sind).

---

## Tokenisierung und maximale Sequenzlänge

Die maximale Sequenzlänge (`max_seq_length`) ist ein kritischer Parameter. Zu kurz: Daten werden abgeschnitten. Zu lang: Mehr VRAM-Bedarf, langsameres Training.

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.2-3B-Instruct")

def analyze_token_lengths(dataset, tokenizer, text_field="output"):
    lengths = []
    for example in dataset:
        text = example.get(text_field, "")
        tokens = tokenizer.encode(text)
        lengths.append(len(tokens))
    return lengths

import numpy as np

train_lengths = analyze_token_lengths(train_data, tokenizer, "output")
print(f"Median Output-Länge:    {np.median(train_lengths):.0f} Tokens")
print(f"95. Perzentile:         {np.percentile(train_lengths, 95):.0f} Tokens")
print(f"99. Perzentile:         {np.percentile(train_lengths, 99):.0f} Tokens")
print(f"Maximale Länge:         {max(train_lengths)} Tokens")

# Empfehlung: max_seq_length auf das 95. Perzentil setzen
# Beispiele, die länger sind, werden abgeschnitten (nur ~5%)
recommended_max_len = int(np.percentile(train_lengths, 95))
print(f"\nEmpfohlene max_seq_length: {recommended_max_len}")
```

**Typische Werte für max_seq_length:**

| Anwendungsfall | Empfohlene max_seq_length |
|---|---|
| Kurze Q&A, Klassifikation | 512–1024 |
| Standard Instruction-Following | 2048 |
| Code-Generierung | 2048–4096 |
| Lange Dokumente / RAG | 4096–8192 |

> [!WARNING]
> VRAM-Bedarf steigt quadratisch mit der Sequenzlänge (Attention-Mechanismus). Eine Verdopplung der Sequenzlänge verdoppelt nicht nur den Speicher – der Attention-Overhead wächst quadratisch. Bei Verwendung von Flash Attention 2 skaliert der VRAM-Bedarf nur linear mit der Sequenzlänge.

---

## Tools

### datasets-Bibliothek (Hugging Face)

```python
from datasets import Dataset, DatasetDict
import pandas as pd

# Von Pandas DataFrame
df = pd.read_json("training_data.jsonl", lines=True)
dataset = Dataset.from_pandas(df)

# Dataset explorieren
print(dataset)
print(dataset[0])

# Spalten umbenennen (z.B. für bestimmte Trainer)
dataset = dataset.rename_column("instruction", "prompt")

# Daten transformieren
def format_alpaca(example):
    if example["input"]:
        prompt = f"### Instruction:\n{example['instruction']}\n\n### Input:\n{example['input']}\n\n### Response:\n"
    else:
        prompt = f"### Instruction:\n{example['instruction']}\n\n### Response:\n"
    return {"text": prompt + example["output"]}

dataset = dataset.map(format_alpaca, remove_columns=dataset.column_names)

# Als JSONL speichern
dataset.to_json("formatted_dataset.jsonl", orient="records", lines=True)

# Als Parquet speichern (schneller für große Datasets)
dataset.save_to_disk("./dataset_dir")
```

### pandas für Datenanalyse

```python
import pandas as pd

df = pd.read_json("training_data.jsonl", lines=True)

# Häufigkeitsanalyse (z.B. Instruction-Längen)
df["instruction_tokens"] = df["instruction"].str.split().str.len()
df["output_tokens"] = df["output"].str.split().str.len()

# Verteilung visualisieren
import matplotlib.pyplot as plt

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
df["instruction_tokens"].hist(bins=50, ax=axes[0])
axes[0].set_title("Instruction-Länge (Wörter)")
df["output_tokens"].hist(bins=50, ax=axes[1])
axes[1].set_title("Output-Länge (Wörter)")
plt.tight_layout()
plt.savefig("dataset_analysis.png", dpi=150)
```

---

## Beispiel: Eigenes Dataset im ShareGPT-Format erstellen

Hier ein vollständiges Beispiel für die Erstellung eines deutschen Kundensupport-Datasets:

```python
import json
import random
from pathlib import Path

# Beispiel-Daten für ein deutsches Kundensupport-Dataset
training_examples = [
    {
        "conversations": [
            {
                "from": "human",
                "value": "Wie lange dauert die Lieferung?"
            },
            {
                "from": "gpt",
                "value": "Standardlieferungen dauern in der Regel 3–5 Werktage innerhalb Deutschlands. Express-Lieferung ist innerhalb von 1–2 Werktagen möglich und kostet 5,99 €. Bei Bestellungen über 50 € ist die Standardlieferung kostenlos."
            }
        ]
    },
    {
        "conversations": [
            {
                "from": "human",
                "value": "Ich möchte meine Bestellung stornieren."
            },
            {
                "from": "gpt",
                "value": "Kein Problem! Eine Stornierung ist möglich, solange die Bestellung noch nicht versandt wurde. Bitte teilen Sie mir Ihre Bestellnummer mit, dann prüfe ich den Status für Sie."
            },
            {
                "from": "human",
                "value": "Meine Bestellnummer ist 12345-DE."
            },
            {
                "from": "gpt",
                "value": "Ich habe Ihre Bestellung 12345-DE geprüft. Diese befindet sich noch im Lager und wurde noch nicht versandt. Ich storniere die Bestellung jetzt für Sie. Sie erhalten innerhalb von 1–3 Werktagen eine Bestätigungsmail, und die Rückerstattung erscheint innerhalb von 5–7 Werktagen auf Ihrem Konto."
            }
        ]
    },
    {
        "conversations": [
            {
                "from": "human",
                "value": "Das Produkt ist beschädigt angekommen. Was kann ich tun?"
            },
            {
                "from": "gpt",
                "value": "Das tut mir sehr leid! Bitte fotografieren Sie den Schaden und kontaktieren Sie uns innerhalb von 14 Tagen nach Erhalt. Wir organisieren dann kostenlos eine Abholung des beschädigten Produkts und senden Ihnen umgehend ein Ersatzprodukt zu – oder erstatten den vollen Kaufpreis, ganz nach Ihrer Wahl."
            }
        ]
    }
]

# Dataset aufteilen
random.seed(42)
random.shuffle(training_examples)

split_idx = int(len(training_examples) * 0.9)
train_data = training_examples[:split_idx]
val_data = training_examples[split_idx:]

# Speichern
output_dir = Path("./customer_support_dataset")
output_dir.mkdir(exist_ok=True)

def save_jsonl(data, filepath):
    with open(filepath, "w", encoding="utf-8") as f:
        for item in data:
            f.write(json.dumps(item, ensure_ascii=False) + "\n")

save_jsonl(train_data, output_dir / "train.jsonl")
save_jsonl(val_data, output_dir / "validation.jsonl")

print(f"Dataset gespeichert in: {output_dir}")
print(f"Training:   {len(train_data)} Konversationen")
print(f"Validation: {len(val_data)} Konversationen")

# Kompatibilitätsprüfung
print("\n=== Beispiel-Eintrag ===")
print(json.dumps(train_data[0], ensure_ascii=False, indent=2))
```

---

## Verwandte Notizen

- [[Fine-Tuning Grundlagen]]
- [[LoRA und QLoRA]]
- [[Unsloth Axolotl LLaMA-Factory]]
