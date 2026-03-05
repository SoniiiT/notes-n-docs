# Fine-Tuning Grundlagen

> [!NOTE]
> Fine-Tuning ist der Prozess, ein bereits vortrainiertes Sprachmodell auf einem spezifischen Datensatz weiter zu trainieren, um es für eine bestimmte Aufgabe oder Domäne zu spezialisieren.

## Was ist Fine-Tuning?

Ein Large Language Model (LLM) wie Llama 3, Mistral oder Qwen wird zunächst auf enormen Mengen an Text-Daten aus dem Internet trainiert (Pre-Training). Dieses Basismodell (Base Model) hat allgemeines Weltwissen, ist aber noch nicht auf spezifische Aufgaben ausgerichtet.

**Fine-Tuning** passt dieses vortrainierte Modell auf einem kleineren, spezialisierten Datensatz weiter an:

```
Pre-trained Base Model
        |
        v
Fine-Tuning auf Domain-Daten
        |
        v
Spezialisiertes Fine-tuned Model
```

Das Modell "lernt" dabei nicht von Grund auf neu – die im Pre-Training erworbenen Sprachfähigkeiten bleiben erhalten und werden durch das Fine-Tuning in eine gewünschte Richtung gelenkt.

---

## Wann ist Fine-Tuning sinnvoll vs. Prompt Engineering?

| Kriterium | Prompt Engineering | Fine-Tuning |
|---|---|---|
| Aufwand | Gering | Hoch (Daten, Compute) |
| Latenz | Höher (langer System-Prompt) | Niedriger |
| Kosten pro Anfrage | Höher (mehr Tokens) | Niedriger |
| Konsistenz | Variabel | Hoch |
| Domänen-Spezifität | Begrenzt | Sehr hoch |
| Eigene Daten einbringen | Schwer | Ideal |
| Stil/Ton anpassen | Eingeschränkt | Sehr gut |

> [!TIP]
> **Faustregel:** Beginne immer mit [[Prompt Engineering Techniken|Prompt Engineering]]. Fine-Tune nur dann, wenn:
> - Prompt Engineering die gewünschte Qualität nicht erreicht
> - Du ein spezifisches Format/Stil konsistent erzwingen willst
> - Die Inferenzkosten durch lange Prompts zu hoch werden
> - Du proprietäre/geheime Daten einbringen willst (kein API-Call zu Drittanbietern)
> - Du ein sehr spezifisches Fachgebiet abdeckst (z.B. medizinische Dokumentation, Rechtswesen)

**Fine-Tuning ist NICHT sinnvoll wenn:**
- Das Problem mit einem 2-3 Shot Prompt gelöst werden kann
- Nur wenige Beispiele (<100) vorhanden sind
- Das Modell aktuelles Wissen benötigt (besser: [[RAG Grundlagen|RAG]])

---

## Full Fine-Tuning vs. Parameter-Efficient Fine-Tuning (PEFT)

### Full Fine-Tuning

Alle Parameter des Modells werden während des Trainings aktualisiert. Bei einem 7B-Modell sind das ~7 Milliarden Parameter.

**Vorteile:**
- Maximale Anpassungsfähigkeit
- Beste mögliche Qualität

**Nachteile:**
- Extrem hoher VRAM-Bedarf (70B-Modell: >140 GB VRAM)
- Gefahr von Catastrophic Forgetting (Modell "vergisst" allgemeines Wissen)
- Sehr teuer und zeitaufwendig

### Parameter-Efficient Fine-Tuning (PEFT)

Nur ein kleiner Bruchteil der Parameter wird trainiert oder neue, kleine Adapter-Schichten werden hinzugefügt.

**Vorteile:**
- Dramatisch reduzierter VRAM-Bedarf
- Viel schneller
- Weniger Overfitting-Risiko
- Originale Modellgewichte bleiben unverändert (Adapter können ausgetauscht werden)

**Nachteile:**
- Minimal geringere maximale Qualität gegenüber Full Fine-Tuning

> [!IMPORTANT]
> In der Praxis ist PEFT (insbesondere LoRA/QLoRA) in fast allen Szenarien die richtige Wahl. Full Fine-Tuning ist nur für Organisationen mit dedizierter GPU-Infrastruktur und sehr großen Datensätzen relevant.

---

## PEFT-Methoden

### LoRA (Low-Rank Adaptation)

LoRA friert alle originalen Modellgewichte ein und fügt trainierbare Niedrig-Rang-Matrizen (sogenannte Adapter) zu bestimmten Layern hinzu. Nur diese Adapter-Gewichte werden trainiert.

- Trainiert typischerweise <1% der originalen Parameter
- Adapter können gespeichert und gewechselt werden
- Sehr gut erforscht und erprobt

Mehr Details: [[LoRA und QLoRA]]

### QLoRA (Quantized LoRA)

Kombination aus LoRA und 4-Bit Quantisierung des Basismodells. Das Basismodell wird in 4-Bit geladen (spart ~75% VRAM), LoRA-Adapter werden in 16-Bit trainiert.

- Ermöglicht Training großer Modelle auf Consumer-GPUs
- 70B-Modell auf einer 4×A100 (80GB) trainierbar

Mehr Details: [[LoRA und QLoRA]]

### Prefix Tuning

Statt Modellgewichte zu ändern, werden trainierbare "Präfix-Tokens" (virtuelle Tokens) an den Kontext angehängt, die das Modellverhalten steuern.

- Sehr wenige trainierbare Parameter
- Besonders gut für Aufgaben mit klaren Eingabe/Ausgabe-Strukturen
- Weniger populär als LoRA in der LLM-Community

### Adapter

Kleine Feed-Forward-Schichten werden zwischen den Transformer-Schichten eingefügt. Nur diese Adapter-Schichten werden trainiert.

- Ältere Methode, mittlerweile weitgehend von LoRA verdrängt
- Etwas höherer Inferenz-Overhead als LoRA (wegen zusätzlicher Schichten)

---

## Supervised Fine-Tuning (SFT) vs. RLHF vs. DPO

### Supervised Fine-Tuning (SFT)

Das Modell lernt direkt aus Beispiel-Paaren (Eingabe → gewünschte Ausgabe). Standardmäßige Methode für die meisten Fine-Tuning-Projekte.

```
Eingabe:  "Erkläre mir, was eine API ist."
Ausgabe:  "Eine API (Application Programming Interface) ist..."
```

Das Modell wird darauf trainiert, die gewünschte Ausgabe zu erzeugen (Token-Prediction Loss).

**Einsatz:** Aufgabenspezifische Anpassung, Stil-Anpassung, Domain-Wissen einbringen.

### RLHF (Reinforcement Learning from Human Feedback)

Dreistufiger Prozess:
1. SFT: Initiales Fine-Tuning auf Demonstrationen
2. Reward Model Training: Menschen bewerten Ausgaben, ein Reward-Modell lernt menschliche Präferenzen
3. RL-Training: Das LLM wird mit PPO optimiert, um maximale Rewards zu erzielen

**Einsatz:** Alignment, Sicherheit, Anpassung an menschliche Präferenzen (verwendet bei ChatGPT, Claude, etc.)

> [!WARNING]
> RLHF ist sehr komplex in der Implementierung, instabil beim Training und erfordert große Mengen an Human-Feedback-Daten. Für die meisten Anwendungsfälle ist RLHF überdimensioniert.

### DPO (Direct Preference Optimization)

Vereinfachte Alternative zu RLHF ohne separates Reward Model. Verwendet Paare von "bevorzugten" und "abgelehnten" Antworten direkt im Training.

```json
{
  "prompt": "Wie erkläre ich Kindern das Internet?",
  "chosen": "Das Internet ist wie ein riesiges Netz...",
  "rejected": "Das Internet ist ein globales Computernetzwerk basierend auf TCP/IP..."
}
```

**Vorteile gegenüber RLHF:**
- Einfacher zu implementieren
- Stabiler im Training
- Kein separates Reward Model nötig
- Ähnlich gute Ergebnisse

**Einsatz:** Alignment, wenn man Preference-Daten (chosen/rejected) hat.

---

## Hardware-Anforderungen (VRAM nach Modellgröße)

> [!NOTE]
> Die folgenden Werte sind Richtwerte für Training mit gemischter Precision (bfloat16). Tatsächlicher Bedarf hängt von Batch-Size, Sequenzlänge und Methode ab.

### Full Fine-Tuning (bfloat16)

| Modellgröße | Modellgewichte | Gradienten + Optimizer | Gesamt (ca.) |
|---|---|---|---|
| 1B | 2 GB | ~6 GB | ~8 GB |
| 3B | 6 GB | ~18 GB | ~24 GB |
| 7B | 14 GB | ~42 GB | ~56 GB |
| 13B | 26 GB | ~78 GB | ~104 GB |
| 70B | 140 GB | ~420 GB | ~560 GB |

### LoRA Fine-Tuning (bfloat16)

| Modellgröße | Gesamt VRAM (ca.) | Empfohlene GPU |
|---|---|---|
| 1B | ~4 GB | RTX 3060 12GB |
| 3B | ~8 GB | RTX 3080 10GB |
| 7B | ~16 GB | RTX 3090 / 4090 |
| 13B | ~28 GB | 2× RTX 3090 |
| 70B | ~140 GB | 4× A100 80GB |

### QLoRA Fine-Tuning (4-Bit Quantisierung)

| Modellgröße | Gesamt VRAM (ca.) | Empfohlene GPU |
|---|---|---|
| 1B | ~2 GB | RTX 3060 12GB |
| 3B | ~4 GB | RTX 3060 12GB |
| 7B | ~6–8 GB | RTX 3060 12GB |
| 13B | ~10–12 GB | RTX 3080 10GB |
| 70B | ~35–40 GB | A100 40GB |

> [!TIP]
> Für Consumer-Hardware (RTX 3090/4090 mit 24 GB VRAM) ist QLoRA auf 7B-Modellen oder LoRA auf 7B-Modellen ideal.

---

## Typischer Fine-Tuning Workflow

```
1. Zielsetzung definieren
   └── Was soll das Modell können?
   └── Welche Metrik messen den Erfolg?

2. Daten sammeln & aufbereiten
   └── Dataset in passendem Format (Alpaca, ShareGPT, JSONL)
   └── Qualitätskontrolle, Duplikate entfernen
   └── Train/Validation-Split (z.B. 95%/5%)

3. Basismodell auswählen
   └── Größe: Tradeoff zwischen Qualität und Ressourcen
   └── Instruction-tuned oder Base Model?

4. Training konfigurieren
   └── Methode: LoRA, QLoRA, Full FT
   └── Hyperparameter: lr, epochs, batch_size, r, alpha
   └── Framework: Unsloth, Axolotl, LLaMA-Factory

5. Training starten & überwachen
   └── Loss-Kurven beobachten
   └── Overfitting erkennen

6. Evaluation
   └── Validation Loss
   └── Manuelle Qualitätsprüfung
   └── Task-spezifische Benchmarks

7. Deployment
   └── Adapter mergen (optional)
   └── Quantisierung für Inferenz (GGUF, AWQ, GPTQ)
   └── Inferenz-Server: Ollama, vLLM, llama.cpp
```

---

## Overfitting erkennen und vermeiden

Overfitting tritt auf, wenn das Modell die Trainingsdaten auswendig lernt, statt zu generalisieren.

### Erkennung

```
Training Loss:   ↓↓↓↓↓↓ (sinkt kontinuierlich)
Validation Loss: ↓↓↑↑↑↑ (steigt wieder an = Overfitting!)
```

Bei Overfitting wird das Modell auf unbekannten Daten schlechter, obwohl der Trainingsverlust weiter sinkt.

### Ursachen und Gegenmaßnahmen

| Ursache | Lösung |
|---|---|
| Zu viele Trainings-Epochen | Früh stoppen (Early Stopping) |
| Zu wenige Trainingsdaten | Mehr Daten sammeln / Data Augmentation |
| Zu hohe Lernrate | Lernrate reduzieren, LR-Schedule verwenden |
| Dataset-Duplikate | Duplikate entfernen |
| Zu hoher LoRA-Rang | r reduzieren |

```python
# Early Stopping in Transformers Trainer
from transformers import TrainingArguments

args = TrainingArguments(
    output_dir="./output",
    num_train_epochs=3,
    eval_strategy="steps",
    eval_steps=100,
    save_strategy="steps",
    save_steps=100,
    load_best_model_at_end=True,   # Bestes Modell nach Training laden
    metric_for_best_model="eval_loss",
    greater_is_better=False,
)
```

> [!WARNING]
> Overfitting ist das häufigste Problem beim Fine-Tuning mit kleinen Datensätzen (<1000 Beispiele). Immer einen Validation-Split verwenden und Validation Loss überwachen.

---

## Evaluation: Wie misst man den Erfolg?

### Quantitative Metriken

**Perplexity / Loss**
- Misst, wie gut das Modell die Validation-Daten vorhersagt
- Niedrigerer Validation Loss = besser
- Gut für: Allgemeine Sprachqualität

**ROUGE Score** (für Textzusammenfassung)
- Misst Überlappung zwischen generierter und Referenz-Zusammenfassung
- ROUGE-1, ROUGE-2, ROUGE-L

**BLEU Score** (für Übersetzung)
- Misst N-Gramm-Überlappung mit Referenzübersetzung

**Accuracy / F1 Score** (für Klassifikation)
- Korrekte Antworten / Gesamtanfragen

### Qualitative Evaluation

> [!IMPORTANT]
> Automatische Metriken reichen selten aus. Manuelle Evaluation durch Menschen oder LLM-as-a-Judge (z.B. GPT-4 bewertet Antworten) ist essenziell.

**LLM-as-a-Judge Ansatz:**
```python
# Beispiel: GPT-4 als Evaluator
evaluation_prompt = """
Bewerte die folgende Antwort auf einer Skala von 1-10 bzgl.:
- Korrektheit
- Vollständigkeit
- Klarheit

Frage: {question}
Antwort: {answer}
Referenzantwort: {reference}

Gib eine Zahl von 1-10 und eine kurze Begründung.
"""
```

---

## Bekannte Benchmarks

| Benchmark | Bereich | Beschreibung |
|---|---|---|
| **MMLU** | Allgemeinwissen | 57 Wissensgebiete, Multiple-Choice |
| **HumanEval** | Code-Generierung | Python-Funktionen schreiben und testen |
| **GSM8K** | Mathematik | Grundschul-Matheaufgaben mit Rechenwegen |
| **HellaSwag** | Reasoning | Satz-Vervollständigung, gesunder Menschenverstand |
| **TruthfulQA** | Wahrheit | Misst, ob Modelle falsche Aussagen vermeiden |
| **ARC-Challenge** | Wissenschaft | Schwere Multiple-Choice-Fragen aus Schulbüchern |
| **MT-Bench** | Konversation | Multi-Turn Gespräche, LLM-as-Judge Bewertung |
| **AlpacaEval** | Instruction Following | Vergleich mit Referenzmodell (GPT-4) |

> [!NOTE]
> Für spezifische Fine-Tuning-Projekte sind eigene, aufgabenspezifische Evaluationen oft aussagekräftiger als allgemeine Benchmarks. MMLU und HumanEval sind gut für allgemeine Qualitätsvergleiche.

---

## Verwandte Notizen

- [[LoRA und QLoRA]]
- [[RLHF und Alignment]]
- [[Prompt Engineering Techniken]]
- [[RAG Grundlagen]]
- [[Datasets vorbereiten]]
- [[Unsloth Axolotl LLaMA-Factory]]

### Externe Ressourcen

- [Hugging Face](https://huggingface.co) — Modelle, Datasets und Trainings-Infrastruktur
