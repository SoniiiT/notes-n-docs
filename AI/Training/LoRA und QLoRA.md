# LoRA und QLoRA

> [!NOTE]
> LoRA (Low-Rank Adaptation) und QLoRA (Quantized LoRA) sind die mit Abstand am häufigsten verwendeten PEFT-Methoden für LLM Fine-Tuning. Sie ermöglichen das Training großer Modelle auf Consumer-Hardware bei minimalem Qualitätsverlust.

## Was ist LoRA (Low-Rank Adaptation)?

LoRA wurde 2021 von Hu et al. (Microsoft) vorgestellt und löst ein fundamentales Problem: Wie kann man ein großes vortrainiertes Modell effizient für eine neue Aufgabe anpassen, ohne alle Milliarden von Gewichten neu zu trainieren?

**Kernidee:** Die Gewichtsänderungen beim Fine-Tuning liegen in einem niedrig-dimensionalen Unterraum. Statt alle Gewichte einer Matrix `W` zu aktualisieren, wird die Änderung `ΔW` als Produkt zweier kleiner Matrizen dargestellt.

### Wie funktioniert LoRA technisch?

In einem Transformer-Modell gibt es große Gewichtsmatrizen (z.B. für Attention: `W_q`, `W_k`, `W_v`, `W_o`). Eine typische Matrix hat z.B. die Dimension `4096 × 4096` = ~16,8 Millionen Parameter.

**Standard Fine-Tuning:**
```
W_new = W_original + ΔW
```
Alle Parameter von `ΔW` (z.B. 16,8M) müssen trainiert werden.

**LoRA:**
```
W_new = W_original + (A × B)

Dabei:
- W_original: eingefroren (frozen), wird nicht trainiert
- A: Matrix der Dimension (d × r), zufällig initialisiert
- B: Matrix der Dimension (r × d), mit Nullen initialisiert
- r: Rang (rank), typischerweise 4–64 (viel kleiner als d)
```

Für eine `4096 × 4096` Matrix mit Rang `r=16`:
- Original: 16,8M Parameter
- LoRA A: `4096 × 16` = 65.536 Parameter
- LoRA B: `16 × 4096` = 65.536 Parameter
- Gesamt LoRA: 131.072 Parameter = **0,78% der Original-Matrix**

```
Originale Gewichtsmatrix W (eingefroren):
┌─────────────────────┐
│  4096 × 4096        │  ← wird NICHT trainiert
│  (16,8M Parameter)  │
└─────────────────────┘
           +
LoRA Adapter:
┌───────┐   ┌───────┐
│  A    │ × │  B    │  ← werden trainiert
│4096×16│   │16×4096│
│(65k)  │   │(65k)  │
└───────┘   └───────┘
```

**Warum B mit Nullen initialisieren?**
Damit das LoRA-Produkt `A × B` am Anfang des Trainings null ist – das Modell verhält sich zu Beginn exakt wie das originale Pre-trained Model.

> [!TIP]
> Nach dem Training können die LoRA-Gewichte (`A` und `B`) mit den originalen Gewichten gemergt werden: `W_merged = W + A × B`. Das ergibt ein vollständiges Modell ohne Inferenz-Overhead.

---

## Was ist QLoRA (Quantized LoRA)?

QLoRA wurde 2023 von Dettmers et al. vorgestellt und kombiniert LoRA mit 4-Bit Quantisierung des Basismodells.

**Prinzip:**
1. Das Basismodell wird in **4-Bit NF4** (NormalFloat4) Quantisierung geladen
2. Für den Vorwärtsdurchlauf werden Gewichte in **bfloat16** dequantisiert
3. Die LoRA-Adapter werden in **bfloat16** trainiert
4. Gradienten werden durch das quantisierte Modell hindurch berechnet

```
QLoRA Speicheraufteilung:
┌─────────────────────────────────────────┐
│ Basismodell (4-Bit NF4)                 │ ← ~25% des bfloat16-Speichers
│ 7B Modell: ~3,5 GB statt ~14 GB        │
└─────────────────────────────────────────┘
           +
┌─────────────────────────────────────────┐
│ LoRA Adapter (bfloat16)                 │ ← sehr klein (~100–500 MB)
└─────────────────────────────────────────┘
           +
┌─────────────────────────────────────────┐
│ Gradienten + Optimizer-States (bfloat16)│ ← nur für Adapter-Gewichte
└─────────────────────────────────────────┘
```

**NF4 (NormalFloat4):** Ein neuer 4-Bit Datentyp, der speziell für normalverteilte Gewichte von Pre-trained Modellen optimiert ist. Besser als standard INT4 für diese Anwendung.

**Double Quantization:** QLoRA quantisiert zusätzlich die Quantisierungs-Konstanten selbst (von float32 auf float8), was weitere ~0,4 Bit pro Parameter spart.

---

## Unterschied LoRA vs. QLoRA

| Merkmal | LoRA | QLoRA |
|---|---|---|
| Basismodell Precision | bfloat16 / float16 | 4-Bit NF4 |
| VRAM (7B Modell) | ~16 GB | ~6–8 GB |
| VRAM (13B Modell) | ~28 GB | ~10–12 GB |
| VRAM (70B Modell) | ~140 GB | ~35–40 GB |
| Trainingsgeschwindigkeit | Schneller | Etwas langsamer |
| Qualität | Sehr gut | Minimal geringer |
| Hauptvorteil | Schnell, geringer VRAM | Extreme VRAM-Einsparung |
| Consumer-GPU (24GB) | 7B–13B Modelle | 13B–34B Modelle |

> [!TIP]
> Für eine RTX 3090/4090 (24 GB VRAM): LoRA für 7B-Modelle, QLoRA für 13B–34B-Modelle. QLoRA macht es möglich, 70B-Modelle auf 2× A100 80GB zu trainieren (statt 4+).

---

## Wichtige LoRA-Parameter

### r (Rank) – Rang der Adapter-Matrizen

**Was es ist:** Der Rang `r` bestimmt die "Kapazität" des LoRA-Adapters. Je höher der Rang, desto mehr Parameter werden trainiert, desto mehr Anpassungspotenzial hat der Adapter.

**Typische Werte:**

| r | Trainierbare Parameter (7B) | Empfehlung |
|---|---|---|
| 4 | ~4M | Minimale Anpassung, sehr schnell |
| 8 | ~8M | Guter Startpunkt für einfache Aufgaben |
| 16 | ~16M | Empfohlen für die meisten Fälle |
| 32 | ~32M | Für komplexere Aufgaben |
| 64 | ~64M | Annäherung an Full Fine-Tuning |

> [!NOTE]
> Höherer Rang = mehr Kapazität, aber auch mehr Risiko für Overfitting bei kleinen Datasets. `r=16` ist ein solider Standard-Wert.

### lora_alpha – Skalierungsfaktor

**Was es ist:** `lora_alpha` skaliert die LoRA-Ausgabe: `output = (lora_alpha / r) × (A × B)`. Effektiv steuert es die "Lernrate" des Adapters.

**Faustregel:** `lora_alpha = 2 × r` oder `lora_alpha = r`

```python
# Häufig verwendete Konfigurationen:
# Konservativ (weniger Einfluss der Adapter):
r=16, lora_alpha=16   # Skalierungsfaktor: 1.0

# Standard (empfohlen):
r=16, lora_alpha=32   # Skalierungsfaktor: 2.0

# Aggressiver (stärkere Anpassung):
r=16, lora_alpha=64   # Skalierungsfaktor: 4.0
```

### lora_dropout – Regularisierung

**Was es ist:** Dropout-Rate für die LoRA-Schichten. Hilft bei Overfitting.

```python
lora_dropout=0.05   # Typischer Wert: 0.0–0.1
lora_dropout=0.0    # Für große Datasets (>10k Beispiele) oft besser
lora_dropout=0.1    # Für kleine Datasets (<1k Beispiele)
```

### target_modules – Welche Layer werden angepasst?

**Was es ist:** Gibt an, welche Gewichtsmatrizen im Transformer mit LoRA-Adaptern versehen werden.

```python
# Llama-Architektur (alle Attention- und MLP-Gewichte):
target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                  "gate_proj", "up_proj", "down_proj"]

# Nur Attention (weniger Parameter, schneller):
target_modules = ["q_proj", "k_proj", "v_proj", "o_proj"]

# Query und Value (Minimum, oft ausreichend):
target_modules = ["q_proj", "v_proj"]

# Mistral-Architektur:
target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                  "gate_proj", "up_proj", "down_proj"]
```

> [!TIP]
> Mehr `target_modules` = mehr trainierbare Parameter = höhere Kapazität, aber auch mehr VRAM. Für die meisten Fälle sind alle Attention-Module (`q_proj`, `k_proj`, `v_proj`, `o_proj`) ein guter Startpunkt.

---

## Wann LoRA, wann QLoRA?

| Szenario | Empfehlung |
|---|---|
| RTX 4090 (24 GB), 7B Modell | LoRA (ausreichend VRAM) |
| RTX 3090 (24 GB), 13B Modell | QLoRA (LoRA würde kaum reinpassen) |
| RTX 3060 (12 GB), 7B Modell | QLoRA |
| A100 40GB, 34B Modell | QLoRA |
| 4× A100 80GB, 70B Modell | LoRA möglich, QLoRA empfohlen |
| Qualität ist oberstes Gebot | LoRA (bfloat16 ist präziser) |
| VRAM ist limitierend | QLoRA |

---

## VRAM-Vergleich: Full FT vs. LoRA vs. QLoRA

Werte bei Batch-Size 1, Sequence Length 2048:

| Modell | Full FT | LoRA (bf16) | QLoRA (4-Bit) |
|---|---|---|---|
| 1B | ~16 GB | ~4 GB | ~2 GB |
| 3B | ~48 GB | ~10 GB | ~5 GB |
| 7B | ~112 GB | ~18 GB | ~8 GB |
| 13B | ~208 GB | ~32 GB | ~14 GB |
| 34B | ~544 GB | ~80 GB | ~36 GB |
| 70B | ~1,1 TB | ~160 GB | ~48 GB |

> [!NOTE]
> Diese Werte sind Näherungswerte. Tatsächlicher VRAM-Bedarf hängt von Batch-Size, Sequence Length, Gradient Checkpointing, und Framework-spezifischen Optimierungen ab.

**Gradient Checkpointing** kann den VRAM-Bedarf für Aktivierungen deutlich reduzieren (auf Kosten von ~20–30% Trainingsgeschwindigkeit):

```python
model.gradient_checkpointing_enable()
```

---

## LoRA Adapter mergen

Nach dem Training existieren zwei separate Teile: das Basismodell und der LoRA-Adapter. Das Mergen kombiniert beide zu einem einzelnen Modell.

**Wann mergen?**
- Für einfachere Deployment ohne PEFT-Bibliothek
- Für Quantisierung (GGUF, AWQ) des fine-getuned Modells
- Für maximale Inferenzgeschwindigkeit

**Wann NICHT mergen?**
- Wenn mehrere Adapter für verschiedene Aufgaben gewechselt werden sollen
- Wenn das Basismodell für verschiedene Projekte geteilt wird

```python
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

base_model_name = "meta-llama/Llama-3.2-3B-Instruct"
adapter_path = "./output/checkpoint-final"
output_path = "./merged_model"

# Basismodell laden
print("Lade Basismodell...")
model = AutoModelForCausalLM.from_pretrained(
    base_model_name,
    torch_dtype=torch.bfloat16,
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(base_model_name)

# LoRA Adapter laden
print("Lade LoRA Adapter...")
model = PeftModel.from_pretrained(model, adapter_path)

# Mergen und speichern
print("Merge Adapter in Basismodell...")
model = model.merge_and_unload()

print(f"Speichere gemergtes Modell nach: {output_path}")
model.save_pretrained(output_path)
tokenizer.save_pretrained(output_path)
print("Fertig!")
```

---

## Praktisches Beispiel mit PEFT-Bibliothek

Ein vollständiges Fine-Tuning-Beispiel mit LoRA und der PEFT-Bibliothek von Hugging Face:

```python
import torch
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    Trainer,
    DataCollatorForSeq2Seq,
)
from peft import (
    LoraConfig,
    get_peft_model,
    TaskType,
    prepare_model_for_kbit_training,
)
from datasets import load_dataset

# ─── Konfiguration ────────────────────────────────────────────────────────────

MODEL_NAME = "meta-llama/Llama-3.2-3B-Instruct"
OUTPUT_DIR = "./lora_output"
DATASET_PATH = "./train.jsonl"
MAX_SEQ_LENGTH = 2048

# ─── Modell und Tokenizer laden ───────────────────────────────────────────────

print("Lade Modell...")
model = AutoModelForCausalLM.from_pretrained(
    MODEL_NAME,
    torch_dtype=torch.bfloat16,     # bfloat16 für VRAM-Effizienz
    device_map="auto",               # GPU automatisch zuweisen
    trust_remote_code=True,
)
model.config.use_cache = False       # Für Training deaktivieren

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
tokenizer.pad_token = tokenizer.eos_token
tokenizer.padding_side = "right"

# ─── LoRA Konfiguration ───────────────────────────────────────────────────────

lora_config = LoraConfig(
    r=16,                            # Rang der Adapter-Matrizen
    lora_alpha=32,                   # Skalierungsfaktor (2 × r empfohlen)
    lora_dropout=0.05,               # Dropout für Regularisierung
    target_modules=[                 # Welche Layer werden angepasst?
        "q_proj", "k_proj",
        "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",
    ],
    bias="none",                     # Bias nicht trainieren
    task_type=TaskType.CAUSAL_LM,    # Aufgabentyp: Causal Language Modeling
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Ausgabe: trainable params: 39,976,960 || all params: 3,252,793,344 || trainable%: 1.23

# ─── Dataset vorbereiten ──────────────────────────────────────────────────────

dataset = load_dataset("json", data_files={"train": DATASET_PATH})

def format_and_tokenize(example):
    """Alpaca-Format zu Chat-Template formatieren und tokenisieren."""
    if example.get("input"):
        text = (
            f"<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\n"
            f"{example['instruction']}\n\n{example['input']}"
            f"<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n"
            f"{example['output']}<|eot_id|>"
        )
    else:
        text = (
            f"<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\n"
            f"{example['instruction']}"
            f"<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n"
            f"{example['output']}<|eot_id|>"
        )

    tokenized = tokenizer(
        text,
        truncation=True,
        max_length=MAX_SEQ_LENGTH,
        padding=False,
    )
    tokenized["labels"] = tokenized["input_ids"].copy()
    return tokenized

tokenized_dataset = dataset["train"].map(
    format_and_tokenize,
    remove_columns=dataset["train"].column_names,
)

# ─── Training Arguments ───────────────────────────────────────────────────────

training_args = TrainingArguments(
    output_dir=OUTPUT_DIR,
    num_train_epochs=3,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,   # Effektive Batch-Size: 2 × 4 = 8
    learning_rate=2e-4,
    lr_scheduler_type="cosine",
    warmup_ratio=0.03,
    weight_decay=0.001,
    bf16=True,                        # bfloat16 Training
    logging_steps=10,
    save_steps=100,
    save_total_limit=3,               # Nur die 3 besten Checkpoints behalten
    evaluation_strategy="no",         # Für Demo ohne Validation
    report_to="none",                 # Kein Wandb/TensorBoard
    gradient_checkpointing=True,      # VRAM sparen
)

# ─── Trainer erstellen und Training starten ───────────────────────────────────

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset,
    data_collator=DataCollatorForSeq2Seq(
        tokenizer,
        model=model,
        padding=True,
        pad_to_multiple_of=8,
    ),
)

print("Starte Training...")
trainer.train()

# ─── Adapter speichern ────────────────────────────────────────────────────────

print(f"Speichere LoRA Adapter nach: {OUTPUT_DIR}/final_adapter")
model.save_pretrained(f"{OUTPUT_DIR}/final_adapter")
tokenizer.save_pretrained(f"{OUTPUT_DIR}/final_adapter")
print("Training abgeschlossen!")
```

### QLoRA Variante (nur die Unterschiede)

```python
from transformers import BitsAndBytesConfig
from peft import prepare_model_for_kbit_training

# 4-Bit Quantisierungs-Konfiguration
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,                          # 4-Bit laden
    bnb_4bit_quant_type="nf4",                  # NormalFloat4
    bnb_4bit_compute_dtype=torch.bfloat16,      # Compute in bfloat16
    bnb_4bit_use_double_quant=True,             # Double Quantization
)

# Modell mit Quantisierung laden
model = AutoModelForCausalLM.from_pretrained(
    MODEL_NAME,
    quantization_config=bnb_config,             # QLoRA-spezifisch
    device_map="auto",
    trust_remote_code=True,
)

# Für kbit-Training vorbereiten (wichtig!)
model = prepare_model_for_kbit_training(model)
model.config.use_cache = False

# Ab hier gleich wie LoRA: lora_config, get_peft_model(), Training...
```

> [!WARNING]
> `prepare_model_for_kbit_training()` ist bei QLoRA zwingend erforderlich. Es aktiviert Gradient Checkpointing korrekt für quantisierte Modelle und stellt sicher, dass die Input-Embeddings Gradienten erhalten.

---

## Inference mit trainiertem LoRA Adapter

```python
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.2-3B-Instruct",
    torch_dtype=torch.bfloat16,
    device_map="auto"
)
model = PeftModel.from_pretrained(model, "./lora_output/final_adapter")
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.2-3B-Instruct")

# Inference
prompt = "<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\nWas ist LoRA?<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n"
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model.generate(
        **inputs,
        max_new_tokens=512,
        temperature=0.7,
        do_sample=True,
        top_p=0.9,
    )

response = tokenizer.decode(outputs[0][inputs["input_ids"].shape[1]:],
                             skip_special_tokens=True)
print(response)
```

---

## Verwandte Notizen

- [[Fine-Tuning Grundlagen]]
- [[Datasets vorbereiten]]
- [[Unsloth Axolotl LLaMA-Factory]]
