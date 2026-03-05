# GitHub Copilot CLI Konfiguration

## Was ist GitHub Copilot CLI?

GitHub Copilot CLI ist ein KI-gestützter Terminal-Assistent von GitHub, der direkt in der Kommandozeile arbeitet. Ähnlich wie Claude Code kann Copilot CLI Code lesen, schreiben, Befehle ausführen und komplexe Entwicklungsaufgaben autonom bearbeiten – alles innerhalb des Terminals.

> [!NOTE]
> Copilot CLI ist Teil des GitHub Copilot Ökosystems und nutzt verschiedene LLM-Modelle (Claude, GPT u.a.). Eine aktive GitHub Copilot Lizenz (Individual, Business oder Enterprise) ist Voraussetzung.

**Kernfähigkeiten:**
- Code-Analyse und -Generierung direkt im Terminal
- Dateien lesen, erstellen und bearbeiten
- Shell-Befehle ausführen (Builds, Tests, Linting)
- Git-Operationen durchführen
- MCP-Server anbinden
- Multi-Agent-Workflows mit Sub-Agents
- Web-Suche und Deep Research

---

## Installation

### Alle Plattformen im Überblick

| Plattform | Methode | Befehl |
|---|---|---|
| Linux / macOS | curl (empfohlen) | `curl -fsSL https://gh.io/copilot-install \| bash` |
| macOS | Homebrew | `brew install copilot-cli` |
| Alle (Node.js) | npm | `npm install -g @github/copilot` |
| Windows | WinGet | `winget install GitHub.Copilot` |

### Installation per curl (Linux / macOS)

```bash
# Installation starten
curl -fsSL https://gh.io/copilot-install | bash

# Installation prüfen
copilot --version
```

### Installation per Homebrew (macOS)

```bash
brew install copilot-cli

# Update
brew upgrade copilot-cli
```

### Installation per npm (Alle Plattformen)

```bash
npm install -g @github/copilot

# Oder mit spezifischer Version
npm install -g @github/copilot@latest
```

### Installation per WinGet (Windows)

```powershell
winget install GitHub.Copilot

# Update
winget upgrade GitHub.Copilot
```

> [!TIP]
> Die curl-Methode ist für Linux und macOS am einfachsten und erfordert keine Abhängigkeiten wie Node.js oder Homebrew.

---

## Starten

Nach der Installation wird Copilot CLI einfach über den Befehl `copilot` gestartet:

```bash
# Copilot CLI starten
copilot

# In einem bestimmten Verzeichnis starten
cd /pfad/zum/projekt && copilot
```

> [!IMPORTANT]
> Copilot CLI arbeitet kontextbezogen im aktuellen Verzeichnis. Es erkennt automatisch Git-Repositories und liest vorhandene Instruktionsdateien ein.

---

## Authentifizierung

### Interaktiver Login

Beim ersten Start oder über den Slash-Command `/login` wird ein interaktiver Login-Flow gestartet:

```bash
# Copilot starten → automatischer Login-Prompt
copilot

# Oder innerhalb einer Session:
/login
```

Der Login erfolgt über den Browser via GitHub OAuth. Nach erfolgreicher Authentifizierung wird die Session gespeichert.

### Authentifizierung per Personal Access Token (PAT)

Für CI/CD-Umgebungen oder automatisierte Setups kann ein PAT über Umgebungsvariablen übergeben werden:

```bash
# Option 1: GH_TOKEN
export GH_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"

# Option 2: GITHUB_TOKEN
export GITHUB_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"

# Dann Copilot starten
copilot
```

> [!WARNING]
> Der PAT benötigt die Permission **"Copilot Requests"**. Ohne diese Berechtigung schlägt die Authentifizierung fehl, auch wenn der Token gültig ist.

### PAT erstellen mit korrekter Permission

1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Token erstellen mit folgenden Permissions:
   - **Copilot Requests** – Pflicht für Copilot CLI
3. Token als `GH_TOKEN` oder `GITHUB_TOKEN` exportieren

```bash
# Dauerhaft in Shell-Profil setzen
echo 'export GH_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"' >> ~/.bashrc
source ~/.bashrc
```

> [!CAUTION]
> Tokens niemals in Repositories committen! Nutze `.env`-Dateien (in `.gitignore`) oder einen Secret Manager.

---

## Modelle und Modellwahl

### Standard-Modell

Copilot CLI nutzt standardmäßig **Claude Sonnet 4.5** als Basis-Modell.

### Verfügbare Modelle

| Modell | Typ | Stärken |
|---|---|---|
| Claude Sonnet 4.5 | Standard (Default) | Ausgewogen, schnell, gute Code-Qualität |
| Claude Sonnet 4 | Standard | Solide Alternative, etwas anders gewichtet |
| GPT-5 | Standard | OpenAI-Modell, gut für bestimmte Aufgaben |
| Weitere Modelle | Variabel | Über `/model` einsehbar |

### Modell wechseln

Das Modell kann jederzeit während einer Session gewechselt werden:

```
/model
```

Nach Eingabe des Commands wird eine Liste der verfügbaren Modelle angezeigt, aus der interaktiv gewählt werden kann.

> [!TIP]
> Das Modell kann während einer laufenden Session gewechselt werden, ohne den Kontext zu verlieren. Das ist praktisch, wenn ein anderes Modell für eine bestimmte Teilaufgabe besser geeignet ist.

### Vergleich: Modellwahl Copilot CLI vs. Claude Code

| Aspekt | Copilot CLI | Claude Code |
|---|---|---|
| Standard-Modell | Claude Sonnet 4.5 | Claude Sonnet 4.6 |
| Modellwechsel | `/model` (interaktiv) | `--model` Flag oder `settings.json` |
| Verfügbare Modelle | Claude, GPT, Gemini u.a. | Nur Claude-Modelle |
| Wechsel in Session | Ja, jederzeit | Ja, über `/model` |

---

## Modi

Copilot CLI bietet drei verschiedene Betriebsmodi, die sich in der Autonomie des Agenten unterscheiden:

### Interactive Mode (Standard)

Der Standard-Modus. Copilot fragt vor jeder Aktion um Bestätigung.

- Copilot schlägt Änderungen vor
- Nutzer bestätigt oder lehnt ab
- Volle Kontrolle über jeden Schritt

### Plan Mode

Im Plan-Modus erstellt Copilot zunächst einen detaillierten Plan, bevor Änderungen durchgeführt werden.

- Copilot analysiert die Aufgabe
- Erstellt einen strukturierten Plan
- Führt nach Bestätigung die Schritte aus

### Autopilot Mode (Experimentell)

Im Autopilot-Modus arbeitet Copilot weitgehend autonom.

- Copilot führt Aktionen selbstständig aus
- Minimale Nutzerinteraktion erforderlich
- Ideal für klar definierte Aufgaben

> [!WARNING]
> Autopilot ist als experimentell markiert. Besonders bei destruktiven Operationen (Datei-Löschung, Git-Pushes) sollte man vorsichtig sein und vorher die Permissions korrekt konfigurieren.

### Zwischen Modi wechseln

```
Shift+Tab  →  Wechselt zwischen Interactive → Plan → Autopilot
```

| Modus | Autonomie | Empfohlen für |
|---|---|---|
| Interactive | Niedrig | Sensible Änderungen, unbekannter Code |
| Plan | Mittel | Komplexe Refactorings, mehrstufige Aufgaben |
| Autopilot | Hoch | Routineaufgaben, gut definierte Tasks |

### Vergleich: Modi Copilot CLI vs. Claude Code

| Aspekt | Copilot CLI | Claude Code |
|---|---|---|
| Bestätigungs-Modus | Interactive (Standard) | Standard (mit Bestätigung) |
| Plan-Modus | Ja (Shift+Tab) | Ja (Shift+Tab) |
| Auto-Modus | Autopilot (experimentell) | Auto-accept (Shift+Tab) |
| Modus-Wechsel | Shift+Tab (zyklisch) | Shift+Tab (zyklisch) |

---

## Wichtige Slash-Commands

### Übersicht aller Commands

| Command | Beschreibung |
|---|---|
| `/model` | KI-Modell auswählen oder wechseln |
| `/mcp` | MCP-Server Konfiguration verwalten |
| `/init` | Copilot-Instruktionsdateien initialisieren |
| `/skills` | Verfügbare Skills verwalten |
| `/agent` | Verfügbare Agents durchsuchen |
| `/fleet` | Parallele Sub-Agent-Ausführung aktivieren |
| `/tasks` | Hintergrund-Tasks anzeigen |
| `/diff` | Änderungen im Diff-Format reviewen |
| `/review` | Code-Review Agent starten |
| `/lsp` | Language Server Konfiguration |
| `/research` | Deep Research mit Web-Suche starten |
| `/instructions` | Custom Instruction Files anzeigen/toggeln |
| `/compact` | Kontext-Fenster komprimieren |
| `/share` | Session als Markdown oder Gist teilen |
| `/login` | GitHub-Authentifizierung durchführen |
| `/allow-all` | Alle Permissions aktivieren |
| `/add-dir` | Erlaubtes Verzeichnis hinzufügen |
| `/list-dirs` | Erlaubte Verzeichnisse auflisten |

### Detailbeschreibung wichtiger Commands

#### /model – Modellwahl

```
/model
```

Zeigt eine interaktive Auswahlliste aller verfügbaren KI-Modelle an. Das gewählte Modell wird sofort für die laufende Session aktiv.

#### /mcp – MCP Server

```
/mcp
```

Verwaltet die Konfiguration von Model Context Protocol (MCP) Servern. Ermöglicht das Hinzufügen, Entfernen und Konfigurieren externer Tool-Server.

#### /init – Instruktionen initialisieren

```
/init
```

Erstellt Basis-Instruktionsdateien im aktuellen Projekt. Erzeugt Vorlagen für Custom Instructions, die das Verhalten von Copilot CLI projektspezifisch steuern.

#### /fleet – Parallele Agents

```
/fleet
```

Aktiviert den Fleet-Modus für parallele Sub-Agent-Ausführung. Mehrere Agents können gleichzeitig an verschiedenen Teilaufgaben arbeiten, was die Bearbeitungszeit bei komplexen Aufgaben deutlich reduziert.

> [!NOTE]
> Fleet-Modus ist besonders nützlich bei großen Refactorings oder wenn mehrere unabhängige Dateien gleichzeitig bearbeitet werden müssen.

#### /research – Deep Research

```
/research
```

Startet einen Deep-Research-Modus mit Web-Suche. Copilot CLI durchsucht das Internet nach relevanten Informationen und fasst die Ergebnisse zusammen. Ideal für:

- Recherche zu APIs und Bibliotheken
- Best Practices für bestimmte Technologien
- Aktuelle Informationen, die über das Training-Datum hinausgehen

#### /diff – Änderungen reviewen

```
/diff
```

Zeigt alle von Copilot vorgenommenen Änderungen im Diff-Format an. Ermöglicht ein schnelles Review vor dem Commit.

#### /compact – Kontext komprimieren

```
/compact
```

Komprimiert das Kontext-Fenster, indem ältere Konversationsteile zusammengefasst werden. Nützlich bei langen Sessions, um Token-Limits nicht zu überschreiten.

#### /share – Session teilen

```
/share
```

Exportiert die aktuelle Session als Markdown-Datei oder als GitHub Gist. Praktisch für Dokumentation oder zum Teilen mit Kollegen.

---

## MCP Server Konfiguration

### Was ist MCP?

Das Model Context Protocol (MCP) ermöglicht es, externe Tool-Server an Copilot CLI anzubinden. Damit kann Copilot auf zusätzliche Datenquellen und Werkzeuge zugreifen.

> [!INFO]
> Für eine ausführliche Einführung in MCP siehe [[Was ist MCP]] und [[MCP Server in Claude Code einbinden]].

### Built-in GitHub MCP Server

Copilot CLI bringt einen integrierten GitHub MCP Server mit, der automatisch verfügbar ist. Dieser bietet Zugriff auf:

- GitHub Issues und Pull Requests
- Repository-Inhalte
- GitHub Actions und Workflows
- Code Search über GitHub

### Custom MCP Server hinzufügen

Über den `/mcp`-Command können eigene MCP-Server konfiguriert werden:

```
/mcp
```

Die Konfiguration wird interaktiv durchgeführt. MCP-Server können verschiedene Transportprotokolle nutzen:

- **stdio** – Lokaler Prozess mit stdin/stdout-Kommunikation
- **SSE** – Server-Sent Events über HTTP

### Beispiel: Filesystem MCP Server

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

### Vergleich: MCP in Copilot CLI vs. Claude Code

| Aspekt | Copilot CLI | Claude Code |
|---|---|---|
| Konfiguration | `/mcp` Command (interaktiv) | `settings.json` → `mcpServers` |
| Built-in Server | GitHub MCP Server | Keiner |
| Transportprotokolle | stdio, SSE | stdio, SSE |
| Hot-Reload | Ja | Ja |

---

## LSP Server Konfiguration

### Was ist LSP im Kontext von Copilot CLI?

Copilot CLI kann Language Server Protocol (LSP) Server nutzen, um bessere Code-Analyse und Navigation zu bieten. LSP-Server liefern Informationen wie:

- Typinformationen und Autocompletion
- Go-to-Definition und References
- Diagnostics (Fehler und Warnungen)
- Symbol-Suche

### Konfigurationsdateien

LSP-Server werden über JSON-Dateien konfiguriert:

| Speicherort | Geltungsbereich |
|---|---|
| `~/.copilot/lsp-config.json` | Global (User-Level) |
| `.github/lsp.json` | Projekt (Repository-Level) |

### Beispiel: TypeScript LSP Konfiguration

**User-Level (`~/.copilot/lsp-config.json`):**

````json
{
  "servers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "languages": ["typescript", "typescriptreact", "javascript", "javascriptreact"],
      "initializationOptions": {
        "preferences": {
          "includeInlayParameterNameHints": "all",
          "includeInlayVariableTypeHints": true
        }
      }
    }
  }
}
````

**Repository-Level (`.github/lsp.json`):**

````json
{
  "servers": {
    "typescript": {
      "command": "npx",
      "args": ["typescript-language-server", "--stdio"],
      "languages": ["typescript", "typescriptreact"]
    }
  }
}
````

### LSP prüfen und konfigurieren

Über den Slash-Command `/lsp` kann die aktuelle LSP-Konfiguration eingesehen und angepasst werden:

```
/lsp
```

> [!TIP]
> LSP-Server verbessern die Code-Qualität von Copilot CLI erheblich, da der Agent dadurch präzisere Typinformationen und Fehlermeldungen erhält. Besonders bei TypeScript- und Rust-Projekten lohnt sich die Konfiguration.

---

## Custom Instructions

### Unterstützte Instruktionsdateien

Copilot CLI liest Custom Instructions aus verschiedenen Dateien ein. Die Dateien werden in folgender Reihenfolge berücksichtigt:

| Datei | Speicherort | Beschreibung |
|---|---|---|
| `CLAUDE.md` | Projekt-Root | Claude-spezifische Instruktionen (wird auch von Copilot gelesen) |
| `GEMINI.md` | Projekt-Root | Gemini-spezifische Instruktionen |
| `AGENTS.md` | Projekt-Root | Allgemeine Agent-Instruktionen |
| `*.instructions.md` | `.github/instructions/` | Modulare Instruktionsdateien |
| `copilot-instructions.md` | `.github/` | GitHub Copilot-spezifische Instruktionen |
| `copilot-instructions.md` | `$HOME/.copilot/` | Globale Copilot-Instruktionen |

> [!IMPORTANT]
> Copilot CLI erkennt sowohl `CLAUDE.md` als auch `AGENTS.md` und die GitHub-spezifischen Instruktionsdateien. Das erleichtert die Migration von Claude Code zu Copilot CLI, da bestehende `CLAUDE.md`-Dateien weiterhin funktionieren.

### Instruktionsdateien initialisieren

```
/init
```

Erstellt Basis-Instruktionsdateien im Projekt. Alternativ manuell anlegen:

```bash
# GitHub-spezifische Instruktionen
mkdir -p .github/instructions
touch .github/copilot-instructions.md

# Modulare Instruktionen
touch .github/instructions/code-style.instructions.md
touch .github/instructions/testing.instructions.md

# Globale Instruktionen
mkdir -p ~/.copilot
touch ~/.copilot/copilot-instructions.md
```

### Beispiel: Projekt-Instruktionen

**`.github/copilot-instructions.md`:**

````markdown
# Projekt-Instruktionen

## Technologie-Stack
- Backend: Node.js mit TypeScript
- Frontend: React mit Next.js
- Datenbank: PostgreSQL mit Prisma ORM
- Tests: Jest und React Testing Library

## Code-Konventionen
- Verwende funktionale Komponenten mit Hooks
- Schreibe Tests für alle neuen Funktionen
- Nutze TypeScript strict mode
- Kommentare auf Englisch

## Git-Konventionen
- Conventional Commits (feat:, fix:, docs:, etc.)
- Feature-Branches mit Prefix: feature/, bugfix/, hotfix/
````

### Beispiel: Modulare Instruktionen

**`.github/instructions/testing.instructions.md`:**

````markdown
# Testing-Instruktionen

## Test-Framework
- Verwende Jest für Unit-Tests
- React Testing Library für Komponenten-Tests
- Playwright für E2E-Tests

## Test-Konventionen
- Dateiname: `*.test.ts` oder `*.spec.ts`
- Beschreibende Test-Namen auf Englisch
- AAA-Pattern: Arrange, Act, Assert
- Mindestens 80% Code-Coverage für neue Dateien
````

### Instruktionen anzeigen und toggeln

```
/instructions
```

Zeigt alle erkannten Instruktionsdateien an und ermöglicht das Aktivieren/Deaktivieren einzelner Dateien für die aktuelle Session.

### Vergleich: Custom Instructions Copilot CLI vs. Claude Code

| Aspekt | Copilot CLI | Claude Code |
|---|---|---|
| Primäre Datei | `.github/copilot-instructions.md` | `CLAUDE.md` |
| Globale Instruktionen | `~/.copilot/copilot-instructions.md` | `~/.claude/CLAUDE.md` |
| Modulare Dateien | `.github/instructions/*.instructions.md` | Nicht direkt unterstützt |
| Cross-Kompatibilität | Liest auch `CLAUDE.md` | Nur `CLAUDE.md` |
| Init-Command | `/init` | Manuell anlegen |

---

## Keyboard Shortcuts

### Übersicht aller Shortcuts

| Shortcut | Funktion | Beschreibung |
|---|---|---|
| `@` | Datei erwähnen | Datei direkt im Prompt referenzieren |
| `Shift+Tab` | Modus wechseln | Zwischen Interactive → Plan → Autopilot |
| `Ctrl+S` | Befehl ausführen | Führt Befehl aus, Input bleibt erhalten |
| `Ctrl+T` | Reasoning anzeigen | Reasoning-Anzeige ein-/ausschalten |
| `Ctrl+O` | Timeline expandieren | Timeline/Verlauf aufklappen |
| `Ctrl+E` | Timeline expandieren | Alternative für Timeline-Expansion |
| `!` | Shell-Befehl | Shell-Befehl direkt ausführen |

### Datei-Referenzierung mit @

Das `@`-Zeichen erlaubt es, Dateien direkt im Prompt zu referenzieren:

```
@src/utils/helpers.ts Erkläre diese Datei

@package.json Welche Dependencies fehlen?

@src/components/ Refactore alle Komponenten in diesem Ordner
```

> [!TIP]
> Mit `@` referenzierte Dateien werden direkt in den Kontext geladen. Das ist effizienter als Copilot die Dateien selbst suchen zu lassen.

### Shell-Befehle mit !

Das `!`-Prefix führt Shell-Befehle direkt aus, ohne dass Copilot sie interpretiert:

```
!git status
!npm test
!ls -la src/
```

---

## Permissions und Sicherheit

### Standard-Permissions

Copilot CLI arbeitet standardmäßig mit eingeschränkten Permissions und fragt vor potenziell gefährlichen Aktionen um Bestätigung.

### Alle Permissions aktivieren

```
/allow-all
```

> [!CAUTION]
> `/allow-all` gibt Copilot CLI volle Kontrolle über Dateioperationen und Shell-Befehle. Nur in vertrauenswürdigen Umgebungen verwenden!

### Verzeichnisse verwalten

Copilot CLI kann auf bestimmte Verzeichnisse beschränkt werden:

```
# Verzeichnis hinzufügen
/add-dir /pfad/zum/verzeichnis

# Erlaubte Verzeichnisse auflisten
/list-dirs
```

### Vergleich: Permissions Copilot CLI vs. Claude Code

| Aspekt | Copilot CLI | Claude Code |
|---|---|---|
| Permission-System | `/allow-all`, `/add-dir` | `settings.json` → `permissions` |
| Granularität | Verzeichnis-basiert | Tool- und Pattern-basiert |
| Deny-Rules | Über Verzeichnisbeschränkung | Explizite Deny-Liste |
| Bestätigungsprompts | Ja (Standard) | Ja (Standard) |

> [!IMPORTANT]
> Bei Copilot CLI erfolgt die Sicherheitskontrolle primär über Verzeichnisbeschränkungen und Bestätigungsprompts. Claude Code bietet dagegen ein feingranulareres Pattern-basiertes Permission-System über `settings.json`.

---

## Vergleich: Copilot CLI vs. Claude Code

### Feature-Vergleich

| Feature | Copilot CLI | Claude Code |
|---|---|---|
| **Hersteller** | GitHub (Microsoft) | Anthropic |
| **Standard-Modell** | Claude Sonnet 4.5 | Claude Sonnet 4.6 |
| **Modellauswahl** | Multi-Anbieter (Claude, GPT, Gemini) | Nur Claude-Modelle |
| **Installation** | curl, brew, npm, winget | npm (`@anthropic-ai/claude-code`) |
| **Authentifizierung** | GitHub OAuth / PAT | Anthropic API Key / Max Plan |
| **MCP Support** | Ja + Built-in GitHub MCP | Ja, über `settings.json` |
| **LSP Support** | Ja, konfigurierbar | Nein (eigene Code-Analyse) |
| **Custom Instructions** | Mehrere Formate + `CLAUDE.md` | `CLAUDE.md` |
| **Hooks** | Nicht dokumentiert | PreToolUse, PostToolUse, Stop, Notification |
| **Permission-System** | Verzeichnis-basiert | Pattern-basiertes Allow/Deny |
| **Sub-Agents** | Ja (`/fleet`, `/agent`) | Ja (automatisch) |
| **Web-Suche** | Ja (`/research`) | Nicht nativ |
| **Session-Sharing** | `/share` (Markdown/Gist) | Nicht nativ |
| **Reasoning-Anzeige** | `Ctrl+T` toggle | Extended Thinking (automatisch) |
| **Preismodell** | GitHub Copilot Lizenz | Anthropic API / Max Subscription |

### Wann welches Tool?

| Szenario | Empfehlung | Begründung |
|---|---|---|
| GitHub-zentrierte Workflows | Copilot CLI | Built-in GitHub MCP, native Integration |
| Multi-Modell Flexibilität | Copilot CLI | Unterstützt Claude, GPT, Gemini |
| Feingranulare Permissions | Claude Code | Pattern-basiertes Allow/Deny-System |
| Hook-basierte Automatisierung | Claude Code | PreToolUse, PostToolUse Hooks |
| Bestehende CLAUDE.md vorhanden | Beide | Copilot CLI liest auch CLAUDE.md |
| Enterprise / Compliance | Copilot CLI | GitHub Enterprise Integration |
| Deep Research | Copilot CLI | Integrierte Web-Suche |

---

## Tipps und Troubleshooting

### Best Practices

> [!TIP]
> **1. Instruktionsdateien pflegen**
> Investiere Zeit in gute Custom Instructions. Je präziser die Instruktionen, desto besser die Ergebnisse. Nutze `.github/copilot-instructions.md` für projektspezifische Vorgaben.

> [!TIP]
> **2. LSP-Server konfigurieren**
> Konfiguriere LSP-Server für deine Hauptsprachen. Copilot CLI arbeitet mit Typ-Informationen deutlich präziser, besonders bei TypeScript und Rust.

> [!TIP]
> **3. Modus passend wählen**
> Nutze Interactive für sensible Änderungen, Plan für komplexe Aufgaben und Autopilot nur für Routine-Tasks in gut getesteten Projekten.

> [!TIP]
> **4. Kontext mit @ steuern**
> Referenziere relevante Dateien direkt mit `@` statt Copilot suchen zu lassen. Das spart Tokens und liefert präzisere Ergebnisse.

> [!TIP]
> **5. /compact regelmäßig nutzen**
> Bei langen Sessions regelmäßig `/compact` ausführen, um das Kontext-Fenster zu komprimieren und Token-Limits zu vermeiden.

### Häufige Probleme und Lösungen

#### Authentifizierung schlägt fehl

```bash
# Token-Permissions prüfen
# → PAT muss "Copilot Requests" Permission haben

# Umgebungsvariablen prüfen
echo $GH_TOKEN
echo $GITHUB_TOKEN

# Neu einloggen
# In Copilot CLI:
/login
```

#### MCP-Server startet nicht

```bash
# Prüfen ob der MCP-Server manuell funktioniert
npx -y @modelcontextprotocol/server-filesystem /tmp

# Node.js Version prüfen (>= 18 empfohlen)
node --version

# MCP-Konfiguration neu laden
/mcp
```

#### Copilot CLI findet Dateien nicht

```bash
# Sicherstellen, dass im richtigen Verzeichnis gestartet
pwd

# Erlaubte Verzeichnisse prüfen
/list-dirs

# Verzeichnis hinzufügen
/add-dir /pfad/zum/projekt
```

#### Kontext-Fenster voll / Langsame Antworten

```
# Kontext komprimieren
/compact

# Oder neue Session starten
# Copilot beenden und neu starten
```

#### LSP-Server wird nicht erkannt

```bash
# Prüfen ob LSP-Server installiert ist
which typescript-language-server

# Installieren falls nötig
npm install -g typescript-language-server typescript

# Konfiguration prüfen
cat ~/.copilot/lsp-config.json
```

### Migration von Claude Code zu Copilot CLI

Wenn du von Claude Code zu Copilot CLI wechseln möchtest:

1. **CLAUDE.md beibehalten** – Copilot CLI liest auch `CLAUDE.md`-Dateien
2. **MCP-Server übertragen** – MCP-Konfiguration aus `settings.json` in Copilot CLI `/mcp` übernehmen
3. **Permissions anpassen** – Copilot CLI nutzt ein anderes Permission-System (Verzeichnis-basiert statt Pattern-basiert)
4. **Hooks ersetzen** – Copilot CLI hat kein direktes Hook-Equivalent; ggf. über CI/CD-Pipelines abbilden
5. **Instruktionsdateien erweitern** – Zusätzlich `.github/copilot-instructions.md` anlegen für Copilot-spezifische Anweisungen

> [!NOTE]
> Beide Tools können parallel genutzt werden. `CLAUDE.md` wird von beiden gelesen, sodass ein sanfter Übergang möglich ist.

---

## Weiterführende Links

- [[Claude Code Konfiguration und Hooks]] – Detaillierte Claude Code Konfiguration
- [[Was ist MCP]] – Grundlagen des Model Context Protocol
- [[MCP Server in Claude Code einbinden]] – MCP-Server Konfiguration (auch für Copilot relevant)
- [[CLAUDE.md schreiben]] – Best Practices für Instruktionsdateien
- [[Multi-Agent Workflows]] – Parallele Agent-Ausführung
