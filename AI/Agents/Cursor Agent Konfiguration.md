# Cursor AI — Agent Konfiguration und Dokumentation

## Was ist Cursor?

Cursor ist eine **AI-powered IDE**, die als Fork von Visual Studio Code entwickelt wurde. Sie behält die volle Kompatibilität mit VS Code Extensions, Themes und Keybindings, erweitert den Editor aber um tiefgreifende KI-Funktionalität.

> [!NOTE]
> Cursor ist **kein Plugin**, sondern eine eigenständige Anwendung. Bestehende VS Code Settings und Extensions können beim ersten Start importiert werden.

Cursor bietet drei zentrale AI-Modi:

| Modus | Zweck | Aktivierung |
|---|---|---|
| **Tab** | Intelligente Autocomplete-Vorschläge | Automatisch beim Tippen |
| **Chat** | Fragen stellen, Code erklären lassen | `Ctrl+L` / `Cmd+L` |
| **Agent** (Composer) | Autonomes Arbeiten über mehrere Dateien | `Ctrl+I` / `Cmd+I` |

Der **Agent Mode** ist der mächtigste Modus — er kann eigenständig planen, Dateien erstellen und bearbeiten, Terminal-Befehle ausführen und iterativ arbeiten.

---

## Installation

### Download und Setup

1. **Download** von [cursor.com](https://cursor.com)
2. Verfügbar für **macOS**, **Windows** und **Linux**
3. Installer ausführen und Cursor starten
4. Beim ersten Start: VS Code Settings importieren (optional)

> [!TIP]
> Cursor kann parallel zu VS Code installiert werden. Beide teilen sich Extensions aus dem gleichen Marketplace.

### Systemvoraussetzungen

- Modernes Betriebssystem (macOS 12+, Windows 10+, Ubuntu 20.04+)
- Mindestens 8 GB RAM empfohlen
- Internetverbindung für AI-Features erforderlich

### VS Code Extensions

Da Cursor ein VS Code Fork ist, sind **alle Extensions kompatibel**. Installation wie gewohnt über den Extensions-Bereich (`Ctrl+Shift+X`).

---

## Die drei Modi

### Tab — Intelligente Autocomplete

Tab ist die Autocomplete-Funktion von Cursor. Sie geht über klassisches IntelliSense hinaus und nutzt ein eigenes Modell (`cursor-small`), um kontextbewusste Vorschläge zu machen.

- Schlägt ganze Codeblöcke vor, nicht nur einzelne Tokens
- Berücksichtigt den umliegenden Code und geöffnete Dateien
- Akzeptieren mit `Tab`, Ablehnen mit `Esc`
- Teilweise akzeptieren mit `Ctrl+→` (Wort für Wort)

### Chat — Fragen und Erklärungen

Der Chat-Modus (`Ctrl+L` / `Cmd+L`) erlaubt direkte Konversation mit dem AI-Modell:

- Code erklären lassen (Code markieren → `Ctrl+L`)
- Fragen zur Codebase stellen
- Code-Vorschläge erhalten (müssen manuell mit "Apply" übernommen werden)
- Unterstützt `@`-Mentions für Kontext

> [!IMPORTANT]
> Im Chat-Modus werden Änderungen **nicht automatisch** angewendet. Man muss explizit auf "Apply" klicken, um vorgeschlagene Änderungen zu übernehmen.

### Agent (Composer) — Autonomes Arbeiten

Der Agent-Modus ist Cursors mächtigstes Feature. Er wird über `Ctrl+I` / `Cmd+I` aktiviert und arbeitet **autonom**:

- Plant die nötigen Schritte
- Erstellt und bearbeitet mehrere Dateien gleichzeitig
- Führt Terminal-Befehle aus (mit Bestätigung)
- Liest Fehlermeldungen und korrigiert sich selbst
- Läuft iterativ bis die Aufgabe erledigt ist

---

## Agent Mode im Detail

### Funktionsweise

Der Agent Mode arbeitet in einem Loop:

```
Aufgabe analysieren → Plan erstellen → Dateien bearbeiten → Terminal nutzen
         ↑                                                        |
         └────────── Ergebnis prüfen ← Fehler korrigieren ←──────┘
```

### Capabilities

| Fähigkeit | Beschreibung |
|---|---|
| **Multi-File Editing** | Bearbeitet beliebig viele Dateien in einem Durchlauf |
| **File Creation** | Erstellt neue Dateien und Verzeichnisse |
| **Terminal Execution** | Führt Shell-Befehle aus (Build, Test, Install) |
| **Error Recovery** | Liest Compiler/Test-Fehler und korrigiert automatisch |
| **Codebase Search** | Durchsucht die gesamte Codebase nach relevantem Kontext |
| **Web Search** | Kann aktuelle Informationen aus dem Web abrufen |

### Workflow-Beispiel

````markdown
Prompt an den Agent:

"Erstelle eine REST API mit Express.js für User-Management.
Nutze TypeScript, Zod für Validation und schreibe Tests mit Vitest."

Der Agent wird dann:
1. `package.json` erstellen und Dependencies installieren
2. TypeScript-Konfiguration anlegen
3. Express Server mit Routen erstellen
4. Zod Schemas definieren
5. Tests schreiben
6. `npm test` ausführen und Fehler beheben
````

> [!WARNING]
> Der Agent führt Terminal-Befehle aus. Standardmäßig wird vor jeder Ausführung eine Bestätigung verlangt. Diese kann in den Settings deaktiviert werden — **nur in vertrauenswürdigen Projekten empfohlen**.

### Yolo Mode

Im sogenannten "Yolo Mode" führt der Agent Terminal-Befehle **ohne Bestätigung** aus. Aktivierung über:

`Settings → Features → Agent → Auto-run mode`

Konfigurierbare Optionen:
- **Off**: Jeder Befehl muss bestätigt werden (Standard)
- **Safe commands only**: Nur sichere Befehle (z.B. `ls`, `cat`) automatisch
- **All commands**: Alle Befehle automatisch ausführen

> [!WARNING]
> Yolo Mode mit "All commands" sollte nur in isolierten Umgebungen (Container, VMs) oder bei vertrauenswürdigen Projekten aktiviert werden.

---

## Custom Rules (.cursor/rules/)

Custom Rules steuern das Verhalten des AI-Agents. Sie definieren Coding-Standards, Projekt-Konventionen und kontextspezifische Anweisungen.

### Legacy: `.cursorrules`

> [!WARNING]
> Die `.cursorrules`-Datei im Projekt-Root ist **deprecated**. Sie funktioniert noch, wird aber von Cursor nicht mehr empfohlen. Stattdessen sollte das neue `.cursor/rules/` System verwendet werden.

### Aktuell: `.cursor/rules/*.mdc`

Das aktuelle System verwendet **MDC-Dateien** (Markdown mit YAML Frontmatter) im Verzeichnis `.cursor/rules/`:

```
projekt/
├── .cursor/
│   └── rules/
│       ├── general.mdc
│       ├── typescript.mdc
│       ├── testing.mdc
│       └── api-design.mdc
├── src/
└── package.json
```

### MDC Format

Jede `.mdc`-Datei besteht aus einem YAML Frontmatter und dem eigentlichen Regeltext:

````markdown
---
description: TypeScript Coding Standards für das Projekt
globs: "**/*.ts,**/*.tsx"
alwaysApply: false
---

# TypeScript Standards

- Verwende `interface` statt `type` für Objekttypen
- Nutze `strict: true` in der tsconfig
- Exportiere Typen explizit mit `export type`
- Verwende `const` als Default, `let` nur wenn nötig
- Keine `any` Types — nutze `unknown` als Fallback

## Error Handling

```typescript
// ✅ Richtig
try {
  await fetchData();
} catch (error) {
  if (error instanceof ApiError) {
    handleApiError(error);
  }
  throw error;
}

// ❌ Falsch
try {
  await fetchData();
} catch (e: any) {
  console.log(e);
}
```
````

### Frontmatter-Felder

| Feld | Typ | Beschreibung |
|---|---|---|
| `description` | `string` | Beschreibung der Regel (wird dem Agent als Kontext angezeigt) |
| `globs` | `string` | File-Pattern für automatische Aktivierung (z.B. `"**/*.py"`) |
| `alwaysApply` | `boolean` | Regel immer aktiv, unabhängig vom Dateikontext |

### Aktivierungslogik

- **`alwaysApply: true`**: Regel wird bei jeder AI-Interaktion geladen
- **`alwaysApply: false` + `globs` gesetzt**: Regel wird automatisch geladen, wenn eine passende Datei im Kontext ist
- **`alwaysApply: false` + kein `globs`**: Regel wird nur geladen, wenn sie über `@rules` explizit referenziert wird

### Rules generieren

Cursor kann Rules automatisch aus dem Chat generieren:

1. Chat öffnen (`Ctrl+L`)
2. Den Befehl `/Generate Cursor Rules` eingeben
3. Beschreiben, welche Regeln generiert werden sollen
4. Cursor erstellt eine passende `.mdc`-Datei

### Prioritätsreihenfolge

Wenn mehrere Rules aktiv sind, gilt folgende Priorität (höchste zuerst):

1. **Team Rules** — Über Cursor for Teams konfiguriert
2. **Project Rules** — `.cursor/rules/*.mdc` Dateien
3. **User Rules** — Globale Rules in den Cursor Settings
4. **Legacy Rules** — `.cursorrules` im Projekt-Root

> [!TIP]
> Project Rules (`.cursor/rules/`) sollten ins Git-Repository committed werden. So teilt das gesamte Team die gleichen AI-Anweisungen.

---

## MCP Server Konfiguration

### Was ist MCP?

Das **Model Context Protocol (MCP)** ermöglicht es, externe Tools und Datenquellen an den AI-Agent anzubinden. Cursor unterstützt MCP nativ.

### Konfiguration über UI

`Settings → Tools & MCP → Add MCP Server`

Hier können MCP Server hinzugefügt, aktiviert und deaktiviert werden.

### Config-Datei: `.cursor/mcp.json`

MCP Server können auch über eine JSON-Datei konfiguriert werden:

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
        "GITHUB_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/projects"
      ]
    },
    "remote-server": {
      "url": "https://mcp.example.com/sse"
    }
  }
}
````

### Transport-Protokolle

| Transport | Typ | Verwendung |
|---|---|---|
| **stdio** | Lokal | Server läuft als lokaler Prozess (Standard) |
| **HTTP+SSE** | Remote | Server über HTTP mit Server-Sent Events |
| **Streamable HTTP** | Remote | Neuere HTTP-basierte Variante |

> [!NOTE]
> Lokale MCP Server (`stdio`) werden von Cursor automatisch gestartet und gestoppt. Remote Server (`url`) müssen eigenständig laufen.

### MCP in der Praxis

Beliebte MCP Server für Cursor:

| Server | Funktion |
|---|---|
| `server-github` | GitHub Issues, PRs, Repos durchsuchen |
| `server-filesystem` | Dateisystem-Zugriff für den Agent |
| `server-postgres` | PostgreSQL-Datenbank abfragen |
| `server-brave-search` | Web-Suche über Brave |
| `server-memory` | Persistenter Speicher für den Agent |

---

## Modelle und Modellwahl

### Unterstützte Modelle

Cursor unterstützt verschiedene AI-Modelle, die je nach Aufgabe gewählt werden können:

| Modell | Anbieter | Stärke |
|---|---|---|
| **Claude Sonnet 4** | Anthropic | Gute Balance aus Speed und Qualität |
| **Claude Opus 4** | Anthropic | Höchste Qualität, langsamer |
| **GPT-4o** | OpenAI | Schnell, gut für einfache Aufgaben |
| **GPT-5** | OpenAI | Neuestes OpenAI-Modell |
| **Gemini 2.5 Pro** | Google | Großes Context Window |
| **cursor-small** | Cursor | Optimiert für Tab/Autocomplete |

### Modellwahl

- **Tab**: Nutzt automatisch `cursor-small` (nicht änderbar)
- **Chat**: Modell über das Dropdown oben im Chat-Panel auswählbar
- **Agent**: Modell über das Dropdown im Composer auswählbar

> [!TIP]
> Für komplexe Agent-Aufgaben (Architektur, Refactoring) eignet sich **Claude Opus 4** oder **Claude Sonnet 4**. Für schnelle Fragen im Chat reicht **GPT-4o** oder **Claude Sonnet**.

### Eigene API Keys

Unter `Settings → Models` können eigene API Keys hinterlegt werden:

- OpenAI API Key
- Anthropic API Key
- Google AI API Key
- Azure OpenAI Endpoint

Mit eigenen Keys entfallen die Cursor-internen Rate Limits und es wird direkt über die jeweilige API abgerechnet.

---

## Context-Features

### @-Mentions

Mit `@`-Mentions kann dem AI-Agent gezielt Kontext mitgegeben werden:

| Mention | Funktion | Beispiel |
|---|---|---|
| `@file` | Einzelne Datei referenzieren | `@src/auth.ts` |
| `@folder` | Ganzen Ordner referenzieren | `@src/components/` |
| `@codebase` | Gesamte Codebase durchsuchen | `@codebase wie funktioniert Auth?` |
| `@web` | Web-Suche durchführen | `@web React 19 neue Features` |
| `@docs` | Dokumentation crawlen | `@docs next.js app router` |
| `@git` | Git-History referenzieren | `@git letzte 5 Commits` |

### Codebase Indexing

Cursor indiziert die gesamte Codebase für **semantische Suche**:

- Aktivierung: `Settings → Features → Codebase Indexing`
- Erstellt Embeddings aller Dateien im Projekt
- Ermöglicht natürlichsprachliche Suche über `@codebase`
- Wird automatisch bei Dateiänderungen aktualisiert

> [!IMPORTANT]
> Die Codebase-Indexierung erfolgt lokal. Der Index wird bei großen Projekten beim ersten Mal einige Minuten benötigen.

### Docs Crawling

Mit `@docs` kann Cursor externe Dokumentation crawlen und als Kontext nutzen:

1. `@docs` im Chat eingeben
2. URL der Dokumentation angeben (z.B. `https://docs.python.org`)
3. Cursor crawlt und indiziert die Seiten
4. Zukünftige `@docs`-Anfragen nutzen den gecachten Index

> [!TIP]
> Häufig genutzte Dokumentationen können unter `Settings → Features → Docs` vorregistriert werden, um sie dauerhaft verfügbar zu machen.

### Image Input

Der Agent und Chat unterstützen **Bild-Input**:

- Screenshots einfügen (Copy & Paste)
- Design-Mockups als Referenz
- Fehlermeldungen als Screenshot
- Unterstützt PNG, JPG, WebP

---

## Keyboard Shortcuts

### Allgemeine Shortcuts

| Shortcut | Aktion |
|---|---|
| `Ctrl+L` / `Cmd+L` | Chat öffnen |
| `Ctrl+I` / `Cmd+I` | Agent (Composer) öffnen |
| `Ctrl+K` / `Cmd+K` | Inline Edit (Code an Cursor-Position bearbeiten) |
| `Ctrl+Shift+L` | Markierten Code zum Chat hinzufügen |
| `Tab` | Autocomplete-Vorschlag akzeptieren |
| `Esc` | Autocomplete-Vorschlag ablehnen |
| `Ctrl+→` | Vorschlag teilweise akzeptieren (Wort) |

### Agent-spezifische Shortcuts

| Shortcut | Aktion |
|---|---|
| `Ctrl+Enter` | Befehl im Agent bestätigen / absenden |
| `Ctrl+Backspace` | Letzte Agent-Aktion rückgängig machen |
| `Escape` | Agent stoppen |

### Navigation

| Shortcut | Aktion |
|---|---|
| `Ctrl+Shift+E` | Explorer öffnen |
| `Ctrl+Shift+F` | Globale Suche |
| `Ctrl+P` | Datei schnell öffnen |
| `Ctrl+Shift+P` | Command Palette |
| `Ctrl+`` ` | Terminal öffnen/schließen |

---

## Vergleich: Cursor vs. Terminal-basierte Agents

### Übersicht

| Eigenschaft | Cursor (Agent Mode) | Terminal Agents (Claude Code, Copilot CLI) |
|---|---|---|
| **Interface** | GUI (IDE) | Terminal / CLI |
| **Editor** | Integriert (VS Code Fork) | Externer Editor nötig |
| **File Editing** | Visuelles Diff + Accept/Reject | Direkte Dateimanipulation |
| **Terminal** | Integriertes Terminal | Native Shell |
| **Multi-File** | ✅ Visuell nachvollziehbar | ✅ Über Tool Calls |
| **Codebase Indexing** | ✅ Eingebaut | ⚠️ Teilweise (tool-abhängig) |
| **MCP Support** | ✅ Nativ | ✅ Nativ |
| **Custom Rules** | `.cursor/rules/*.mdc` | `CLAUDE.md`, `.github/copilot-instructions.md` |
| **CI/CD Integration** | ❌ Manuell | ✅ Headless möglich |
| **Offline** | ❌ (API-basiert) | ❌ (API-basiert) |
| **Kosten** | Cursor Abo ($20/Monat Pro) | API-Kosten / Tool-Abo |

### Wann Cursor?

- **Visuelles Arbeiten**: Wenn man Änderungen sehen und Review-en möchte
- **Frontend-Entwicklung**: Design → Code mit Bild-Input
- **Exploration**: Codebase durchsuchen und verstehen
- **Onboarding**: Neues Projekt kennenlernen mit `@codebase`

### Wann Terminal Agents?

- **Automation**: CI/CD Pipelines, Batch-Operationen
- **Server-Umgebungen**: Headless / SSH Sessions
- **Komplexe Multi-Step Tasks**: Lange autonome Workflows
- **Scripting**: Einbindung in bestehende Shell-Workflows

> [!NOTE]
> Cursor und Terminal Agents schließen sich nicht aus. Viele Entwickler nutzen Cursor für die tägliche Entwicklung und Terminal Agents für Automation und CI/CD-Aufgaben.

---

## Modi im Vergleich

| Feature | Tab | Chat | Agent |
|---|---|---|---|
| Autocomplete | ✅ | ❌ | ❌ |
| Fragen beantworten | ❌ | ✅ | ✅ |
| Code erklären | ❌ | ✅ | ✅ |
| Dateien editieren | ❌ | ⚠️ Apply nötig | ✅ Automatisch |
| Dateien erstellen | ❌ | ❌ | ✅ |
| Terminal nutzen | ❌ | ❌ | ✅ |
| Multi-File Editing | ❌ | ❌ | ✅ |
| Web-Suche | ❌ | ✅ | ✅ |
| Bild-Input | ❌ | ✅ | ✅ |
| MCP Tools | ❌ | ❌ | ✅ |
| Codebase-Suche | ❌ | ✅ | ✅ |

---

## Tipps und Best Practices

### Effektive Prompts

> [!TIP]
> **Sei spezifisch.** Statt "Mach die API besser" lieber:
> "Refaktoriere die User API in `src/api/users.ts` — extrahiere die Validation in Zod Schemas und füge Error Handling mit Custom Error Classes hinzu."

### Agent Mode optimal nutzen

1. **Kontext mitgeben**: Relevante Dateien mit `@file` referenzieren
2. **Schrittweise arbeiten**: Lieber mehrere kleine Aufgaben als eine riesige
3. **Ergebnisse prüfen**: Agent-Änderungen immer reviewen bevor sie committed werden
4. **Rules pflegen**: Projekt-spezifische Rules halten die Agent-Qualität konsistent

### Custom Rules Best Practices

- **Eine Rule pro Thema**: `typescript.mdc`, `testing.mdc`, `api-design.mdc`
- **Globs nutzen**: Rules nur für relevante Dateien aktivieren
- **Beispiele geben**: Code-Beispiele in Rules sind effektiver als abstrakte Anweisungen
- **Im Git committen**: Rules gehören ins Repository für Team-Konsistenz

### Performance

- **Codebase Indexing** für große Projekte aktivieren
- **`.cursorignore`** nutzen um irrelevante Dateien auszuschließen (ähnlich `.gitignore`)
- **Kontext begrenzen**: Nur relevante Dateien referenzieren statt `@codebase` für alles

### Sicherheit

> [!WARNING]
> - Keine Secrets oder API Keys in Prompts oder Rules einfügen
> - Yolo Mode nur in isolierten Umgebungen aktivieren
> - Agent-generierte Terminal-Befehle immer prüfen (besonders `rm`, `sudo`, etc.)
> - MCP Server Berechtigungen minimal halten

### `.cursorignore` Beispiel

```
# Build Artifacts
node_modules/
dist/
build/
.next/

# Secrets
.env
.env.local
*.pem
*.key

# Große Dateien
*.sql
*.csv
data/
```

---

## Weiterführende Ressourcen

| Ressource | Link |
|---|---|
| Cursor Dokumentation | [docs.cursor.com](https://docs.cursor.com) |
| Cursor Forum | [forum.cursor.com](https://forum.cursor.com) |
| Cursor Changelog | [cursor.com/changelog](https://cursor.com/changelog) |
| MCP Spezifikation | [modelcontextprotocol.io](https://modelcontextprotocol.io) |
| MCP Server Registry | [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) |

---

> [!NOTE]
> Diese Dokumentation bezieht sich auf den Stand von Cursor im Jahr 2025. Da Cursor aktiv weiterentwickelt wird, können sich Features und Konfigurationsoptionen ändern. Die offizielle Dokumentation unter [docs.cursor.com](https://docs.cursor.com) ist immer die aktuellste Quelle.
