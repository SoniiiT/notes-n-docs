# LLM API Provider und Router

> [!NOTE]
> LLM API Provider sind die **Backend-Dienste**, die Zugang zu Sprachmodellen über eine API bereitstellen. Sie sind nicht die Tools oder Agents selbst, sondern die Infrastruktur, an die sich [[AI Coding Agents Vergleich|Coding Agents]], Apps und Entwickler-Tools anbinden. Diese Notiz erklärt das gesamte Ökosystem: Direkte Provider, Inference-Dienste, Router und lokale Lösungen.

---

## Was sind LLM API Provider?

Wenn ein [[Was ist ein AI Agent|AI Agent]] wie [[Aider Konfiguration|Aider]], [[Cline Konfiguration|Cline]] oder [[Claude Code Konfiguration und Hooks|Claude Code]] eine Antwort generiert, passiert im Hintergrund Folgendes:

```
┌─────────────┐     HTTP/REST      ┌──────────────────┐     Inferenz     ┌───────────┐
│  Coding     │ ──────────────────▶│  API Provider    │ ───────────────▶│  LLM      │
│  Agent      │    API Request     │  (z.B. OpenAI,   │   GPU-Cluster   │  Modell   │
│  (Aider,    │◀────────────────── │   OpenRouter,    │◀─────────────── │  (GPT,    │
│   Cline)    │    API Response    │   Groq)          │   Tokens        │   Claude) │
└─────────────┘                    └──────────────────┘                 └───────────┘
```

**Der Agent selbst führt kein Modell aus.** Er sendet einen Prompt an eine API und bekommt eine Antwort zurück. Welchen Provider man wählt, bestimmt:

- **Welche Modelle** verfügbar sind
- **Was es kostet** (pro Token, Abo oder kostenlos)
- **Wie schnell** die Antwort kommt (Latenz und Tokens/Sekunde)
- **Wo die Daten** verarbeitet werden (Cloud vs. lokal)
- **Welche Features** unterstützt werden (Tool Use, Vision, Streaming)

> [!TIP]
> Die meisten Coding Agents unterstützen mehrere Provider. Man ist nicht an einen einzigen gebunden – man kann z.B. Claude für komplexe Aufgaben und Groq für schnelle Iterationen verwenden.

---

## Kategorien von Providern

Es gibt vier grundlegend verschiedene Arten, wie man an LLM-Inferenz kommt:

| Kategorie | Beschreibung | Beispiele | Eigene Modelle? |
|-----------|-------------|-----------|-----------------|
| **Direkte Provider** | Betreiben eigene proprietäre Modelle | Anthropic, OpenAI, Google, xAI, Mistral | ✅ Ja |
| **Inference Provider** | Hosten Open-Source-Modelle auf eigener Infrastruktur | Together AI, Groq, DeepInfra, Fireworks | ❌ Nein (Open-Source) |
| **Router/Aggregatoren** | Leiten Anfragen an verschiedene Provider weiter | OpenRouter | ❌ Nein (Vermittler) |
| **Lokal** | Modelle laufen auf eigener Hardware | Ollama, LM Studio, vLLM | ❌ Nein (Open-Source) |

```
                    ┌─────────────────────────────────┐
                    │      Direkte Provider           │
                    │  (Anthropic, OpenAI, Google)     │
                    │  → Eigene Modelle, eigene GPUs   │
                    └────────────────┬────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
┌───────────────┐         ┌──────────────────┐         ┌──────────────────┐
│  Inference    │         │  Router          │         │  Lokal           │
│  Provider     │         │  (OpenRouter)    │         │  (Ollama,        │
│  (Groq,       │         │  → Einheitliche  │         │   LM Studio)     │
│   Together)   │         │    API zu allen   │         │  → Eigene HW     │
│  → OS-Modelle │         │    Providern     │         │  → Kostenlos     │
│    auf eigener│         └──────────────────┘         └──────────────────┘
│    Hardware   │
└───────────────┘
```

---

## OpenRouter — Der universelle Router

### Was ist OpenRouter?

OpenRouter ist ein **API-Gateway**, das eine einheitliche, OpenAI-kompatible API bereitstellt und Anfragen an über **300 Modelle** von allen großen Providern weiterleitet. Statt für jeden Provider einen eigenen API-Key und eine eigene Integration zu pflegen, nutzt man einen einzigen OpenRouter-Key.

> [!IMPORTANT]
> OpenRouter ist **kein** eigener Inference-Provider. Es betreibt keine eigenen GPUs und trainiert keine Modelle. Es fungiert als intelligenter Vermittler zwischen dem Nutzer und den eigentlichen Providern (Anthropic, OpenAI, Google, Together AI, etc.).

### Technische Details

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://openrouter.ai/api/v1` |
| **API-Kompatibilität** | OpenAI-kompatibel (Drop-in Replacement) |
| **Authentifizierung** | Bearer Token (`Authorization: Bearer $OPENROUTER_API_KEY`) |
| **Modelle** | 300+ von allen großen Providern |
| **Pricing** | Pay-as-you-go, Modellpreise vom Provider durchgereicht |
| **Aufschlag** | Kein Markup auf Modellkosten, ~5,5% Payment Processing Fee |
| **Free Models** | Ja, mehrere kostenlose Modelle zum Testen |
| **Rate Limits** | Abhängig vom gewählten Modell und Provider |

### BYOK — Bring Your Own Key

> [!TIP]
> Mit BYOK kann man seine **eigenen Provider-API-Keys** bei OpenRouter hinterlegen. Die Anfragen werden dann direkt über den eigenen Account beim Provider abgerechnet. OpenRouter berechnet dafür **nichts** – man bekommt **1 Million Requests pro Monat kostenlos** über BYOK.

**Vorteile von BYOK:**
- Keine OpenRouter-Gebühren (0% Aufschlag)
- Eigene Rate Limits und Kontingente
- Direkte Abrechnung beim Provider
- Trotzdem OpenRouters einheitliche API nutzen

**Einrichtung:** Im OpenRouter Dashboard unter *Settings → Integrations* den API-Key des jeweiligen Providers hinterlegen.

### Pricing und Budget-Kontrolle

OpenRouter bietet granulare Budget-Kontrolle über das Dashboard: Monthly Budget, Per-Request Limit und Credit Alerts unter *Settings → Limits*.

### OpenRouter in Coding Agents einrichten

**Umgebungsvariablen (universell):**

```bash
export OPENROUTER_API_KEY="sk-or-v1-..."
export OPENROUTER_BASE_URL="https://openrouter.ai/api/v1"
```

**Aider:**

```bash
# Direkt als Argument
aider --model openrouter/anthropic/claude-sonnet-4

# Oder in ~/.aider.conf.yml
model: openrouter/anthropic/claude-sonnet-4
```

**OpenCode:**

```yaml
# In der OpenCode-Konfiguration
# Provider hinzufügen über /connect oder config.json
{
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-v1-...",
      "baseURL": "https://openrouter.ai/api/v1"
    }
  }
}
```

**Andere Tools (Open WebUI, AnythingLLM etc.):** Einfach als "OpenAI-Compatible API" hinzufügen mit Base URL `https://openrouter.ai/api/v1` und dem OpenRouter API Key.

**cURL-Beispiel:**

````bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "anthropic/claude-sonnet-4",
    "messages": [
      {"role": "user", "content": "Erkläre Kubernetes in 3 Sätzen."}
    ]
  }'
````

---

## Direkte Provider

Direkte Provider betreiben **eigene proprietäre Modelle** auf eigener Infrastruktur. Man zahlt direkt beim Anbieter und bekommt Zugang zu dessen Modellen – ohne Zwischenhändler.

### Anthropic — Claude

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.anthropic.com/v1` |
| **API-Key Variable** | `ANTHROPIC_API_KEY` |
| **Modelle** | Claude Opus 4.5, Sonnet 4.5, Haiku 4.5 |
| **Pricing** | Pay-per-Token |
| **Besonderheiten** | Extended Thinking, Prompt Caching, 200K Context |

> [!TIP]
> **Prompt Caching** ist ein Alleinstellungsmerkmal von Anthropic: Wiederkehrende System-Prompts werden gecacht und kosten bei Wiederverwendung nur 10% des normalen Preises. Für Coding Agents mit langen System-Prompts spart das erheblich.

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# Aider
aider --model claude-sonnet-4
# Claude Code (nutzt automatisch ANTHROPIC_API_KEY)
claude
```

Siehe auch: [[Modell Übersicht#Anthropic – Claude]]

### OpenAI — GPT

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.openai.com/v1` |
| **API-Key Variable** | `OPENAI_API_KEY` |
| **Modelle** | GPT-4o, GPT-4o-mini, o1, o3, o4-mini |
| **Pricing** | Pay-per-Token + optionale Subscription Tiers |
| **Besonderheiten** | Function Calling, Vision, Audio, DALL-E, Codex |

OpenAI bietet auch **Subscription-Stufen** an:
- **Free:** Limitierter Zugang über ChatGPT
- **Plus ($20/Monat):** Mehr Nutzung, Zugang zu neusten Modellen
- **Pro ($200/Monat):** Unlimitiert, höchste Priorität
- **API:** Separat, rein pay-per-token

```bash
export OPENAI_API_KEY="sk-..."
aider --model gpt-4o
```

Siehe auch: [[Modell Übersicht#OpenAI – GPT & Reasoning-Modelle]]

### Google — Gemini

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://generativelanguage.googleapis.com/v1beta` |
| **API-Key Variable** | `GEMINI_API_KEY` |
| **Modelle** | Gemini 2.5 Pro, Gemini 2.5 Flash, Gemini 2.0 Flash |
| **Pricing** | Free Tier verfügbar, danach Pay-per-Token |
| **Besonderheiten** | 1M+ Context Window, multimodal, kostenloser Einstieg |

> [!TIP]
> Google bietet einen **großzügigen Free Tier** für Gemini an: 15 Requests/Minute für Gemini 2.5 Flash kostenlos. Ideal zum Einstieg und Experimentieren, ohne Kreditkarte hinterlegen zu müssen.

```bash
export GEMINI_API_KEY="AI..."
aider --model gemini/gemini-2.5-pro
```

Siehe auch: [[Modell Übersicht#Google DeepMind – Gemini]]

### xAI — Grok

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.x.ai/v1` |
| **API-Key Variable** | `XAI_API_KEY` |
| **Modelle** | Grok 3, Grok 3 Mini |
| **Besonderheiten** | Echtzeit-Wissen, OpenAI-kompatible API |

### Mistral

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.mistral.ai/v1` |
| **API-Key Variable** | `MISTRAL_API_KEY` |
| **Modelle** | Mistral Large, Medium, Codestral, Pixtral |
| **Besonderheiten** | Europäischer Anbieter (Frankreich), DSGVO-freundlich |

> [!NOTE]
> Mistral ist ein **europäischer Anbieter** mit Sitz in Paris. Für europäische Unternehmen kann das bezüglich DSGVO und Datenverarbeitung relevant sein.

### DeepSeek

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.deepseek.com/v1` |
| **API-Key Variable** | `DEEPSEEK_API_KEY` |
| **Modelle** | DeepSeek R1, DeepSeek V3 |
| **Pricing** | Extrem günstig ($0.14/1M Input, $2.19/1M Output für R1) |
| **Besonderheiten** | Open-Source-Modelle, Chain-of-Thought Reasoning |

> [!WARNING]
> DeepSeek ist ein **chinesischer Anbieter**. Für sensible oder unternehmenskritische Daten sollte man abwägen, ob die direkte API genutzt wird, oder ob man DeepSeek-Modelle lieber über westliche Inference Provider (Together AI, Groq) oder [[Lokale LLMs|lokal]] betreibt.

---

## Inference Provider — Open-Source-Modelle hosten

Inference Provider betreiben **keine eigenen Modelle**, sondern hosten populäre Open-Source-Modelle (Llama, Mistral, DeepSeek, Qwen, etc.) auf ihrer eigenen GPU-Infrastruktur. Der Vorteil: Man bekommt Open-Source-Modelle als komfortable API, ohne eigene Hardware verwalten zu müssen.

> [!NOTE]
> Die gleichen Open-Source-Modelle können bei verschiedenen Inference Providern unterschiedlich schnell und unterschiedlich teuer sein – die Provider unterscheiden sich in Hardware, Optimierung und Pricing.

### Together AI

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.together.xyz/v1` |
| **Modelle** | 200+ Open-Source-Modelle |
| **Pricing** | $3–7 pro 1M Tokens (je nach Modell) |
| **Geschwindigkeit** | ~134 Tokens/Sekunde |
| **Free Tier** | $5 Startguthaben |
| **Best für** | Breite Modellauswahl, Finetuning |

Together AI hat die **größte Auswahl** an gehosteten Open-Source-Modellen. Zusätzlich bieten sie Finetuning als Service an – man kann eigene Modellvarianten auf ihrer Infrastruktur trainieren.

```bash
export TOGETHER_API_KEY="..."
# OpenAI-kompatibel
aider --model together_ai/meta-llama/Llama-3.1-405B-Instruct-Turbo
```

### Groq

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.groq.com/openai/v1` |
| **Modelle** | Llama 3.x, Mixtral, Gemma, DeepSeek R1 |
| **Pricing** | Ab $0.05/1M Tokens (Llama 3.1 8B), $0.59/1M (Llama 3.3 70B) |
| **Geschwindigkeit** | **275+ Tokens/Sekunde** (schnellster Provider) |
| **Free Tier** | Ja, mit Rate Limits |
| **Best für** | Geschwindigkeit, interaktive Nutzung |

> [!IMPORTANT]
> Groq nutzt **eigene LPU-Hardware** (Language Processing Unit) statt herkömmlicher GPUs. Das ermöglicht extrem niedrige Latenz und die höchste Inferenz-Geschwindigkeit am Markt. Für iterative Coding-Workflows, bei denen man viele schnelle Antworten braucht, ist Groq ideal.

### DeepInfra

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.deepinfra.com/v1/openai` |
| **Modelle** | 100+ Open-Source-Modelle |
| **Pricing** | Ab $0.03/1M Tokens (günstigster Provider), Llama 405B: $0.55/1M |
| **Geschwindigkeit** | ~100 Tokens/Sekunde |
| **Free Tier** | $1.80 Startguthaben |
| **Best für** | Kostenoptimierung, Budget-Projekte |

DeepInfra ist der **günstigste Inference Provider** für die meisten Modelle.

### Fireworks AI

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.fireworks.ai/inference/v1` |
| **Modelle** | 100+ Modelle, inkl. multimodal |
| **Pricing** | $0.10–5.00 pro 1M Tokens |
| **Geschwindigkeit** | Bis zu **1000 Tokens/Sekunde** (Speculative Decoding) |
| **Zertifizierungen** | HIPAA, SOC2 |
| **Best für** | Enterprise, regulierte Branchen |

> [!NOTE]
> Fireworks AI ist der einzige Inference Provider mit **HIPAA- und SOC2-Zertifizierung**. Für Unternehmen in regulierten Branchen ist das oft Voraussetzung.

### Replicate

| Eigenschaft | Wert |
|------------|------|
| **Base URL** | `https://api.replicate.com/v1` |
| **Modelle** | Tausende Community-Modelle (nicht nur LLMs) |
| **Pricing** | Pay-per-Second (GPU-Zeit), nicht per Token |
| **Best für** | Prototyping, Modell-Sharing, multimodale Modelle |

### Lambda & Lepton AI

- **Lambda** (`https://api.lambda.ai/v1`): Primär GPU-Cloud-Anbieter, der auch Inference-APIs anbietet. Interessant für Teams, die Training + Inference auf dedizierter Hardware brauchen.
- **Lepton AI** (`https://api.lepton.ai/v1`): Serverless Inference – man zahlt nur für tatsächliche Nutzung, ohne sich um Skalierung kümmern zu müssen.

---

## Lokale Inference

Lokale Modelle laufen auf der **eigenen Hardware** – kein Internet, keine API-Keys, keine laufenden Kosten (nach Hardware-Kauf).

> [!TIP]
> Für ausführliche Anleitungen zur lokalen Einrichtung siehe [[Lokale LLMs]].

### Übersicht der lokalen Tools

| Tool | Beschreibung | Best für |
|------|-------------|---------|
| **[[Lokale LLMs#Ollama\|Ollama]]** | CLI-Tool, einfachste Installation, `ollama run llama3` | Einsteiger, CLI-Nutzer |
| **LM Studio** | Desktop-App mit GUI, Download-Manager | GUI-Nutzer, Experimentieren |
| **vLLM** | High-Performance Inference Server | Produktion, Multi-GPU |
| **llama.cpp** | C++ Inference Engine, minimaler Overhead | Maximale Performance, Embedding |

### Vorteile lokaler Inference

- ✅ **Kostenlos** – Keine laufenden API-Kosten
- ✅ **Privat** – Daten verlassen nie den eigenen Rechner
- ✅ **Offline** – Funktioniert ohne Internet
- ✅ **Keine Rate Limits** – Beliebig viele Anfragen
- ✅ **DSGVO-konform** – Keine Datenweitergabe an Dritte

### Nachteile

- ❌ **Hardware-Anforderungen** – Gute Modelle brauchen viel VRAM (16-48 GB+)
- ❌ **Langsamere Modelle** – Kleine lokale Modelle sind qualitativ schwächer als Cloud-Flaggschiffe
- ❌ **Setup-Aufwand** – Installation und Konfiguration nötig
- ❌ **Kein Extended Thinking** – Proprietäre Features wie Claudes Extended Thinking nicht verfügbar

Siehe auch: [[Lokale LLMs]] und [[Context Window und Token Management]]

---

## Große Vergleichstabelle

| Provider | Typ | Modelle | Preis (1M Input) | Geschwindigkeit | Free Tier | BYOK | Best für |
|----------|-----|---------|-------------------|-----------------|-----------|------|----------|
| **Anthropic** | Direkt | Claude-Familie | $3.00 (Sonnet 4) | ~90 tok/s | ❌ | — | Coding, Analyse |
| **OpenAI** | Direkt | GPT, o-Serie | $2.50 (GPT-4o) | ~100 tok/s | Limitiert | — | Allround, Ecosystem |
| **Google** | Direkt | Gemini-Familie | $1.25 (2.5 Pro) | ~150 tok/s | ✅ Großzügig | — | Langer Context, Einstieg |
| **xAI** | Direkt | Grok | $2.00 (Grok 3) | ~80 tok/s | ❌ | — | Echtzeit-Wissen |
| **Mistral** | Direkt | Mistral/Codestral | $2.00 (Large) | ~100 tok/s | ✅ Klein | — | EU-Compliance |
| **DeepSeek** | Direkt | R1, V3 | $0.14 (R1) | ~50 tok/s | ✅ | — | Budget-Reasoning |
| **OpenRouter** | Router | 300+ (alle) | Durchgereicht | Vom Provider | ✅ Free Models | ✅ | Flexibilität, Vergleich |
| **Together AI** | Inference | 200+ OS | $0.10–7.00 | ~134 tok/s | $5 Start | ❌ | Modellvielfalt |
| **Groq** | Inference | ~20 OS | $0.05–0.99 | **275 tok/s** | ✅ | ❌ | Speed, Iteration |
| **DeepInfra** | Inference | 100+ OS | **$0.03–0.55** | ~100 tok/s | $1.80 Start | ❌ | Kosten sparen |
| **Fireworks** | Inference | 100+ OS | $0.10–5.00 | Bis 1000 tok/s | ✅ Klein | ❌ | Enterprise, Compliance |
| **Replicate** | Inference | 1000+ | GPU-Sekunden | Variabel | ❌ | ❌ | Prototyping |
| **Ollama** | Lokal | Alle GGUF | Kostenlos (HW) | HW-abhängig | ✅ | — | Privatsphäre, Offline |
| **LM Studio** | Lokal | Alle GGUF | Kostenlos (HW) | HW-abhängig | ✅ | — | GUI, Experimentieren |
| **vLLM** | Lokal | Alle HF | Kostenlos (HW) | HW-abhängig | ✅ | — | Produktion, Multi-GPU |

> [!NOTE]
> Preise Stand Mitte 2025. Preise ändern sich regelmäßig – tendenziell werden sie günstiger. Aktuelle Preise immer auf der jeweiligen Provider-Website prüfen.

---

## Provider in Coding Agents einrichten

### Aider

[[Aider Konfiguration|Aider]] unterstützt viele Provider direkt:

```bash
# Direkte Provider
export ANTHROPIC_API_KEY="sk-ant-..."
aider --model claude-sonnet-4

export OPENAI_API_KEY="sk-..."
aider --model gpt-4o

export GEMINI_API_KEY="AI..."
aider --model gemini/gemini-2.5-pro

# OpenRouter
export OPENROUTER_API_KEY="sk-or-v1-..."
aider --model openrouter/anthropic/claude-sonnet-4
aider --model openrouter/google/gemini-2.5-pro
aider --model openrouter/meta-llama/llama-3.1-405b-instruct

# Groq
export GROQ_API_KEY="gsk_..."
aider --model groq/llama-3.3-70b-versatile

# DeepSeek
export DEEPSEEK_API_KEY="sk-..."
aider --model deepseek/deepseek-chat

# Konfigurationsdatei (~/.aider.conf.yml)
model: openrouter/anthropic/claude-sonnet-4
weak-model: openrouter/google/gemini-2.5-flash
editor-model: openrouter/deepseek/deepseek-chat-v3
```

### OpenCode

[[OpenCode Konfiguration|OpenCode]] unterstützt Provider-Konfiguration:

```jsonc
// ~/.config/opencode/config.json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-..."
    },
    "openrouter": {
      "apiKey": "sk-or-v1-...",
      "baseURL": "https://openrouter.ai/api/v1"
    },
    "groq": {
      "apiKey": "gsk_...",
      "baseURL": "https://api.groq.com/openai/v1"
    }
  }
}
```

### Cline

[[Cline Konfiguration|Cline]] bietet ein Provider-Dropdown in den VS Code Settings unter *API Provider* – einfach den gewünschten Provider auswählen und API Key eingeben. Unterstützt Anthropic, OpenAI, OpenRouter, Google Gemini, AWS Bedrock, GCP Vertex und Ollama.

### Cursor

[[Cursor Agent Konfiguration|Cursor]]: Unter *Settings → Models → Use own API Key* aktivieren und Keys eintragen. OpenRouter als Custom Provider einrichten über *Override API Base URL* → `https://openrouter.ai/api/v1`.

### Open WebUI & Claude Code

- **Open WebUI:** Unter *Admin Panel → Settings → Connections* beliebig viele OpenAI-kompatible APIs hinzufügen
- **[[Claude Code Konfiguration und Hooks|Claude Code]]:** Nutzt `ANTHROPIC_API_KEY` direkt. Für OpenRouter: `ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"` setzen

---

## Kosten optimieren

### Strategie nach Einsatzzweck

```
┌────────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│  Entwicklung       │     │  Review/Test         │     │  Produktion      │
│  ────────────────  │     │  ─────────────────── │     │  ──────────────  │
│  Groq / DeepInfra  │     │  Claude Sonnet /     │     │  Claude Opus /   │
│  (schnell, billig) │────▶│  GPT-4o              │────▶│  GPT-4o          │
│  $0.05-0.55/1M     │     │  (gute Qualität)     │     │  (beste Qualität)│
│                    │     │  $2.50-3.00/1M       │     │  $15.00/1M       │
└────────────────────┘     └─────────────────────┘     └──────────────────┘
```

### Konkrete Tipps

**1. Modell-Stacking in Aider:**

```yaml
# ~/.aider.conf.yml
weak-model: groq/llama-3.3-70b-versatile      # Günstig für einfache Edits
editor-model: openrouter/deepseek/deepseek-chat-v3  # Mittel für Editor-Aufgaben
model: anthropic/claude-sonnet-4               # Premium für komplexe Fragen
```

**2. Anthropic Prompt Caching:** Anthropic cached automatisch lange System-Prompts. Bei Coding Agents, die große Codebasen im Context halten, reduziert das die Kosten um bis zu **90%** für den gecachten Teil.

**3. Token-Verbrauch abschätzen:** 1 Token ≈ 4 Zeichen (EN) / 2-3 Zeichen (DE). Eine Quelldatei (100 Zeilen) ≈ 1.000-2.000 Tokens. Eine typische Coding-Session ≈ 50.000-200.000 Tokens.

**Kostenbeispiel einer Coding-Session (100K Input + 20K Output):**

| Provider/Modell | Kosten |
|-----------------|--------|
| Groq Llama 3.3 70B | ~$0.07 |
| DeepInfra Llama 405B | ~$0.07 |
| DeepSeek R1 (direkt) | ~$0.06 |
| Claude Sonnet 4 | ~$0.60 |
| GPT-4o | ~$0.45 |
| Claude Opus 4.5 | ~$3.00 |

> [!IMPORTANT]
> Der Preisunterschied zwischen Open-Source-Modellen bei Inference Providern und proprietären Flaggschiffen kann **Faktor 10-50** betragen. Für Experimente und Iteration lohnt sich der Einsatz günstigerer Modelle fast immer.

**4. OpenRouter für Kostenvergleich:** Dashboard zeigt Kosten pro Anfrage – verschiedene Modelle direkt vergleichen.

**5. Lokale Modelle:** Für einfache Aufgaben (Commit Messages, Refactorings) lohnt sich [[Lokale LLMs#Ollama|Ollama]] mit einem 7-14B Modell – komplett kostenlos.

---

## Entscheidungshilfe: Welcher Provider wann?

### Nach Use Case

| Situation | Empfehlung | Begründung |
|-----------|-----------|------------|
| **Hobby / Lernen** | Google Free Tier, Groq Free, Ollama | Kein Geld ausgeben, trotzdem loslegen |
| **Professionelle Entwicklung** | Anthropic (Claude) oder OpenRouter | Beste Code-Qualität, flexible Modellwahl |
| **Enterprise / Team** | Anthropic oder OpenAI direkt | SLAs, Support, Compliance |
| **Speed-kritisch** | Groq | Schnellste Inferenz (275+ tok/s) |
| **Kosten-kritisch** | DeepInfra, DeepSeek, Groq | Günstigste Preise pro Token |
| **Privatsphäre / DSGVO** | Ollama + Open-Source-Modelle | Daten verlassen das Netzwerk nie |
| **Flexibilität** | OpenRouter | Ein Key, alle Modelle, schnell wechseln |
| **EU-Compliance** | Mistral (oder lokal) | Europäischer Anbieter, EU-Datenverarbeitung |
| **Regulierte Branchen** | Fireworks AI, oder Direkt-Provider | HIPAA/SOC2-Zertifizierung |
| **Experimentieren** | OpenRouter + Free Models | Verschiedene Modelle kostenlos testen |

### Entscheidungsbaum

```
Brauche ich maximale Privatsphäre?
├── JA → Ollama / LM Studio (lokal)
└── NEIN
    ├── Budget sehr begrenzt?
    │   ├── JA → Google Free Tier / Groq Free / DeepInfra
    │   └── NEIN
    │       ├── Brauche ich verschiedene Modelle?
    │       │   ├── JA → OpenRouter
    │       │   └── NEIN
    │       │       ├── Bestes Coding-Modell?
    │       │       │   └── Anthropic (Claude)
    │       │       ├── Bestes Allround-Modell?
    │       │       │   └── OpenAI (GPT-4o)
    │       │       └── Größter Context?
    │       │           └── Google (Gemini)
    │       └── Enterprise / Compliance nötig?
    │           ├── JA → Direkte Provider mit SLA
    │           └── Reguliert? → Fireworks AI
    └── Geschwindigkeit wichtigste Priorität?
        └── JA → Groq
```

---

## OpenAI-kompatible API

### Warum das wichtig ist

Die meisten LLM-Tools wurden ursprünglich für die OpenAI-API gebaut. Viele Provider bieten deshalb eine **OpenAI-kompatible API** an – sie akzeptieren die gleichen Request-Formate und antworten im gleichen Format. Das bedeutet: Ein Tool, das für OpenAI gebaut wurde, funktioniert mit diesen Providern, indem man **nur die Base URL und den API-Key ändert**.

### Wie es funktioniert

Alle Aufrufe nutzen das gleiche Format – nur Base URL und Key ändern:

```bash
# Gleicher Request-Body, unterschiedliche Provider:
curl $BASE_URL/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -d '{"model": "MODEL_NAME", "messages": [{"role": "user", "content": "Hello"}]}'

# Beispiel Base URLs:
# OpenAI:     https://api.openai.com/v1
# OpenRouter: https://openrouter.ai/api/v1
# Groq:       https://api.groq.com/openai/v1
# Ollama:     http://localhost:11434/v1
```

### Provider mit OpenAI-kompatibler API

| Provider | Base URL | Kompatibilität |
|----------|---------|----------------|
| OpenAI | `https://api.openai.com/v1` | ✅ Original |
| OpenRouter | `https://openrouter.ai/api/v1` | ✅ Voll kompatibel |
| Groq | `https://api.groq.com/openai/v1` | ✅ Voll kompatibel |
| Together AI | `https://api.together.xyz/v1` | ✅ Voll kompatibel |
| DeepInfra | `https://api.deepinfra.com/v1/openai` | ✅ Voll kompatibel |
| Fireworks | `https://api.fireworks.ai/inference/v1` | ✅ Voll kompatibel |
| Mistral | `https://api.mistral.ai/v1` | ✅ Voll kompatibel |
| DeepSeek | `https://api.deepseek.com/v1` | ✅ Voll kompatibel |
| xAI | `https://api.x.ai/v1` | ✅ Voll kompatibel |
| Ollama | `http://localhost:11434/v1` | ✅ Voll kompatibel |
| LM Studio | `http://localhost:1234/v1` | ✅ Voll kompatibel |
| Anthropic | `https://api.anthropic.com/v1` | ❌ Eigenes Format |

> [!WARNING]
> **Anthropic ist die Ausnahme.** Die Claude-API hat ein eigenes Request-Format (z.B. `system` als separates Feld, `anthropic-version` Header). Tools, die nur OpenAI-Format unterstützen, können nicht direkt mit Anthropic kommunizieren – deshalb ist **OpenRouter** so nützlich: Es übersetzt OpenAI-Format-Anfragen automatisch in Anthropics Format.

### Praktische Konsequenz

Dank der OpenAI-Kompatibilität kann man in den meisten Tools einfach **zwei Umgebungsvariablen** setzen, um den Provider zu wechseln:

```bash
# Statt OpenAI → Groq verwenden
export OPENAI_API_KEY="gsk_..."          # Groq-Key
export OPENAI_BASE_URL="https://api.groq.com/openai/v1"

# Statt OpenAI → Together AI verwenden
export OPENAI_API_KEY="..."              # Together-Key
export OPENAI_BASE_URL="https://api.together.xyz/v1"

# Statt OpenAI → Lokal (Ollama) verwenden
export OPENAI_API_KEY="not-needed"
export OPENAI_BASE_URL="http://localhost:11434/v1"
```

> [!TIP]
> Das ist auch der Grund, warum **OpenRouter** so mächtig ist: Es bietet eine einzige OpenAI-kompatible API, hinter der alle Provider stehen – auch Anthropic. Tools müssen nur einen Endpunkt unterstützen, um Zugang zu 300+ Modellen zu haben.

---

## Weiterführende Links

- [[Modell Übersicht]] — Detaillierte Beschreibung aller Modell-Familien
- [[Modell Vergleiche]] — Benchmarks und Qualitätsvergleiche
- [[Lokale LLMs]] — Ausführliche Anleitung für lokale Modelle
- [[Context Window und Token Management]] — Token-Limits und Strategien
- [[AI Coding Agents Vergleich]] — Übersicht aller Coding Agents
- [[Aider Konfiguration]] — Aider Setup und Provider-Konfiguration
- [[OpenCode Konfiguration]] — OpenCode Setup
- [[Cline Konfiguration]] — Cline Setup und Provider-Auswahl
