---
tags: [ai, agents, cli, openai-codex, konfiguration]
---

# OpenAI Codex CLI Konfiguration

## Was ist OpenAI Codex CLI?

OpenAI Codex CLI ist ein Open-Source, terminal-basierter KI-Coding-Agent von OpenAI, der direkt in der Kommandozeile arbeitet. Er kombiniert die Fähigkeiten von GPT-Modellen mit der Möglichkeit, Code zu lesen, zu schreiben, Shell-Befehle auszuführen und komplexe Entwicklungsaufgaben autonom zu bearbeiten – alles innerhalb des Terminals.

> [!NOTE]
> Codex CLI war eines der ersten Tools, das `AGENTS.md` als Standard-Instruktionsdatei eingeführt hat. Dieses Format wird mittlerweile auch von GitHub Copilot CLI und anderen Agents unterstützt.

**Kernfähigkeiten:**
- Code-Analyse und -Generierung direkt im Terminal
- Dateien lesen, erstellen und bearbeiten
- Shell-Befehle ausführen (Builds, Tests, Linting)
- Interaktives TUI (Terminal User Interface)
- Konfigurierbares Sandbox-System für sichere Ausführung
- Multi-Agent-Modus für parallele Aufgaben
- MCP-Server-Integration für externe Tools
- Web-Suche (cached oder live)

---

## Voraussetzungen

### Systemanforderungen

| Anforderung | Details |
|---|---|
| Node.js | Version 18+ (20+ empfohlen) |
| Betriebssystem | macOS, Linux |
| Windows | Nur über WSL2 (experimentell) |
| Account | ChatGPT Plus ($20/Mo), Pro ($200/Mo), Business ($30/user/Mo) oder API Key |

> [!IMPORTANT]
> Codex CLI benötigt entweder einen aktiven ChatGPT-Plan (Plus $20/Mo, Pro $200/Mo oder Business $30/user/Mo) für die OAuth-Authentifizierung oder einen OpenAI API Key. Ohne eines von beiden kann Codex CLI nicht genutzt werden.

### Node.js installieren

```bash
# Node.js Version prüfen
node --version

# Falls nicht installiert oder veraltet – über nvm installieren
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
nvm install 22
nvm use 22
```

---

## Installation

### Alle Plattformen im Überblick

| Plattform | Methode | Befehl |
|---|---|---|
| Alle (Node.js) | npm (empfohlen) | `npm install -g @openai/codex` |
| macOS | Homebrew | `brew install codex` |

### Installation per npm (Empfohlen)

```bash
# Installation
npm install -g @openai/codex

# Installation prüfen
codex --version

# Update auf neueste Version
npm update -g @openai/codex
```

### Installation per Homebrew (macOS)

```bash
brew install codex

# Update
brew upgrade codex
```

> [!TIP]
> Die npm-Methode ist plattformübergreifend und die empfohlene Installationsart. Homebrew ist eine bequeme Alternative für macOS-Nutzer.

---

## Starten

Nach der Installation wird Codex CLI über den Befehl `codex` gestartet:

```bash
# Codex CLI im interaktiven Modus starten
codex

# In einem bestimmten Verzeichnis starten
cd /pfad/zum/projekt && codex

# Direkt mit einem Prompt starten
codex "Erkläre die Struktur dieses Projekts"
```

> [!IMPORTANT]
> Codex CLI arbeitet kontextbezogen im aktuellen Verzeichnis. Es erkennt automatisch Git-Repositories und liest vorhandene Instruktionsdateien (`AGENTS.md`) ein.

---

## Authentifizierung

### OAuth via Browser (ChatGPT Account)

Beim ersten Start wird automatisch ein OAuth-Login-Flow über den Browser gestartet:

```bash
# Codex starten → automatischer Login-Prompt
codex

# Codex öffnet den Browser für die Anmeldung
# → Mit ChatGPT-Account (Plus/Pro/Team/Business/Enterprise) einloggen
# → Token wird lokal gespeichert
```

Der Login erfolgt über den Browser via OpenAI OAuth. Nach erfolgreicher Authentifizierung wird die Session gespeichert und muss nicht bei jedem Start wiederholt werden.

### Authentifizierung per API Key

Für API-basierte Nutzung oder CI/CD-Umgebungen kann ein OpenAI API Key verwendet werden:

```bash
# API Key als Umgebungsvariable setzen
export OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxxxxxx"

# Dann Codex starten
codex
```

> [!WARNING]
> Der API Key wird direkt über die OpenAI API abgerechnet. Im Gegensatz zur OAuth-Methode (die im ChatGPT-Plan enthalten ist) fallen hier separate API-Kosten an.

### API Key dauerhaft konfigurieren

```bash
# In Shell-Profil setzen (bash)
echo 'export OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxxxxxx"' >> ~/.bashrc
source ~/.bashrc

# Für zsh
echo 'export OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxxxxxx"' >> ~/.zshrc
source ~/.zshrc
```

> [!CAUTION]
> API Keys niemals in Repositories committen! Nutze `.env`-Dateien (in `.gitignore`) oder einen Secret Manager.

---

## Konfiguration (config.toml)

### Konfigurationsdatei

Codex CLI wird über eine zentrale TOML-Konfigurationsdatei gesteuert:

| Speicherort | Beschreibung |
|---|---|
| `~/.codex/config.toml` | Globale Konfiguration (User-Level) |

### Wichtige Konfigurationsoptionen

| Option | Werte | Beschreibung |
|---|---|---|
| `model` | `"gpt-5.3-codex"`, `"o4-mini"`, `"o3"` u.a. | Zu verwendendes KI-Modell |
| `model_reasoning_effort` | `"high"`, `"medium"`, `"low"` | Reasoning-Intensität des Modells |
| `approval_policy` | `"on-request"`, `"never"`, `"untrusted"` | Wann Nutzer-Bestätigung erforderlich ist |
| `sandbox_mode` | `"read-only"`, `"workspace-write"`, `"danger-full-access"` | Dateisystem-Zugriffslevel |
| `web_search` | `"cached"`, `"live"` | Web-Suche Modus |

### Vollständiges Konfigurationsbeispiel

````toml
# ~/.codex/config.toml

# Modell-Konfiguration
model = "gpt-5.3-codex"
model_reasoning_effort = "high"

# Approval-Verhalten
approval_policy = "on-request"

# Sandbox-Modus
sandbox_mode = "workspace-write"

# Web-Suche
web_search = "cached"

# Feature-Flags
[features]
multi_agent = true
shell_tool = true
````

### Konfigurationsoptionen im Detail

#### model – Modellwahl

Das `model`-Feld bestimmt, welches KI-Modell verwendet wird:

| Modell | Typ | Stärken |
|---|---|---|
| `gpt-5.3-codex` | Codex-optimiert | Speziell für Coding-Aufgaben trainiert, schnell |
| `o4-mini` | Reasoning | Gutes Reasoning bei geringeren Kosten |
| `o3` | Reasoning (Premium) | Maximale Reasoning-Fähigkeiten |
| `gpt-4.1` | General Purpose | Allgemein stark, gut für komplexe Aufgaben |

#### model_reasoning_effort – Reasoning-Intensität

Steuert, wie intensiv das Modell über Probleme nachdenkt:

```toml
# Tiefes Nachdenken – für komplexe Architektur-Entscheidungen
model_reasoning_effort = "high"

# Ausgewogen – für die meisten Aufgaben
model_reasoning_effort = "medium"

# Schnelle Antworten – für einfache Aufgaben
model_reasoning_effort = "low"
```

> [!TIP]
> Für komplexe Refactorings oder Architektur-Entscheidungen empfiehlt sich `"high"`. Für einfache Code-Generierung reicht `"low"` und spart Tokens.

#### approval_policy – Bestätigungsverhalten

| Wert | Verhalten |
|---|---|
| `"on-request"` | Bestätigung wird bei potenziell riskanten Aktionen angefragt (Standard) |
| `"never"` | Keine automatische Bestätigung – alles muss manuell bestätigt werden |
| `"untrusted"` | Nur bei Aktionen aus nicht vertrauenswürdigen Quellen |

#### sandbox_mode – Dateisystem-Zugriff

| Modus | Lesen | Schreiben | Beschreibung |
|---|---|---|---|
| `"read-only"` | ✅ | ❌ | Kann nur Dateien lesen, keine Änderungen |
| `"workspace-write"` | ✅ | ✅ (Workspace) | Kann innerhalb des Workspaces schreiben |
| `"danger-full-access"` | ✅ | ✅ (Überall) | Vollzugriff auf das gesamte Dateisystem |

> [!WARNING]
> `"danger-full-access"` sollte nur in isolierten Umgebungen (Container, VMs) verwendet werden. In Produktionsumgebungen immer `"workspace-write"` oder `"read-only"` nutzen.

---

## Custom Instructions (AGENTS.md)

### Die Rolle von AGENTS.md

Codex CLI war eines der ersten Tools, das `AGENTS.md` als Standard-Instruktionsdatei etabliert hat. Diese Datei steuert das Verhalten des Agenten projektspezifisch.

> [!INFO]
> `AGENTS.md` wird mittlerweile auch von GitHub Copilot CLI gelesen. Wer bereits eine `AGENTS.md`-Datei hat, profitiert automatisch bei beiden Tools. Für Details zu Custom Instructions allgemein siehe [[Custom Instructions für AI Coding Agents]].

### Unterstützte Instruktionsdateien

| Datei | Speicherort | Beschreibung |
|---|---|---|
| `AGENTS.md` | Projekt-Root | Primäre Instruktionsdatei für Codex CLI |
| `AGENTS.md` | Unterverzeichnisse | Kontextspezifische Instruktionen für Teilbereiche |

### AGENTS.md erstellen

````markdown
# AGENTS.md

## Projekt-Kontext
Dieses Projekt ist eine REST-API auf Basis von Node.js und TypeScript.

## Code-Konventionen
- Verwende TypeScript strict mode
- Funktionale Programmierung bevorzugen
- Alle Funktionen mit JSDoc kommentieren
- Error Handling mit Custom Error Classes

## Testing
- Jest für Unit-Tests
- Supertest für API-Tests
- Testdateien: `*.test.ts`
- Mindestens 80% Code-Coverage

## Git-Konventionen
- Conventional Commits (feat:, fix:, docs:, refactor:)
- Feature-Branches: `feature/beschreibung`
- Squash-Merge in main
````

### Hierarchische AGENTS.md

Codex CLI unterstützt `AGENTS.md`-Dateien in Unterverzeichnissen. Die Instruktionen werden hierarchisch zusammengeführt:

```
projekt/
├── AGENTS.md              ← Globale Projekt-Instruktionen
├── src/
│   ├── AGENTS.md          ← Spezifisch für src/
│   └── api/
│       └── AGENTS.md      ← Spezifisch für api/
└── tests/
    └── AGENTS.md          ← Spezifisch für Tests
```

> [!TIP]
> Nutze hierarchische `AGENTS.md`-Dateien, um verschiedenen Teilen des Projekts unterschiedliche Instruktionen zu geben. Zum Beispiel könnten Frontend und Backend unterschiedliche Coding-Styles haben.

---

## Approval Modi und Sandbox

### Approval Modi (CLI-Flags)

Codex CLI bietet drei Approval Modi, die über CLI-Flags beim Start gesetzt werden:

| Flag | Modus | Verhalten |
|---|---|---|
| `--suggest` | Suggest | Manuelle Bestätigung für **alles** – sicherster Modus |
| `--auto-edit` | Auto-Edit | Datei-Edits automatisch, riskante Aktionen mit Bestätigung |
| `--full-auto` | Full Auto | Voller Zugriff, keine Bestätigung nötig |

#### --suggest (Empfohlen für sensible Arbeit)

```bash
codex --suggest "Refactore die Auth-Logik"
```

- Jede Dateiänderung muss manuell bestätigt werden
- Jeder Shell-Befehl muss bestätigt werden
- Maximale Kontrolle über den Agenten

#### --auto-edit (Standard für die meisten Aufgaben)

```bash
codex --auto-edit "Füge Unit-Tests für die UserService-Klasse hinzu"
```

- Datei-Edits werden automatisch angewandt
- Shell-Befehle und andere riskante Aktionen erfordern Bestätigung
- Guter Kompromiss zwischen Geschwindigkeit und Sicherheit

#### --full-auto (Nur mit Vorsicht verwenden!)

```bash
codex --full-auto "Führe alle Tests aus und fixe die fehlschlagenden"
```

> [!CAUTION]
> `--full-auto` gibt dem Agenten vollen Zugriff ohne jede Bestätigung. Nur in isolierten Umgebungen (Container, VMs) oder bei gut verstandenen, klar definierten Aufgaben verwenden. In Kombination mit `sandbox_mode = "danger-full-access"` kann dies zu unbeabsichtigten Systemänderungen führen.

### Sandbox Modi

Die Sandbox Modi kontrollieren den Dateisystem-Zugriff unabhängig vom Approval Modus:

```bash
# Read-only: Agent kann nur lesen
codex --suggest --sandbox read-only "Analysiere diesen Code"

# Workspace-write: Agent kann im Projektverzeichnis schreiben
codex --auto-edit --sandbox workspace-write "Erstelle eine README"

# Full Access: Agent hat vollen Zugriff (Vorsicht!)
codex --full-auto --sandbox danger-full-access "Installiere die Dependencies"
```

### Kombinationsmatrix: Approval × Sandbox

| Approval Modus | Sandbox | Sicherheit | Empfohlen für |
|---|---|---|---|
| `--suggest` | `read-only` | ⬛⬛⬛⬛⬛ Maximal | Code-Review, Analyse |
| `--suggest` | `workspace-write` | ⬛⬛⬛⬛ Hoch | Sensible Änderungen |
| `--auto-edit` | `workspace-write` | ⬛⬛⬛ Mittel | Tägliche Entwicklung |
| `--auto-edit` | `danger-full-access` | ⬛⬛ Niedrig | Erfahrene Nutzer |
| `--full-auto` | `danger-full-access` | ⬛ Minimal | Nur Container/VMs |

---

## Befehle und Nutzung

### Übersicht der CLI-Befehle

| Befehl | Beschreibung |
|---|---|
| `codex` | Interaktives TUI starten |
| `codex "prompt"` | Einmaliger Prompt mit interaktiver Ausgabe |
| `codex exec "task"` | Nicht-interaktive Ausführung (für Skripte/CI) |
| `codex resume` | Letzte Session fortsetzen |

### Interaktiver Modus (TUI)

```bash
# TUI starten
codex

# TUI mit spezifischem Modell
codex --model o4-mini

# TUI mit Approval Modus
codex --auto-edit
```

Das TUI bietet eine vollständige interaktive Oberfläche mit:
- Eingabefeld für Prompts
- Live-Ausgabe der Agenten-Aktionen
- Bestätigungs-Dialoge für Änderungen
- Diff-Anzeige für Dateiänderungen

### Einmaliger Prompt

```bash
# Direkte Aufgabe
codex "Erstelle eine .gitignore für ein Node.js Projekt"

# Mit Approval Modus
codex --auto-edit "Füge TypeScript-Konfiguration hinzu"

# Mit spezifischem Modell
codex --model gpt-5.3-codex "Optimiere die Datenbankabfragen"
```

### Nicht-interaktiver Modus (exec)

```bash
# Für CI/CD Pipelines
codex exec "Führe die Tests aus und erstelle einen Report"

# Für Skripte
codex exec "Formatiere alle Python-Dateien mit black"
```

> [!NOTE]
> Der `exec`-Modus ist ideal für Automatisierung und CI/CD-Pipelines, da er keine interaktive Eingabe erwartet und nach Abschluss der Aufgabe automatisch beendet wird.

### Session fortsetzen

```bash
# Letzte Session fortsetzen
codex resume

# Session-ID anzeigen und gezielt fortsetzen
codex resume --session <session-id>
```

> [!TIP]
> `codex resume` ist nützlich, wenn eine Session unerwartet beendet wurde oder du eine komplexe Aufgabe am nächsten Tag fortsetzen möchtest. Der gesamte Kontext wird wiederhergestellt.

---

## Multi-Agent Modus

### Aktivierung

Der Multi-Agent Modus ermöglicht parallele Aufgabenbearbeitung durch mehrere Sub-Agents:

```toml
# In ~/.codex/config.toml
[features]
multi_agent = true
```

### Funktionsweise

Im Multi-Agent Modus kann Codex CLI:
- Aufgaben in Teilaufgaben zerlegen
- Mehrere Sub-Agents parallel starten
- Ergebnisse zusammenführen und koordinieren

```bash
# Multi-Agent Aufgabe starten
codex --full-auto "Refactore alle API-Endpunkte auf das neue Schema und aktualisiere die Tests"
```

> [!NOTE]
> Der Multi-Agent Modus ist besonders effektiv bei großen Refactorings, wo mehrere unabhängige Dateien gleichzeitig bearbeitet werden können. Für Details zu Multi-Agent-Workflows allgemein siehe [[Multi-Agent Workflows]].

### Vergleich: Multi-Agent in verschiedenen Tools

| Aspekt | Codex CLI | Copilot CLI | Claude Code |
|---|---|---|---|
| Multi-Agent | `[features] multi_agent = true` | `/fleet` Command | Background-Tasks mit `task` Tool |
| Parallelität | Ja | Ja (Fleet-Modus) | Ja (Sub-Agents) |
| Konfiguration | `config.toml` | Interaktiv | Automatisch |

---

## MCP Server Integration

### Was ist MCP?

Das Model Context Protocol (MCP) ermöglicht es, externe Tool-Server an Codex CLI anzubinden. Damit kann Codex auf zusätzliche Datenquellen und Werkzeuge zugreifen.

> [!INFO]
> Für eine ausführliche Einführung in MCP siehe [[Was ist MCP]] und [[MCP Server in Claude Code einbinden]].

### MCP Server konfigurieren

Codex CLI unterstützt MCP-Server über eine JSON-Konfiguration:

````json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/projects"
      ],
      "env": {}
    }
  }
}
````

### Beispiel: GitHub MCP Server

````json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
````

### Beispiel: Datenbank MCP Server

````json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://localhost:5432/mydb"
      ],
      "env": {
        "PGPASSWORD": "geheim"
      }
    }
  }
}
````

> [!TIP]
> MCP-Server erweitern die Fähigkeiten von Codex CLI erheblich. Mit einem GitHub MCP Server kann Codex Issues und PRs lesen, mit einem Datenbank-Server direkt Queries ausführen.

---

## Vergleich: Codex CLI vs. Claude Code vs. Copilot CLI

### Allgemeiner Vergleich

| Aspekt | Codex CLI | Claude Code | Copilot CLI |
|---|---|---|---|
| Hersteller | OpenAI | Anthropic | GitHub (Microsoft) |
| Open Source | ✅ Ja | ❌ Nein | ❌ Nein |
| Lizenz | Apache 2.0 | Proprietär | Proprietär |
| Sprache | TypeScript/Rust | Rust | TypeScript |
| LLM-Modelle | OpenAI (GPT, o-Serie) | Claude-Modelle | Claude, GPT, Gemini u.a. |

### Authentifizierung

| Aspekt | Codex CLI | Claude Code | Copilot CLI |
|---|---|---|---|
| OAuth | ChatGPT Account (Browser) | Anthropic Console | GitHub Account |
| API Key | `OPENAI_API_KEY` | `ANTHROPIC_API_KEY` | `GH_TOKEN` / `GITHUB_TOKEN` |
| Kostenmodell | ChatGPT-Plan oder API | API oder Max-Plan | GitHub Copilot Lizenz |

### Konfiguration

| Aspekt | Codex CLI | Claude Code | Copilot CLI |
|---|---|---|---|
| Konfigurationsdatei | `~/.codex/config.toml` (TOML) | `~/.claude/settings.json` (JSON) | `~/.copilot/config.json` (JSON) |
| Instruktionsdatei | `AGENTS.md` | `CLAUDE.md` | `CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md` |
| Sandbox | `read-only` / `workspace-write` / `danger-full-access` | Container-basiert (Docker) | Verzeichnis-basiert |
| Approval Modi | `--suggest` / `--auto-edit` / `--full-auto` | Standard / Auto-accept | Interactive / Plan / Autopilot |

### Funktionen

| Funktion | Codex CLI | Claude Code | Copilot CLI |
|---|---|---|---|
| TUI (Terminal UI) | ✅ | ✅ | ✅ |
| MCP Support | ✅ | ✅ | ✅ |
| Multi-Agent | ✅ (config.toml) | ✅ (Sub-Agents) | ✅ (Fleet-Modus) |
| Web-Suche | ✅ (cached/live) | ❌ | ✅ (Deep Research) |
| Session Resume | ✅ (`codex resume`) | ✅ (`--resume`) | ❌ |
| Non-Interactive | ✅ (`codex exec`) | ✅ (`--print`) | ❌ |
| LSP Integration | ❌ | ❌ | ✅ |
| Hooks | ❌ | ✅ (Pre/Post Hooks) | ❌ |

### Wann welches Tool nutzen?

| Szenario | Empfehlung | Grund |
|---|---|---|
| OpenAI-Modelle bevorzugt | Codex CLI | Direkter Zugang zu GPT und o-Serie |
| Claude-Modelle bevorzugt | Claude Code | Optimiert für Claude, tiefste Integration |
| Multi-Modell-Flexibilität | Copilot CLI | Unterstützt Claude, GPT und Gemini |
| Open-Source wichtig | Codex CLI | Einziges OSS-Tool der drei |
| Enterprise / Compliance | Copilot CLI | GitHub Enterprise Integration |
| Bestehende AGENTS.md | Codex CLI / Copilot CLI | Beide lesen AGENTS.md |
| Bestehende CLAUDE.md | Claude Code / Copilot CLI | Copilot liest auch CLAUDE.md |

---

## Tipps und Troubleshooting

### Best Practices

> [!TIP]
> **1. AGENTS.md sorgfältig pflegen**
> Eine gut gepflegte `AGENTS.md` verbessert die Ergebnisse erheblich. Definiere Code-Konventionen, Testing-Strategien und Projektstruktur klar und präzise.

> [!TIP]
> **2. Sandbox-Modus passend wählen**
> Starte mit `workspace-write` für die tägliche Arbeit. Nutze `read-only` für Code-Reviews und Analysen. Reserviere `danger-full-access` für Container und VMs.

> [!TIP]
> **3. Approval Modus der Aufgabe anpassen**
> `--suggest` für sensible Produktionscode-Änderungen, `--auto-edit` für die meisten Entwicklungsaufgaben, `--full-auto` nur für klar definierte, risikoarme Tasks.

> [!TIP]
> **4. Reasoning Effort steuern**
> Setze `model_reasoning_effort = "high"` für komplexe Architektur-Entscheidungen und `"low"` für einfache Code-Generierung. Das spart Tokens und beschleunigt die Antworten.

> [!TIP]
> **5. Sessions fortsetzen statt neu starten**
> Nutze `codex resume`, um unterbrochene Sessions fortzusetzen. Der Kontext bleibt erhalten, und du sparst dir die erneute Erklärung der Aufgabe.

### Häufige Probleme und Lösungen

#### Authentifizierung schlägt fehl

```bash
# API Key prüfen
echo $OPENAI_API_KEY

# Key neu setzen
export OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxxxxxx"

# OAuth erneut durchführen
codex
# → Browser öffnet sich für Login
```

#### Node.js Version zu alt

```bash
# Version prüfen
node --version

# Falls < 18: Update über nvm
nvm install 22
nvm use 22

# Codex erneut installieren
npm install -g @openai/codex
```

#### Sandbox blockiert Dateioperationen

```bash
# Aktuellen Sandbox-Modus prüfen
# In config.toml nachschauen:
cat ~/.codex/config.toml | grep sandbox

# Sandbox-Modus anpassen (falls nötig)
# In ~/.codex/config.toml:
# sandbox_mode = "workspace-write"
```

> [!NOTE]
> Wenn der Agent Dateien nicht schreiben kann, ist meist der Sandbox-Modus auf `"read-only"` gesetzt. Ändere ihn in `"workspace-write"` für normale Entwicklungsarbeit.

#### MCP-Server startet nicht

```bash
# Prüfen ob der MCP-Server manuell funktioniert
npx -y @modelcontextprotocol/server-filesystem /tmp

# Node.js Version prüfen (>= 18 erforderlich)
node --version

# npx Cache leeren bei Problemen
npx --yes clear-npx-cache
```

#### Codex CLI reagiert nicht / hängt

```bash
# Prozess beenden und neu starten
# Terminal schließen oder Ctrl+C

# Cache leeren
rm -rf ~/.codex/cache

# Neu starten
codex
```

### Migration von Claude Code zu Codex CLI

Wenn du von Claude Code zu Codex CLI wechseln möchtest:

1. **CLAUDE.md → AGENTS.md umbenennen** – Codex CLI liest `AGENTS.md`, nicht `CLAUDE.md`. Die Syntax ist kompatibel, meist reicht ein Umbenennen
2. **MCP-Server übertragen** – MCP-Konfiguration aus Claude Code `settings.json` in das Codex CLI MCP-Format übernehmen
3. **Hooks anpassen** – Codex CLI hat kein direktes Hook-System; ggf. über CI/CD-Pipelines oder Shell-Skripte abbilden
4. **API Key wechseln** – Von `ANTHROPIC_API_KEY` zu `OPENAI_API_KEY` oder ChatGPT-OAuth nutzen
5. **Konfiguration migrieren** – Von JSON (`settings.json`) zu TOML (`config.toml`) Format

> [!IMPORTANT]
> Wer sowohl Claude Code als auch Copilot CLI nutzt, kann `AGENTS.md` als gemeinsame Instruktionsdatei verwenden. Copilot CLI liest sowohl `AGENTS.md` als auch `CLAUDE.md`, Claude Code nur `CLAUDE.md`.

---

## Weiterführende Links

- [[Custom Instructions für AI Coding Agents]] – Best Practices für AGENTS.md und CLAUDE.md
- [[GitHub Copilot CLI Konfiguration]] – Detaillierte Copilot CLI Konfiguration
- [[Claude Code Konfiguration und Hooks]] – Claude Code Konfiguration zum Vergleich
- [[Was ist MCP]] – Grundlagen des Model Context Protocol
- [[MCP Server in Claude Code einbinden]] – MCP-Server Konfiguration (auch für Codex relevant)
- [[Multi-Agent Workflows]] – Parallele Agent-Ausführung
- [[Was ist ein AI Agent]] – Grundlagen zu AI Agents
