# Lokale LLMs

> [!NOTE]
> Lokale LLMs laufen vollständig auf der eigenen Hardware – kein Internet, kein API-Key, keine Datenweitergabe. Diese Notiz gibt einen Überblick über Tools, Hardware und Modell-Empfehlungen. Ausführliche Installationsanleitungen für die einzelnen Tools befinden sich unter `AI/Tools/`.

**Stand: März 2026**

---

## Warum lokale LLMs?

### Datenschutz und Compliance

Der stärkste Grund für lokale LLMs ist **volle Datenkontrolle**. Bei Cloud-APIs werden alle Eingaben an externe Server gesendet – auch sensible Informationen wie:

- Interne Firmendaten und Strategiepapiere
- Quellcode mit proprietärer Logik
- Persönliche Daten (DSGVO-relevant)
- Medizinische oder rechtliche Informationen
- Zugangsdaten oder API-Keys in Code-Prompts

Mit lokalen LLMs verlassen diese Daten das eigene Netzwerk nie.

### Kostenstruktur

| Ansatz | Kosten-Modell | Break-Even |
|--------|--------------|-----------|
| Cloud-API | Pay-per-Token, laufende Kosten | - |
| Lokal (GPU-Kauf) | Einmalige Hardware, dann ~0 | Bei hoher Nutzung: 3-12 Monate |
| Lokal (vorhandene Hardware) | Praktisch kostenlos | Sofort |

**Beispielrechnung:**
- Claude Sonnet 4.5: $3 / 1M Input-Tokens
- 1.000 tägliche Anfragen × 500 Tokens = 500.000 Tokens/Tag = $1,50/Tag = $45/Monat
- Eine RTX 4090 (ca. 1.500€) läuft Llama 4 Scout lokal (MoE, Q4) → Break-Even nach ~33 Monaten
- Aber: Qualität ist bei Cloud-Modellen aktuell noch deutlich besser

### Offline-Fähigkeit

Lokale LLMs funktionieren ohne Internetverbindung:
- Air-gapped Environments (Sicherheitskritische Systeme)
- Reisen mit schlechter Konnektivität
- Keine API-Ausfälle, kein Rate-Limiting
- Garantierte Verfügbarkeit für kritische Anwendungen

> [!WARNING]
> Lokale LLMs sind (Stand März 2026) in der Qualität hinter den Top-Cloud-Modellen, aber der Abstand schrumpft rapide. Modelle wie Llama 4 Scout (MoE 109B/17B aktiv), DeepSeek V3.2 und Mistral Small 3.2 erreichen auf vielen Benchmarks GPT-4-Niveau oder darüber. Für kritische Aufgaben (komplexes Reasoning, Coding) können Cloud-APIs dennoch überlegen sein.

---

## Hardware-Anforderungen

### Die entscheidende Ressource: VRAM

Das wichtigste für lokale LLMs ist der **VRAM (Video RAM) der GPU**. Ein Modell muss vollständig in den VRAM passen, um GPU-beschleunigt zu laufen. Passt es nicht rein, fällt die Ausführung auf die CPU zurück – was 10-50x langsamer ist.

**Faustformel (fp16/bf16, ohne Quantisierung):**
```
VRAM (GB) ≈ Modellparameter (Milliarden) × 2
Beispiel: 7B-Modell ≈ 14 GB VRAM
```

**Mit Q4-Quantisierung (~60% Reduktion):**
```
VRAM (GB) ≈ Modellparameter (Milliarden) × 0,7
Beispiel: 7B-Modell mit Q4 ≈ 4-5 GB VRAM
```

### VRAM-Tabelle: Modelle und passende GPUs

| Modell | VRAM (fp16) | VRAM (Q4) | VRAM (Q8) | Passende GPUs |
|--------|------------|----------|----------|---------------|
| Phi-4 3.8B | ~7,5 GB | ~3 GB | ~4,5 GB | GTX 1660 Ti, RTX 3060 (8GB) |
| Llama 3.2 3B | ~6 GB | ~2,5 GB | ~3,5 GB | Jede moderne GPU mit 4+ GB |
| Llama 3.1 8B | ~16 GB | ~5,5 GB | ~9 GB | RTX 3080 (10GB), RTX 4070 |
| Mistral Small 3.2 24B | ~48 GB | ~16 GB | ~26 GB | RTX 4090 (24GB) bei Q4 |
| Codestral 25.01 22B | ~44 GB | ~15 GB | ~24 GB | RTX 4090 (24GB) bei Q4 |
| Mistral Nemo 12B | ~24 GB | ~8 GB | ~13 GB | RTX 3090/4090 (24GB) bei Q4 |
| Gemma2 9B | ~18 GB | ~6,5 GB | ~10 GB | RTX 3090 bei Q8, RTX 4070 bei Q4 |
| Llama 4 Scout 109B (MoE, 17B aktiv) | ~30 GB* | ~12 GB* | ~20 GB* | RTX 4090 bei Q4 (MoE: nur aktive Parameter) |
| DeepSeek V3.2 (MoE) | ~80 GB* | ~28 GB* | ~45 GB* | 2x RTX 4090 bei Q4 |
| DeepSeek R1 (MoE) | ~80 GB* | ~28 GB* | ~45 GB* | 2x RTX 4090 bei Q4 |
| Llama 3.3 70B | ~140 GB | ~48 GB | ~75 GB | 2x RTX 4090 bei Q4 |
| Qwen2.5 72B | ~144 GB | ~50 GB | ~78 GB | 2-3x RTX 4090 bei Q4 |
| Llama 4 Maverick 400B (MoE, 17B aktiv) | ~60 GB* | ~24 GB* | ~40 GB* | RTX 4090 bei Q4, H100 |
| Llama 3.1 405B | ~810 GB | ~280 GB | ~430 GB | 8x A100 80GB (fp16) |

> \* MoE-Modelle (Mixture of Experts) laden nur die aktiven Parameter pro Token. VRAM-Angaben beziehen sich auf die tatsächlich benötigten aktiven Parameter, nicht die Gesamtparameterzahl.

### GPU-Empfehlungen nach Budget

| Budget | GPU | VRAM | Empfohlene Modelle |
|--------|-----|------|-------------------|
| ~300€ | RTX 3060 12GB | 12 GB | Bis 8B-Modelle (Q4/Q5) |
| ~500€ | RTX 4060 Ti 16GB | 16 GB | 8B-Modelle (fp16), 13B (Q4) |
| ~800€ | RTX 4070 Ti Super | 16 GB | 8B (fp16), 13B (Q4) – schneller |
| ~1.500€ | RTX 4090 | 24 GB | Llama 4 Scout (Q4), Mistral Small 3.2, 70B (Q4 mit Offloading) |
| ~2.500€ | 2x RTX 3090 | 48 GB | 70B-Modelle (Q4/Q5), Llama 4 Maverick (Q4) |
| ~10.000€ | A100 80GB / H100 | 80 GB | 70B (fp16), Llama 4 Scout/Maverick, DeepSeek V3.2 |

> [!TIP]
> Für den Einstieg ist ein **Llama 3.1 8B mit Q5-Quantisierung** auf einer RTX 4060 (8GB) oder ähnlichen GPU ein hervorragender Start. Alternativ bieten MoE-Modelle wie **Llama 4 Scout** überraschend starke Leistung auf Consumer-Hardware – Q4-quantisiert passt es auf eine RTX 4090.

### RAM (System-RAM)

Neben VRAM ist auch System-RAM wichtig, besonders wenn Modell-Teile auf die CPU ausgelagert werden:

| Szenario | Benötigter RAM |
|----------|---------------|
| Modell passt komplett in VRAM | 16 GB reichen |
| Teilweises CPU-Offloading | 32 GB empfohlen |
| Großes Modell (70B+) mit Offloading | 64-128 GB |

---

## Quantisierung: Q4, Q5, Q8 erklärt

### Was ist Quantisierung?

Quantisierung reduziert die **Präzision** der Modellgewichte. Statt 16-Bit-Gleitkommazahlen (fp16) werden die Gewichte auf 4-8 Bit komprimiert. Das reduziert:
- **VRAM-Bedarf** drastisch
- **Ladezeit** des Modells
- **Inferenz-Geschwindigkeit** verbessert sich oft

**Dafür wird etwas Qualität eingebüßt.**

### Quantisierungs-Stufen im Vergleich

| Format | Bits/Gewicht | VRAM vs. fp16 | Qualitätsverlust | Empfehlung |
|--------|-------------|---------------|-----------------|-----------|
| fp32 | 32 Bit | 200% | Keine (Referenz) | Nur für Training |
| fp16 / bf16 | 16 Bit | 100% (Baseline) | Minimal | Wenn VRAM reicht |
| Q8_0 | 8 Bit | ~50% | Sehr gering (~1%) | Wenn VRAM verfügbar |
| Q6_K | 6 Bit | ~37% | Gering (~2%) | Guter Kompromiss |
| Q5_K_M | 5 Bit | ~31% | Moderat (~3%) | Empfohlen für 70B |
| Q4_K_M | 4 Bit | ~25% | Merklich (~5-8%) | Standard-Empfehlung |
| Q3_K_M | 3 Bit | ~19% | Deutlich (~10-15%) | Nur wenn nötig |
| Q2_K | 2 Bit | ~12% | Stark (~20-30%) | Notlösung |

**Die `_K_M` / `_K_S` Suffixe** (z.B. Q4_K_M): "K" bedeutet k-quantization (wichtige Schichten werden weniger quantisiert), "M" = Medium (Balance), "S" = Small (stärkere Kompression).

> [!TIP]
> **Empfehlung für die meisten Anwender:** Nutze **Q4_K_M** wenn VRAM knapp ist, **Q5_K_M** wenn etwas mehr VRAM verfügbar ist. Der Qualitätsunterschied zwischen Q4_K_M und Q8 ist für die meisten Aufgaben kaum wahrnehmbar.

### Quantisierungs-Auswirkung auf die Praxis

```
Aufgabe: "Erkläre Rekursion in Python"

fp16 Llama 3.1 8B:
→ Klare, präzise Erklärung mit korrektem Beispielcode

Q4_K_M Llama 3.1 8B:
→ Ebenfalls korrekte Erklärung, minimal weniger eloquent
→ Code-Qualität vergleichbar

Q2_K Llama 3.1 8B:
→ Sichtbarer Qualitätsverlust, manche Erklärungen ungenau
→ Für produktiven Einsatz nicht empfohlen
```

---

## Lokale LLM-Tools

### Ollama

**Ollama** ist das meistgenutzte Tool für lokale LLMs. Es bietet eine einfache CLI und eine OpenAI-kompatible API.

```bash
# Schnellstart (nach Installation)
ollama pull llama3.1:8b
ollama run llama3.1:8b "Erkläre mir Quantisierung in einfachen Worten"

# Neue Modelle (Stand März 2026)
ollama run llama4:scout       # Llama 4 Scout – MoE 109B/17B aktiv, 10M Context
ollama run llama4:maverick    # Llama 4 Maverick – MoE 400B/17B aktiv, 1M Context (24GB+ VRAM)
ollama run deepseek-v3.2      # DeepSeek V3.2 – extrem leistungsstark, 128K Context
ollama run deepseek-r1        # DeepSeek R1 – Reasoning-Modell
ollama run mistral-small3.2   # Mistral Small 3.2 (24B), 262K Context
ollama run codestral          # Codestral 25.01 (22B), Code-Spezialist, 256K Context
ollama run phi4               # Phi-4 – Microsofts kleines Modell

# Als Server starten (OpenAI-API-kompatibel auf Port 11434)
ollama serve

# Verfügbare Modelle auflisten
ollama list

# Modell-Informationen
ollama show llama4:scout
```

**API-Nutzung:**
```python
# Ollama ist OpenAI-API-kompatibel
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # Beliebiger String
)

response = client.chat.completions.create(
    model="llama3.1:8b",
    messages=[{"role": "user", "content": "Hallo!"}]
)
print(response.choices[0].message.content)
```

Ausführliche Installationsanleitung: [[Ollama]]

---

### LM Studio

**LM Studio** ist eine Desktop-Applikation (Windows, macOS, Linux) für das Herunterladen und Ausführen von LLMs mit grafischer Benutzeroberfläche.

**Features:**
- Integrierter Modell-Browser (Hugging Face-Integration)
- Chat-Interface direkt in der App
- Lokaler API-Server (OpenAI-kompatibel)
- Modell-Vergleiche nebeneinander
- GPU/CPU-Auslastungsanzeige
- Einfaches Wechseln zwischen Modellen

**Typischer Workflow:**
1. LM Studio öffnen
2. Im "Discover"-Tab nach Modell suchen (z.B. "llama 3.1")
3. GGUF-Variante mit passendem Q-Level auswählen
4. Herunterladen
5. In "Chat" oder als "Local Server" starten

> [!NOTE]
> LM Studio ist ideal für **Einsteiger und visuelle Nutzer**. Kein Terminal nötig, alles per GUI bedienbar. Für Automatisierung und Scripting ist Ollama flexibler.

---

### Jan.ai

**Jan.ai** ist eine Open-Source-Alternative zu LM Studio (MIT-Lizenz).

**Unterschiede zu LM Studio:**
- Vollständig Open Source (GitHub: `janhq/jan`)
- Mehr Kontrolle und Anpassungsmöglichkeiten
- Extensions-System (erweiterbar)
- Etwas weniger poliertes UI als LM Studio
- Plattformübergreifend (Windows, macOS, Linux)

```bash
# Jan.ai API (ebenfalls OpenAI-kompatibel)
curl http://localhost:1337/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1-8b-instruct",
    "messages": [{"role": "user", "content": "Hallo Jan!"}]
  }'
```

---

### llama.cpp

**llama.cpp** ist die C++-Implementierung für das Ausführen von LLaMA-Modellen (und viele andere). Es ist die **Grundlage für Ollama, LM Studio und Jan.ai**.

**Warum direkt nutzen?**
- Maximale Kontrolle über alle Parameter
- Minimaler Overhead (keine zusätzliche Abstraktionsschicht)
- Scripting und Integration in eigene Tools
- Beste Performance auf spezifischer Hardware (Metal auf Apple Silicon, CUDA, Vulkan)

```bash
# Kompilieren
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON && cmake --build build  # Mit CUDA-Unterstützung

# Modell ausführen
./build/bin/llama-cli -m models/llama-3.1-8b-q4_k_m.gguf \
  -p "Erkläre mir Rekursion" \
  -n 512 \
  --temp 0.7 \
  --ctx-size 8192

# Als Server starten
./build/bin/llama-server -m models/llama-3.1-8b-q4_k_m.gguf \
  --port 8080 \
  --ctx-size 8192 \
  --n-gpu-layers 35  # Anzahl Layer auf GPU
```

---

## Empfohlene Modelle nach Größe

### 1B-4B: Kleinste Modelle (Edge/Mobile)

| Modell | VRAM (Q4) | Stärken | Schwächen |
|--------|----------|---------|-----------|
| Llama 3.2 1B | ~1 GB | Extrem schnell, minimal | Sehr begrenzte Fähigkeiten |
| Llama 3.2 3B | ~2 GB | Gut für einfache Tasks | Kein komplexes Reasoning |
| Phi-4 3.8B | ~2,5 GB | Microsofts bestes kleines Modell, starkes Reasoning | Kurzer Context bei kleiner GPU |
| Gemma2 2B | ~1,5 GB | Gut für Klassifizierung | Limitierte Generierung |

**Empfehlung:** Phi-4 – trotz 3.8B Parameter hat es durch optimiertes Training erstaunlich gute Qualität.

### 7B-24B: Sweet Spot für die meisten Nutzer

| Modell | VRAM (Q4) | Stärken | Schwächen |
|--------|----------|---------|-----------|
| Llama 3.1 8B | ~5,5 GB | Sehr gute Balance, 128K Context | Kein Mittelding zu 70B |
| Mistral Small 3.2 24B | ~16 GB | 262K Context, effizient, stark in Reasoning | Braucht RTX 4090 für Q4 |
| Codestral 25.01 22B | ~15 GB | Code-Spezialist, 256K Context, 80+ Sprachen | Nur für Code-Aufgaben optimiert |
| Mistral Nemo 12B | ~8 GB | Apache 2.0, stark in Reasoning | Etwas mehr VRAM |
| Gemma2 9B | ~6,5 GB | Sehr gute Performance für Größe | Kürzerer Context (8K) |
| Qwen2.5 7B | ~5 GB | Gute Mehrsprachigkeit, starker Code | Weniger Community-Support |

**Empfehlung:** **Mistral Small 3.2 24B** (262K Context, sehr effizient, starkes Reasoning) wenn eine RTX 4090 vorhanden ist, sonst **Llama 3.1 8B** (breite Community, viele Fine-Tunes verfügbar). Für Code-Aufgaben: **Codestral 25.01**.

### 70B+ / MoE: Höchste lokale Qualität

| Modell | VRAM (Q4) | Stärken | Anforderungen |
|--------|----------|---------|--------------|
| Llama 4 Scout 109B (MoE) | ~12 GB* | 10M Context (!), MoE 17B aktiv, GPT-4-Niveau | RTX 4090, oder H100 für fp16 |
| Llama 4 Maverick 400B (MoE) | ~24 GB* | 1M Context, MoE 17B aktiv, extrem stark | 24GB+ VRAM, H100 empfohlen |
| DeepSeek V3.2 (MoE) | ~28 GB* | Extrem leistungsstark, 128K Context | 2x RTX 4090 bei Q4 |
| DeepSeek R1 (MoE) | ~28 GB* | Reasoning-Spezialist, Chain-of-Thought | 2x RTX 4090 bei Q4 |
| Llama 3.3 70B | ~48 GB | Verbesserte Version, 128K Context | 2x RTX 4090 bei Q4 |
| Qwen2.5 72B | ~50 GB | Sehr stark in Code und Mehrsprachigkeit | 2-3x RTX 4090 bei Q4 |

> \* MoE-Architektur: VRAM bezieht sich auf aktive Parameter. Llama 4 Scout hat 109B Gesamtparameter, aber nur 17B sind pro Token aktiv – dadurch läuft es auf deutlich weniger Hardware als ein traditionelles 70B-Modell.

> [!TIP]
> **Llama 4 Scout** ist der neue Sweet Spot für hohe Qualität auf Consumer-Hardware: Dank MoE-Architektur passt es quantisiert auf eine einzelne RTX 4090 und bietet dabei 10M Token Context und GPT-4-Niveau-Qualität. Für maximale Qualität ohne Cloud: **DeepSeek V3.2** auf 2x RTX 4090.

---

## GGUF-Format

### Was ist GGUF?

**GGUF** (GGML Unified Format) ist das Standard-Dateiformat für quantisierte Modelle, die mit llama.cpp (und kompatiblen Tools wie Ollama, LM Studio) ausgeführt werden können.

- Nachfolger von GGML
- Enthält Modellgewichte + Metadaten in einer einzigen Datei
- Optimiert für schnelles Laden und effiziente Inferenz
- Plattformübergreifend kompatibel

**Dateiname-Konvention:**
```
llama-3.1-8b-instruct.Q4_K_M.gguf
├── Modellname: llama-3.1-8b-instruct
├── Quantisierung: Q4_K_M
└── Format: .gguf
```

### Wo findet man GGUF-Modelle?

**Hugging Face** ist die primäre Quelle. Besonders empfehlenswert:

- **TheBloke** (der bekannteste GGUF-Ersteller, hunderte Modelle): `huggingface.co/TheBloke`
- **bartowski** (modernere Quantisierungen, oft aktueller): `huggingface.co/bartowski`
- **Offizielle Repos** (z.B. `meta-llama`, `mistralai`, `microsoft`)

```bash
# GGUF-Modell von Hugging Face herunterladen
# Option 1: Direkt mit wget/curl
wget https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf

# Option 2: Mit huggingface_hub (Python)
pip install huggingface_hub

python3 -c "
from huggingface_hub import hf_hub_download
hf_hub_download(
    repo_id='bartowski/Meta-Llama-3.1-8B-Instruct-GGUF',
    filename='Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf',
    local_dir='./models'
)
"

# Option 3: Über Ollama (automatisch)
ollama pull llama3.1:8b  # Ollama kümmert sich um Download und Format
```

> [!NOTE]
> Beim Herunterladen auf Hugging Face immer die **Modell-Karte** (Model Card) lesen! Dort stehen wichtige Infos zu: Lizenz, Trainingsdetails, empfohlene Parameter (Temperature, System-Prompt), bekannte Einschränkungen.

---

## Vergleich: Lokale LLMs vs. Cloud-APIs

| Kriterium | Lokal | Cloud-API |
|-----------|-------|-----------|
| **Qualität** | Gut (7B-13B), sehr gut (70B+) | Hervorragend (Top-Modelle) |
| **Datenschutz** | Vollständig (Daten bleiben lokal) | Abhängig von Anbieter/AGB |
| **Kosten (einmalig)** | Hardware (0-10.000€+) | Keine |
| **Kosten (laufend)** | Stromkosten (~50-200W) | Pay-per-Token |
| **Latenz** | Niedrig (lokal, keine Netzwerklatenz) | Variiert (50-500ms + Generierung) |
| **Geschwindigkeit** | Mittel (10-50 T/s auf Consumer-GPU) | Hoch (100+ T/s bei großen APIs) |
| **Verfügbarkeit** | 24/7, offline | Abhängig von API-Verfügbarkeit |
| **Skalierbarkeit** | Durch Hardware begrenzt | Praktisch unbegrenzt |
| **Setup-Aufwand** | Mittel-Hoch | Minimal (API-Key) |
| **Updates** | Manuell | Automatisch durch Anbieter |
| **Anpassung (Finetuning)** | Vollständig möglich | Begrenzt (nur über offizielle Services) |
| **Context Window** | 8K-10M (je nach Modell/Tool) | Bis 2M Tokens (Gemini) |

### Empfohlene Hybrid-Strategie

```
Aufgabe mit sensiblen Daten?
    → Lokales LLM (Llama 4 Scout, DeepSeek V3.2 oder Llama 3.1 8B)

Aufgabe erfordert höchste Qualität?
    → Claude Sonnet 4.6 / GPT-5.2

Hohe Anfragevolumen (>10.000 Anfragen/Tag)?
    → Kosten vergleichen: Lokal vs. Cloud

Entwicklung/Prototyping?
    → Cloud-API (kein Hardware-Setup nötig)

Code-Aufgaben lokal?
    → Codestral 25.01 oder DeepSeek V3.2

Produktionssystem, hohe Qualitätsanforderungen?
    → Cloud-API mit Datenschutz-Vertrag (DPA)
```

---

## Verwandte Notizen

- [[Modell Übersicht]]
- [[Context Window und Token Management]]
- [[Modell Vergleiche]]
- [[Ollama]]
- [[Open WebUI]]

### Externe Ressourcen

- [Ollama](https://ollama.com) — Download und Dokumentation
- [Hugging Face](https://huggingface.co) — Modelle und GGUF-Downloads
