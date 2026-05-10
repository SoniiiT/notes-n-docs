---
tags: [ai, agents, ide, cline, konfiguration]
---

# Cline Konfiguration

> [!info] Überblick
> **Cline** (ehemals „Claude Dev") ist ein autonomer AI Coding Agent als VS Code Extension.
> Mit ~57.000 GitHub Stars eines der beliebtesten Open-Source AI-Coding-Tools.
> GitHub: [clinebot/cline](https://github.com/clinebot/cline)

---

## Was ist Cline?

Cline ist eine **VS Code Extension**, die als vollständig autonomer AI-Agent arbeitet. Im Gegensatz zu reinen Autocomplete-Tools wie GitHub Copilot kann Cline eigenständig:

- **Dateien erstellen, bearbeiten und löschen**
- **Terminal-Befehle ausführen** (Build, Test, Install)
- **Browser automatisieren** (Screenshots, Interaktion)
- **Multi-File Refactoring** durchführen
- **Projektstrukturen analysieren** und verstehen

> [!tip] Kernkonzept
> Cline arbeitet nach dem **Human-in-the-Loop** Prinzip: Jede Aktion (Dateiänderung, Terminal-Befehl) wird dem Nutzer zur Genehmigung vorgelegt, bevor sie ausgeführt wird.

### Architektur

```
VS Code
 └── Cline Extension
      ├── Chat UI ──────── Permission Layer
      │                         │
      ├── Agent Core (Planung, Ausführung, Diff)
      │    ├── File Operations
      │    ├── Terminal
      │    └── Browser
      │
      └── LLM API (beliebiger Provider)
```

---

## Installation

### VS Code Extension installieren

1. **VS Code** öffnen
2. **Extensions Sidebar** öffnen (`Ctrl+Shift+X`)
3. Nach **„Cline"** suchen
4. **Install** klicken

Alternativ über die Kommandozeile:

```bash
code --install-extension saoudrizwan.claude-dev
```

> [!warning] Extension ID
> Die Extension heißt intern noch `saoudrizwan.claude-dev` (alter Name „Claude Dev"),
> im Marketplace wird sie aber als **Cline** geführt.

### Erste Schritte

1. Cline-Icon in der **Activity Bar** anklicken → API Provider und Key eingeben
2. Erste Nachricht im Chat eingeben — Cline beginnt zu arbeiten

---

## Provider und Modell-Konfiguration

Cline unterstützt eine Vielzahl von LLM-Providern. Die Konfiguration erfolgt im **Cline Settings Panel**.

### Unterstützte Provider

| Provider | Modelle (Beispiele) | Bemerkung |
|---|---|---|
| **Anthropic** | Claude Sonnet 4.6, Claude Opus 4.6 | Empfohlen für beste Ergebnisse |
| **OpenAI** | GPT-5.2, GPT-5.3 Codex | Gute Alternative |
| **Google** | Gemini 2.5 Pro/Flash, 3.1 Pro | Großes Context Window |
| **AWS Bedrock** | Claude via AWS | Für Enterprise-Umgebungen |
| **Google Vertex AI** | Gemini/Claude via GCP | Google Cloud Integration |
| **OpenRouter** | Zugang zu vielen Modellen | Meta-Provider, flexibel |
| **Ollama** | Llama, Mistral, CodeLlama | Lokal, kostenlos, privat |
| **LM Studio** | Beliebige GGUF-Modelle | Lokale GUI für Modelle |
| **vLLM / llama.cpp** | Beliebige Modelle | Self-Hosted Inference |

### Provider konfigurieren

1. **Cline Panel** öffnen → **Zahnrad-Icon** (Settings) klicken
2. **API Provider** aus Dropdown wählen → **API Key** eingeben → **Modell** auswählen

### Beispiel: Anthropic API

```
Provider:  Anthropic
API Key:   sk-ant-api03-xxxxxxxxxxxxx
Model:     claude-sonnet-4-20250514
```

### Beispiel: Ollama (lokal)

```bash
ollama pull llama3.1
# In Cline: Provider → Ollama, Base URL → http://localhost:11434, Model → llama3.1
```

### Beispiel: OpenRouter

```
Provider:  OpenRouter
API Key:   sk-or-v1-xxxxxxxxxxxxx
Model:     anthropic/claude-sonnet-4
```

> [!tip] Kostenübersicht
> Cline zeigt nach jeder Nachricht die **API-Kosten** an. So behält man den Überblick über die Ausgaben pro Session und insgesamt.

---

## Custom Instructions (.clinerules)

### Globale Custom Instructions

Im **Cline Settings Panel** gibt es ein Textfeld für **Custom Instructions**, die bei jeder Nachricht als System Prompt mitgesendet werden:

```text
Du bist ein erfahrener Senior Developer. Antworte immer auf Deutsch.
Verwende TypeScript mit strikten Typen.
```

### Projekt-spezifische Rules: `.clinerules`

Im **Projekt-Root** kann eine `.clinerules`-Datei angelegt werden. Diese wird automatisch erkannt und bei jeder Interaktion im jeweiligen Projekt mitgeladen.

````markdown
# .clinerules

## Projekt-Konventionen
- Sprache: TypeScript mit strict mode
- Framework: Next.js 15 mit App Router
- Styling: Tailwind CSS
- Tests: Vitest + Testing Library

## Code-Style
- Funktionale Komponenten mit React Hooks
- Named Exports bevorzugen

## Dateistruktur
```
src/
├── app/          # Next.js App Router
├── components/   # Wiederverwendbare Komponenten
├── lib/          # Utility-Funktionen
└── types/        # TypeScript Definitionen
```

## Verboten
- Keine `any` Types
- Keine `console.log` in Production Code
````

> [!info] Mehrere Rules-Dateien
> Cline unterstützt auch einen `.clinerules/`-**Ordner** mit mehreren `.md`-Dateien, um Rules thematisch zu trennen (z.B. `coding.md`, `testing.md`, `architecture.md`).

---

## Autonomer Agent Modus

Im Standard-Modus arbeitet Cline als **autonomer Agent**: Aufgabe analysieren → Dateien lesen → Änderungen als Diff vorschlagen → Terminal-Befehle ausführen → bei Fehlern selbstständig iterieren.

### Beispiel-Workflow

```
User: "Erstelle eine REST API mit Express für User-Management mit CRUD und SQLite."

Cline:
  1. → Erstellt package.json und führt `npm install` aus
  2. → Erstellt src/database.ts, src/routes/users.ts, src/index.ts
  3. → Führt Build und Start aus, testet mit curl
  4. → Korrigiert eventuelle Fehler automatisch
```

> [!warning] Achtung bei autonomem Modus
> Auch wenn Cline autonom arbeitet, wird **jede Aktion** einzeln zur Genehmigung vorgelegt. Man kann jederzeit ablehnen oder den Ansatz korrigieren.

### Auto-Approve Optionen

In den Cline-Settings können bestimmte Aktionen automatisch genehmigt werden:

| Option | Beschreibung |
|---|---|
| **Read files** | Dateien lesen ohne Nachfrage |
| **Write files** | Dateien schreiben ohne Nachfrage |
| **Execute commands** | Terminal-Befehle ohne Nachfrage |
| **Use browser** | Browser-Aktionen ohne Nachfrage |
| **Use MCP servers** | MCP-Tool-Aufrufe ohne Nachfrage |

> [!danger] Sicherheitshinweis
> Auto-Approve für **Write files** und **Execute commands** nur in isolierten Umgebungen (Container, VM) aktivieren. In Produktionsumgebungen immer manuell prüfen.

---

## Plan Mode

Der **Plan Mode** ist eine sicherere Alternative zum direkten autonomen Modus.

### Funktionsweise

1. **Plan Mode aktivieren** — Toggle im Chat-Interface (Act/Plan Schalter)
2. **Aufgabe beschreiben** — Cline erstellt einen detaillierten Plan
3. **Plan reviewen** — Nutzer prüft und passt an
4. **Ausführung** — Zurück zu Act Mode, Cline führt den Plan aus

```
Plan Mode → Review & Edit → Act Mode → (Iteration bei Anpassungen) → Plan Mode
```

---

## MCP Server Integration

Cline unterstützt das **Model Context Protocol (MCP)** zur Integration externer Tools und Datenquellen.

### Was ist MCP?

MCP ermöglicht es, externe **Tools**, **Ressourcen** und **Prompts** als Server bereitzustellen, die Cline nutzen kann.

### MCP Server konfigurieren

Die MCP-Konfiguration erfolgt über die Cline-Settings oder die Datei `cline_mcp_settings.json`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx" }
    }
  }
}
```

### Beliebte MCP Server

| Server | Funktion |
|---|---|
| **filesystem** | Erweiterter Dateizugriff |
| **github** | GitHub API Integration |
| **postgres** | Datenbank-Abfragen |
| **brave-search** | Web-Suche |
| **puppeteer** | Browser-Automation |

> [!tip] MCP im Cline-Kontext
> Cline kann MCP Server auch **selbst installieren und konfigurieren**. Man kann Cline einfach bitten: „Installiere den GitHub MCP Server" — und Cline erledigt die Einrichtung.

---

## Browser Automation

Cline kann einen **eingebetteten Browser** (headless Chromium) steuern:

### Fähigkeiten

- **Screenshots** aufnehmen und analysieren, **Klicken** auf Elemente
- **Text eingeben** in Formulare, **Navigieren** zu URLs
- **Console Logs** auslesen

### Anwendungsbeispiel

```
User: "Starte den Dev Server und prüfe, ob die Login-Seite korrekt rendert."

Cline:
  1. → Führt `npm run dev` aus
  2. → Öffnet Browser auf http://localhost:3000/login
  3. → Macht Screenshot und analysiert das Ergebnis
  4. → Gibt Ergebnis zurück
```

---

## Permission System

Das Permission System ist das **zentrale Sicherheitsfeature** von Cline.

### Aktionstypen und Genehmigungen

| Aktion | Genehmigung erforderlich | Risiko |
|---|---|---|
| Dateien lesen | Optional (Auto-Approve möglich) | 🟢 Niedrig |
| Dateien schreiben | Ja (Standard) | 🟡 Mittel |
| Dateien löschen | Ja (immer) | 🔴 Hoch |
| Terminal-Befehle | Ja (Standard) | 🔴 Hoch |
| Browser öffnen | Ja (Standard) | 🟡 Mittel |
| MCP Tools nutzen | Ja (Standard) | 🟡 Mittel |

### Diff View

Bei jeder Dateiänderung zeigt Cline einen **vollständigen Diff** an. Der Nutzer kann dann:
- ✅ **Approve** — Änderung wird übernommen
- ❌ **Reject** — Änderung wird verworfen
- 💬 **Feedback geben** — Cline überarbeitet den Vorschlag

---

## Context Features (@-Mentions)

Cline unterstützt **@-Mentions** im Chat, um gezielt Kontext bereitzustellen.

| Mention | Funktion | Beispiel |
|---|---|---|
| `@file` | Einzelne Datei referenzieren | `@src/index.ts` |
| `@folder` | Ganzen Ordner referenzieren | `@src/components` |
| `@url` | Webseite als Kontext laden | `@https://docs.example.com` |
| `@problems` | VS Code Probleme/Fehler | Aktuelle Diagnostics |
| `@terminal` | Terminal-Output referenzieren | Letzter Terminal-Output |
| `@git` | Git-Änderungen referenzieren | Aktuelle Diff-Ansicht |

### Beispiele

```
"Refactore @src/utils/helpers.ts nach dem Muster in @src/utils/format.ts"
"Behebe die Fehler in @problems"
"Implementiere das API laut @https://api-docs.example.com/v2/users"
```

---

## Memory Bank

Die **Memory Bank** ermöglicht persistenten Projekt-Kontext über Sessions hinweg.

### Konzept

Da LLMs keinen dauerhaften Speicher haben, verwendet die Memory Bank Markdown-Dateien im Projekt, um wichtigen Kontext über Sessions hinweg festzuhalten.

### Struktur

Die Memory Bank besteht aus Markdown-Dateien im Ordner `memory-bank/`:

````markdown
# memory-bank/projectbrief.md

## Projektbeschreibung
E-Commerce Platform mit Next.js und Stripe Integration.

## Technologie-Stack
- Frontend: Next.js 15, React 19, Tailwind CSS
- Backend: tRPC, Prisma, PostgreSQL
````

### Typische Memory Bank Dateien

| Datei | Inhalt |
|---|---|
| `projectbrief.md` | Projektübersicht und Ziele |
| `productContext.md` | Warum existiert das Produkt, UX-Ziele |
| `techContext.md` | Technologie-Stack, Architektur |
| `activeContext.md` | Aktueller Arbeitsstand |
| `systemPatterns.md` | Architektur-Patterns |
| `progress.md` | Was funktioniert, was fehlt |

> [!tip] Memory Bank initialisieren
> Man kann Cline bitten: „Initialisiere eine Memory Bank für dieses Projekt" — Cline erstellt dann die passende Struktur und befüllt sie mit Kontext aus dem bestehenden Code.

---

## Vergleich: Cline vs. Cursor vs. CLI Agents

| Feature | Cline | Cursor | CLI Agents |
|---|---|---|---|
| **Typ** | VS Code Extension | Eigener Editor | Terminal-basiert |
| **LLM-Flexibilität** | Jedes LLM | Cursor-Modelle + eigene | Provider-spezifisch |
| **MCP Support** | ✅ Ja | ✅ Ja | ✅ Teilweise |
| **Browser Automation** | ✅ Ja | ❌ Nein | ❌ Nein |
| **Kosten** | Nur API-Kosten | Abo + API | Nur API-Kosten |
| **Open Source** | ✅ Ja (Apache 2.0) | ❌ Nein | Teilweise |
| **Plan Mode** | ✅ Ja | ❌ Nein | Teilweise |
| **Lokale Modelle** | ✅ Ja (Ollama) | ❌ Nein | ❌ Nein |

### Wann welches Tool?

> [!tip] Empfehlung
> - **Cline**: Wenn man VS Code behalten will, lokale Modelle nutzen möchte oder maximale Flexibilität braucht
> - **Cursor**: Wenn man einen integrierten Editor mit AI bevorzugt und ein Abo-Modell akzeptiert
> - **CLI Agents**: Wenn man Terminal-basiert arbeitet, CI/CD-Integration braucht oder Headless-Umgebungen nutzt

---

## Tipps und Best Practices

### 1. Kontext gezielt bereitstellen

```
❌ Schlecht:  "Behebe den Bug"
✅ Besser:   "Behebe den TypeError in @src/api/users.ts Zeile 42. 
              Der Fehler tritt auf wenn `userId` undefined ist."
```

### 2. .clinerules pflegen

```markdown
# .clinerules — Beispiel
- Verwende immer `pnpm` statt `npm`
- Teste mit `pnpm test` nach jeder Änderung
```

### 3. Plan Mode für komplexe Tasks

Für größere Aufgaben zuerst im **Plan Mode** arbeiten, dann Plan reviewen, dann Act Mode.

### 4. Kosten im Blick behalten

- Cline zeigt **Token-Verbrauch und Kosten** pro Nachricht
- Kürzere Konversationen sind günstiger als lange
- Neue Tasks → Neuer Chat (statt langer Konversation)
- Lokale Modelle via **Ollama** für Experimentieren nutzen

### 5. Auto-Approve bewusst einsetzen

| Option | Empfohlen | Aggressiv (nur isoliert) |
|---|---|---|
| Read files | ✅ Auto | ✅ Auto |
| Write files | ❌ Manuell | ✅ Auto |
| Execute commands | ❌ Manuell | ✅ Auto |
| Browser | ❌ Manuell | ❌ Manuell |

### 6. Memory Bank und MCP nutzen

- Für wiederkehrende Projekte eine **Memory Bank** einrichten (Tech Stack, Architektur, offene Tasks)
- Häufige externe Integrationen als **MCP Server** konfigurieren (Datenbank, API-Docs, CI/CD)
- **Bild-Input** für UI-Arbeit: Screenshot in den Chat ziehen, Cline analysiert und korrigiert

---

## Nützliche Ressourcen

| Ressource | Link |
|---|---|
| GitHub Repository | [clinebot/cline](https://github.com/clinebot/cline) |
| VS Code Marketplace | Extensions → „Cline" suchen |
| MCP Protokoll | [modelcontextprotocol.io](https://modelcontextprotocol.io) |

---

> [!summary] Zusammenfassung
> Cline ist ein leistungsstarker, **Open-Source AI Coding Agent** für VS Code mit maximaler LLM-Flexibilität. Durch das granulare Permission System, den Plan Mode und die MCP-Integration bietet es eine sichere und erweiterbare Umgebung für AI-gestütztes Coding. Die Möglichkeit, lokale Modelle über Ollama zu nutzen, macht es besonders attraktiv für datenschutzbewusste Entwickler.

---

*Zuletzt aktualisiert: Juli 2025*

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Übersicht und Vergleich aller AI Coding Agents
- [[Was ist ein AI Agent]] — Grundlagen und Definition von AI Agents
- [[Custom Instructions für AI Coding Agents]] — Best Practices für Instruktionsdateien und Konfiguration
- [[MCP Tools Resources Prompts]] — Detaillierte Erklärung der MCP-Komponenten Tools, Resources und Prompts
