# Modell Übersicht – Large Language Models

> [!NOTE]
> Diese Übersicht spiegelt den Stand von **März 2026** wider. Das LLM-Feld entwickelt sich sehr schnell – Modelle, Preise und Fähigkeiten ändern sich regelmäßig.

## Was ist ein Large Language Model?

Ein Large Language Model (LLM) ist ein neuronales Netz auf Basis der Transformer-Architektur, das auf enormen Textmengen trainiert wurde. LLMs lernen statistische Muster in Sprache und können damit Text generieren, übersetzen, zusammenfassen, Code schreiben und komplexe Aufgaben lösen.

**Kernbegriffe:**
- **Parameter:** Die Gewichte des Modells – mehr Parameter bedeuten (grob) mehr Kapazität
- **Training:** Vortraining auf rohem Text (Next-Token-Prediction), danach oft Finetuning (RLHF, RLAIF)
- **Inferenz:** Das Ausführen des Modells zur Laufzeit (Antworten generieren)

---

## Modell-Familien im Überblick

### Anthropic – Claude

Anthropic wurde 2021 von ehemaligen OpenAI-Mitarbeitern gegründet mit dem Fokus auf **AI Safety**. Claude-Modelle sind bekannt für sehr langen Context, präzise Instruction-Following und geringes Halluzinationsrisiko.

**Constitutional AI (CAI):** Anthropics eigener Ansatz, Modelle durch selbst-generiertes Feedback sicherer zu machen.

| Modell | Context Window | Preis (Input/Output) | Stärken | Einsatzgebiete |
|--------|---------------|---------------------|---------|----------------|
| **Claude Opus 4.6** | 1M (Beta) / 200K | $5 / $25 pro 1M Tokens | Höchste Intelligenz, Extended Thinking, 128K max Output | Forschung, komplexe Analysen, schwierige Code-Aufgaben |
| **Claude Sonnet 4.6** | 1M (Beta) / 200K | $3 / $15 pro 1M Tokens | 98% der Opus-Qualität, ausgewogenes Kosten-Leistungs-Verhältnis | Produktionsanwendungen, Agenten, allgemeine Aufgaben |
| **Claude Haiku 4.5** | 200K | Günstigstes Claude-Modell | Schnellstes Claude-Modell (kein 4.6-Update bisher) | Echtzeitanwendungen, einfache Klassifizierung, Chatbots |

> [!TIP]
> Claude Sonnet 4.6 ist für die meisten produktiven Anwendungsfälle die beste Wahl – es bietet 98% der Opus-Qualität zu einem Bruchteil der Kosten.

**Besonderheiten von Claude:**
- **Extended Thinking:** Claude Opus 4.6 kann sichtbare Denkschritte ausführen (ähnlich wie o1/o3)
- **Tool Use / Function Calling:** Nativ unterstützt
- **Prompt Caching:** Reduziert Kosten bei wiederholten langen System-Prompts erheblich
- **Vision:** Alle aktuellen Claude-Modelle unterstützen Bild-Input
- **1M Token Context (Beta):** Opus 4.6 und Sonnet 4.6 unterstützen bis zu 1 Million Tokens im Beta-Modus

---

### OpenAI – GPT & Reasoning-Modelle

OpenAI ist der bekannteste LLM-Anbieter und betreibt ChatGPT sowie die API.

#### GPT-5.x Familie

| Modell | Beschreibung | Preis (Input/Output) | Stärken |
|--------|-------------|---------------------|---------|
| **GPT-5.3 Codex** | Neuestes Modell (März 2026), Real-Time-Coding-Fokus | TBD | Echtzeit-Code-Generierung und -Analyse |
| **GPT-5.2** | Aktuelles Flaggschiff, 400K Context | $1,75 / $14 pro 1M Tokens | Drei Varianten: Instant / Thinking / Pro |
| **GPT-5.1** | Wird schrittweise eingestellt (Sunset) | – | Noch verfügbar, Migration empfohlen |

> [!WARNING]
> GPT-5 wird aktuell eingestellt (Retired). GPT-4o, o1 und o3 sind abgekündigt und nur noch als Legacy verfügbar. Neue Projekte sollten auf GPT-5.2+ setzen.

**GPT-5.2 Varianten:**
- **Instant:** Schnellste Antwort, keine Denkschritte
- **Thinking:** Chain-of-Thought Reasoning (vergleichbar mit ehemaligem o3)
- **Pro:** Höchste Qualität, längere Denkzeit, teurer

---

### Google – Gemini

Google hat seine KI-Strategie rund um die **Gemini**-Familie neu ausgerichtet (Nachfolger von PaLM 2 / Bard).

| Modell | Context Window | Status | Stärken |
|--------|---------------|--------|---------|
| **Gemini 3.1 Pro** | TBD | Neuestes Modell (Feb 2026) | Fortgeschrittenes Reasoning, State-of-the-Art |
| **Gemini 3 Pro Preview** | TBD | Wird durch 3.1 ersetzt | Erste Generation 3 |
| **Gemini 2.5 Pro** | 1 Mio. Tokens | Stabil, empfohlener Fallback | Sehr langer Context, zuverlässig |
| **Gemini 2.5 Flash** | 1 Mio. Tokens | Aktiv | Schnell und günstig für Produktion |
| **Gemini 2.5 Flash-Lite** | 1 Mio. Tokens | Aktiv | Budget-Option, extrem kosteneffizient |
| **Gemini 2.0 Flash** | 1 Mio. Tokens | Wird abgekündigt (Juni 2026) | Noch nutzbar, Migration empfohlen |

**Besonderheiten:**
- Native Integration in Google Workspace (Docs, Sheets, Gmail)
- **Google Search Grounding:** Kann in Echtzeit auf Google-Suche zugreifen
- Multimodal von Haus aus: Text, Bild, Audio, Video, Code

> [!NOTE]
> Gemini 2.5 Pro mit 1 Million Token Context bleibt der empfohlene Fallback für sehr lange Dokumente wie gesamte Codebases oder Bücher. Gemini 3.1 Pro bietet verbessertes Reasoning. Gemini 2.0 Flash wird im Juni 2026 abgekündigt – rechtzeitig auf 2.5 Flash migrieren.

---

### Meta – Llama 4 & 3.x (Open-Weight)

Metas **Llama**-Familie ist die wichtigste Open-Weight-LLM-Serie. "Open-Weight" bedeutet: Die Modellgewichte werden öffentlich zur Verfügung gestellt, aber das Training-Setup und die Daten nicht vollständig.

#### Llama 4 (aktuell)

| Modell | Architektur | Context Window | Besonderheit |
|--------|-------------|---------------|-------------|
| **Llama 4 Scout** | MoE 109B / 17B aktiv | 10M Tokens (!) | Multimodal, läuft auf einzelner H100 |
| **Llama 4 Maverick** | MoE 400B / 17B aktiv | 1M Tokens | Übertrifft GPT-4o, multimodal |
| **Llama 4 Behemoth** | 2T Parameter | TBD | Limitierte Beta, größtes Open-Weight-Modell |

#### Llama 3.x (weiterhin verfügbar)

| Modell | Parameter | Besonderheit |
|--------|-----------|-------------|
| **Llama 3.1 8B** | 8 Milliarden | Sehr effizient, gut für lokale Nutzung |
| **Llama 3.1 70B** | 70 Milliarden | Bewährte Open-Weight-Qualität |
| **Llama 3.1 405B** | 405 Milliarden | Konkurriert mit GPT-4o |
| **Llama 3.2 1B/3B** | 1-3 Milliarden | Für Edge-Devices und mobile Nutzung |
| **Llama 3.2 11B/90B** | 11-90 Milliarden | Vision-Modelle (Text + Bild) |
| **Llama 3.3 70B** | 70 Milliarden | Verbessertes 70B-Modell |

**Lizenz:** Llama-Modelle haben eine eigene Meta-Lizenz (kein reines Open Source). Kommerzielle Nutzung ist erlaubt, aber mit Einschränkungen (z.B. für Unternehmen über 700 Mio. monatliche aktive Nutzer gelten Sonderbedingungen).

> [!TIP]
> Llama 4 Scout ist ein Durchbruch: 10 Millionen Token Context bei nur 17B aktiven Parametern dank MoE-Architektur – läuft auf einer einzelnen H100. Für lokale Nutzung mit moderaterem VRAM-Bedarf bleibt Llama 3.1 70B weiterhin der Sweet Spot.

---

### Mistral AI – Mistral-Familie (Open & Closed)

Mistral AI ist ein französisches KI-Startup, das für sehr effiziente Modelle bekannt ist. Sie veröffentlichen sowohl Open-Weight als auch proprietäre Modelle.

| Modell | Typ | Context Window | Stärken |
|--------|-----|---------------|---------|
| **Mistral Large 3** | Proprietär | 256K Tokens | 675B Parameter, Flagship-Modell, Multilingual, starkes Reasoning |
| **Mistral Medium 3** | Proprietär | 131K Tokens | Gute Balance aus Qualität und Kosten |
| **Mistral Small 3.2** | Open-Weight | 262K Tokens | 24B Parameter, edge-freundlich, sehr effizient |
| **Codestral 25.01** | Proprietär | 256K Tokens | 22B Parameter, Code-spezialisiert, 80+ Sprachen |
| **Devstral** | Open-Weight | – | Entwickler-fokussiertes Modell |
| **Magistral** | Proprietär | – | Reasoning-optimiertes Modell |

**Mixture of Experts (MoE):** Einige Mistral-Modelle nutzen MoE-Architektur und aktivieren bei jeder Inferenz nur einen Teil der Parameter – dadurch sind sie schneller als ihr Parametercount vermuten lässt.

> [!NOTE]
> Mistral Small 3.2 (24B) ist eine hervorragende Wahl für Edge-Deployment und lokale Nutzung – 262K Context bei nur 24B Parametern. Codestral 25.01 ist der Code-Spezialist mit Unterstützung für 80+ Programmiersprachen.

---

### DeepSeek – DeepSeek V3 & R1

**DeepSeek** ist ein chinesisches KI-Labor, das sich mit extrem kosteneffizienten Modellen als ernstzunehmender Konkurrent etabliert hat. Die Modelle bieten Top-Qualität zu einem Bruchteil der üblichen Preise.

| Modell | Context Window | Preis (Input/Output) | Stärken |
|--------|---------------|---------------------|---------|
| **DeepSeek V3.2** | 128K-164K Tokens | $0,25 / $0,38 pro 1M Tokens | Extrem günstig, sehr gute Allround-Qualität |
| **DeepSeek R1** | 128K Tokens | $0,70 / $2,50 pro 1M Tokens | Reasoning-Modell, Chain-of-Thought, Open-Weight |

**Besonderheiten:**
- **Preisbrecher:** DeepSeek V3.2 kostet einen Bruchteil vergleichbarer Modelle (~10× günstiger als GPT-5.2)
- **Open-Weight:** DeepSeek R1 ist Open-Weight und kann lokal betrieben werden
- **Reasoning:** R1 bietet sichtbare Denkschritte ähnlich wie o3 / Claude Extended Thinking

> [!TIP]
> DeepSeek V3.2 ist die beste Budget-Option für Produktionsanwendungen: $0,25 / $0,38 pro 1M Tokens bei konkurrenzfähiger Qualität. Ideal wenn Kosten eine Rolle spielen.

---

### Microsoft – Phi-3.5 (Small Language Models)

Microsofts **Phi**-Familie zeigt, dass kleine Modelle mit den richtigen Trainingsdaten erstaunlich leistungsfähig sein können.

| Modell | Parameter | Besonderheit |
|--------|-----------|-------------|
| **Phi-3.5 Mini** | 3,8 Milliarden | Hervorragend für seine Größe, läuft auf Smartphones |
| **Phi-3.5 MoE** | 16 × 3,8B (aktiv: ~6,6B) | MoE-Variante für bessere Reasoning-Fähigkeiten |
| **Phi-3.5 Vision** | 4,2 Milliarden | Multimodal (Text + Bild) |

**Philosophie:** "Small but Mighty" – Phi-Modelle werden auf hochqualitativen, kuratierten Daten trainiert statt auf Massen von Web-Daten. Dadurch erreichen 3-4B-Modelle die Qualität früherer 7B-Modelle.

**Einsatzgebiete:**
- Edge-Computing und IoT
- Mobile Apps
- Geräte ohne Internetverbindung
- Kostensensitive Anwendungen

---

### Alibaba – Qwen2.5

Die **Qwen**-Familie von Alibaba ist besonders stark im asiatischen Sprachraum und hat sich zu einem ernsthaften Konkurrenten entwickelt.

| Modell | Parameter | Besonderheit |
|--------|-----------|-------------|
| **Qwen2.5 7B** | 7 Milliarden | Sehr gute Performance für die Größe |
| **Qwen2.5 72B** | 72 Milliarden | Konkurriert mit Llama 3.1 70B |
| **Qwen2.5-Coder 32B** | 32 Milliarden | Spezialisiertes Code-Modell |
| **Qwen2.5-Math** | 7B / 72B | Mathematik-spezialisiert |

**Lizenz:** Qwen2.5-Modelle unter 72B verwenden Apache 2.0, das 72B-Modell hat eine eigene Qwen-Lizenz.

**Stärken:** Chinesisch und Englisch auf sehr hohem Niveau, gute mehrsprachige Fähigkeiten, starke Code-Performance.

---

### Google – Gemma2

**Gemma2** ist Googles Open-Weight-Modellfamilie (parallel zu Gemini als Cloud-API).

| Modell | Parameter | Besonderheit |
|--------|-----------|-------------|
| **Gemma2 2B** | 2 Milliarden | Kleinste Variante, für Edge-Anwendungen |
| **Gemma2 9B** | 9 Milliarden | Sehr gute Qualität für Open-Weight-Modell |
| **Gemma2 27B** | 27 Milliarden | Beste Gemma-Qualität, läuft auf einer A100 |

**Lizenz:** Gemma License (kein echtes Open Source, aber für Forschung und Kommerzielle Nutzung zugelassen).

**Technologie:** Gemma2 nutzt Group Query Attention und Sliding Window Attention für verbesserte Effizienz.

---

## Große Vergleichstabelle

| Modell | Anbieter | Typ | Context Window | Besonderheit | Lizenz |
|--------|---------|-----|----------------|-------------|--------|
| Claude Opus 4.6 | Anthropic | Proprietär | 1M (Beta) / 200K | Extended Thinking, höchste Qualität, 128K max Output | Proprietär |
| Claude Sonnet 4.6 | Anthropic | Proprietär | 1M (Beta) / 200K | 98% Opus-Qualität, beste Balance | Proprietär |
| Claude Haiku 4.5 | Anthropic | Proprietär | 200K Tokens | Schnellstes Claude-Modell | Proprietär |
| GPT-5.3 Codex | OpenAI | Proprietär | TBD | Neuestes Modell, Real-Time-Coding | Proprietär |
| GPT-5.2 | OpenAI | Proprietär | 400K Tokens | Flaggschiff, 3 Varianten (Instant/Thinking/Pro) | Proprietär |
| GPT-5.1 | OpenAI | Proprietär | – | Wird eingestellt (Sunset) | Proprietär |
| Gemini 3.1 Pro | Google | Proprietär | TBD | Neuestes Gemini, Advanced Reasoning | Proprietär |
| Gemini 2.5 Pro | Google | Proprietär | 1M Tokens | Stabil, empfohlener Fallback | Proprietär |
| Gemini 2.5 Flash | Google | Proprietär | 1M Tokens | Schnell und günstig | Proprietär |
| Gemini 2.5 Flash-Lite | Google | Proprietär | 1M Tokens | Budget-Option | Proprietär |
| DeepSeek V3.2 | DeepSeek | Proprietär | 128-164K Tokens | Extrem günstig ($0,25/$0,38 pro 1M) | Proprietär |
| DeepSeek R1 | DeepSeek | Open-Weight | 128K Tokens | Reasoning, sehr günstig | MIT |
| Llama 4 Scout | Meta | Open-Weight | 10M Tokens | MoE 109B/17B aktiv, multimodal, 1× H100 | Meta Llama |
| Llama 4 Maverick | Meta | Open-Weight | 1M Tokens | MoE 400B/17B aktiv, übertrifft GPT-4o | Meta Llama |
| Llama 4 Behemoth | Meta | Open-Weight | TBD | 2T Parameter, limitierte Beta | Meta Llama |
| Llama 3.1 70B | Meta | Open-Weight | 128K Tokens | Bewährte Open-Weight-Balance | Meta Llama 3 |
| Llama 3.1 405B | Meta | Open-Weight | 128K Tokens | GPT-4o-Niveau, Open-Weight | Meta Llama 3 |
| Mistral Large 3 | Mistral | Proprietär | 256K Tokens | 675B, Multilingual, starkes Reasoning | Proprietär |
| Mistral Small 3.2 | Mistral | Open-Weight | 262K Tokens | 24B, edge-freundlich | Apache 2.0 |
| Codestral 25.01 | Mistral | Proprietär | 256K Tokens | 22B, Code-spezialisiert, 80+ Sprachen | Proprietär |
| Phi-3.5 Mini | Microsoft | Open-Weight | 128K Tokens | Kleinste Modelle mit hoher Qualität | MIT |
| Phi-3.5 MoE | Microsoft | Open-Weight | 128K Tokens | MoE, ~6,6B aktive Parameter | MIT |
| Qwen2.5 72B | Alibaba | Open-Weight | 128K Tokens | Starke Mehrsprachigkeit | Qwen License |
| Qwen2.5-Coder 32B | Alibaba | Open-Weight | 128K Tokens | Top Code-Modell im Open-Weight-Bereich | Apache 2.0 |
| Gemma2 9B | Google | Open-Weight | 8K Tokens | Sehr effizient für die Größe | Gemma License |
| Gemma2 27B | Google | Open-Weight | 8K Tokens | Beste Gemma-Qualität | Gemma License |

---

## Closed vs. Open-Weight Modelle

### Closed-Source / Proprietäre Modelle

Die Modellgewichte sind nicht öffentlich zugänglich. Der Zugriff erfolgt ausschließlich über eine API.

**Vorteile:**
- Höchste Qualität (in der Regel)
- Einfache Integration über API
- Kein Hardware-Aufwand
- Kontinuierliche Updates durch den Anbieter
- Skaliert automatisch

**Nachteile:**
- Datenschutz: Daten verlassen das eigene System
- Laufende Kosten pro Token
- Abhängigkeit vom Anbieter (Vendor Lock-in)
- Keine Anpassung (Finetuning nur über offizielle APIs/Services)
- Ausfall des Anbieters = Ausfall des eigenen Dienstes

**Beispiele:** Claude, GPT-5.2, Gemini Pro, DeepSeek V3.2

### Open-Weight Modelle

Die Modellgewichte werden zum Download bereitgestellt. Der Begriff "Open Source" ist hier oft ungenau – viele Modelle haben Nutzungsbeschränkungen.

**Vorteile:**
- Vollständige Datenkontrolle (lokal oder auf eigener Infrastruktur)
- Keine laufenden API-Kosten nach der Hardware-Investition
- Finetuning auf eigene Daten möglich
- Offline-Fähigkeit
- Kein Vendor Lock-in

**Nachteile:**
- Schlechtere Durchschnittsqualität als proprietäre Top-Modelle
- Erhebliche Hardware-Anforderungen für große Modelle
- Selbst-Hosting erfordert DevOps-Kompetenz
- Sicherheit und Updates müssen selbst verwaltet werden

**Beispiele:** Llama 4 Scout/Maverick, Llama 3.x, Mistral Small 3.2, DeepSeek R1, Phi-3.5, Gemma2

> [!WARNING]
> "Open-Weight" ist nicht dasselbe wie "Open Source". Bei echtem Open Source sind Trainingsdata, Code und Gewichte frei verfügbar. Bei Open-Weight-Modellen wie Llama sind nur die Gewichte frei – aber Trainingsdaten und vollständiger Code sind nicht öffentlich. Nur wenige Modelle (z.B. Mistral Nemo, Phi-3.5 mit MIT-Lizenz) sind wirklich für alle kommerziellen Zwecke unbeschränkt nutzbar.

### Lizenz-Übersicht

| Lizenz | Beschreibung | Kommerzielle Nutzung |
|--------|-------------|---------------------|
| Apache 2.0 | Echtes Open Source, sehr permissiv | Ja, ohne Einschränkungen |
| MIT | Sehr permissiv, minimale Anforderungen | Ja, ohne Einschränkungen |
| Meta Llama License | Proprietäre Lizenz mit Ausnahmen | Ja, mit Einschränkungen (>700M MAU) |
| Gemma License | Google-spezifisch | Ja, mit Nutzungsbedingungen |
| Qwen License | Alibaba-spezifisch | Ja, mit Nutzungsbedingungen |
| Proprietär | Nur API-Zugriff | Nur über API |

---

## Verwandte Notizen

- [[Context Window und Token Management]]
- [[Lokale LLMs]]
- [[Modell Vergleiche]]
- [[Prompting/Grundlagen]]
- [[Agents/Überblick]]
