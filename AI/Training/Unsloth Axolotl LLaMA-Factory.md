---
tags: [ai, training, unsloth, axolotl, llama-factory, framework]
---

# Unsloth, Axolotl & LLaMA-Factory

> [!NOTE]
> Drei der wichtigsten Frameworks für praktisches LLM Fine-Tuning. Unsloth für maximale Geschwindigkeit, Axolotl für YAML-basierte Flexibilität, LLaMA-Factory für Benutzerfreundlichkeit.

---

# Unsloth

## Was ist Unsloth?

Unsloth ist ein Open-Source-Framework, das LLM Fine-Tuning durch handgeschriebene Triton-Kernel und mathematische Optimierungen um **2–5× schneller** macht und dabei **30–70% weniger VRAM** benötigt als Standard Hugging Face Transformers.

**Entwickelt von:** Daniel und Michael Han (seit 2023)
**GitHub:** https://github.com/unslothai/unsloth

**Hauptvorteile:**
- 2–5× schnelleres Training (keine Qualitätsverluste)
- 30–70% weniger VRAM durch optimierte Kernels
- Einfache Python-API, ähnlich zu Transformers
- Natives 4-Bit QLoRA-Support (bitsandbytes)
- Unterstützt Flash Attention 2 automatisch
- Kann Modelle in GGUF exportieren (direkt für Ollama nutzbar)

> [!TIP]
> Unsloth ist die erste Wahl für schnelles Fine-Tuning auf einer einzelnen GPU. Die API ist einfach zu erlernen und die Performance-Gewinne sind erheblich.

---

## Installation

### pip (empfohlen für Colab / bestehende Umgebungen)

```bash
# CUDA 12.1 (für neuere GPUs: RTX 30xx, 40xx, A100, H100)
pip install "unsloth[colab-new] @ git+https://github.com/unslothai/unsloth.git"

# CUDA 11.8 (ältere GPUs)
pip install "unsloth[cu118-torch240] @ git+https://github.com/unslothai/unsloth.git"

# Mit bitsandbytes (für QLoRA)
pip install bitsandbytes

# Abhängigkeiten
pip install xformers trl peft accelerate datasets
```

### conda (empfohlen für lokale Installationen)

```bash
# Neue conda-Umgebung erstellen
conda create --name unsloth_env python=3.11 -y
conda activate unsloth_env

# PyTorch mit CUDA installieren
conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia -y

# Unsloth installieren
pip install "unsloth[colab-new] @ git+https://github.com/unslothai/unsloth.git"
pip install bitsandbytes xformers trl peft accelerate datasets
```

### Installationscheck

```python
import unsloth
print(f"Unsloth Version: {unsloth.__version__}")

import torch
print(f"PyTorch: {torch.__version__}")
print(f"CUDA verfügbar: {torch.cuda.is_available()}")
print(f"GPU: {torch.cuda.get_device_name(0)}")
```

---

## Unterstützte Modelle

| Modell-Familie | Beispiele |
|---|---|
| Llama 3.x | Llama-3.2-1B/3B, Llama-3.1-8B/70B |
| Mistral | Mistral-7B-v0.3, Mistral-Nemo-12B |
| Phi | Phi-3-mini/medium, Phi-3.5-mini |
| Qwen | Qwen2.5-0.5B/1.5B/3B/7B/14B/32B |
| Gemma | Gemma-2B/7B, Gemma-2-2B/9B/27B |
| CodeLlama | CodeLlama-7B/13B/34B |
| TinyLlama | TinyLlama-1.1B |

> [!NOTE]
> Die vollständige und aktuelle Liste der unterstützten Modelle ist unter https://github.com/unslothai/unsloth zu finden. Neue Modelle werden regelmäßig hinzugefügt.

---

## Einfaches Fine-Tuning Beispiel mit Unsloth

```python
# ─── Imports ──────────────────────────────────────────────────────────────────

from unsloth import FastLanguageModel
from trl import SFTTrainer
from transformers import TrainingArguments
from datasets import load_dataset
import torch

# ─── Konfiguration ────────────────────────────────────────────────────────────

MODEL_NAME    = "unsloth/Llama-3.2-3B-Instruct"  # Unsloth-optimierte Versionen
MAX_SEQ_LEN   = 2048
OUTPUT_DIR    = "./unsloth_output"
DATASET_PATH  = "./train.jsonl"

# ─── Modell laden (4-Bit QLoRA) ───────────────────────────────────────────────

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name=MODEL_NAME,
    max_seq_length=MAX_SEQ_LEN,
    dtype=None,             # Auto-detect: bfloat16 für Ampere+, float16 für ältere
    load_in_4bit=True,      # QLoRA: 4-Bit Quantisierung
)

# ─── LoRA Adapter hinzufügen ──────────────────────────────────────────────────

model = FastLanguageModel.get_peft_model(
    model,
    r=16,                   # LoRA Rang
    lora_alpha=32,          # Skalierungsfaktor
    lora_dropout=0.05,
    target_modules=[
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",
    ],
    bias="none",
    use_gradient_checkpointing="unsloth",  # Unsloth's optimiertes Gradient Checkpointing
    random_state=42,
    use_rslora=False,       # Rank-Stabilized LoRA (optional, oft besser)
)

model.print_trainable_parameters()

# ─── Dataset laden und formatieren ───────────────────────────────────────────

dataset = load_dataset("json", data_files={"train": DATASET_PATH}, split="train")

# Alpaca-Prompt Template
alpaca_prompt = """Below is an instruction that describes a task. Write a response that appropriately completes the request.

### Instruction:
{}

### Input:
{}

### Response:
{}"""

EOS_TOKEN = tokenizer.eos_token

def format_prompts(examples):
    instructions = examples["instruction"]
    inputs       = examples["input"]
    outputs      = examples["output"]
    texts = []
    for instruction, inp, output in zip(instructions, inputs, outputs):
        text = alpaca_prompt.format(instruction, inp, output) + EOS_TOKEN
        texts.append(text)
    return {"text": texts}

dataset = dataset.map(format_prompts, batched=True)

# ─── Training Arguments ───────────────────────────────────────────────────────

training_args = TrainingArguments(
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,     # Effektive Batch-Size: 8
    warmup_steps=5,
    num_train_epochs=3,
    learning_rate=2e-4,
    fp16=not torch.cuda.is_bf16_supported(),
    bf16=torch.cuda.is_bf16_supported(),
    logging_steps=1,
    optim="adamw_8bit",                # 8-Bit AdamW (weniger VRAM)
    weight_decay=0.01,
    lr_scheduler_type="linear",
    seed=42,
    output_dir=OUTPUT_DIR,
    report_to="none",
)

# ─── Trainer ──────────────────────────────────────────────────────────────────

trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    dataset_text_field="text",
    max_seq_length=MAX_SEQ_LEN,
    dataset_num_proc=2,
    packing=False,                     # Packing: mehrere kurze Sequenzen zusammenfassen
    args=training_args,
)

# ─── Training starten ─────────────────────────────────────────────────────────

print("Starte Training mit Unsloth...")
trainer_stats = trainer.train()
print(f"Training abgeschlossen in {trainer_stats.metrics['train_runtime']:.1f} Sekunden")

# ─── Modell speichern ─────────────────────────────────────────────────────────

# Nur LoRA Adapter
model.save_pretrained(f"{OUTPUT_DIR}/lora_adapter")
tokenizer.save_pretrained(f"{OUTPUT_DIR}/lora_adapter")

# Als GGUF für Ollama/llama.cpp exportieren
model.save_pretrained_gguf(
    f"{OUTPUT_DIR}/gguf_model",
    tokenizer,
    quantization_method="q4_k_m",     # 4-Bit Quantisierung für GGUF
)
print(f"GGUF-Modell gespeichert: {OUTPUT_DIR}/gguf_model")
```

---

## Unsloth vs. Standard Transformers Vergleich

| Metrik | Standard Transformers | Unsloth | Verbesserung |
|---|---|---|---|
| Trainingsgeschwindigkeit (7B) | 1× | 2–3× | +100–200% |
| VRAM-Nutzung (7B QLoRA) | ~12 GB | ~7–8 GB | −30–40% |
| Trainingsgeschwindigkeit (70B) | 1× | 4–5× | +300–400% |
| VRAM-Nutzung (70B QLoRA) | ~48 GB | ~30 GB | −35% |
| Qualität des Outputs | Baseline | Identisch | 0% Verlust |
| Setup-Aufwand | Mittel | Gering | Einfacher |

> [!NOTE]
> Unsloth erzielt diese Speedups durch handgeschriebene Triton-Kernel für kritische Operationen (RoPE Embeddings, Cross-Entropy Loss, RMS Norm) und mathematische Tricks im Backpropagation. Es gibt **keinen Qualitätsverlust** im Vergleich zu Standard-Training.

---

# Axolotl

## Was ist Axolotl?

Axolotl ist ein konfigurations-basiertes Training Framework für LLMs, bei dem alle Trainingsparameter in einer einzigen **YAML-Datei** definiert werden. Es ist modular, gut dokumentiert und unterstützt viele Modelle und Trainingsmethoden.

**GitHub:** https://github.com/axolotl-ai-cloud/axolotl
**Besonderheiten:**
- Alles in einer YAML-Config-Datei
- Unterstützt Alpaca, ShareGPT, JSONL und viele weitere Formate direkt
- Integrierte Unterstützung für [[LoRA und QLoRA|LoRA, QLoRA]], Full FT, [[RLHF und Alignment|RLHF]], DPO
- Multi-GPU-Training ohne extra Konfiguration
- Gut für reproduzierbare Experimente (Config-Datei = vollständige Dokumentation)

---

## Installation

```bash
# System-Abhängigkeiten
pip install packaging ninja

# Axolotl installieren (CUDA 12.1, PyTorch 2.3)
pip install axolotl[flash-attn,deepspeed]

# Oder aus dem Repository (für neueste Version)
git clone https://github.com/axolotl-ai-cloud/axolotl
cd axolotl
pip install -e ".[flash-attn,deepspeed]"
```

---

## Grundlegende config.yaml Struktur

Eine vollständige Axolotl-Konfiguration für LoRA Fine-Tuning auf einem 7B-Modell:

```yaml
# ─── Modell ───────────────────────────────────────────────────────────────────

base_model: meta-llama/Llama-3.1-8B-Instruct
model_type: LlamaForCausalLM
tokenizer_type: AutoTokenizer

# ─── Quantisierung (QLoRA) ────────────────────────────────────────────────────

load_in_4bit: true               # QLoRA aktivieren
load_in_8bit: false

# ─── Adapter ──────────────────────────────────────────────────────────────────

adapter: lora                    # lora, qlora, oder leer für Full FT
lora_r: 16
lora_alpha: 32
lora_dropout: 0.05
lora_target_modules:
  - q_proj
  - k_proj
  - v_proj
  - o_proj
  - gate_proj
  - up_proj
  - down_proj
lora_modules_to_save:            # Diese Layer vollständig trainieren (z.B. lm_head)
  - embed_tokens
  - lm_head

# ─── Dataset ──────────────────────────────────────────────────────────────────

datasets:
  - path: ./train.jsonl          # Lokale Datei
    type: alpaca                 # Format: alpaca, sharegpt, completion, etc.
    # Für ShareGPT:
    # type: sharegpt
    # conversation: chatml       # Template: chatml, llama3, mistral, etc.

  # Mehrere Datasets kombinieren:
  # - path: teknium/OpenHermes-2.5
  #   type: sharegpt
  #   split: train[:10%]         # Nur 10% verwenden

dataset_prepared_path: ./prepared_dataset   # Cache für aufbereitete Daten

# ─── Sequenzlänge ─────────────────────────────────────────────────────────────

sequence_len: 2048
sample_packing: false            # Mehrere kurze Sequenzen zusammenpacken
pad_to_sequence_len: true

# ─── Output ───────────────────────────────────────────────────────────────────

output_dir: ./axolotl_output

# ─── Training Hyperparameter ──────────────────────────────────────────────────

num_epochs: 3
micro_batch_size: 2              # Batch-Size pro GPU
gradient_accumulation_steps: 4  # Effektive Batch-Size: 2 × 4 = 8
learning_rate: 0.0002
lr_scheduler: cosine
warmup_ratio: 0.03

optimizer: adamw_bnb_8bit        # 8-Bit AdamW (weniger VRAM)
weight_decay: 0.0
max_grad_norm: 1.0

# ─── Precision ────────────────────────────────────────────────────────────────

bf16: auto                       # Automatisch: bfloat16 wenn möglich, sonst float16
tf32: false

# ─── Speichern und Logging ────────────────────────────────────────────────────

logging_steps: 10
save_steps: 100
save_total_limit: 3
eval_steps: 100
evaluation_strategy: steps

# ─── Optimierungen ────────────────────────────────────────────────────────────

gradient_checkpointing: true
flash_attention: true            # Flash Attention 2 (schneller, weniger VRAM)
xformers_attention: false        # Alternative zu Flash Attention

# ─── Weights & Biases (optional) ──────────────────────────────────────────────

wandb_project: my_finetuning
wandb_run_id:                    # Auto-generiert wenn leer
# wandb_mode: disabled           # Auskommentieren um Wandb zu deaktivieren

# ─── Seed für Reproduzierbarkeit ──────────────────────────────────────────────

seed: 42
```

---

## Training starten

```bash
# Einfaches Training auf einer GPU
axolotl train config.yaml

# Multi-GPU Training (2 GPUs mit accelerate)
accelerate launch -m axolotl.cli.train config.yaml

# Mit torchrun (4 GPUs)
torchrun --nproc_per_node=4 -m axolotl.cli.train config.yaml

# Training debuggen (nur erste 10 Steps)
axolotl train config.yaml --debug

# Config validieren ohne Training
axolotl train config.yaml --validate
```

**Training überwachen:**
```bash
# VRAM-Nutzung live beobachten
watch -n 1 nvidia-smi

# Logs in Echtzeit
tail -f axolotl_output/trainer_log.jsonl | python -c "
import sys, json
for line in sys.stdin:
    data = json.loads(line)
    if 'loss' in data:
        print(f\"Step {data.get('step','?'):5d} | Loss: {data['loss']:.4f}\")
"
```

**Nach dem Training – Adapter mergen:**
```bash
# LoRA Adapter in Basismodell mergen
axolotl merge-lora config.yaml --lora-model-dir ./axolotl_output
```

---

## Wann Axolotl statt Unsloth?

| Szenario | Empfehlung |
|---|---|
| Schnelles Experiment auf einer GPU | Unsloth |
| Multi-GPU Training (2+ GPUs) | Axolotl |
| Reproduzierbare Experimente (YAML als Dokumentation) | Axolotl |
| DPO oder RLHF Training | Axolotl |
| Sehr einfaches Setup, Einsteiger | Unsloth |
| Kombination mehrerer Datasets mit Gewichtung | Axolotl |
| GGUF-Export direkt beim Training | Unsloth |
| Maximale Trainingsgeschwindigkeit (eine GPU) | Unsloth |

> [!TIP]
> Axolotl und Unsloth schließen sich nicht aus. Axolotl kann Unsloth als Backend verwenden: `unsloth_cross_entropy_loss: true` in der config.yaml aktivieren.

---

# LLaMA-Factory

## Was ist LLaMA-Factory?

LLaMA-Factory ist ein umfassendes Fine-Tuning-Framework mit einer **Web-Benutzeroberfläche (WebUI)** und CLI-Unterstützung. Es unterstützt über 100 Modelle, alle gängigen Training-Methoden und ist für seine Benutzerfreundlichkeit bekannt.

**GitHub:** https://github.com/hiyouga/LLaMA-Factory
**Besonderheiten:**
- Vollwertige WebUI im Browser (kein Python-Code nötig)
- Unterstützt 100+ Modelle
- Alle Methoden: SFT, DPO, PPO, ORPO, KTO
- Multi-GPU und Distributed Training
- Integriertes Monitoring und Evaluation
- One-Click GGUF/AWQ/GPTQ Export

---

## Installation

```bash
# Voraussetzungen
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# LLaMA-Factory installieren
git clone --depth 1 https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
pip install -e ".[torch,metrics]"

# Optionale Extras (empfohlen)
pip install -e ".[torch,metrics,deepspeed,bitsandbytes,vllm]"

# Installationstest
llamafactory-cli version
```

> [!NOTE]
> LLaMA-Factory benötigt Python 3.8+ und PyTorch 2.0+. Für die WebUI wird zusätzlich `gradio` installiert (automatisch durch die Installation).

---

## WebUI nutzen

```bash
# WebUI starten
llamafactory-cli webui

# Auf einem Server mit Port-Forwarding
llamafactory-cli webui --server_name 0.0.0.0 --server_port 7860

# Mit SSH Tunnel vom lokalen Rechner
# ssh -L 7860:localhost:7860 user@server
```

Nach dem Start im Browser öffnen: `http://localhost:7860`

---

## Wichtige Einstellungen in der WebUI

Die WebUI ist in Tabs aufgeteilt. Hier sind die wichtigsten Einstellungen im **Train-Tab:**

### Model Name
- Auswahl aus der integrierten Modell-Liste oder eigenes Modell-Pfad eingeben
- Beispiel: `LLaMA-3.2-3B-Instruct` oder `/path/to/model`

### Finetuning Method
- `lora`: LoRA Fine-Tuning (empfohlen für die meisten Fälle)
- `full`: Full Fine-Tuning (braucht viel VRAM)
- `freeze`: Nur bestimmte Layer trainieren

### Quantization Bit
- `none`: Kein Quantisierung (bfloat16)
- `4`: 4-Bit QLoRA
- `8`: 8-Bit Training

### Dataset
- Aus der Dropdown-Liste wählen oder eigenes Dataset hochladen
- Format: Alpaca (`alpaca_format`) oder ShareGPT (`sharegpt_format`)

### Wichtige Hyperparameter (in der WebUI):

| Parameter | Empfohlener Wert | Beschreibung |
|---|---|---|
| Learning Rate | 1e-4 bis 5e-4 | Lernrate |
| Epochs | 3–5 | Trainings-Epochen |
| Batch Size | 2–4 | Pro GPU |
| Gradient Accumulation | 4–8 | Effektive Batch-Size |
| Cutoff Length | 1024–2048 | Maximale Sequenzlänge |
| LoRA Rank | 8–32 | r-Parameter |
| LoRA Alpha | 16–64 | Alpha-Parameter |

> [!TIP]
> In der WebUI gibt es einen **Preview** Button, der den finalen Trainingsbefehl als Shell-Kommando anzeigt. Das ist sehr hilfreich um zu verstehen, was unter der Haube passiert, und um das Training später per CLI zu reproduzieren.

---

## Eigenes Dataset in LLaMA-Factory einbinden

LLaMA-Factory verwaltet Datasets über eine `dataset_info.json` Datei.

```bash
# Dataset-Dateien platzieren
cp ./mein_dataset.jsonl LLaMA-Factory/data/
```

Dann `LLaMA-Factory/data/dataset_info.json` anpassen:

```json
{
  "mein_dataset": {
    "file_name": "mein_dataset.jsonl",
    "formatting": "alpaca",
    "columns": {
      "prompt": "instruction",
      "query": "input",
      "response": "output"
    }
  },
  "mein_sharegpt_dataset": {
    "file_name": "mein_chat_dataset.jsonl",
    "formatting": "sharegpt",
    "columns": {
      "messages": "conversations"
    },
    "tags": {
      "role_tag": "from",
      "content_tag": "value",
      "user_tag": "human",
      "assistant_tag": "gpt"
    }
  }
}
```

Nach dem Neustart der WebUI erscheint das Dataset in der Dropdown-Liste.

---

## CLI-Training

```bash
# SFT Training mit LoRA
llamafactory-cli train \
  --model_name_or_path meta-llama/Llama-3.2-3B-Instruct \
  --stage sft \
  --do_train true \
  --finetuning_type lora \
  --quantization_bit 4 \
  --dataset mein_dataset \
  --template llama3 \
  --lora_rank 16 \
  --lora_alpha 32 \
  --lora_dropout 0.05 \
  --output_dir ./llama_factory_output \
  --num_train_epochs 3 \
  --per_device_train_batch_size 2 \
  --gradient_accumulation_steps 4 \
  --learning_rate 2e-4 \
  --lr_scheduler_type cosine \
  --warmup_ratio 0.03 \
  --logging_steps 10 \
  --save_steps 500 \
  --bf16 true \
  --plot_loss true   # Loss-Kurve als PNG speichern
```

**Als YAML-Konfigurationsdatei (empfohlen für Reproduzierbarkeit):**

```yaml
# llama_factory_config.yaml
model_name_or_path: meta-llama/Llama-3.2-3B-Instruct
stage: sft
do_train: true
finetuning_type: lora
quantization_bit: 4

dataset: mein_dataset
template: llama3
cutoff_len: 2048
overwrite_cache: true
preprocessing_num_workers: 4

output_dir: ./llama_factory_output
logging_steps: 10
save_steps: 500
plot_loss: true
overwrite_output_dir: true

per_device_train_batch_size: 2
gradient_accumulation_steps: 4
learning_rate: 2.0e-4
num_train_epochs: 3.0
lr_scheduler_type: cosine
warmup_ratio: 0.03
bf16: true
ddp_timeout: 180000000

lora_rank: 16
lora_alpha: 32
lora_dropout: 0.05
lora_target: all   # Alle Standard-Attention-Module
```

```bash
# Training mit YAML-Config
llamafactory-cli train llama_factory_config.yaml

# Multi-GPU mit DeepSpeed
deepspeed --num_gpus=4 \
  $(which llamafactory-cli) train llama_factory_config.yaml \
  --deepspeed examples/deepspeed/ds_z3_config.json
```

---

## Inference und Export in LLaMA-Factory

```bash
# Chat mit trainiertem Modell (CLI)
llamafactory-cli chat \
  --model_name_or_path meta-llama/Llama-3.2-3B-Instruct \
  --adapter_name_or_path ./llama_factory_output \
  --template llama3 \
  --finetuning_type lora

# LoRA Adapter mergen
llamafactory-cli export \
  --model_name_or_path meta-llama/Llama-3.2-3B-Instruct \
  --adapter_name_or_path ./llama_factory_output \
  --template llama3 \
  --finetuning_type lora \
  --export_dir ./merged_model \
  --export_size 2 \       # Modell auf 2 Dateien aufteilen
  --export_device cpu     # Auf CPU mergen (spart VRAM)

# Als GGUF exportieren (für Ollama/llama.cpp)
llamafactory-cli export \
  --model_name_or_path meta-llama/Llama-3.2-3B-Instruct \
  --adapter_name_or_path ./llama_factory_output \
  --template llama3 \
  --finetuning_type lora \
  --export_dir ./gguf_model \
  --export_quantization_bit 4 \
  --export_quantization_dataset data/c4_demo.json
```

---

## Vorteil: Benutzerfreundlichkeit und Modellunterstützung

LLaMA-Factory glänzt besonders durch:

**100+ unterstützte Modelle**, darunter:

| Familie | Modelle |
|---|---|
| Llama | 2, 3, 3.1, 3.2, 3.3 (alle Größen) |
| Mistral | 7B v0.1–v0.3, Mixtral, Mistral-Nemo |
| Qwen | 1.5, 2, 2.5 (0.5B–72B) |
| Phi | 1.5, 2, 3, 3.5 (mini/small/medium) |
| Gemma | 2B, 7B, Gemma 2 (2B, 9B, 27B) |
| InternLM | 2, 2.5, 3 |
| DeepSeek | DeepSeek-R1, DeepSeek-V2 |
| Yi | Yi-1.5 (6B, 9B, 34B) |

**Alle Training-Methoden in einer UI:**

| Methode | LLaMA-Factory Support |
|---|---|
| SFT (LoRA/QLoRA/Full) | Ja |
| DPO | Ja |
| PPO (RLHF) | Ja |
| ORPO | Ja |
| KTO | Ja |
| Reward Model Training | Ja |

---

## Framework-Vergleich

| Kriterium | Unsloth | Axolotl | LLaMA-Factory |
|---|---|---|---|
| Einstiegshürde | Niedrig | Mittel | Sehr niedrig |
| Konfiguration | Python-Code | YAML | WebUI / YAML |
| Trainingsgeschwindigkeit | Sehr hoch | Hoch | Hoch |
| Multi-GPU | Eingeschränkt | Sehr gut | Sehr gut |
| Modell-Support | ~20 Modelle | ~50 Modelle | 100+ Modelle |
| DPO / RLHF | Eingeschränkt | Gut | Sehr gut |
| WebUI | Nein | Nein | Ja |
| GGUF-Export | Ja (direkt) | Nein | Ja |
| Dokumentation | Gut | Gut | Sehr gut |
| Community | Groß | Groß | Sehr groß |

> [!TIP]
> **Empfehlung für Einsteiger:** Mit LLaMA-Factory (WebUI) starten – kein Code nötig, sofort produktiv. Dann zu Unsloth wechseln wenn Geschwindigkeit wichtig wird, oder zu Axolotl wenn reproduzierbare Multi-GPU-Experimente gebraucht werden.

---

## Verwandte Notizen

- [[Fine-Tuning Grundlagen]]
- [[LoRA und QLoRA]]
- [[Datasets vorbereiten]]
