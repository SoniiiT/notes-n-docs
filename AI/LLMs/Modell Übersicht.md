# Modell Übersicht – Large Language Models

> [!NOTE]
> Diese Übersicht spiegelt den Stand von **August 2025** wider. Das LLM-Feld entwickelt sich sehr schnell – Modelle, Preise und Fähigkeiten ändern sich regelmäßig.

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

| Modell | Stärken | Einsatzgebiete |
|--------|---------|----------------|
| **Claude Opus 4.5** | Höchste Intelligenz, Extended Thinking, komplexes Reasoning | Forschung, komplexe Analysen, schwierige Code-Aufgaben |
| **Claude Sonnet 4.5** | Ausgewogenes Kosten-Leistungs-Verhältnis, sehr schnell | Produktionsanwendungen, Agenten, allgemeine Aufgaben |
| **Claude Haiku 4.5** | Schnellstes Claude-Modell, günstig | Echtzeitanwendungen, einfache Klassifizierung, Chatbots |

> [!TIP]
> Claude Sonnet 4.5 ist für die meisten produktiven Anwendungsfälle die beste Wahl – es bietet ~80% der Opus-Qualität zu einem Bruchteil der Kosten.

**Besonderheiten von Claude:**
- **Extended Thinking:** Claude Opus 4.5 kann sichtbare Denkschritte ausführen (ähnlich wie o1/o3)
- **Tool Use / Function Calling:** Nativ unterstützt
- **Prompt Caching:** Reduziert Kosten bei wiederholten langen System-Prompts erheblich
- **Vision:** Alle aktuellen Claude-Modelle unterstützen Bild-Input

---

### OpenAI – GPT & Reasoning-Modelle

OpenAI ist der bekannteste LLM-Anbieter und betreibt ChatGPT sowie die API.

#### GPT-4o Familie

| Modell | Beschreibung | Stärken |
|--------|-------------|---------|
| **GPT-4o** | Multimodales Flaggschiff-Modell | Text, Bild, Audio; sehr breite Fähigkeiten |
| **GPT-4o-mini** | Kleines, günstiges GPT-4-Modell | Schnell, günstig, gut für einfache Tasks |

**GPT-4o** steht für "omni" – das Modell verarbeitet Text, Bilder und Audio nativ.

#### Reasoning-Modelle: o1 und o3

Diese Modelle unterscheiden sich grundlegend von klassischen LLMs: Sie führen einen internen **Chain-of-Thought**-Prozess durch, bevor sie antworten. Das macht sie langsamer und teurer, aber deutlich besser bei:
- Mathematik und Logik
- Komplexem Code-Debugging
- Mehrstufigen Planungsaufgaben

| Modell | Besonderheit |
|--------|-------------|
| **o1** | Reasoning-Modell, erstes seiner Klasse von OpenAI |
| **o3** | Signifikant stärker als o1, State-of-the-Art auf vielen Benchmarks |
| **o3-mini** | Günstigere Reasoning-Variante |

> [!IMPORTANT]
> Bei o1/o3 zahlt man auch für die "Thinking Tokens" – die versteckten Denkschritte. Das kann teuer werden. Nur einsetzen wenn nötig.

---

### Google – Gemini

Google hat seine KI-Strategie rund um die **Gemini**-Familie neu ausgerichtet (Nachfolger von PaLM 2 / Bard).

| Modell | Context Window | Stärken |
|--------|---------------|---------|
| **Gemini 1.5 Pro** | 1 Mio. Tokens (experimentell: 2 Mio.) | Sehr langer Context, Multimodal, Video-Verständnis |
| **Gemini 1.5 Flash** | 1 Mio. Tokens | Schnell und günstig, für Produktion geeignet |
| **Gemini 2.0 Pro** | 2 Mio. Tokens | Stärkstes Gemini-Modell, Multimodal |
| **Gemini 2.0 Flash** | 1 Mio. Tokens | Schnell, günstig, sehr gute Allroundleistung |

**Besonderheiten:**
- Native Integration in Google Workspace (Docs, Sheets, Gmail)
- **Google Search Grounding:** Kann in Echtzeit auf Google-Suche zugreifen
- Multimodal von Haus aus: Text, Bild, Audio, Video, Code

> [!NOTE]
> Gemini 1.5 Pro mit 1 Million Token Context ist ideal für sehr lange Dokumente wie gesamte Codebases oder Bücher. In der Praxis nimmt die Recall-Qualität bei sehr vollem Context aber ab ("lost in the middle"-Problem).

---

### Meta – Llama 3.x (Open-Weight)

Metas **Llama**-Familie ist die wichtigste Open-Weight-LLM-Serie. "Open-Weight" bedeutet: Die Modellgewichte werden öffentlich zur Verfügung gestellt, aber das Training-Setup und die Daten nicht vollständig.

| Modell | Parameter | Besonderheit |
|--------|-----------|-------------|
| **Llama 3.1 8B** | 8 Milliarden | Sehr effizient, gut für lokale Nutzung |
| **Llama 3.1 70B** | 70 Milliarden | Beste Open-Weight-Qualität für diese Größe |
| **Llama 3.1 405B** | 405 Milliarden | Konkurriert mit GPT-4o |
| **Llama 3.2 1B/3B** | 1-3 Milliarden | Für Edge-Devices und mobile Nutzung |
| **Llama 3.2 11B/90B** | 11-90 Milliarden | Vision-Modelle (Text + Bild) |
| **Llama 3.3 70B** | 70 Milliarden | Verbessertes 70B-Modell |

**Lizenz:** Llama-Modelle haben eine eigene Meta-Lizenz (kein reines Open Source). Kommerzielle Nutzung ist erlaubt, aber mit Einschränkungen (z.B. für Unternehmen über 700 Mio. monatliche aktive Nutzer gelten Sonderbedingungen).

> [!TIP]
> Llama 3.1 70B ist der Sweet Spot für lokale Nutzung: Sehr gute Qualität, läuft mit Quantisierung auf 2× RTX 3090/4090 oder einer GPU mit 48GB+ VRAM (z.B. A6000).

---

### Mistral AI – Mistral-Familie (Open & Closed)

Mistral AI ist ein französisches KI-Startup, das für sehr effiziente Modelle bekannt ist. Sie veröffentlichen sowohl Open-Weight als auch proprietäre Modelle.

| Modell | Typ | Stärken |
|--------|-----|---------|
| **Mistral Large** | Proprietär | Flagship-Modell, Multilingual, starkes Reasoning |
| **Mistral Small** | Proprietär | Günstiger, gute Balance |
| **Mistral Nemo** | Open-Weight (Apache 2.0) | 12B Parameter, sehr gute Effizienz, echtes Open Source |
| **Codestral** | Proprietär | Speziell für Code, unterstützt 80+ Programmiersprachen |
| **Mixtral 8x7B** | Open-Weight | Mixture-of-Experts-Architektur, sehr effizient |
| **Mixtral 8x22B** | Open-Weight | Größeres MoE-Modell |

**Mixture of Experts (MoE):** Mixtral-Modelle aktivieren bei jeder Inferenz nur einen Teil der Parameter – dadurch sind sie schneller als ihr Parametercount vermuten lässt.

> [!NOTE]
> Mistral Nemo (Apache 2.0 Lizenz) ist eine hervorragende Wahl für kommerzielle Open-Source-Projekte – keine Lizenzeinschränkungen.

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
| Claude Opus 4.5 | Anthropic | Proprietär | 200K Tokens | Extended Thinking, höchste Qualität | Proprietär |
| Claude Sonnet 4.5 | Anthropic | Proprietär | 200K Tokens | Beste Balance Qualität/Kosten | Proprietär |
| Claude Haiku 4.5 | Anthropic | Proprietär | 200K Tokens | Schnellstes Claude-Modell | Proprietär |
| GPT-4o | OpenAI | Proprietär | 128K Tokens | Multimodal (Text/Bild/Audio) | Proprietär |
| GPT-4o-mini | OpenAI | Proprietär | 128K Tokens | Günstige GPT-4-Qualität | Proprietär |
| o1 | OpenAI | Proprietär | 200K Tokens | Reasoning-Modell | Proprietär |
| o3 | OpenAI | Proprietär | 200K Tokens | Stärkstes Reasoning | Proprietär |
| Gemini 1.5 Pro | Google | Proprietär | 1M Tokens | Längster stabiler Context | Proprietär |
| Gemini 1.5 Flash | Google | Proprietär | 1M Tokens | Schnell und günstig | Proprietär |
| Gemini 2.0 Pro | Google | Proprietär | 2M Tokens | Stärkstes Gemini | Proprietär |
| Gemini 2.0 Flash | Google | Proprietär | 1M Tokens | Beste Allround-Flash-Option | Proprietär |
| Llama 3.1 8B | Meta | Open-Weight | 128K Tokens | Effizient, lokal nutzbar | Meta Llama 3 |
| Llama 3.1 70B | Meta | Open-Weight | 128K Tokens | Beste Open-Weight-Balance | Meta Llama 3 |
| Llama 3.1 405B | Meta | Open-Weight | 128K Tokens | GPT-4o-Niveau, Open-Weight | Meta Llama 3 |
| Llama 3.2 3B | Meta | Open-Weight | 128K Tokens | Für Edge/Mobile | Meta Llama 3 |
| Llama 3.3 70B | Meta | Open-Weight | 128K Tokens | Verbessertes 70B | Meta Llama 3 |
| Mistral Large | Mistral | Proprietär | 128K Tokens | Multilingual, starkes Reasoning | Proprietär |
| Mistral Nemo 12B | Mistral | Open-Weight | 128K Tokens | Echtes Open Source (Apache 2.0) | Apache 2.0 |
| Codestral | Mistral | Proprietär | 32K Tokens | Code-spezialisiert, 80+ Sprachen | Proprietär |
| Mixtral 8x7B | Mistral | Open-Weight | 32K Tokens | MoE-Architektur, effizient | Apache 2.0 |
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

**Beispiele:** Claude, GPT-4o, Gemini Pro

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

**Beispiele:** Llama 3.x, Mistral Nemo, Phi-3.5, Gemma2

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
