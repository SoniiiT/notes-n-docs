---
tags: [ai, agents, ide, vscode, copilot, konfiguration]
aliases: [VSCode Copilot, VS Code AI, GitHub Copilot IDE]
---

# VS Code als AI-IDE mit GitHub Copilot

> [!ABSTRACT]
> VS Code ist seit 2024 eine vollwertige AI-IDE mit GitHub Copilot als eingebautem AI-System. Ab **v1.110** (Februar 2026) ist VS Code eine echte Multi-Agent-Plattform: Agent Plugins, Browser-Steuerung durch Agents, parallele Subagents, GPT-5.4, Session Memory und `/fork`-Chat-Sessions. Kein Fork, kein Drittanbieter nötig.

---

## Was ist VS Code als AI-IDE?

Visual Studio Code hat sich seit 2024 von einem reinen Code-Editor zu einer vollwertigen AI-integrierten Entwicklungsumgebung gewandelt. GitHub Copilot ist nicht mehr nur ein Autocomplete-Plugin, sondern ein eingebettetes AI-System mit eigenem Chat-Panel, Agenten-Modus und tiefer Editor-Integration.

Mit **VS Code v1.99** (April 2025) ist das erste Release erschienen, das vollständig auf Copilot Chat ausgerichtet ist. Seitdem hat sich die Plattform rasant weiterentwickelt — **v1.109** (Januar 2026) brachte Multi-Agent-Workflows, Agent Skills und Terminal Sandboxing; **v1.110** (Februar 2026) führte Agent Plugins, nativen Browser-Zugriff für Agents, forkbare Chat-Sessions und GPT-5.4 ein.

> [!NOTE] Abgrenzung zu anderen AI-IDEs
> Im Gegensatz zu [[Cursor Agent Konfiguration]] (VS-Code-Fork) oder [[Cline Konfiguration]] (VS-Code-Extension) ist GitHub Copilot **direkt in VS Code integriert** — kein Fork, keine Drittanbieter-Extension nötig. Dafür ist Copilot an das GitHub-Ökosystem gebunden.

---

## Installation & Setup

### Voraussetzungen

- VS Code **v1.110** oder neuer (Download: [code.visualstudio.com](https://code.visualstudio.com))
- GitHub-Konto (kostenlos für Free Tier)
- GitHub Copilot aktiviert (Free, Pro oder Business)

### Einrichtung

1. **Extensions installieren:** `GitHub Copilot` + `GitHub Copilot Chat` im Marketplace (`Ctrl+Shift+X`)
2. **Einloggen:** `Ctrl+Shift+P` → *GitHub Copilot: Sign In*
3. **Agent Mode aktivieren** in `settings.json`:

```json
{
  "chat.agent.enabled": true,
  "github.copilot.chat.agent.runTasks": true,
  "github.copilot.chat.agent.autoFix": true,
  "github.copilot.nextEditSuggestions.enabled": true,
  "github.copilot.editor.enableCodeActions": true,
  "github.copilot.chat.localeOverride": "de"
}
```

> [!TIP] settings.json öffnen
> `Ctrl+Shift+P` → *Preferences: Open User Settings (JSON)*

---

## Die drei Chat-Modi

VS Code Copilot bietet drei klar getrennte Arbeitsmodi, die per Dropdown im Chat-Panel gewählt werden. Alle Modi öffnen das Chat-Panel mit `Ctrl+Alt+I`.

| Modus | Verwendung | Dateiänderungen |
|-------|-----------|-----------------|
| **Ask** | Fragen stellen, Code erklären, Recherche | Keine |
| **Edit** | Gezielte Änderungen an ausgewählten Dateien | Manuell akzeptieren |
| **Agent** | Autonomes Arbeiten, Multi-File, Terminal | Automatisch, mit Bestätigung |

### Ask Mode

Standard-Modus für Konversation und Erklärungen. Copilot antwortet, macht aber keine direkten Dateiänderungen. Ideal für:
- Code-Reviews und Erklärungen (`/explain`)
- Architekturentscheidungen diskutieren
- Fehlerursachen analysieren

### Edit Mode

Copilot nimmt gezielte Änderungen an explizit angegebenen Dateien vor. Alle Änderungen werden als Diff angezeigt und können selektiv akzeptiert oder abgelehnt werden.

### Agent Mode

Der mächtigste Modus — Copilot arbeitet autonom:

- **Multi-File-Edits:** Ändert beliebig viele Dateien in einem Schritt
- **Terminal-Ausführung:** Führt Build-, Test- und Install-Befehle aus
- **Self-Healing:** Erkennt Fehler im Terminal-Output und korrigiert sich selbst
- **Parallele Sessions:** Mehrere Agent-Sessions gleichzeitig möglich
- **Planungsphase:** Copilot erstellt vor der Ausführung einen Plan, der überprüft werden kann
- **Subagents (ab v1.109):** Tasks an spezialisierte Subagents delegieren — parallel ausführbar
- **Claude Agent Support (ab v1.109):** Aufgaben direkt an Anthropic Claude Agents übergeben

```text
Beispiel-Prompt für Agent Mode:
"Refaktoriere alle API-Calls in src/services/ auf async/await,
 ersetze axios durch fetch, und passe die zugehörigen Tests an."
```

> [!WARNING] Agent Mode überwachen
> Im Agent Mode führt Copilot Terminal-Befehle aus. VS Code fragt vor potenziell destruktiven Befehlen nach Bestätigung — trotzdem bei produktivem Code genau prüfen, was ausgeführt wird.

---

## Inline Chat

Inline Chat ermöglicht AI-Assistenz direkt im Editor, ohne das Chat-Panel zu öffnen:

| Aktion | Shortcut |
|--------|---------|
| Inline Chat öffnen | `Ctrl+I` |
| Code markieren + erklären/ändern | Auswahl treffen → `Ctrl+I` |
| Vorschlag akzeptieren | `Tab` |
| Vorschlag ablehnen | `Esc` |
| Nächste Variante | `Alt+]` |

> [!TIP] Inline Chat für fokussierte Refactorings
> Code-Block markieren → `Ctrl+I` → Anweisung tippen. Copilot bearbeitet nur den markierten Bereich — schneller und präziser als über das Chat-Panel.

---

## Next Edit Suggestions (NES)

Next Edit Suggestions ist seit **Februar 2025 GA** — ein qualitativ neues Feature: Copilot analysiert laufend den Kontext und schlägt proaktiv vor, **was als nächstes geändert werden sollte**, ohne dass der Nutzer explizit fragt.

**Funktionsweise:**
- Beobachtet Tipp-Muster und erkennt zusammenhängende Änderungen
- Ändert man z.B. einen Variablennamen, schlägt NES vor, alle anderen Vorkommen anzupassen
- Vorschläge erscheinen als subtiler Pfeil (`→`) im Editor
- `Tab` → Vorschlag annehmen, `Esc` → ablehnen

**Aktivierung:**
```json
{
  "github.copilot.nextEditSuggestions.enabled": true
}
```

> [!NOTE] NES vs. klassische Autocomplete
> Klassische Completions ergänzen den Text am aktuellen Cursor. NES springt an andere Stellen im Dokument und schlägt dort Änderungen vor — ein grundlegend anderes Paradigma, das Refactoring-Aufwand deutlich reduziert.

---

## Modellauswahl

GitHub Copilot unterstützt mehrere LLM-Backends, die per Dropdown im Chat-Panel gewählt werden können:

| Modell | Stärken | Empfohlen für |
|--------|---------|---------------|
| **GPT-4o** | Schnell, ausgewogen | Alltägliche Coding-Aufgaben |
| **GPT-5.4** | Agentic Coding, Multi-Step | Komplexe Agent-Workflows, CI-Integration |
| **o3** | Starkes Reasoning | Komplexe Algorithmen, Debugging |
| **o4-mini** | Schnell + günstig | Einfache Aufgaben, hoher Durchsatz |
| **Claude 3.7 Sonnet** | Instruktionsfolgen, langer Kontext | Große Codebases, Refactoring |
| **Claude Opus** | Höchste Qualität | Architekturentscheidungen |
| **Gemini 2.5 Pro** | Langer Kontext, multimodal | Dokumentation, Code-Review |

> [!TIP] Modell je nach Aufgabe wählen
> Schnelle Completions → GPT-4o. Komplexe Agent-Tasks → Claude Sonnet oder o3. Die Wahl beeinflusst nicht das monatliche Chat-Kontingent bei Pro-Plänen.

---

## Custom Instructions

Custom Instructions erlauben **projektweite Verhaltensanweisungen** für Copilot — GA seit Februar 2025.

### Datei anlegen

Erstelle `.github/copilot-instructions.md` im Projektstamm:

```markdown
# Copilot Instructions

Antworte immer auf Deutsch.
Verwende TypeScript mit striktem Typing (`strict: true`).
Füge JSDoc-Kommentare zu allen öffentlichen Funktionen hinzu.
Schreibe Tests mit Vitest, nicht mit Jest.
Halte Funktionen unter 30 Zeilen.
Bevorzuge `const` über `let`, vermeide `var`.
```

> [!IMPORTANT] Datei-Pfad ist fest
> Die Datei muss exakt unter `.github/copilot-instructions.md` liegen. Copilot lädt sie automatisch als System-Kontext für alle Chat-Anfragen im Workspace — kein manuelles Einbinden nötig.

> [!WARNING] Keine Secrets in Custom Instructions
> `.github/copilot-instructions.md` wird von Copilot als Kontext gesendet. Niemals API-Keys, Tokens oder Passwörter dort hinterlegen — auch nicht als Beispiel.

Mehr dazu: [[Custom Instructions für AI Coding Agents]]

---

## MCP-Integration

VS Code unterstützt das **Model Context Protocol (MCP)**, wodurch Copilot auf externe Tools und Datenquellen zugreifen kann. MCP-Tools stehen primär im Agent Mode zur Verfügung.

### Konfiguration

Erstelle `.mcp.json` im Projektordner:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${env:DATABASE_URL}"]
    }
  }
}
```

> [!TIP] MCP im Agent Mode
> Mit MCP-Servern kann Copilot im Agent Mode direkt Datenbankabfragen stellen, GitHub-Issues lesen oder Dateisystemoperationen ausführen — ohne Copy-Paste-Workflow.

Weiterführend: [[Was ist MCP]] · [[MCP Grundlagen]] · [[MCP Server in Claude Code einbinden]]

---

## Slash Commands & Kontext

### Slash Commands im Chat

| Befehl | Funktion |
|--------|---------|
| `/explain` | Markierten Code erklären |
| `/fix` | Fehler im markierten Code beheben |
| `/tests` | Unit-Tests für den markierten Code generieren |
| `/doc` | JSDoc/Docstrings für den Code schreiben |
| `/new` | Neues Projekt oder neue Datei erstellen |
| `/clear` | Chat-Verlauf zurücksetzen |

### Kontextvariablen (`#`)

Mit `#`-Variablen wird explizit Kontext in den Chat gezogen:

| Variable | Beschreibung |
|----------|-------------|
| `#file:src/app.ts` | Bestimmte Datei als Kontext |
| `#selection` | Aktuell markierter Code-Block |
| `#codebase` | Gesamter Workspace (semantische Suche) |
| `#terminalLastCommand` | Output des zuletzt ausgeführten Terminal-Befehls |
| `#problems` | Aktuelle Fehler und Warnings aus dem Problems-Panel |

> [!TIP] `#terminalLastCommand` bei Fehlern nutzen
> Build oder Test schlägt fehl? Nicht manuell kopieren — einfach `#terminalLastCommand fix` im Chat tippen. Copilot liest den Fehler direkt aus dem Terminal und schlägt eine Korrektur vor.

> [!TIP] `#codebase` im Agent Mode
> Mit `#codebase` durchsucht Copilot den gesamten Workspace semantisch — ideal für Fragen wie *"Wo wird diese Funktion aufgerufen?"* oder *"Gibt es ähnliche Implementierungen?"*

---

## Settings-Referenz

Vollständige empfohlene `settings.json`-Konfiguration:

```json
{
  // Agent Mode
  "chat.agent.enabled": true,
  "github.copilot.chat.agent.runTasks": true,
  "github.copilot.chat.agent.autoFix": true,

  // Next Edit Suggestions
  "github.copilot.nextEditSuggestions.enabled": true,

  // Code Actions (Glühbirnen-Menü mit Copilot-Optionen)
  "github.copilot.editor.enableCodeActions": true,

  // Chat-Sprache auf Deutsch
  "github.copilot.chat.localeOverride": "de",

  // Inline Suggestions
  "editor.inlineSuggest.enabled": true,
  "editor.inlineSuggest.showToolbar": "onHover",

  // Copilot per Sprache steuern (optional)
  "github.copilot.enable": {
    "*": true,
    "plaintext": false,
    "markdown": true
  }
}
```

---

## Neuigkeiten 2026

### v1.109 — Januar 2026

#### Multi-Agent & Subagents

Ab v1.109 kann Copilot Aufgaben an **spezialisierte Subagents** delegieren, die **parallel** statt sequentiell laufen. Das beschleunigt komplexe Workflows erheblich:

```text
Beispiel: Code-Review einer großen Codebase
├── Subagent A: Analysiert src/api/ auf Sicherheitsprobleme
├── Subagent B: Prüft src/database/ auf Performance-Issues
└── Subagent C: Checkt Tests auf Coverage-Lücken
→ Alle drei laufen gleichzeitig, Ergebnisse werden zusammengeführt
```

**Claude Agent Support** (Public Preview): Aufgaben können direkt an Anthropic-Claude-Agents übergeben werden. Verschiedene Modelle (GPT, Claude, Gemini) lassen sich so in einem Workflow kombinieren.

#### Agent Skills

**Agent Skills** erlauben es, wiederverwendbare Fähigkeiten für Copilot zu definieren — als Verzeichnis mit einer `SKILL.md`-Datei:

```
.github/
  copilot-skills/
    docker-expert/
      SKILL.md          ← Beschreibung + Anweisungen
      templates/        ← Ressourcen für diesen Skill
      scripts/          ← Hilfsskripte
```

```markdown
# SKILL.md — Docker Expert
Dieser Skill hilft beim Erstellen optimierter Dockerfiles.
Verwende immer Multi-Stage-Builds und non-root Users.
Prüfe auf bekannte Sicherheitslücken in Base-Images.
```

> [!TIP] Agent Skills für Teams
> Skills können im Repo eingecheckt werden und gelten für das gesamte Team. Ideal für domänenspezifische Workflows, die immer wiederkehren (z.B. "API-Design-Review", "Migration-Helper", "Security-Checker").

#### Terminal Sandboxing & Auto-Approval Rules

- **Terminal Command Sandboxing** (experimentell, macOS/Linux): Agent-vorgeschlagene Terminal-Befehle laufen in einer abgeschirmten Umgebung
- **Auto-Approval Rules:** Vertrauenswürdige Aktionen (z.B. `npm install`, `git status`) können automatisch freigegeben werden — ohne ständige Bestätigung

```json
// settings.json — Auto-Approval für read-only Befehle
{
  "github.copilot.chat.agent.autoApprovePatterns": [
    "git status",
    "git log",
    "npm test",
    "cat *"
  ]
}
```

#### GPT-5.4 & Copilot Memory

- **GPT-5.4** jetzt verfügbar — optimiert für agentic Coding, Multi-Step-Tasks und Tool-Integration
- **Copilot Memory:** Agents behalten relevanten Projekt-Kontext über Sessions hinweg, verbessert Reasoning bei wiederkehrenden Aufgaben

---

### v1.110 — Februar 2026

#### Agent Plugins

**Agent Plugins** sind installierbare Pakete (Bündel aus Skills, Tools, Hooks), die direkt über den Extensions-Bereich installiert werden können — ähnlich wie Browser-Extensions für AI:

```text
Quellen für Agent Plugins:
- VS Code Marketplace (öffentlich)
- Private GitHub Repositories
- Direkte Git-URLs
- Lokale Dateipfade
```

> [!NOTE] Agent Plugins vs. Agent Skills
> **Skills** (v1.109) sind projektspezifische Fähigkeiten die im Repo liegen. **Plugins** (v1.110) sind installierbare Pakete die editor-weit gelten und komplexere Tool-Bundles mitbringen können.

#### Agentic Browser Tools

Agents können jetzt den **integrierten Browser in VS Code direkt steuern** — kein externes Tool, kein Puppeteer MCP-Server nötig:

| Browser-Fähigkeit | Beschreibung |
|------------------|-------------|
| DOM-Interaktion | Elemente lesen, klicken, ausfüllen |
| Screenshots | Vollseiten- oder Bereichs-Screenshots für Debugging |
| Console Logs | Browser-Konsole auslesen |
| UI-Verifikation | Automatisierte Prüfung ob Änderungen korrekt rendern |

```text
Beispiel-Prompt im Agent Mode:
"Öffne http://localhost:3000, mach einen Screenshot der Login-Seite
und prüfe ob der Button 'Anmelden' korrekt positioniert ist."
```

> [!TIP] Browser Tools für Frontend-Entwicklung
> Besonders nützlich bei CSS-Fixes und UI-Bugs: Agent öffnet den Browser, macht einen Screenshot, vergleicht mit dem erwarteten Layout und schlägt Korrekturen vor — alles ohne VS Code zu verlassen.

#### Forkbare Chat-Sessions (`/fork`)

Chat-Sessions können jetzt **gegabelt** werden, um alternative Lösungsansätze zu erkunden, ohne den ursprünglichen Verlauf zu verlieren:

```text
Chat A: "Implementiere Authentifizierung mit JWT"
→ Lösung gefällt nicht, zu komplex

/fork  ← Erstellt Chat B als Kopie von Chat A

Chat B: "Implementiere Authentifizierung mit Sessions statt JWT"
→ Beide Ansätze vergleichbar, ohne Context-Verlust
```

#### Agent Debug Panel

Neues Debug-Panel mit Echtzeit-Einblick in:
- Agent-Events und Tool Calls (welches Tool wurde aufgerufen mit welchen Parametern)
- Geladene Plugins und Skills
- MCP-Server-Verbindungen
- Subagent-Status und -Ergebnisse

Ideal für die Entwicklung eigener Agent Plugins und Workflows.

#### Context Compaction & Session Memory

- **`/compact`-Befehl:** Komprimiert den Chat-Verlauf, um Context-Space für neue Anfragen freizugeben — ohne relevante Entscheidungen zu verlieren
- **Session Memory:** Agents persistieren "Pläne und Leitfäden" über Konversationsrunden hinweg

#### Weitere v1.110 Neuerungen

- **Co-authored-by Trailer:** AI-generierter Code erhält automatisch `Co-authored-by: GitHub Copilot` im Commit
- **MCP Server Sandboxing:** Lokale MCP-Server laufen in isolierten Umgebungen
- **Plan Mode:** Copilot erstellt zuerst einen Implementierungsplan, wartet auf Freigabe, führt dann aus
- **Redesigned Model Picker:** Übersichtlichere Modellauswahl mit Kategorien und Empfehlungen

---

## Preise & Kontingente

| Plan | Preis | Completions | Chat-Nachrichten |
|------|-------|-------------|-----------------|
| **Free** | kostenlos | 2.000 / Monat | 50 / Monat |
| **Pro** | $10 / Monat | Unbegrenzt | Unbegrenzt |
| **Business** | $19 / Nutzer / Monat | Unbegrenzt | Org-Management + Policies |

> [!NOTE] Free Tier als Einstieg
> Der Free Tier reicht für gelegentliche Nutzung. Bei aktivem Agent-Mode-Einsatz sind 50 Chat-Nachrichten schnell aufgebraucht — Pro lohnt sich ab täglicher Nutzung.

---

## Vergleich mit anderen IDE-Agents

| Feature | VS Code + Copilot | Cursor | Cline |
|---------|------------------|--------|-------|
| **Basis** | VS Code (original) | VS Code Fork | VS Code Extension |
| **Agent Mode** | ✅ GA seit v1.99 | ✅ | ✅ |
| **Subagents / Multi-Agent** | ✅ seit v1.109 | ⚠️ begrenzt | ❌ |
| **Agent Plugins** | ✅ seit v1.110 | ❌ | ❌ |
| **Browser-Steuerung** | ✅ nativ seit v1.110 | ❌ | ✅ via MCP |
| **Modellwahl** | ✅ 7+ Modelle inkl. GPT-5.4 | ✅ | ✅ beliebig (BYOK) |
| **MCP-Support** | ✅ + Sandboxing | ✅ | ✅ |
| **Custom Instructions** | ✅ `.github/copilot-instructions.md` | ✅ `.cursorrules` | ✅ |
| **Agent Skills** | ✅ seit v1.109 | ❌ | ❌ |
| **Terminal Sandboxing** | ✅ experimentell | ❌ | ❌ |
| **Lokal / Offline** | ❌ | ❌ | ✅ via Ollama |
| **GitHub-Integration** | ✅ Nativ | Über Extension | Über Extension |
| **Preis** | Free / $10 Pro | $20 Pro | Free (eigener API-Key) |

Detaillierter Vergleich: [[AI Coding Agents Vergleich]]

---

## Tipps & Best Practices

> [!TIP] Custom Instructions von Anfang an einrichten
> Erstelle `.github/copilot-instructions.md` direkt beim Projektstart. Konventionen (Sprache, Framework, Test-Setup, Code-Style) einmal definiert — Copilot hält sie in allen Sessions automatisch ein.

> [!TIP] Agent Mode mit kleinen Aufgaben beginnen
> Erste Agent-Mode-Aufgaben: klar definiert und überschaubar halten, z.B. *"Schreib Tests für alle Funktionen in `utils.ts`"*. Erst nach dem Aufbau von Vertrauen in das Verhalten größere Refactorings delegieren.

> [!TIP] Modell bewusst wählen
> Für iteratives Arbeiten (viele kleine Anfragen): GPT-4o oder o4-mini. Für komplexe Refactoring-Sessions: Claude Sonnet. Das spart Zeit durch schnellere Antworten bei Routine-Aufgaben.

Weiterführendes Prompting: [[Prompting Grundlagen]]

---

## Verwandte Notizen

- [[GitHub Copilot CLI Konfiguration]] — Copilot im Terminal statt in der IDE
- [[AI Coding Agents Vergleich]] — Cursor, Cline, Copilot und weitere im Vergleich
- [[Cursor Agent Konfiguration]] — Alternativer AI-IDE-Ansatz als VS-Code-Fork
- [[Cline Konfiguration]] — Open-Source Agent Mode als VS-Code-Extension
- [[Custom Instructions für AI Coding Agents]] — Anweisungsdateien tiefer erklärt
- [[Was ist MCP]] — Model Context Protocol Grundlagen
- [[MCP Grundlagen]] — MCP-Server einrichten und nutzen
- [[Prompting Grundlagen]] — Effektiv mit AI kommunizieren
- [[AI Index]] — Übersicht aller AI-Themen
