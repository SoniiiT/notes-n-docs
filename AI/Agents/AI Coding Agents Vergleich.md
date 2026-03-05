# AI Coding Agents Vergleich (2025)

> [!NOTE] Über dieses Dokument
> Dieser Vergleich deckt **alle AI Coding Agents** ab, die in diesem Vault dokumentiert sind. Ziel: Die richtige Wahl für den richtigen Anwendungsfall treffen. Letzte Aktualisierung: 2025.

---

## 1. Übersicht: Die Landschaft der AI Coding Agents (2025)

Die Welt der AI Coding Agents hat sich 2025 in drei klare Kategorien aufgeteilt:

| Kategorie | Beschreibung | Vertreter |
|---|---|---|
| **CLI (Commercial)** | Terminal-basiert, von großen AI-Unternehmen | Claude Code, Copilot CLI, Gemini CLI, Codex CLI |
| **CLI (Open Source)** | Terminal-basiert, Community-getrieben | OpenCode, Aider, Goose, Open Interpreter |
| **IDE-basiert** | In den Editor integriert | Cursor, Cline |
| **Multi-Channel** | Über Chat-Apps steuerbar | OpenClaw |

Warum so viele? Jeder Ansatz hat andere Stärken: CLI-Agents sind schnell und scriptbar, IDE-Agents bieten visuelles Feedback, und Multi-Channel-Agents ermöglichen Zugriff von überall. Es gibt keinen "besten" Agent — nur den besten **für deinen Workflow**.

Die wichtigsten Trends 2025:
- **MCP (Model Context Protocol)** wird zum De-facto-Standard für Tool-Integration
- **Multi-Provider-Support** ist der Normalfall — kein Lock-in mehr nötig
- **Custom Instructions** (`CLAUDE.md`, `AGENTS.md`, etc.) geben Entwicklern Kontrolle über Agent-Verhalten
- **Lokale Modelle** via Ollama machen AI-Coding auch ohne Cloud möglich
- **Multi-Agent-Workflows** (ein Agent orchestriert andere) entstehen als neues Paradigma

> [!TIP] Individuelle Dokumentation
> Jeder Agent hat eine eigene Konfigurationsseite in diesem Vault. Dieser Vergleich hilft bei der Auswahl, die Details findest du in den verlinkten Dokumenten.
> 
> Für Grundlagen siehe [[Was ist ein AI Agent]]. Für Custom Instructions über alle Agents hinweg siehe [[Custom Instructions für AI Coding Agents]].

---

## 2. Große Vergleichstabelle

| Agent | Typ | Kosten | Modelle | Custom Instructions | MCP | Sandbox | Multi-Agent | Offline/Lokal | Best für |
|---|---|---|---|---|---|---|---|---|---|
| **[[Claude Code Konfiguration und Hooks\|Claude Code]]** | CLI (Commercial) | API-basiert (pay-per-use) | Claude Sonnet 4.6, Claude Opus 4.6 | `CLAUDE.md` | ✅ | ✅ | ✅ | ❌ | Komplexe Refactorings, Code-Qualität |
| **[[GitHub Copilot CLI Konfiguration\|Copilot CLI]]** | CLI (Commercial) | Subscription ($10–39/Monat) | Claude, GPT, Gemini (multi) | `copilot-instructions.md` | ✅ | ✅ | ✅ | ❌ | GitHub-Workflows, Team-Projekte |
| **[[Gemini CLI Konfiguration\|Gemini CLI]]** | CLI (Commercial) | Freemium (1K Req/Tag frei) | Gemini 2.5 Pro | `GEMINI.md` | ✅ | ✅ (Docker) | ❌ | ❌ | Multimodal, große Codebases |
| **[[OpenAI Codex CLI Konfiguration\|Codex CLI]]** | CLI (Commercial) | API-basiert / ChatGPT Plus | GPT-5.2, GPT-5.3 Codex | `AGENTS.md` | ✅ | ✅ | ✅ | ❌ | OpenAI-Ökosystem, schnelle Iteration |
| **[[OpenCode Konfiguration\|OpenCode]]** | CLI (OSS) | Kostenlos (BYOK) | 75+ Provider, Ollama | `AGENTS.md` | ✅ | ❌ | ✅ | ✅ (Ollama) | Vendor-neutral, Privacy |
| **[[Aider Konfiguration\|Aider]]** | CLI (OSS) | Kostenlos (BYOK) | OpenAI, Anthropic, Ollama | `.aider.conf.yml` | ❌ | ❌ | ❌ | ✅ (Ollama) | Git-native Workflows, Quick Fixes |
| **[[Goose Konfiguration\|Goose]]** | CLI (OSS) | Kostenlos (BYOK) | OpenAI, Anthropic, Google, lokal | `goose-hints` | ✅ (nativ) | ❌ | ❌ | ✅ | Performance, MCP-Extensions |
| **[[Open Interpreter Konfiguration\|Open Interpreter]]** | CLI (OSS) | Kostenlos (BYOK) | OpenAI, diverse Provider | System Prompt | ❌ | ❌ | ❌ | ✅ | Data Science, Shell-Automation |
| **[[Cursor Agent Konfiguration\|Cursor]]** | IDE | Freemium / Pro ($20/Monat) | Claude, GPT-4, Gemini | `.cursor/rules/` | ✅ | ❌ | ✅ | ❌ | Visuelles Arbeiten, Einsteiger |
| **[[Cline Konfiguration\|Cline]]** | IDE (Extension) | Kostenlos (BYOK) | Claude, OpenAI, Ollama, Bedrock | `.clinerules` | ✅ | ❌ | ❌ | ✅ (Ollama) | VS-Code-Nutzer, Multi-Provider |
| **[[OpenClaw Konfiguration\|OpenClaw]]** | Multi-Channel | Kostenlos (Self-Hosted) | Claude, OpenAI, Gemini, Ollama | Skills/Plugin-System | ✅ | ❌ | ❌ | ✅ (Ollama) | Remote-Zugriff, Messaging |

> [!IMPORTANT] BYOK = Bring Your Own Key
> Bei allen Open-Source-Agents und Cline zahlst du nur die API-Kosten deines gewählten Providers. Der Agent selbst ist kostenlos.

---

## 3. Entscheidungsbaum: Welcher Agent für welchen Anwendungsfall?

````
Brauchst du einen AI Coding Agent?
│
├─ Brauchst du eine IDE mit visuellem Feedback?
│  ├─ Ja, komplett integriert ──────────────► Cursor
│  └─ Ja, aber als VS Code Extension ──────► Cline
│
├─ Terminal / CLI bevorzugt?
│  │
│  ├─ Commercial mit Support gewünscht?
│  │  ├─ Maximale Code-Qualität ────────────► Claude Code
│  │  ├─ GitHub-Integration wichtig ────────► Copilot CLI
│  │  ├─ Kostenloser Einstieg ──────────────► Gemini CLI (1K Req/Tag frei)
│  │  └─ OpenAI-Ökosystem / GPT-4 ─────────► Codex CLI
│  │
│  └─ Open Source / eigene API Keys?
│     ├─ Maximale Provider-Flexibilität ────► OpenCode (75+ Provider)
│     ├─ Git-native, leichtgewichtig ───────► Aider
│     ├─ Performance + MCP-Extensions ──────► Goose (Rust-basiert)
│     └─ Code-Ausführung / Data Science ────► Open Interpreter
│
├─ Zugriff über Chat-Apps (Telegram, Discord…)?
│  └─ Ja ───────────────────────────────────► OpenClaw
│
└─ Kostenlos + vollständig lokal?
   ├─ CLI mit vielen Features ──────────────► OpenCode + Ollama
   ├─ CLI, minimal + Git-fokussiert ────────► Aider + Ollama
   └─ IDE, Open Source ─────────────────────► Cline + Ollama
````

---

## 4. Detailvergleiche nach Kategorie

### 4.1 CLI Agents: Claude Code vs. Copilot CLI vs. Gemini CLI vs. Codex CLI

Die vier großen AI-Unternehmen bieten jeweils eigene CLI-Agents. Hier die Unterschiede:

| Aspekt | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|---|---|---|---|---|
| **Hersteller** | Anthropic | GitHub/Microsoft | Google | OpenAI |
| **Stärke** | Code-Qualität, Hooks | GitHub-Ökosystem | Kostenloses Tier, Multimodal | Schnelle Iteration, TUI |
| **Preismodell** | Pay-per-token | Subscription | Freemium | Pay-per-token / Plus |
| **Modellauswahl** | Nur Claude | Multi-Model | Nur Gemini | Nur GPT |
| **MCP-Support** | ✅ Voll | ✅ Voll | ✅ Voll | ✅ Voll |
| **Sandbox** | ✅ | ✅ | ✅ Docker/Podman | ✅ |
| **Hooks/Lifecycle** | ✅ (Pre/Post-Hooks) | ❌ | ❌ | ❌ |
| **Multi-Agent** | ✅ Sub-Agents | ✅ Task-Agents | ❌ | ✅ |
| **Image-Input** | ✅ | ✅ | ✅ (nativ multimodal) | ✅ |
| **Web-Suche** | ✅ | ✅ | ✅ | ✅ |

> [!TIP] Wann welchen Commercial CLI?
> - **Claude Code**: Wenn Code-Qualität und präzise Refactorings Priorität haben. Das Hook-System ist einzigartig.
> - **Copilot CLI**: Wenn du bereits im GitHub-Ökosystem arbeitest (Issues, PRs, Actions).
> - **Gemini CLI**: Wenn du kostenlos starten willst oder mit Bildern/Multimodal arbeitest.
> - **Codex CLI**: Wenn du im OpenAI-Ökosystem bist und ein poliertes TUI schätzt.

**Typischer Workflow-Vergleich: "Refactore eine Funktion"**

````bash
# Claude Code — präzise, mit Hooks
claude "Refactore die Funktion parseConfig() in src/config.ts"
# → Erstellt chirurgische Edits, Hook kann Linter automatisch laufen lassen

# Copilot CLI — GitHub-integriert
copilot "Refactore parseConfig und erstelle einen PR"
# → Kann direkt PR erstellen, Labels setzen, Reviewer zuweisen

# Gemini CLI — multimodal
gemini "Refactore parseConfig basierend auf diesem Screenshot der Architektur"
# → Kann Bilder als Kontext verarbeiten

# Codex CLI — interaktiv
codex "Refactore parseConfig"
# → TUI zeigt Vorschau, Sandbox testet Änderungen
````

> [!WARNING] Vendor Lock-in
> Alle vier Commercial CLI-Agents binden dich an ihren jeweiligen LLM-Provider. Wechsel zwischen Anthropic, Google und OpenAI bedeutet Wechsel des gesamten Tools. Nur Copilot CLI bietet Multi-Model-Support.

### 4.2 Open Source CLI: OpenCode vs. Aider vs. Goose vs. Open Interpreter

| Aspekt | OpenCode | Aider | Goose | Open Interpreter |
|---|---|---|---|---|
| **GitHub Stars** | ~97K | ~40K | ~30K | ~62K |
| **Sprache** | Go | Python | Rust | Python |
| **Kernkonzept** | Universeller Agent | Git-native Editor | MCP-native Agent | Code-Executor |
| **Provider-Support** | 75+ (breitester Markt) | Viele (OpenAI, Anthropic, Ollama…) | Mehrere + MCP | OpenAI + konfigurierbar |
| **Git-Integration** | ✅ | ✅✅ (Auto-Commits, native) | ✅ | ❌ (nicht fokussiert) |
| **LSP-Support** | ✅ | ❌ | ❌ | ❌ |
| **Code-Ausführung** | ❌ | ❌ | Via MCP-Extensions | ✅✅ (Kernfeature) |
| **Lokal/Ollama** | ✅ | ✅ | ✅ | ✅ |
| **Plan/Build-Modi** | ✅ | ✅ (Architect Mode) | ❌ | ❌ |
| **TUI/Interface** | ✅ (Vim-artig) | Minimal (Chat) | ✅ (TUI) | Chat-basiert |
| **Lizenz** | MIT | Apache 2.0 | MIT | AGPL-3.0 |

> [!NOTE] Die Open-Source-Philosophie
> Alle vier Agents sind kostenlos nutzbar. Die Kosten entstehen durch die API-Keys der LLM-Provider. Mit Ollama sind alle vier **komplett kostenlos** und **lokal** nutzbar — allerdings mit Qualitätseinbußen bei lokalen Modellen.

**Wann welchen OSS-Agent?**

- **OpenCode**: Wenn du maximale Flexibilität bei der Provider-Wahl brauchst und ein poliertes TUI schätzt. Der "Schweizer Taschenmesser"-Ansatz.
- **Aider**: Wenn Git dein Workflow-Zentrum ist. Auto-Commits, Repository-Mapping und die leichtgewichtige Architektur machen Aider ideal für schnelle, fokussierte Änderungen.
- **Goose**: Wenn Performance zählt (Rust-basiert) und du MCP-Extensions aktiv nutzt. Durch Block (Square) enterprise-gebacken.
- **Open Interpreter**: Wenn du Code nicht nur schreiben, sondern **direkt ausführen** willst. Ideal für Data Science, Datenanalyse und Shell-Automation.

**Typischer Workflow-Vergleich: "Analysiere dieses Projekt"**

````bash
# OpenCode — wähle deinen Provider
opencode --provider anthropic "Analysiere die Architektur dieses Projekts"
# Oder wechsle zu einem lokalen Modell:
opencode --provider ollama --model codellama "Analysiere die Architektur"

# Aider — git-nativ
aider --model claude-3-sonnet
# > Analysiere die Architektur und committe eine ARCHITECTURE.md
# → Auto-Commit mit sinnvoller Commit-Message

# Goose — MCP-Extensions
goose "Analysiere die Architektur"
# → Kann via MCP auf Datenbanken, APIs, etc. zugreifen

# Open Interpreter — führt Code direkt aus
interpreter "Analysiere alle Python-Dateien und erstelle ein Abhängigkeitsdiagramm"
# → Führt tatsächlich Code aus, installiert Pakete, erstellt Grafiken
````

> [!WARNING] Open Interpreter Sicherheit
> Open Interpreter führt Code **direkt auf deinem System** aus — ohne Sandbox. Prüfe immer den generierten Code, bevor du die Ausführung bestätigst. Details: [[Open Interpreter Konfiguration]]

### 4.3 IDE vs. CLI: Cursor & Cline vs. Terminal Agents

| Aspekt | IDE (Cursor / Cline) | CLI Agents |
|---|---|---|
| **Visuelles Feedback** | ✅ Inline-Diffs, Syntax-Highlighting | ❌ Text-basiert |
| **Lernkurve** | Niedrig (vertraute IDE) | Mittel bis hoch |
| **Automatisierung / Scripting** | ❌ Schwierig | ✅ Pipes, Scripts, CI/CD |
| **Ressourcenverbrauch** | Hoch (Electron/VS Code) | Niedrig |
| **Multi-File Navigation** | ✅ Nativ (Tabs, Explorer) | ✅ (aber ohne visuellen Kontext) |
| **Remote / SSH** | Möglich, aber umständlich | ✅ Nativ perfekt |
| **Headless / CI** | ❌ | ✅ |

**Cursor vs. Cline im Detail:**

| Aspekt | Cursor | Cline |
|---|---|---|
| **Typ** | Eigenständiger VS-Code-Fork | VS-Code-Extension |
| **Kosten** | Pro: $20/Monat | Kostenlos (BYOK) |
| **Modelle** | Built-in (Claude, GPT, Gemini) | BYOK (beliebig) |
| **Vorteil** | Nahtloses Erlebnis, Tab-Completion | Keine Vendor-Abhängigkeit |
| **Nachteil** | Proprietär, eigener Editor nötig | Konfigurationsaufwand |
| **Browser-Automation** | ❌ | ✅ |

> [!TIP] IDE oder CLI?
> - **IDE wählen**, wenn: du visuelles Feedback brauchst, Einsteiger bist, oder in einem Team mit IDE-Standard arbeitest.
> - **CLI wählen**, wenn: du remote arbeitest, Automation brauchst, oder Ressourcen sparen willst.
> - **Beides nutzen**: Cursor/Cline für explorative Arbeit, CLI-Agent für Batch-Operationen und CI/CD.

**Workflow-Beispiel: Bug-Fix mit IDE vs. CLI**

Mit **Cursor** (IDE):
1. Öffne die Datei im Editor
2. Markiere die fehlerhafte Stelle
3. Drücke Cmd+K → "Fix this bug"
4. Sieh den Inline-Diff, akzeptiere oder lehne ab
5. Starte Tests über die IDE

Mit **Claude Code** (CLI):
````bash
claude "Der Test test_user_login schlägt fehl. Finde und fixe den Bug."
# → Agent navigiert selbst durch die Codebase, findet den Bug, fixt ihn
# → Kein manuelles Markieren oder Navigieren nötig
````

Die IDE-Variante gibt dir **mehr Kontrolle**, die CLI-Variante ist **schneller bei klaren Aufgaben**.

### 4.4 OpenClaw: Wann ein Multi-Channel Agent Sinn macht

[[OpenClaw Konfiguration|OpenClaw]] ist der einzige Agent in dieser Sammlung, der über **Chat-Apps** gesteuert wird: Telegram, Discord, WhatsApp, Slack, Signal und sogar iMessage.

**Sinnvoll wenn:**
- Du unterwegs bist und keinen Laptop-Zugriff hast
- Du Coding-Aufgaben von deinem Handy aus anstoßen willst
- Du einen Self-Hosted AI-Assistenten für dein Team brauchst
- Du einen "Butler" für deinen Homelab-Server willst

**Weniger sinnvoll wenn:**
- Du präzise, interaktive Coding-Sessions brauchst
- Du sensiblen Code bearbeitest (Chat-Apps als Übertragungsweg)
- Du Echtzeit-Feedback beim Coden brauchst

> [!WARNING] Sicherheit bei OpenClaw
> OpenClaw hat standardmäßig Shell-Zugriff auf den Host. In Produktionsumgebungen **immer** mit Docker isolieren und Berechtigungen einschränken. Siehe [[OpenClaw Konfiguration]] für Details.

---

## 5. Kosten im Vergleich

| Agent | Gratis-Tier | Subscription | Pay-per-Token | BYOK | Lokal (kostenlos) |
|---|---|---|---|---|---|
| **Claude Code** | ❌ | ❌ | ✅ (Anthropic API) | — | ❌ |
| **Copilot CLI** | ✅ (limitiert) | $10–39/Monat | ❌ | ❌ | ❌ |
| **Gemini CLI** | ✅ (1K Req/Tag) | ❌ | ✅ (Vertex AI) | — | ❌ |
| **Codex CLI** | ❌ | Via ChatGPT Plus ($20) | ✅ (OpenAI API) | — | ❌ |
| **OpenCode** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Aider** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Goose** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Open Interpreter** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Cursor** | ✅ (limitiert) | Pro $20/Monat | ❌ | ❌ | ❌ |
| **Cline** | ✅ (Extension selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **OpenClaw** | ✅ (Self-Hosted) | ❌ | Via Provider | ✅ | ✅ (Ollama) |

> [!NOTE] Kostenoptimierung
> Die günstigste Option ist immer **OSS-Agent + Ollama** (komplett kostenlos). Für professionelle Nutzung rechne mit **$20–50/Monat** für API-Kosten bei moderatem Einsatz eines Cloud-Modells.

---

## 6. Feature-Matrix

| Feature | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode | Aider | Goose | Interpreter | Cursor | Cline | OpenClaw |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **MCP-Support** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅✅ | ❌ | ✅ | ✅ | ✅ |
| **LSP-Integration** | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| **Git-Integration** | ✅ | ✅✅ | ✅ | ✅ | ✅ | ✅✅✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| **Multi-File Edit** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Auto-Commit** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Image-Input** | ✅ | ✅ | ✅✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Web-Suche** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | Via MCP | ❌ | ✅ | Via MCP | ✅ |
| **Browser-Automation** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | Via MCP | ✅ | ❌ | ✅ | ✅ |
| **Sandbox/Isolation** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Code-Ausführung** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅✅✅ | ✅ | ✅ | ✅ |
| **Custom Instructions** | `CLAUDE.md` | `copilot-instructions.md` | `GEMINI.md` | `AGENTS.md` | `AGENTS.md` | `.aider.conf.yml` | `goose-hints` | System Prompt | `.cursor/rules/` | `.clinerules` | Skills |
| **Hooks/Lifecycle** | ✅✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Multi-Provider** | ❌ | ✅ | ❌ | ❌ | ✅✅✅ | ✅✅ | ✅ | ✅ | ✅ | ✅✅ | ✅ |
| **Voice-Input** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |

> [!IMPORTANT] Legende
> ✅ = Unterstützt | ✅✅ = Besonders stark | ✅✅✅ = Kernfeature / Alleinstellungsmerkmal | ❌ = Nicht unterstützt

Siehe auch: [[Custom Instructions für AI Coding Agents]] für Details zu den verschiedenen Instruktions-Formaten.

---

## 7. Stärken und Schwächen (Pro/Contra)

### Claude Code
| ✅ Pro | ❌ Contra |
|---|---|
| Hervorragende Code-Qualität und Verständnis | Nur Claude-Modelle, kein Multi-Provider |
| Einzigartiges Hook-System (Pre/Post-Tool) | API-Kosten können bei intensiver Nutzung hoch werden |
| Starke Multi-Agent-Architektur (Sub-Agents) | Kein kostenloses Tier |
| Präzise, chirurgische Edits | An Anthropic-Ökosystem gebunden |

### GitHub Copilot CLI
| ✅ Pro | ❌ Contra |
|---|---|
| Tiefe GitHub-Integration (Issues, PRs, Actions) | Subscription-Modell (nicht pay-per-use) |
| Multi-Model-Unterstützung (Claude, GPT, Gemini) | An GitHub-Konto gebunden |
| Enterprise-ready mit Seat-Licensing | Weniger Flexibilität als OSS-Alternativen |
| Sandbox und Multi-Agent-Support | |

### Gemini CLI
| ✅ Pro | ❌ Contra |
|---|---|
| **1.000 Requests/Tag kostenlos** — bester Free-Tier | Nur Gemini-Modelle |
| Exzellente multimodale Fähigkeiten | Kein Multi-Agent-Modus |
| Docker/Podman-Sandbox nativ | Für Skalierung Vertex AI nötig |
| Einfache Google-Authentifizierung | Weniger ausgereiftes Ökosystem als Claude/Copilot |

### OpenAI Codex CLI
| ✅ Pro | ❌ Contra |
|---|---|
| Poliertes TUI-Interface | Nur OpenAI-Modelle |
| Starke Sandbox-Isolation | Erfordert ChatGPT Plus oder API-Key |
| Multi-Agent-Unterstützung | API-Kosten nicht transparent vorhersagbar |
| Web-Suche (cached & live) | |

### OpenCode
| ✅ Pro | ❌ Contra |
|---|---|
| **75+ Provider** — breiteste Modellunterstützung | Kein Sandbox-Modus |
| LSP-Integration für besseres Code-Verständnis | Relativ jung, schnelle Änderungen |
| Plan/Build-Modi für strukturiertes Arbeiten | Kann für Einsteiger überwältigend sein |
| Vim-artiger Editor, persistent Conversations | |

### Aider
| ✅ Pro | ❌ Contra |
|---|---|
| **Git-native**: Auto-Commits als Kernfeature | Kein MCP-Support |
| Leichtgewichtig und schnell aufgesetzt | Keine Sandbox, keine Hooks |
| Repository-Mapping für Kontext-Verständnis | Minimalistisches Interface |
| Architect Mode für größere Refactorings | Weniger Features als OpenCode/Goose |

### Goose
| ✅ Pro | ❌ Contra |
|---|---|
| **Rust-basiert** — extrem schnelle Performance | Kleineres Ökosystem als Alternativen |
| MCP-native: Extensions als First-Class-Feature | Weniger Provider als OpenCode |
| Enterprise-Backing durch Block (Square) | Dokumentation noch ausbaufähig |
| Modulares Extension-System | |

### Open Interpreter
| ✅ Pro | ❌ Contra |
|---|---|
| **Direkte Code-Ausführung** als Kernfeature | AGPL-Lizenz (nicht für alle geeignet) |
| Ideal für Data Science und Automation | Weniger auf Code-Editing fokussiert |
| Computer Use (GUI-Automation) | Sicherheitsrisiko bei ungeprüfter Ausführung |
| Unterstützt Python, JS, Shell, R | |

### Cursor
| ✅ Pro | ❌ Contra |
|---|---|
| **Nahtloses IDE-Erlebnis** mit AI-Integration | Proprietär, eigener Editor nötig |
| Tab-Autocomplete + Agent-Mode (Composer) | Pro-Tier ($20/Monat) für volle Features |
| Alle VS-Code-Extensions kompatibel | Hoher Ressourcenverbrauch |
| Niedrige Einstiegshürde | Vendor Lock-in |

### Cline
| ✅ Pro | ❌ Contra |
|---|---|
| **Open Source** VS-Code-Extension | Nur VS Code (kein anderer Editor) |
| Multi-Provider mit BYOK | Konfigurationsaufwand höher als Cursor |
| Browser-Automation integriert | API-Kosten können überraschen |
| Permission-basiert (Human-in-the-Loop) | |

### OpenClaw
| ✅ Pro | ❌ Contra |
|---|---|
| **Multi-Channel**: Telegram, Discord, WhatsApp, Slack… | Shell-Zugriff ist Sicherheitsrisiko |
| Self-Hosted, volle Kontrolle | Braucht dedizierte Hardware/Server |
| Skills/Plugin-System erweiterbar | Weniger präzise als dedizierte CLI-Agents |
| Web-Dashboard + CLI verfügbar | |

---

## 8. Empfehlungen nach Rolle

| Rolle | Empfehlung | Warum |
|---|---|---|
| **Solo-Entwickler (Hobby)** | [[Aider Konfiguration\|Aider]] + Ollama oder [[OpenCode Konfiguration\|OpenCode]] | Komplett kostenlos, lokal, keine API-Kosten |
| **Professional Developer** | [[Claude Code Konfiguration und Hooks\|Claude Code]] oder [[GitHub Copilot CLI Konfiguration\|Copilot CLI]] | Beste Code-Qualität bzw. bestes Ökosystem |
| **Team / Enterprise** | [[GitHub Copilot CLI Konfiguration\|Copilot CLI]] oder [[Cursor Agent Konfiguration\|Cursor]] | Seat-Licensing, GitHub-Integration, IDE-Standard |
| **Data Scientist** | [[Open Interpreter Konfiguration\|Open Interpreter]] | Direkte Code-Ausführung, Datenanalyse, R/Python |
| **DevOps / Infra** | [[Goose Konfiguration\|Goose]] oder [[Claude Code Konfiguration und Hooks\|Claude Code]] | MCP-Extensions für Infra-Tools, Hooks für Automation |
| **Mobile / Unterwegs** | [[OpenClaw Konfiguration\|OpenClaw]] | Steuerung via Telegram/Discord vom Handy |
| **Privacy-fokussiert** | [[OpenCode Konfiguration\|OpenCode]] + Ollama oder [[Aider Konfiguration\|Aider]] + lokales LLM | Kein Code verlässt deinen Rechner |
| **Einsteiger** | [[Cursor Agent Konfiguration\|Cursor]] oder [[GitHub Copilot CLI Konfiguration\|Copilot CLI]] | Visuell / einfacher Einstieg |

> [!TIP] Kein Entweder-Oder
> Du musst dich nicht für einen Agent entscheiden. Viele Entwickler nutzen 2–3 Agents parallel für verschiedene Aufgaben. Siehe Abschnitt 9.

---

## 9. Kombinationen: Mehrere Agents gleichzeitig nutzen

Die stärksten Workflows entstehen oft durch die **Kombination** mehrerer Agents:

### Bewährte Kombinationen

| Kombination | Workflow |
|---|---|
| **Aider + Claude Code** | Aider für schnelle, git-native Fixes. Claude Code für komplexe Multi-File-Refactorings. |
| **Cursor + CLI-Agent** | Cursor für visuelles Explorieren und Verstehen. CLI-Agent (Claude Code / Copilot) für Batch-Änderungen. |
| **OpenCode + Goose** | OpenCode als Haupt-Agent. Goose-MCP-Extensions für spezialisierte Infra-Aufgaben. |
| **Cline + Aider** | Cline für explorative Browser-Tests. Aider für die schnellen Code-Commits. |
| **OpenClaw + beliebiger CLI** | OpenClaw für unterwegs (Handy). CLI-Agent am Desktop für präzise Arbeit. |
| **Gemini CLI + Claude Code** | Gemini CLI für kostenloses Prototyping und Exploration. Claude Code für die finale, hochwertige Implementierung. |
| **Open Interpreter + Aider** | Interpreter für Datenanalyse und Exploration. Aider für die resultierenden Code-Änderungen. |

### Warum mehrere Agents?

Kein einzelner Agent ist in allem der Beste. Typische Gründe für Multi-Agent-Setups:

1. **Kosten-Optimierung**: Nutze Gemini CLI (kostenlos) für einfache Aufgaben und Claude Code (bezahlt) nur für komplexe
2. **Stärken-Kombination**: Aiders Git-Integration + Claude Codes Refactoring-Qualität
3. **Kontext-Wechsel**: IDE für explorative Arbeit, CLI für fokussierte Aufgaben
4. **Verfügbarkeit**: Wenn ein Provider down ist, wechsle zum anderen
5. **Evaluation**: Lass zwei Agents dieselbe Aufgabe lösen und vergleiche die Ergebnisse

### AGENTS.md als universelle Brücke

Mehrere Agents unterstützen `AGENTS.md` als Instruktionsdatei (Codex CLI, OpenCode). Andere haben eigene Formate. Für maximale Kompatibilität empfiehlt sich eine **Multi-File-Strategie**:

````markdown
# Empfohlene Projekt-Struktur für Multi-Agent-Support

├── AGENTS.md          # Codex CLI, OpenCode
├── CLAUDE.md          # Claude Code
├── GEMINI.md          # Gemini CLI
├── .github/
│   └── copilot-instructions.md  # Copilot CLI
├── .cursor/
│   └── rules/         # Cursor (mehrere Rule-Dateien möglich)
├── .clinerules        # Cline
├── .aider.conf.yml    # Aider
└── .goosehints        # Goose
````

**Tipp**: Halte die Kern-Regeln (Code-Style, Architektur-Entscheidungen, verbotene Patterns) in einer zentralen Datei und inkludiere sie in den agent-spezifischen Dateien. Beispiel:

````markdown
# CLAUDE.md
<!-- Kern-Regeln aus AGENTS.md übernommen -->
- TypeScript strict mode verwenden
- Fehler immer mit custom Error-Klassen werfen
- Tests mit Vitest schreiben
- Keine console.log in Production-Code

# Claude-spezifisch:
- Nutze Sub-Agents für parallele Aufgaben
- Pre-Tool-Hook: Linter vor jedem Edit laufen lassen
````

> [!TIP] Gemeinsame Regeln
> Erstelle eine zentrale Datei (z.B. `AGENTS.md`) mit den Kern-Regeln und referenziere sie in den agent-spezifischen Dateien. So vermeidest du Duplikation. Details: [[Custom Instructions für AI Coding Agents]]

Siehe auch: [[Multi-Agent Workflows]] für fortgeschrittene Multi-Agent-Strategien.

---

## 10. Fazit und Ausblick

### Der Stand der Dinge (2025)

Die AI-Coding-Agent-Landschaft ist **reif, aber fragmentiert**. Es gibt keinen einzelnen Agent, der in allen Bereichen dominiert:

- **Claude Code** führt bei Code-Qualität und komplexen Refactorings
- **Copilot CLI** ist der Standard für GitHub-zentrierte Teams
- **Gemini CLI** bietet den besten kostenlosen Einstieg
- **Codex CLI** punktet mit poliertem TUI und starker Sandbox
- **OpenCode** ist der flexibelste Open-Source-Agent (75+ Provider)
- **Aider** bleibt der Goldstandard für git-native Workflows
- **Goose** setzt auf Performance (Rust) und MCP-Extensions
- **Open Interpreter** ist einzigartig mit direkter Code-Ausführung
- **Cursor** senkt die Einstiegshürde für IDE-Nutzer
- **Cline** bringt Agent-Power als Open-Source-Extension in VS Code
- **OpenClaw** erschließt neue Kanäle jenseits von Terminal und IDE

### Drei Szenarien für den Einstieg

**Szenario 1: "Ich will sofort loslegen, kostenlos"**
→ Installiere [[Gemini CLI Konfiguration|Gemini CLI]] (1K Requests/Tag frei) oder [[Aider Konfiguration|Aider]] + Ollama

**Szenario 2: "Ich will die beste Qualität, Kosten sind sekundär"**
→ Nutze [[Claude Code Konfiguration und Hooks|Claude Code]] mit Claude Sonnet/Opus und konfiguriere Hooks

**Szenario 3: "Ich will flexibel bleiben und nichts verpassen"**
→ Starte mit [[OpenCode Konfiguration|OpenCode]] (75+ Provider, wechselbar) und teste verschiedene Modelle

### Was sich 2025 schnell ändert

- **MCP** etabliert sich als Standard-Protokoll — Agents ohne MCP-Support (wie Aider) werden Module nachrüsten oder zurückfallen
- **Multi-Agent-Workflows** werden ausgereifter (Sub-Agents, Orchestrierung) — siehe [[Multi-Agent Workflows]]
- **Lokale Modelle** (Ollama, llama.cpp) verbessern sich rapide und schließen die Qualitätslücke zu Cloud-Modellen
- **Custom Instructions** konvergieren: `AGENTS.md` als möglicher universeller Standard — siehe [[Custom Instructions für AI Coding Agents]]
- **Spezialisierung**: Agents werden sich stärker auf Nischen fokussieren statt Generalisten zu sein
- **Kosten sinken**: API-Preise fallen kontinuierlich, lokale Modelle werden brauchbarer

> [!IMPORTANT] Dieses Dokument aktuell halten
> Die AI-Agent-Landschaft ändert sich monatlich. Prüfe die individuellen Konfigurationsseiten auf aktuelle Details und aktualisiere diese Vergleichstabelle regelmäßig.

---

### Weiterführende Dokumente

- [[Custom Instructions für AI Coding Agents]] — Alle Instruktions-Formate im Detail
- [[CLAUDE.md schreiben]] — Best Practices für Claude Code Instructions
- [[Multi-Agent Workflows]] — Fortgeschrittene Multi-Agent-Strategien
- [[Was ist ein AI Agent]] — Grundlagen und Konzepte
