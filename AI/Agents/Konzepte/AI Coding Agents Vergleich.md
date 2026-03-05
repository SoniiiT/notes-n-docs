# AI Coding Agents Vergleich (2026)

> [!NOTE] Über dieses Dokument
> Dieser Vergleich deckt **alle AI Coding Agents** ab, die in diesem Vault dokumentiert sind. Ziel: Die richtige Wahl für den richtigen Anwendungsfall treffen. Letzte Aktualisierung: 2026.

---

## 1. Übersicht: Die Landschaft der AI Coding Agents (2026)

Die Welt der AI Coding Agents hat sich 2026 in mehrere Kategorien aufgeteilt:

| Kategorie | Beschreibung | Vertreter |
|---|---|---|
| **CLI (Commercial)** | Terminal-basiert, von großen AI-Unternehmen | Claude Code, Copilot CLI, Gemini CLI, Codex CLI, Amp, Amazon Q |
| **CLI (Open Source)** | Terminal-basiert, Community-getrieben | OpenCode, Aider, Goose, Open Interpreter, Crush |
| **IDE-basiert** | In den Editor integriert | Cursor, Cline, Windsurf, Kiro, Tabnine, Continue |
| **Multi-Channel** | Über Chat-Apps steuerbar | OpenClaw |
| **AI Terminal** | Modernisierter Terminal-Emulator mit KI | Warp |

Warum so viele? Jeder Ansatz hat andere Stärken: CLI-Agents sind schnell und scriptbar, IDE-Agents bieten visuelles Feedback, und Multi-Channel-Agents ermöglichen Zugriff von überall. Es gibt keinen "besten" Agent — nur den besten **für deinen Workflow**.

Die wichtigsten Trends 2026:
- **MCP (Model Context Protocol)** wird zum De-facto-Standard für Tool-Integration
- **[[Skills]]** ermöglichen modulare, wiederverwendbare Agent-Expertise (Agent Skills Spec)
- **Multi-Provider-Support** ist der Normalfall — kein Lock-in mehr nötig
- **Custom Instructions** (`CLAUDE.md`, `AGENTS.md`, etc.) geben Entwicklern Kontrolle über Agent-Verhalten
- **Lokale Modelle** via Ollama machen AI-Coding auch ohne Cloud möglich
- **Multi-Agent-Workflows** (ein Agent orchestriert andere) entstehen als neues Paradigma
- **Spec-driven Development** (Kiro) strukturiert Code-Generierung über formale Spezifikationen

> [!TIP] Individuelle Dokumentation
> Jeder Agent hat eine eigene Konfigurationsseite in diesem Vault. Dieser Vergleich hilft bei der Auswahl, die Details findest du in den verlinkten Dokumenten.
> 
> Für Grundlagen siehe [[Was ist ein AI Agent]]. Für Custom Instructions über alle Agents hinweg siehe [[Custom Instructions für AI Coding Agents]]. Für das Skills-System siehe [[Skills]].

---

## 2. Große Vergleichstabelle

| Agent                                                    | Typ              | Kosten                                                                    | Modelle                            | Custom Instructions       | MCP       | Sandbox    | Multi-Agent | Offline/Lokal | Best für                             |
| -------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------- | ---------------------------------- | ------------------------- | --------- | ---------- | ----------- | ------------- | ------------------------------------ |
| **[[Claude Code Konfiguration und Hooks\|Claude Code]]** | CLI (Commercial) | Subscription (Pro $20/Mo, Max $100/Mo) oder API                           | Claude Sonnet 4.6, Claude Opus 4.6 | `CLAUDE.md`               | ✅         | ✅          | ✅           | ❌             | Komplexe Refactorings, Code-Qualität |
| **[[GitHub Copilot CLI Konfiguration\|Copilot CLI]]**    | CLI (Commercial) | Subscription (Pro $10/Mo, Pro+ $39/Mo)                                    | Claude, GPT, Gemini (multi)        | `copilot-instructions.md` | ✅         | ✅          | ✅           | ❌             | GitHub-Workflows, Team-Projekte      |
| **[[Gemini CLI Konfiguration\|Gemini CLI]]**             | CLI (Commercial) | Freemium (1K Req/Tag) oder Subscription (Pro €21,99/Mo, Ultra €274,99/Mo) | Gemini 2.5 Pro                     | `GEMINI.md`               | ✅         | ✅ (Docker) | ❌           | ❌             | Multimodal, große Codebases          |
| **[[OpenAI Codex CLI Konfiguration\|Codex CLI]]**        | CLI (Commercial) | Subscription (Plus $20/Mo, Pro $200/Mo) oder API                          | GPT-5.2, GPT-5.3 Codex             | `AGENTS.md`               | ✅         | ✅          | ✅           | ❌             | OpenAI-Ökosystem, schnelle Iteration |
| **[[OpenCode Konfiguration\|OpenCode]]**                 | CLI (OSS)        | Kostenlos (BYOK)                                                          | 75+ Provider, Ollama               | `AGENTS.md`               | ✅         | ❌          | ✅           | ✅ (Ollama)    | Vendor-neutral, Privacy              |
| **[[Aider Konfiguration\|Aider]]**                       | CLI (OSS)        | Kostenlos (BYOK)                                                          | OpenAI, Anthropic, Ollama          | `.aider.conf.yml`         | ❌         | ❌          | ❌           | ✅ (Ollama)    | Git-native Workflows, Quick Fixes    |
| **[[Goose Konfiguration\|Goose]]**                       | CLI (OSS)        | Kostenlos (BYOK)                                                          | OpenAI, Anthropic, Google, lokal   | `goose-hints`             | ✅ (nativ) | ❌          | ❌           | ✅             | Performance, MCP-Extensions          |
| **[[Open Interpreter Konfiguration\|Open Interpreter]]** | CLI (OSS)        | Kostenlos (BYOK)                                                          | OpenAI, diverse Provider           | System Prompt             | ❌         | ❌          | ❌           | ✅             | Data Science, Shell-Automation       |
| **[[Cursor Agent Konfiguration\|Cursor]]**               | IDE              | Freemium oder Subscription (Pro $20/Mo, Pro+ $60/Mo, Ultra $200/Mo)       | Claude, GPT-4, Gemini              | `.cursor/rules/`          | ✅         | ❌          | ✅           | ❌             | Visuelles Arbeiten, Einsteiger       |
| **[[Cline Konfiguration\|Cline]]**                       | IDE (Extension)  | Kostenlos (BYOK)                                                          | Claude, OpenAI, Ollama, Bedrock    | `.clinerules`             | ✅         | ❌          | ❌           | ✅ (Ollama)    | VS-Code-Nutzer, Multi-Provider       |
| **[[OpenClaw Konfiguration\|OpenClaw]]**                 | Multi-Channel    | Kostenlos (Self-Hosted)                                                   | Claude, OpenAI, Gemini, Ollama     | [[Skills]]                | ✅         | ❌          | ❌           | ✅ (Ollama)    | Remote-Zugriff, Messaging            |
| **[[Amp Konfiguration\|Amp]]**                           | CLI (Commercial) | Free + Credits, Teams/Enterprise (Custom)                                 | Claude, GPT (Multi-Modell)         | `AGENT.md`                | ❌         | ❌          | ❌           | ❌             | Multi-Repo, Codebase-Suche           |
| **[[Windsurf Konfiguration\|Windsurf]]**                 | IDE              | Free, Pro ($15/Mo), Teams ($30/user/Mo), Enterprise ($60/user/Mo)         | SWE-1.5 (eigen), BYOK             | `.windsurfrules`          | ✅         | ✅          | ✅           | ❌             | Autonome Multi-File-Edits            |
| **[[Amazon Q Developer CLI Konfiguration\|Amazon Q]]**   | CLI (Commercial) | Free ($0/Mo), Pro ($19/user/Mo)                                           | Amazon eigene Modelle              | AWS-Konfiguration         | ❌         | ✅          | ✅           | ❌             | AWS-Entwicklung, IaC                 |
| **[[Kiro Konfiguration\|Kiro]]**                         | IDE              | Free (50 Credits), Pro ($20/Mo), Power ($200/Mo)                          | Multi-Modell (AWS)                 | Steering Files            | ✅         | ❌          | ✅           | ❌             | Spec-driven Development              |
| **[[Augment Code Konfiguration\|Augment Code]]**         | CLI/IDE          | Indie ($20/Mo), Standard ($60/Mo), Max ($200/Mo)                          | Proprietär + konfigurierbar        | —                         | ❌         | ❌          | ✅✅         | ❌             | Context Engine, Multi-Agent          |
| **[[Warp Konfiguration\|Warp]]**                         | AI Terminal      | Free ($0/Mo), Build ($18/Mo), Max ($180/Mo)                               | OpenAI, Anthropic, Gemini (BYOK)   | —                         | ❌         | ❌          | ❌           | ❌             | Modernes Terminal, DevOps            |
| **[[Crush Konfiguration\|Crush]]**                       | CLI (OSS)        | Kostenlos (BYOK)                                                          | OpenAI, Anthropic, Gemini, Ollama  | `crush.json`              | ✅         | ❌          | ❌           | ✅ (Ollama)    | LSP-Integration, leichtgewichtig     |
| **[[Continue Konfiguration\|Continue]]**                 | IDE/CLI (OSS)    | Free, Team ($10/user/Mo)                                                  | Alle Provider, Ollama, LM Studio   | `config.json` + Rules     | ❌         | ❌          | ✅ (CI/CD)   | ✅ (Ollama)    | Open Source, CI/CD-Agents            |
| **[[Tabnine Konfiguration\|Tabnine]]**                   | IDE              | Basic (Free), Dev (~$12/Mo), Enterprise ($39–59/user/Mo)                  | Tabnine eigen + BYOM               | —                         | ❌         | ❌          | ❌           | ✅ (On-Prem)   | Enterprise, Compliance, Air-Gapped   |

> [!IMPORTANT] BYOK = Bring Your Own Key
> Bei allen Open-Source-Agents und Cline zahlst du nur die API-Kosten deines gewählten Providers. Der Agent selbst ist kostenlos.

---

## 3. Entscheidungsbaum: Welcher Agent für welchen Anwendungsfall?

````
Brauchst du einen AI Coding Agent?
│
├─ Brauchst du eine IDE mit visuellem Feedback?
│  ├─ Ja, komplett integriert ──────────────► Cursor / Windsurf / Kiro
│  ├─ Ja, als VS Code Extension ───────────► Cline / Continue / Tabnine
│  └─ Enterprise / On-Premises ────────────► Tabnine (Air-Gapped)
│
├─ Terminal / CLI bevorzugt?
│  │
│  ├─ Commercial mit Support gewünscht?
│  │  ├─ Maximale Code-Qualität ────────────► Claude Code
│  │  ├─ GitHub-Integration wichtig ────────► Copilot CLI
│  │  ├─ Kostenloser Einstieg ──────────────► Gemini CLI (1K Req/Tag frei)
│  │  ├─ OpenAI-Ökosystem / GPT ───────────► Codex CLI
│  │  ├─ Multi-Repo / Codebase-Graph ──────► Amp (Sourcegraph)
│  │  ├─ AWS-Ökosystem / IaC ──────────────► Amazon Q Developer CLI
│  │  └─ Deep Context Engine ──────────────► Augment Code (Auggie CLI)
│  │
│  └─ Open Source / eigene API Keys?
│     ├─ Maximale Provider-Flexibilität ────► OpenCode (75+ Provider)
│     ├─ Git-native, leichtgewichtig ───────► Aider
│     ├─ Performance + MCP-Extensions ──────► Goose (Rust-basiert)
│     ├─ LSP + MCP, minimalistisch ────────► Crush (Go-basiert)
│     └─ Code-Ausführung / Data Science ────► Open Interpreter
│
├─ Zugriff über Chat-Apps (Telegram, Discord…)?
│  └─ Ja ───────────────────────────────────► OpenClaw
│
├─ Modernes AI-Terminal statt Agent?
│  └─ Ja ───────────────────────────────────► Warp
│
└─ Kostenlos + vollständig lokal?
   ├─ CLI mit vielen Features ──────────────► OpenCode + Ollama
   ├─ CLI, minimal + Git-fokussiert ────────► Aider + Ollama
   ├─ CLI, leichtgewichtig + LSP ──────────► Crush + Ollama
   └─ IDE, Open Source ─────────────────────► Cline / Continue + Ollama
````

---

## 4. Detailvergleiche nach Kategorie

### 4.1 CLI Agents: Claude Code vs. Copilot CLI vs. Gemini CLI vs. Codex CLI

Die vier großen AI-Unternehmen bieten jeweils eigene CLI-Agents. Hier die Unterschiede:

| Aspekt | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|---|---|---|---|---|
| **Hersteller** | Anthropic | GitHub/Microsoft | Google | OpenAI |
| **Stärke** | Code-Qualität, Hooks | GitHub-Ökosystem | Kostenloses Tier, Multimodal | Schnelle Iteration, TUI |
| **Preismodell** | Subscription (Pro $20/Mo, Max $100/Mo) oder API | Subscription (Pro $10/Mo, Pro+ $39/Mo) | Freemium oder Subscription (Pro €21,99/Mo, Ultra €274,99/Mo) | Subscription (Plus $20/Mo, Pro $200/Mo) oder API |
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

### 4.2 Weitere Commercial CLI: Amp, Amazon Q, Augment Code

| Aspekt | Amp (Sourcegraph) | Amazon Q Developer | Augment Code |
|---|---|---|---|
| **Hersteller** | Sourcegraph | AWS | Augment AI |
| **Stärke** | Multi-Repo-Suche, Code-Graph | AWS-Integration, IaC | Context Engine, Multi-Agent |
| **Preismodell** | Free Credits, Enterprise (Custom) | Free ($0/Mo), Pro ($19/user/Mo) | Indie ($20/Mo), Standard ($60/Mo), Max ($200/Mo) |
| **Modellauswahl** | Multi-Modell (Claude, GPT) | AWS eigene Modelle | Proprietär + konfigurierbar |
| **MCP-Support** | ❌ | ❌ | ❌ |
| **Sandbox** | ❌ | ✅ | ❌ |
| **Multi-Agent** | ❌ | ✅ | ✅✅ (Kernfeature) |
| **Besonderheit** | Codebase-Graph über alle Repos | Security Scanning, Code Transformation | Next Edit + Instructions + Auggie CLI |

> [!TIP] Wann welchen Spezialisten?
> - **Amp**: Wenn du in großen Monorepos oder Multi-Repo-Setups arbeitest (Sourcegraph-Stärke)
> - **Amazon Q**: Wenn du AWS-Entwickler bist und IaC, Security Scanning oder Code-Migration brauchst
> - **Augment Code**: Wenn tiefes Codebase-Verständnis über die gesamte Organisation wichtig ist

### 4.3 Open Source CLI: OpenCode vs. Aider vs. Goose vs. Open Interpreter vs. Crush

| Aspekt | OpenCode | Aider | Goose | Open Interpreter | Crush |
|---|---|---|---|---|---|
| **GitHub Stars** | ~97K | ~40K | ~30K | ~62K | ~10K |
| **Sprache** | Go | Python | Rust | Python | Go |
| **Kernkonzept** | Universeller Agent | Git-native Editor | MCP-native Agent | Code-Executor | LSP+MCP Agent |
| **Provider-Support** | 75+ (breitester Markt) | Viele (OpenAI, Anthropic, Ollama…) | Mehrere + MCP | OpenAI + konfigurierbar | OpenAI, Anthropic, Gemini, Ollama |
| **Git-Integration** | ✅ | ✅✅ (Auto-Commits, native) | ✅ | ❌ (nicht fokussiert) | ✅ |
| **LSP-Support** | ✅ | ❌ | ❌ | ❌ | ✅✅ |
| **MCP-Support** | ✅ | ❌ | ✅✅ (Kernfeature) | ❌ | ✅ |
| **Code-Ausführung** | ❌ | ❌ | Via MCP-Extensions | ✅✅ (Kernfeature) | ✅ |
| **Lokal/Ollama** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Plan/Build-Modi** | ✅ | ✅ (Architect Mode) | ❌ | ❌ | ❌ |
| **TUI/Interface** | ✅ (Vim-artig) | Minimal (Chat) | ✅ (TUI) | Chat-basiert | Minimal (Chat) |
| **Lizenz** | MIT | Apache 2.0 | MIT | AGPL-3.0 | MIT |

> [!NOTE] Die Open-Source-Philosophie
> Alle vier Agents sind kostenlos nutzbar. Die Kosten entstehen durch die API-Keys der LLM-Provider. Mit Ollama sind alle vier **komplett kostenlos** und **lokal** nutzbar — allerdings mit Qualitätseinbußen bei lokalen Modellen.

**Wann welchen OSS-Agent?**

- **OpenCode**: Wenn du maximale Flexibilität bei der Provider-Wahl brauchst und ein poliertes TUI schätzt. Der "Schweizer Taschenmesser"-Ansatz.
- **Aider**: Wenn Git dein Workflow-Zentrum ist. Auto-Commits, Repository-Mapping und die leichtgewichtige Architektur machen Aider ideal für schnelle, fokussierte Änderungen.
- **Goose**: Wenn Performance zählt (Rust-basiert) und du MCP-Extensions aktiv nutzt. Durch Block (Square) enterprise-gebacken.
- **Crush**: Wenn du LSP+MCP in einem leichtgewichtigen Go-Paket willst. Ideal für Entwickler, die präzise Code-Navigation schätzen.
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

### 4.4 IDE Agents: Cursor vs. Windsurf vs. Kiro vs. Cline vs. Continue vs. Tabnine

| Aspekt | Cursor | Windsurf | Kiro | Cline | Continue | Tabnine |
|---|---|---|---|---|---|---|
| **Typ** | VS-Code-Fork | VS-Code-Fork | Eigenständige IDE (AWS) | VS-Code-Extension | VS-Code + JetBrains Extension | Multi-IDE Plugin |
| **Kosten** | Pro $20/Mo, Pro+ $60/Mo, Ultra $200/Mo | Pro $15/Mo, Teams $30/user/Mo | Pro $20/Mo, Power $200/Mo | Kostenlos (BYOK) | Free, Team $10/user/Mo | Dev ~$12/Mo, Enterprise $39–59/user/Mo |
| **Agent-Modus** | Composer (Agent Mode) | Cascade (autonome Sessions) | Spec-driven Development | Chat + Agent-Tools | Chat + CI/CD-Agents | Code Completion + Chat |
| **Modelle** | Built-in (Claude, GPT, Gemini) | SWE-1.5 (eigen) + BYOK | Multi-Modell (AWS) | BYOK (beliebig) | BYOK (alle Provider) | Eigen + BYOM |
| **Stärke** | Nahtloses Erlebnis, Tab-Completion | Autonome Multi-File-Edits | Formale Spezifikationen | Keine Vendor-Abhängigkeit | Open Source (Apache 2.0), CI/CD | Enterprise-Compliance, Air-Gapped |
| **MCP-Support** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Custom Instructions** | `.cursor/rules/` | `.windsurfrules` | Steering Files | `.clinerules` | `config.json` + Rules | — |
| **Open Source** | ❌ | ❌ | ❌ | ✅ (Apache 2.0) | ✅ (Apache 2.0) | ❌ |
| **On-Premises** | ❌ | ❌ | ❌ | ❌ | ✅ (Self-Hosted) | ✅✅ (Air-Gapped) |

> [!TIP] IDE-Agent wählen
> - **Cursor**: Bester Allrounder für IDE-First-Entwickler. Nahtlose Tab-Completion.
> - **Windsurf**: Wenn du autonome, mehrstufige Edits willst (Cascade-Agent).
> - **Kiro**: Wenn formale Spezifikationen und Requirements-Tracking wichtig sind.
> - **Cline**: Wenn du VS Code behalten und keine Vendor-Abhängigkeit willst (BYOK).
> - **Continue**: Wenn du Open Source willst und CI/CD-Agents brauchst.
> - **Tabnine**: Wenn Enterprise-Compliance (SOC2, HIPAA) oder Air-Gapped-Deployment nötig ist.

### 4.5 OpenClaw: Wann ein Multi-Channel Agent Sinn macht

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
| **Claude Code** | ❌ | ✅ Pro ($20/Mo), Max ($100/Mo) | ✅ (Anthropic API) | — | ❌ |
| **Copilot CLI** | ✅ (limitiert) | ✅ Pro ($10/Mo), Pro+ ($39/Mo) | ❌ | ❌ | ❌ |
| **Gemini CLI** | ✅ (1K Req/Tag) | ✅ Plus (€7,99/Mo), Pro (€21,99/Mo), Ultra (€274,99/Mo) | ✅ (Vertex AI) | — | ❌ |
| **Codex CLI** | ❌ | ✅ Plus ($20/Mo), Pro ($200/Mo), Business ($30/user/Mo) | ✅ (OpenAI API) | — | ❌ |
| **Amp** | ✅ (limitierte Credits) | Teams/Enterprise (Custom) | ❌ | ❌ | ❌ |
| **Amazon Q** | ✅ (Free Tier) | ✅ Pro ($19/user/Mo) | ❌ | ❌ | ❌ |
| **Augment Code** | ❌ | ✅ Indie ($20/Mo), Standard ($60/Mo), Max ($200/Mo) | ❌ | ❌ | ❌ |
| **OpenCode** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Aider** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Goose** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Open Interpreter** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Crush** | ✅ (Agent selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Cursor** | ✅ (Hobby Free) | ✅ Pro ($20/Mo), Pro+ ($60/Mo), Ultra ($200/Mo) | ❌ | ❌ | ❌ |
| **Windsurf** | ✅ (Free Tier) | ✅ Pro ($15/Mo), Teams ($30/user/Mo), Enterprise ($60/user/Mo) | ❌ | ✅ (BYOK) | ❌ |
| **Kiro** | ✅ (50 Credits) | ✅ Pro ($20/Mo), Power ($200/Mo) | ❌ | ❌ | ❌ |
| **Continue** | ✅ (Extension selbst) | ✅ Team ($10/user/Mo) | Via Provider | ✅ | ✅ (Ollama) |
| **Tabnine** | ✅ (Basic Free) | ✅ Dev (~$12/Mo), Enterprise ($39–59/user/Mo) | ❌ | ❌ | ✅ (On-Prem) |
| **Cline** | ✅ (Extension selbst) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **OpenClaw** | ✅ (Self-Hosted) | ❌ | Via Provider | ✅ | ✅ (Ollama) |
| **Warp** | ✅ (Free Tier) | ✅ Build ($18/Mo), Max ($180/Mo) | ❌ | ✅ (BYOK) | ❌ |

> [!NOTE] Kostenoptimierung
> Die günstigste Option ist immer **OSS-Agent + Ollama** (komplett kostenlos). Für professionelle Nutzung rechne mit **$20–50/Monat** für API-Kosten bei moderatem Einsatz eines Cloud-Modells.

---

## 6. Feature-Matrix

| Feature | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode | Aider | Goose | Interpreter | Cursor | Cline | OpenClaw | Amp | Windsurf | Amazon Q | Kiro | Augment | Warp | Crush | Continue | Tabnine |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **MCP-Support** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅✅ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| **LSP-Integration** | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅✅ | ❌ | ❌ |
| **Git-Integration** | ✅ | ✅✅ | ✅ | ✅ | ✅ | ✅✅✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Multi-File Edit** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Auto-Commit** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Image-Input** | ✅ | ✅ | ✅✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Web-Suche** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | Via MCP | ❌ | ✅ | Via MCP | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Sandbox/Isolation** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Code-Ausführung** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅✅✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ❌ |
| **Multi-Provider** | ❌ | ✅ | ❌ | ❌ | ✅✅✅ | ✅✅ | ✅ | ✅ | ✅ | ✅✅ | ✅ | ✅ | ✅ (BYOK) | ❌ | ✅ | ✅ | ✅ (BYOK) | ✅ | ✅ | ✅ (BYOM) |
| **Multi-Agent** | ✅✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅✅ | ❌ | ❌ | ✅ | ❌ |
| **Custom Instructions** | `CLAUDE.md` | `copilot-instructions.md` | `GEMINI.md` | `AGENTS.md` | `AGENTS.md` | `.aider.conf.yml` | `goose-hints` | System Prompt | `.cursor/rules/` | `.clinerules` | [[Skills]] | `AGENT.md` | `.windsurfrules` | AWS-Config | Steering Files | — | — | `crush.json` | `config.json` | — |
| **Hooks/Lifecycle** | ✅✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (Agent Hooks) | ❌ | ❌ | ❌ | ❌ | ❌ |

> [!IMPORTANT] Legende
> ✅ = Unterstützt | ✅✅ = Besonders stark | ✅✅✅ = Kernfeature / Alleinstellungsmerkmal | ❌ = Nicht unterstützt

Siehe auch: [[Custom Instructions für AI Coding Agents]] für Details zu den verschiedenen Instruktions-Formaten.

---

## 7. Stärken und Schwächen (Pro/Contra)

### Claude Code
| ✅ Pro | ❌ Contra |
|---|---|
| Hervorragende Code-Qualität und Verständnis | Nur Claude-Modelle, kein Multi-Provider |
| Einzigartiges Hook-System (Pre/Post-Tool) | Kein kostenloses Tier |
| Starke Multi-Agent-Architektur (Sub-Agents) | An Anthropic-Ökosystem gebunden |
| Subscription-Pläne: Pro ($20/Mo) und Max ($100/Mo) | API-Kosten bei Pay-per-Use schwer vorhersagbar |

### GitHub Copilot CLI
| ✅ Pro | ❌ Contra |
|---|---|
| Tiefe GitHub-Integration (Issues, PRs, Actions) | Subscription-Modell (Pro $10/Mo, Pro+ $39/Mo) |
| Multi-Model-Unterstützung (Claude, GPT, Gemini) | An GitHub-Konto gebunden |
| Enterprise-ready mit Seat-Licensing | Weniger Flexibilität als OSS-Alternativen |
| Sandbox und Multi-Agent-Support | |

### Gemini CLI
| ✅ Pro | ❌ Contra |
|---|---|
| **1.000 Requests/Tag kostenlos** — bester Free-Tier | Nur Gemini-Modelle |
| Exzellente multimodale Fähigkeiten | Kein Multi-Agent-Modus |
| Subscription-Pläne: Plus (€7,99/Mo), Pro (€21,99/Mo), Ultra (€274,99/Mo) | Für Skalierung Vertex AI nötig |
| Docker/Podman-Sandbox nativ | Weniger ausgereiftes Ökosystem als Claude/Copilot |

### OpenAI Codex CLI
| ✅ Pro | ❌ Contra |
|---|---|
| Poliertes TUI-Interface | Nur OpenAI-Modelle |
| Starke Sandbox-Isolation | Subscription nötig: Plus ($20/Mo), Pro ($200/Mo), Business ($30/user/Mo) |
| Multi-Agent-Unterstützung | API-Kosten bei Pay-per-Use schwer vorhersagbar |
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
| Tab-Autocomplete + Agent-Mode (Composer) | Subscription: Pro ($20/Mo), Pro+ ($60/Mo), Ultra ($200/Mo) |
| Alle VS-Code-Extensions kompatibel | Hoher Ressourcenverbrauch |
| Niedrige Einstiegshürde, Hobby-Tier kostenlos | Vendor Lock-in |

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

### Amp (Sourcegraph)
| ✅ Pro | ❌ Contra |
|---|---|
| **Multi-Repo-Suche** über gesamte Codebases | Kein MCP-Support |
| Code-Graph für semantisches Verständnis | An Sourcegraph-Ökosystem gebunden |
| Multi-Modell-Support (Claude, GPT) | Enterprise-Fokus, Pricing nicht transparent |
| Thread-basierte Kontextverwaltung | Relativ neues Produkt |

### Windsurf (Codeium)
| ✅ Pro | ❌ Contra |
|---|---|
| **Cascade-Agent** für autonome Multi-File-Edits | Proprietärer Editor (VS-Code-Fork) |
| Eigenes SWE-Modell (SWE-1.5) optimiert für Coding | Weniger Community als Cursor |
| BYOK für beliebige Modelle möglich | Pricing steigt schnell bei Teams |
| App-Deployment direkt aus der IDE | |

### Amazon Q Developer CLI
| ✅ Pro | ❌ Contra |
|---|---|
| **AWS-Integration** — nativer Zugriff auf alle Services | Nur AWS-Ökosystem, kein Multi-Cloud |
| Security Scanning und Code Transformation | Erfordert AWS-Konto und IAM-Setup |
| Großzügiger Free Tier | Weniger ausgereift als Claude/Copilot |
| IaC-Generierung (CloudFormation, CDK, Terraform) | |

### Kiro (AWS)
| ✅ Pro | ❌ Contra |
|---|---|
| **Spec-driven Development** — strukturierte Codegenerierung | Sehr neues Produkt (Preview) |
| Agent Hooks für automatisierte Workflows | An AWS-Ökosystem gebunden |
| EARS-Syntax für formale Anforderungen | Weniger Flexibilität als freie Agents |
| Steering Files für Projekt-Konventionen | |

### Augment Code
| ✅ Pro | ❌ Contra |
|---|---|
| **Context Engine** — versteht gesamte Codebase tiefgreifend | Relativ teuer ($20–200/Mo) |
| Echte **Multi-Agent-Orchestrierung** (Next Edit, Instructions, Auggie) | Kein Open Source |
| Enterprise-ready, mehrere IDE-Integrationen | Weniger bekannt als Cursor/Copilot |
| Auggie CLI für Terminal-Nutzung | |

### Warp
| ✅ Pro | ❌ Contra |
|---|---|
| **Modernes Terminal** mit Block-basierter Ausgabe | Kein Coding Agent im eigentlichen Sinne |
| AI-Kommandogenerierung und Fehlererklärung | Ersetzt nicht Terminal + Agent-Kombo |
| Warp Drive für Team-Sharing | Warp selbst ist proprietär |
| BYOK für verschiedene AI-Provider | |

### Crush
| ✅ Pro | ❌ Contra |
|---|---|
| **Open Source** (MIT), Go-basiert und leichtgewichtig | Noch sehr frühes Projekt |
| LSP-Integration für präzises Code-Verständnis | Kleinere Community als Aider/OpenCode |
| MCP-Support und Session-Management | Weniger Features als etablierte Alternativen |
| Schnelle Performance durch Go | |

### Continue
| ✅ Pro | ❌ Contra |
|---|---|
| **Open Source** (Apache 2.0), VS Code + JetBrains | Kein eigener Editor (nur Extension) |
| CI/CD-Agents für automatisierte Code-Reviews | Konfigurationsaufwand höher |
| Continue Hub für Konfigurationsmanagement | API-Kosten können überraschen |
| Maximale BYOK-Flexibilität (alle Provider) | |

### Tabnine
| ✅ Pro | ❌ Contra |
|---|---|
| **Enterprise-Compliance**: SOC2, ISO 27001, HIPAA | Weniger "Agent"-Features als Konkurrenz |
| **Air-Gapped** On-Premises-Deployment möglich | Eigene Modelle qualitativ unter GPT/Claude |
| Trainiert nie auf Kundendaten | Fokus mehr auf Autocomplete als Agent |
| Breite IDE-Unterstützung (15+ Editoren) | |

---

## 8. Empfehlungen nach Rolle

| Rolle | Empfehlung | Warum |
|---|---|---|
| **Solo-Entwickler (Hobby)** | [[Aider Konfiguration\|Aider]] + Ollama oder [[OpenCode Konfiguration\|OpenCode]] | Komplett kostenlos, lokal, keine API-Kosten |
| **Professional Developer** | [[Claude Code Konfiguration und Hooks\|Claude Code]] oder [[GitHub Copilot CLI Konfiguration\|Copilot CLI]] | Beste Code-Qualität bzw. bestes Ökosystem |
| **Team / Enterprise** | [[GitHub Copilot CLI Konfiguration\|Copilot CLI]], [[Tabnine Konfiguration\|Tabnine]] oder [[Augment Code Konfiguration\|Augment Code]] | Seat-Licensing, Compliance, Context Engine |
| **Data Scientist** | [[Open Interpreter Konfiguration\|Open Interpreter]] | Direkte Code-Ausführung, Datenanalyse, R/Python |
| **DevOps / Infra** | [[Goose Konfiguration\|Goose]], [[Amazon Q Developer CLI Konfiguration\|Amazon Q]] oder [[Warp Konfiguration\|Warp]] | MCP-Extensions, AWS-Integration, AI-Terminal |
| **AWS-Entwickler** | [[Amazon Q Developer CLI Konfiguration\|Amazon Q]] oder [[Kiro Konfiguration\|Kiro]] | Natives AWS-Ökosystem, IaC, Spec-driven Development |
| **Multi-Repo / Monorepo** | [[Amp Konfiguration\|Amp]] oder [[Augment Code Konfiguration\|Augment Code]] | Code-Graph, Context Engine über Repo-Grenzen |
| **IDE-First (visuell)** | [[Cursor Agent Konfiguration\|Cursor]], [[Windsurf Konfiguration\|Windsurf]] oder [[Kiro Konfiguration\|Kiro]] | Nahtloses IDE-Erlebnis mit AI |
| **Mobile / Unterwegs** | [[OpenClaw Konfiguration\|OpenClaw]] | Steuerung via Telegram/Discord vom Handy |
| **Privacy / Compliance** | [[Tabnine Konfiguration\|Tabnine]], [[OpenCode Konfiguration\|OpenCode]] + Ollama | Air-Gapped, On-Premises, kein Code verlässt Rechner |
| **Einsteiger** | [[Cursor Agent Konfiguration\|Cursor]], [[Continue Konfiguration\|Continue]] oder [[GitHub Copilot CLI Konfiguration\|Copilot CLI]] | Visuell / einfacher Einstieg / Open Source |
| **Open Source Fan** | [[Crush Konfiguration\|Crush]], [[Continue Konfiguration\|Continue]] oder [[Cline Konfiguration\|Cline]] | MIT/Apache-Lizenz, BYOK, Community-driven |

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
| **Windsurf + Claude Code** | Windsurf für autonome Cascade-Sessions. Claude Code für präzise Hooks-basierte Workflows. |
| **OpenCode + Goose** | OpenCode als Haupt-Agent. Goose-MCP-Extensions für spezialisierte Infra-Aufgaben. |
| **Cline + Aider** | Cline für explorative Browser-Tests. Aider für die schnellen Code-Commits. |
| **Continue + Crush** | Continue in IDE für tägliche Arbeit. Crush im Terminal für schnelle CLI-Aufgaben. |
| **OpenClaw + beliebiger CLI** | OpenClaw für unterwegs (Handy). CLI-Agent am Desktop für präzise Arbeit. |
| **Gemini CLI + Claude Code** | Gemini CLI für kostenloses Prototyping und Exploration. Claude Code für die finale, hochwertige Implementierung. |
| **Amazon Q + Kiro** | Amazon Q CLI für IaC und Cloud-Operationen. Kiro IDE für Spec-driven Feature-Development. |
| **Augment Code + Copilot CLI** | Augment für tiefes Codebase-Verständnis. Copilot für GitHub-Integration. |
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
├── AGENT.md           # Amp (Sourcegraph)
├── .github/
│   └── copilot-instructions.md  # Copilot CLI
├── .cursor/
│   └── rules/         # Cursor (mehrere Rule-Dateien möglich)
├── .windsurfrules     # Windsurf
├── .clinerules        # Cline
├── .aider.conf.yml    # Aider
├── .goosehints        # Goose
├── .kiro/
│   └── steering/      # Kiro (Steering Files)
├── .continue/
│   └── config.json    # Continue
└── crush.json         # Crush
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

### Der Stand der Dinge (2026)

Die AI-Coding-Agent-Landschaft ist **reif, vielfältig und stark gewachsen**. Es gibt keinen einzelnen Agent, der in allen Bereichen dominiert:

- **Claude Code** führt bei Code-Qualität und komplexen Refactorings
- **Copilot CLI** ist der Standard für GitHub-zentrierte Teams
- **Gemini CLI** bietet den besten kostenlosen Einstieg
- **Codex CLI** punktet mit poliertem TUI und starker Sandbox
- **Amp** (Sourcegraph) dominiert bei Multi-Repo-Suche und Code-Graphen
- **Windsurf** bietet autonome Cascade-Sessions mit eigenem SWE-Modell
- **Amazon Q** ist der go-to für AWS-Entwickler mit IaC-Support
- **Kiro** bringt Spec-driven Development als neues Paradigma
- **Augment Code** setzt Maßstäbe bei Context Engine und Multi-Agent-Orchestrierung
- **Warp** modernisiert das Terminal-Erlebnis mit AI
- **OpenCode** ist der flexibelste Open-Source-Agent (75+ Provider)
- **Aider** bleibt der Goldstandard für git-native Workflows
- **Goose** setzt auf Performance (Rust) und MCP-Extensions
- **Crush** bietet LSP+MCP als leichtgewichtige Go-Alternative
- **Open Interpreter** ist einzigartig mit direkter Code-Ausführung
- **Cursor** senkt die Einstiegshürde für IDE-Nutzer
- **Cline** bringt Agent-Power als Open-Source-Extension in VS Code
- **Continue** erweitert den OSS-IDE-Markt mit CI/CD-Agents
- **Tabnine** bedient Enterprise-Anforderungen mit Air-Gapped-Deployment
- **OpenClaw** erschließt neue Kanäle jenseits von Terminal und IDE

### Drei Szenarien für den Einstieg

**Szenario 1: "Ich will sofort loslegen, kostenlos"**
→ Installiere [[Gemini CLI Konfiguration|Gemini CLI]] (1K Requests/Tag frei) oder [[Aider Konfiguration|Aider]] + Ollama

**Szenario 2: "Ich will die beste Qualität, Kosten sind sekundär"**
→ Nutze [[Claude Code Konfiguration und Hooks|Claude Code]] mit Claude Sonnet/Opus und konfiguriere Hooks

**Szenario 3: "Ich will flexibel bleiben und nichts verpassen"**
→ Starte mit [[OpenCode Konfiguration|OpenCode]] (75+ Provider, wechselbar) und teste verschiedene Modelle

### Was sich 2026 schnell ändert

- **MCP** etabliert sich als Standard-Protokoll — Agents ohne MCP-Support rüsten nach
- **[[Skills]]** und **Agent Skills Spec** etablieren modulare, wiederverwendbare Agent-Expertise
- **Spec-driven Development** (Kiro, Augment) strukturiert AI-Coding über formale Spezifikationen
- **Multi-Agent-Workflows** werden ausgereifter (Sub-Agents, Orchestrierung) — siehe [[Multi-Agent Workflows]]
- **Context Engines** (Augment, Amp) verarbeiten ganze Codebases für tieferes Verständnis
- **Lokale Modelle** (Ollama, llama.cpp) verbessern sich rapide und schließen die Qualitätslücke zu Cloud-Modellen
- **Custom Instructions** konvergieren: `AGENTS.md` als möglicher universeller Standard — siehe [[Custom Instructions für AI Coding Agents]]
- **Spezialisierung**: Agents fokussieren sich stärker auf Nischen (AWS → Q/Kiro, Enterprise → Tabnine, Multi-Repo → Amp)
- **Kosten sinken**: API-Preise fallen kontinuierlich, Free Tiers werden großzügiger

> [!IMPORTANT] Dieses Dokument aktuell halten
> Die AI-Agent-Landschaft ändert sich monatlich. Prüfe die individuellen Konfigurationsseiten auf aktuelle Details und aktualisiere diese Vergleichstabelle regelmäßig.

---

### Weiterführende Dokumente

- [[Custom Instructions für AI Coding Agents]] — Alle Instruktions-Formate im Detail
- [[Custom Instructions für AI Coding Agents]] — Best Practices für Claude Code Instructions
- [[Skills]] — Modulare Agent-Expertise und Agent Skills Spec
- [[Multi-Agent Workflows]] — Fortgeschrittene Multi-Agent-Strategien
- [[Was ist ein AI Agent]] — Grundlagen und Konzepte
