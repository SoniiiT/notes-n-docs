---
tags: [ai, agents, cli, opencode, konfiguration]
---

# OpenCode — AI Coding Agent für das Terminal

## Was ist OpenCode?

OpenCode ist ein **open-source AI Coding Agent**, der direkt im Terminal läuft. Mit ~97.000 GitHub-Stars gehört es zu den populärsten CLI-basierten Coding-Assistenten überhaupt. Das Projekt verfolgt einen **model-agnostischen, vendor-neutralen** Ansatz und unterstützt über **75 LLM-Provider** — von OpenAI über Anthropic und Google bis hin zu lokalen Modellen via Ollama.

> [!info] GitHub Repository
> **Repository:** [opencode-ai/opencode](https://github.com/opencode-ai/opencode)
> **Sprache:** Go (TUI mit Bubble Tea)
> **Lizenz:** MIT
> **Stars:** ~97K

### Kernfeatures

| Feature                  | Beschreibung                                                  |
| ------------------------ | ------------------------------------------------------------- |
| 75+ Provider             | OpenAI, Anthropic, Google, Ollama, AWS Bedrock, Azure u.v.m.  |
| Terminal UI (TUI)        | Interaktive Oberfläche mit Bubble Tea                         |
| Multi-Agent Modus        | Mehrere Agents parallel ausführen                             |
| Plan & Build Modi        | Strukturiertes Planen und Umsetzen                            |
| LSP Integration          | Language Server Protocol für Code Intelligence                |
| MCP Support              | Model Context Protocol für erweiterte Tool-Nutzung            |
| Vim-Style Editor         | Vertraute Tastenkürzel für Entwickler                         |
| Permission System        | Granulare Kontrolle über Aktionen (ask/allow/deny)            |
| Persistent Conversations | Gespräche werden lokal gespeichert                            |
| AGENTS.md                | Custom Instructions pro Projekt                               |
| Local-first              | Alle Daten bleiben auf dem eigenen Rechner                    |

---

## Installation

OpenCode kann über verschiedene Paketmanager installiert werden.

### npm / npx

````bash
# Global installieren
npm install -g opencode

# Oder direkt ausführen ohne Installation
npx opencode
````

### Homebrew (macOS / Linux)

````bash
brew install opencode
````

### Chocolatey (Windows)

````bash
choco install opencode
````

### Scoop (Windows)

````bash
scoop install opencode
````

### Docker

````bash
docker run -it --rm \
  -v "$(pwd):/workspace" \
  -w /workspace \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  ghcr.io/opencode-ai/opencode
````

> [!tip] Empfehlung
> Für den schnellsten Einstieg reicht `npx opencode` — keine Installation nötig. Für regelmäßige Nutzung empfiehlt sich die globale Installation via npm oder Homebrew.

### Voraussetzungen

- **Node.js** ≥ 18 (für npm-Installation)
- Mindestens ein **API-Key** eines unterstützten LLM-Providers
- Terminal mit **256-Farb-Unterstützung** (empfohlen für TUI)

---

## Konfiguration und Provider Setup

Die Konfiguration erfolgt über eine `opencode.json` Datei im Projektverzeichnis oder im Home-Verzeichnis unter `~/.config/opencode/config.json`.

### Grundkonfiguration

````json
{
  "provider": {
    "name": "anthropic",
    "model": "claude-sonnet-4-20250514"
  }
}
````

### Provider konfigurieren

OpenCode unterstützt über 75 Provider. Die wichtigsten:

| Provider   | Env-Variable             | Beispiel-Modell              |
| ---------- | ------------------------ | ---------------------------- |
| OpenAI     | `OPENAI_API_KEY`         | `gpt-4o`                     |
| Anthropic  | `ANTHROPIC_API_KEY`      | `claude-sonnet-4-20250514`   |
| Google     | `GOOGLE_API_KEY`         | `gemini-2.5-pro`             |
| Ollama     | —                        | `llama3:latest`              |
| AWS Bedrock| `AWS_ACCESS_KEY_ID`      | `anthropic.claude-3-sonnet`  |
| Azure      | `AZURE_OPENAI_API_KEY`   | `gpt-4o`                     |

### Anthropic Setup

````bash
export ANTHROPIC_API_KEY="sk-ant-..."
````

````json
{
  "provider": {
    "name": "anthropic",
    "model": "claude-sonnet-4-20250514"
  }
}
````

### OpenAI Setup

````bash
export OPENAI_API_KEY="sk-..."
````

````json
{
  "provider": {
    "name": "openai",
    "model": "gpt-4o"
  }
}
````

### Ollama (Lokale Modelle)

````bash
# Ollama muss lokal laufen
ollama serve

# Modell herunterladen
ollama pull llama3:latest
````

````json
{
  "provider": {
    "name": "ollama",
    "model": "llama3:latest",
    "baseURL": "http://localhost:11434"
  }
}
````

> [!warning] API-Keys sicher aufbewahren
> API-Keys niemals in die `opencode.json` schreiben, sondern immer über **Umgebungsvariablen** setzen. Am besten in `.bashrc`, `.zshrc` oder einem Secret Manager.

### Provider zur Laufzeit wechseln

Mit dem `/connect` Command kann der Provider direkt in der laufenden Session gewechselt werden:

````
/connect anthropic claude-sonnet-4-20250514
/connect openai gpt-4o
/connect ollama llama3:latest
````

---

## Custom Instructions (AGENTS.md)

OpenCode unterstützt eine `AGENTS.md` Datei im Projektstamm, um dem Agent projektspezifische Anweisungen zu geben — vergleichbar mit Cursor Rules oder GitHub Copilot Instructions.

### Aufbau einer AGENTS.md

````markdown
# Project Instructions

## Code Style
- Use TypeScript strict mode
- Follow ESLint Airbnb config
- Use functional components with hooks

## Architecture
- Clean Architecture mit Layers: Domain, Application, Infrastructure
- Repository Pattern für Datenzugriff
- DTOs für API-Grenzen

## Testing
- Jest für Unit Tests
- Playwright für E2E Tests
- Mindestens 80% Coverage

## Conventions
- Commit Messages: Conventional Commits
- Branch Naming: feature/ticket-id-beschreibung
- PR Description in Deutsch
````

> [!tip] Best Practice
> Die `AGENTS.md` sollte im Repository eingecheckt werden, damit alle Teammitglieder die gleichen Instruktionen nutzen. Sie kann auch verschachtelte Dateien referenzieren.

---

## Modi (Plan, Build)

OpenCode bietet zwei Hauptmodi, die den Workflow strukturieren.

### Plan Modus

Im **Plan Modus** analysiert der Agent das Projekt und erstellt einen strukturierten Umsetzungsplan, **ohne Änderungen vorzunehmen**.

- Analyse der bestehenden Codebasis
- Identifikation betroffener Dateien
- Erstellung eines Schritt-für-Schritt-Plans
- Risikobewertung und Abhängigkeiten

````
/plan Implementiere eine REST API für User-Management mit CRUD Operationen
````

Der Agent gibt dann einen detaillierten Plan aus, z.B.:

```
1. Erstelle User-Model in src/models/user.ts
2. Erstelle User-Repository in src/repositories/userRepository.ts
3. Implementiere CRUD-Endpoints in src/routes/users.ts
4. Füge Validierung mit Zod hinzu
5. Schreibe Unit Tests
```

### Build Modus

Im **Build Modus** setzt der Agent Änderungen aktiv um — er erstellt, bearbeitet und löscht Dateien.

````
/build Implementiere den Plan aus der vorherigen Analyse
````

> [!note] Plan → Build Workflow
> Der empfohlene Workflow ist: Erst mit `/plan` einen Plan erstellen lassen, diesen reviewen und anpassen, dann mit `/build` umsetzen. So behält man die volle Kontrolle über den Prozess.

---

## Multi-Agent Modus

OpenCode kann **mehrere Agents parallel** ausführen, um komplexe Aufgaben effizienter zu bearbeiten.

### Funktionsweise

- Ein **Orchestrator-Agent** koordiniert die Arbeit
- Mehrere **Worker-Agents** arbeiten parallel an Teilaufgaben
- Ergebnisse werden zusammengeführt und validiert

### Anwendungsbeispiele

| Szenario                          | Agents                                         |
| --------------------------------- | ---------------------------------------------- |
| Feature mit Tests                 | Agent 1: Code, Agent 2: Tests                  |
| Refactoring über mehrere Dateien  | Je ein Agent pro Modul/Dateigruppe              |
| Code Review + Fix                 | Agent 1: Analyse, Agent 2: Fixes               |
| API + Frontend                    | Agent 1: Backend, Agent 2: Frontend             |

> [!info] Parallelität
> Der Multi-Agent Modus ist besonders nützlich bei großen Refactorings oder wenn unabhängige Teilaufgaben gleichzeitig bearbeitet werden können. Die Agents teilen sich den Kontext, arbeiten aber unabhängig.

---

## MCP und LSP Integration

### MCP (Model Context Protocol)

OpenCode unterstützt das **Model Context Protocol** von Anthropic, um externe Tools und Datenquellen einzubinden.

````json
{
  "mcp": {
    "servers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
      },
      "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {
          "GITHUB_TOKEN": "ghp_..."
        }
      }
    }
  }
}
````

Unterstützte MCP-Server (Beispiele):

- **Filesystem** — Dateizugriff und -verwaltung
- **GitHub** — Issues, PRs, Repositories
- **PostgreSQL** — Datenbankabfragen
- **Brave Search** — Websuche
- **Memory** — Persistenter Wissensspeicher

### LSP (Language Server Protocol)

OpenCode integriert sich mit **Language Servern**, um Code Intelligence zu nutzen:

- **Go to Definition** — Zur Definition springen
- **Find References** — Alle Referenzen finden
- **Diagnostics** — Fehler und Warnungen in Echtzeit
- **Completions** — Kontextbezogene Vorschläge
- **Hover** — Typ-Informationen und Dokumentation

Die LSP-Integration ist automatisch aktiv, wenn ein passender Language Server im System installiert ist (z.B. `typescript-language-server`, `gopls`, `pyright`).

> [!tip] LSP-Vorteil
> Durch die LSP-Integration kann OpenCode präzisere Code-Änderungen vornehmen, da es die Codebasis semantisch versteht — nicht nur textbasiert sucht.

---

## Permission System

OpenCode implementiert ein **granulares Permission System**, das Kontrolle über alle Aktionen gibt.

### Permission Levels

| Level   | Beschreibung                                      |
| ------- | ------------------------------------------------- |
| `ask`   | Vor jeder Aktion wird um Erlaubnis gefragt        |
| `allow` | Aktion wird automatisch erlaubt                   |
| `deny`  | Aktion wird automatisch blockiert                 |

### Konfiguration

````json
{
  "permissions": {
    "file_write": "ask",
    "file_delete": "ask",
    "command_execute": "ask",
    "network_access": "deny"
  }
}
````

### Aktionstypen

| Aktion             | Standard | Beschreibung                          |
| ------------------ | -------- | ------------------------------------- |
| `file_write`       | `ask`    | Dateien erstellen oder bearbeiten     |
| `file_delete`      | `ask`    | Dateien löschen                       |
| `command_execute`  | `ask`    | Shell-Befehle ausführen               |
| `network_access`   | `deny`   | Netzwerkzugriffe                      |

> [!warning] Sicherheitshinweis
> Für Produktionsprojekte sollte `command_execute` auf `ask` bleiben. Niemals `allow` für `file_delete` oder `network_access` in unbekannten Projekten setzen.

---

## Wichtige Commands

### Slash Commands

| Command         | Beschreibung                                       |
| --------------- | -------------------------------------------------- |
| `/init`         | Projekt scannen und Kontext aufbauen               |
| `/connect`      | Provider/Modell wechseln                           |
| `/plan`         | Plan Modus aktivieren                              |
| `/build`        | Build Modus aktivieren                             |
| `/clear`        | Konversation zurücksetzen                           |
| `/compact`      | Kontext komprimieren                               |
| `/help`         | Hilfe anzeigen                                     |
| `/quit`         | OpenCode beenden                                   |

### Tastenkürzel (TUI)

| Kürzel       | Aktion                          |
| ------------ | ------------------------------- |
| `i`          | Insert Mode (Eingabe)           |
| `Esc`        | Normal Mode                     |
| `Ctrl+C`     | Abbrechen / Beenden             |
| `Ctrl+L`     | Bildschirm leeren               |
| `j` / `k`    | Scrollen (Vim-Style)            |
| `g` / `G`    | Anfang / Ende                   |
| `/`          | Slash Command starten           |

### Typischer Workflow

````bash
# 1. OpenCode im Projektverzeichnis starten
cd mein-projekt
opencode

# 2. Projekt scannen
/init

# 3. Plan erstellen
/plan Erstelle eine REST API mit Express und TypeScript

# 4. Plan reviewen und dann umsetzen
/build

# 5. Bei Bedarf Provider wechseln
/connect anthropic claude-sonnet-4-20250514
````

---

## Vergleich mit anderen CLI Agents

| Feature                  | OpenCode      | Claude Code   | Aider         | GitHub Copilot CLI |
| ------------------------ | ------------- | ------------- | ------------- | ------------------ |
| Open Source               | ✅            | ❌            | ✅            | ❌                 |
| Provider-Agnostisch       | ✅ (75+)      | ❌ (nur Claude)| ✅            | ❌ (nur GitHub)    |
| TUI                       | ✅ (Bubble Tea)| ✅           | ❌            | ❌                 |
| Multi-Agent               | ✅            | ❌            | ❌            | ❌                 |
| MCP Support               | ✅            | ✅            | ❌            | ✅                 |
| LSP Integration           | ✅            | ✅            | ❌            | ❌                 |
| Plan/Build Modi           | ✅            | ✅            | ❌            | ❌                 |
| Permission System         | ✅            | ✅            | ❌            | ✅                 |
| Lokale Modelle (Ollama)   | ✅            | ❌            | ✅            | ❌                 |
| Custom Instructions       | ✅ (AGENTS.md)| ✅ (CLAUDE.md)| ✅ (.aider)   | ✅                 |

> [!note] Stärken von OpenCode
> Der größte Vorteil von OpenCode ist die **Kombination aus Provider-Agnostik und lokalem Betrieb**. Man kann frei zwischen Cloud- und lokalen Modellen wechseln, ohne den Workflow zu ändern.

---

## Tipps und Troubleshooting

### Performance-Tipps

1. **`/init` beim Start nutzen** — Gibt dem Agent besseren Projektkontext
2. **`/compact` bei langen Sessions** — Komprimiert den Kontext und spart Tokens
3. **AGENTS.md pflegen** — Je besser die Instruktionen, desto bessere Ergebnisse
4. **Plan vor Build** — Reduziert Fehler und unnötige Iterationen
5. **Lokale Modelle für Prototyping** — Spart API-Kosten bei Experimenten

### Häufige Probleme

> [!warning] TUI rendert nicht korrekt
> **Problem:** Zeichen werden falsch dargestellt oder die UI ist verzerrt.
> **Lösung:** Terminal auf UTF-8 und 256 Farben konfigurieren. Moderne Terminals wie Kitty, Alacritty oder WezTerm werden empfohlen.

> [!warning] API-Key wird nicht erkannt
> **Problem:** `Error: No API key found for provider`
> **Lösung:** Sicherstellen, dass die Umgebungsvariable korrekt gesetzt ist:
> ````bash
> echo $ANTHROPIC_API_KEY  # Sollte den Key ausgeben
> export ANTHROPIC_API_KEY="sk-ant-..."
> ````

> [!warning] Ollama-Verbindung fehlgeschlagen
> **Problem:** `Connection refused` bei lokalem Ollama
> **Lösung:** Prüfen ob Ollama läuft und der Port stimmt:
> ````bash
> ollama serve           # Ollama starten
> curl localhost:11434   # Verbindung testen
> ````

### Konversationen verwalten

OpenCode speichert alle Konversationen **lokal persistent**. Sie können nach einem Neustart fortgesetzt werden.

````bash
# Gespeicherte Konversationen befinden sich unter:
~/.local/share/opencode/conversations/
````

---

## Status und Nachfolger (Crush)

> [!caution] Projekt archiviert
> Das OpenCode-Repository wurde **Ende 2025 archiviert**. Das Projekt wird nicht mehr aktiv weiterentwickelt. Der offizielle Nachfolger ist **Crush**.

### Was bedeutet das?

- **Keine neuen Features** — OpenCode erhält keine Updates mehr
- **Keine Bugfixes** — Bekannte Probleme werden nicht behoben
- **Bestehende Installationen funktionieren** — Der Agent ist weiterhin nutzbar
- **API-Kompatibilität** — Solange die Provider-APIs kompatibel bleiben, funktioniert OpenCode

### Crush — Der Nachfolger

**Crush** ist der offizielle Nachfolger von OpenCode und wird vom gleichen Team entwickelt. Es baut auf den Erkenntnissen von OpenCode auf und bietet:

- Verbesserte Architektur
- Noch mehr Provider-Unterstützung
- Erweiterte Multi-Agent-Fähigkeiten
- Bessere Performance und Token-Effizienz

> [!tip] Migration
> Wer OpenCode produktiv nutzt, sollte den Umstieg auf Crush planen. Die Konfiguration ist weitgehend kompatibel, sodass eine Migration relativ unkompliziert ist.

### Empfehlung

| Situation                             | Empfehlung          |
| ------------------------------------- | ------------------- |
| Neues Projekt starten                 | → Crush verwenden   |
| OpenCode läuft stabil                 | → Weiterverwenden   |
| Neue Features benötigt                | → Zu Crush wechseln |
| Lokale Modelle wichtig                | → Beide evaluieren  |

---

## Weiterführende Links

- [OpenCode GitHub (archiviert)](https://github.com/opencode-ai/opencode)
- [Model Context Protocol](https://modelcontextprotocol.io)
- [Bubble Tea TUI Framework](https://github.com/charmbracelet/bubbletea)
- [Ollama](https://ollama.ai)

---

> [!quote] Fazit
> OpenCode war ein wegweisendes Projekt, das gezeigt hat, wie ein **offener, provider-agnostischer AI Coding Agent** im Terminal aussehen kann. Trotz der Archivierung bleibt es eine wertvolle Referenz — und sein Nachfolger Crush führt die Vision weiter.

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Übersicht und Vergleich aller AI Coding Agents
- [[Was ist ein AI Agent]] — Grundlagen und Definition von AI Agents
- [[Custom Instructions für AI Coding Agents]] — Best Practices für Instruktionsdateien und Konfiguration
- [[LLM API Provider und Router]] — Überblick über LLM-Anbieter und API-Router
