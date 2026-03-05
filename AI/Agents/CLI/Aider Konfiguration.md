# Aider Konfiguration

## Was ist Aider?

Aider ist ein leichtgewichtiger, git-nativer KI-Coding-Assistent für das Terminal, entwickelt von Paul Gauthier. Mit über 40.000 GitHub-Sternen ist Aider eines der beliebtesten Open-Source-Tools für KI-gestütztes Programmieren. Das zentrale Designprinzip: Jede KI-Änderung wird automatisch als Git-Commit erfasst – mit einer aussagekräftigen Commit-Nachricht.

> [!NOTE]
> Aider unterscheidet sich von schwergewichtigen Agents wie Claude Code oder Codex CLI dadurch, dass es bewusst schlank bleibt. Es liest und schreibt Dateien, erstellt Git-Commits und nutzt verschiedene LLMs – ohne eigene Sandbox, Hook-Systeme oder MCP-Server.

**Kernfähigkeiten:**
- KI-gestütztes Code-Editing direkt im Terminal
- Automatische Git-Commits mit beschreibenden Nachrichten
- Multi-Datei-Bearbeitung über mehrere Dateien gleichzeitig
- Unterstützung für Cloud- und lokale LLMs (OpenAI, Anthropic, Google, Ollama, etc.)
- Architect-Modus (Planen, dann Ausführen)
- Watch-Modus für automatische Reaktion auf Dateiänderungen
- Repository-Map für intelligentes Kontextmanagement
- Spracheingabe und Bilderkennung
- CONVENTIONS.md und AGENTS.md für Custom Instructions

**GitHub:** [paul-gauthier/aider](https://github.com/paul-gauthier/aider)

---

## Installation

### Alle Methoden im Überblick

| Methode | Befehl | Empfehlung |
|---|---|---|
| pip | `pip install aider-chat` | Für bestehende Python-Umgebungen |
| pipx (empfohlen) | `pipx install aider-chat` | Isolierte Installation, kein Konflikt mit anderen Paketen |
| Docker | `docker run -it ghcr.io/paul-gauthier/aider` | Für containerisierte Umgebungen |
| Homebrew | `brew install aider` | macOS-Nutzer |

### Installation per pipx (Empfohlen)

```bash
# pipx installieren (falls nicht vorhanden)
pip install pipx
pipx ensurepath

# Aider installieren
pipx install aider-chat

# Installation prüfen
aider --version

# Update auf neueste Version
pipx upgrade aider-chat
```

### Installation per pip

```bash
# Installation
pip install aider-chat

# In einer virtuellen Umgebung (empfohlen)
python -m venv .venv
source .venv/bin/activate
pip install aider-chat
```

### Installation per Docker

```bash
# Einmalig ausführen
docker run -it \
  -v $(pwd):/app \
  -e OPENAI_API_KEY=$OPENAI_API_KEY \
  ghcr.io/paul-gauthier/aider

# Alias für regelmäßige Nutzung
alias aider='docker run -it -v $(pwd):/app -e OPENAI_API_KEY ghcr.io/paul-gauthier/aider'
```

> [!IMPORTANT]
> Aider benötigt Python 3.9 oder höher. Bei der Installation per `pip` kann es zu Konflikten mit anderen Python-Paketen kommen – `pipx` vermeidet dieses Problem, indem es Aider in einer isolierten Umgebung installiert.

---

## Schnellstart

### Voraussetzungen

1. Git-Repository (Aider funktioniert nur innerhalb eines Git-Repos)
2. API-Key für mindestens einen LLM-Provider

### Erster Start

```bash
# In ein Git-Repository wechseln
cd ~/projects/mein-projekt

# API-Key setzen (z.B. OpenAI)
export OPENAI_API_KEY="sk-..."

# Aider starten
aider

# Oder mit spezifischem Modell starten
aider --model gpt-4o

# Oder mit Anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
aider --model claude-sonnet-4-20250514
```

### Erste Interaktion

Nach dem Start befindet man sich in einer interaktiven Chat-Sitzung:

```
Aider v0.82.0
Model: gpt-4o with diff edit format
Git repo: .git with 42 files
> Erstelle eine Python-Funktion, die Fibonacci-Zahlen berechnet
```

Aider analysiert die Anfrage, erstellt oder bearbeitet die entsprechenden Dateien und committet die Änderungen automatisch.

> [!TIP]
> Beim ersten Start zeigt Aider eine Übersicht der erkannten Dateien im Repository. Mit `/add` können gezielt Dateien zum Kontext hinzugefügt werden, damit die KI den bestehenden Code versteht.

---

## Konfiguration (.aider.conf.yml)

Aider wird über YAML-Dateien konfiguriert. Es gibt zwei Ebenen:

| Speicherort | Geltungsbereich | Beschreibung |
|---|---|---|
| `~/.aider.conf.yml` | Global | Gilt für alle Projekte des Nutzers |
| `.aider.conf.yml` (Projektroot) | Projekt | Gilt nur für dieses spezifische Projekt |

> [!NOTE]
> Projekt-Konfiguration überschreibt globale Konfiguration. Kommandozeilen-Argumente überschreiben beide. Die Reihenfolge ist: CLI-Argumente > Projekt-Datei > Globale Datei.

### Vollständige .aider.conf.yml – Beispiel

```yaml
## Modell-Einstellungen
model: claude-sonnet-4-20250514
edit-format: diff
architect: true

## Git-Einstellungen
auto-commits: true
dirty-commits: true
auto-lint: true
auto-test: false
commit-prompt: "Write a concise commit message in conventional commit format."

## Kontext-Einstellungen
map-tokens: 1024
map-refresh: auto
subtree-only: false

## Darstellung
dark-mode: true
pretty: true
stream: true
show-diffs: true

## Verlauf
chat-history-file: .aider.chat.history.md
input-history-file: .aider.input.history

## Lint & Test
lint-cmd: "ruff check --fix"
test-cmd: "pytest -x"

## Dateien, die immer gelesen werden sollen
read:
  - CONVENTIONS.md
  - docs/architecture.md

## Dateien/Verzeichnisse ignorieren
aiderignore: .aiderignore
```

### .aiderignore – Dateien ausschließen

Ähnlich wie `.gitignore` können Dateien und Verzeichnisse von Aider ausgeschlossen werden:

```gitignore
# .aiderignore
node_modules/
dist/
build/
*.min.js
*.lock
.env
__pycache__/
*.pyc
```

> [!TIP]
> Eine `.aiderignore`-Datei ist besonders nützlich in großen Repositories, um irrelevante Dateien aus dem Repository-Map auszuschließen und Token zu sparen.

---

## Custom Instructions (CONVENTIONS.md & AGENTS.md)

### CONVENTIONS.md

Aider liest automatisch eine `CONVENTIONS.md`-Datei im Projektroot, um projektspezifische Coding-Konventionen zu befolgen.

````markdown
# CONVENTIONS.md – Beispiel

## Code-Stil
- Verwende TypeScript strict mode
- Bevorzuge `const` über `let`
- Nutze async/await statt Promises mit `.then()`

## Benennung
- React-Komponenten: PascalCase
- Utility-Funktionen: camelCase
- Konstanten: UPPER_SNAKE_CASE

## Tests
- Jede neue Funktion braucht mindestens einen Unit-Test
- Verwende `describe` / `it` Blöcke mit Vitest
- Test-Dateien: `*.test.ts` neben der Quelldatei

## Git
- Conventional Commits verwenden
- Kein Code ohne zugehörigen Test committen

## Dokumentation
- JSDoc für alle exportierten Funktionen
- README aktuell halten bei API-Änderungen
````

### AGENTS.md

Aider liest ebenfalls `AGENTS.md` – das Format, das ursprünglich von OpenAI Codex CLI eingeführt wurde und mittlerweile von mehreren Agents unterstützt wird.

````markdown
# AGENTS.md – Beispiel

## Projektübersicht
Dies ist eine FastAPI-Anwendung mit PostgreSQL-Datenbank.

## Architektur
- `src/api/` – API-Endpunkte (FastAPI Router)
- `src/models/` – SQLAlchemy-Modelle
- `src/services/` – Business-Logik
- `src/utils/` – Hilfsfunktionen

## Befehle
```bash
# Tests ausführen
pytest -x --tb=short

# Linting
ruff check .

# Server starten
uvicorn src.main:app --reload
```

## Regeln
- Keine direkten Datenbankzugriffe außerhalb von `src/services/`
- Alle API-Endpunkte müssen Pydantic-Modelle für Request/Response verwenden
````

> [!IMPORTANT]
> `CONVENTIONS.md` wird von Aider nativ unterstützt und automatisch gelesen. `AGENTS.md` wird ebenfalls gelesen, wobei `CONVENTIONS.md` die bevorzugte Methode für Aider-spezifische Konventionen ist. Wer mehrere Agents nutzt, kann beide Dateien parallel pflegen.

---

## Dateiverwaltung (/add, /drop, /map)

Aider arbeitet nur mit Dateien, die explizit zum Kontext hinzugefügt wurden. Dies spart Token und verbessert die Qualität der KI-Antworten.

### Dateien hinzufügen und entfernen

| Befehl | Beschreibung |
|---|---|
| `/add datei.py` | Einzelne Datei zum Kontext hinzufügen (editierbar) |
| `/add src/*.py` | Mehrere Dateien per Glob-Pattern hinzufügen |
| `/add-read docs/api.md` | Datei nur lesend hinzufügen (wird nicht editiert) |
| `/drop datei.py` | Datei aus dem Kontext entfernen |
| `/drop *` | Alle Dateien aus dem Kontext entfernen |
| `/ls` | Alle aktuell im Kontext befindlichen Dateien auflisten |

```
> /add src/auth/login.py src/auth/utils.py
Added src/auth/login.py to the chat.
Added src/auth/utils.py to the chat.

> /add-read README.md
Added README.md as read-only to the chat.

> /ls
  src/auth/login.py
  src/auth/utils.py
  README.md (read-only)

> /drop src/auth/utils.py
Removed src/auth/utils.py from the chat.
```

### Repository Map

Aider erstellt automatisch eine Repository-Map – eine komprimierte Übersicht aller Dateien, Klassen und Funktionen im Repository. Diese Map hilft der KI, den Gesamtkontext zu verstehen, ohne alle Dateien vollständig lesen zu müssen.

```yaml
# In .aider.conf.yml
map-tokens: 1024        # Token-Budget für die Map (Standard: 1024)
map-refresh: auto        # auto, always, files, manual
subtree-only: false      # Nur Unterverzeichnis mappen
```

> [!TIP]
> Bei großen Repositories kann `map-tokens` erhöht werden (z.B. 2048 oder 4096), um der KI mehr Kontext zu geben. Bei sehr kleinen Projekten kann die Map mit `map-tokens: 0` deaktiviert werden.

---

## Edit-Formate (whole, diff, udiff)

Aider unterstützt drei verschiedene Formate, in denen die KI Code-Änderungen vorschlägt:

| Format | Beschreibung | Beste für |
|---|---|---|
| `whole` | Gesamte Datei wird neu geschrieben | Kleine Dateien, neue Dateien |
| `diff` | Search-and-Replace-Blöcke | Gezielte Änderungen in großen Dateien |
| `udiff` | Unified-Diff-Format | Komplexe Änderungen, erfahrene Nutzer |

### Format setzen

```bash
# Per Kommandozeile
aider --edit-format diff

# Per Konfiguration (.aider.conf.yml)
# edit-format: diff

# Während der Sitzung wechseln
> /chat-mode diff
```

### Vergleich der Formate

**whole** – die KI schreibt die gesamte Datei neu:
- ✅ Einfach und zuverlässig
- ❌ Verbraucht viele Token bei großen Dateien
- ❌ Kann unbeabsichtigt Code entfernen

**diff** – Search-and-Replace-Blöcke:
- ✅ Token-effizient
- ✅ Präzise Änderungen
- ❌ Kann bei ungenauen Suchblöcken fehlschlagen

**udiff** – Unified-Diff-Format:
- ✅ Kompaktes Format
- ✅ Vertraut für erfahrene Entwickler
- ❌ Höhere Fehlerquote bei manchen Modellen

> [!NOTE]
> Das `diff`-Format ist der Standard für die meisten Modelle und bietet den besten Kompromiss aus Token-Effizienz und Zuverlässigkeit. `whole` eignet sich besser für kleinere Modelle, die mit Diff-Formaten Schwierigkeiten haben.

---

## Modi (Architect, Watch, Map)

### Architect-Modus

Im Architect-Modus plant die KI zunächst die Änderungen und führt sie anschließend aus. Dies ist besonders nützlich für komplexe, mehrstufige Aufgaben.

```bash
# Architect-Modus aktivieren
aider --architect

# Oder in der Konfiguration
# architect: true
```

Ablauf:
1. **Architect-LLM** (z.B. ein stärkeres Modell) erstellt einen detaillierten Plan
2. **Editor-LLM** (z.B. ein schnelleres Modell) setzt den Plan um

```yaml
# Verschiedene Modelle für Architect und Editor
model: claude-sonnet-4-20250514
editor-model: claude-haiku-4-20250514
architect: true
```

> [!TIP]
> Der Architect-Modus eignet sich besonders für umfangreiche Refactorings oder Feature-Implementierungen, bei denen eine Planungsphase die Qualität deutlich verbessert.

### Watch-Modus

Der Watch-Modus überwacht Dateien auf Änderungen und reagiert automatisch. Kommentare mit speziellen Markierungen lösen KI-Aktionen aus.

```bash
# Watch-Modus starten
aider --watch
```

Funktionsweise: Aider überwacht Dateien im Repository. Wenn eine Datei mit einem speziellen Kommentar gespeichert wird, führt Aider die Anweisung aus:

```python
# ai: Füge Error-Handling für den Datenbankzugriff hinzu
def get_user(user_id: int):
    return db.query(User).filter(User.id == user_id).first()
```

Aider erkennt den `# ai:`-Kommentar und bearbeitet die Funktion entsprechend.

### Map-Modus

Der Map-Modus generiert eine strukturierte Übersicht des Repositories, die als Kontext für die KI dient:

```bash
# Map anzeigen
> /map

# Map-Refresh erzwingen
> /map-refresh
```

---

## Git-Integration (Auto-Commits)

Die Git-Integration ist das Herzstück von Aider. Jede KI-Änderung wird automatisch als separater Git-Commit erfasst.

### Funktionsweise

```
> Füge eine Validierung für E-Mail-Adressen in der User-Klasse hinzu

# Aider bearbeitet die Datei(en) und erstellt automatisch einen Commit:
# "feat: add email validation to User class"
```

### Git-Konfiguration

```yaml
# .aider.conf.yml
auto-commits: true          # Automatische Commits (Standard: true)
dirty-commits: true         # Uncommitted Changes erlauben (Standard: true)
auto-lint: true             # Linting vor dem Commit
auto-test: false            # Tests vor dem Commit ausführen
commit-prompt: "Write a concise commit message in conventional commit format."
```

### Änderungen rückgängig machen

```
> /undo
# Macht den letzten Aider-Commit rückgängig (git reset)

> /diff
# Zeigt die Änderungen des letzten Commits
```

> [!IMPORTANT]
> Wenn `dirty-commits: false` gesetzt ist, weigert sich Aider zu starten, wenn es uncommitted Changes im Repository gibt. Dies stellt sicher, dass KI-Änderungen immer sauber von manuellen Änderungen getrennt sind.

### Commit-Nachrichten

Aider generiert automatisch beschreibende Commit-Nachrichten basierend auf den vorgenommenen Änderungen. Das Format kann über `commit-prompt` angepasst werden:

```yaml
# Conventional Commits
commit-prompt: "Write a concise conventional commit message. Use feat:, fix:, refactor:, docs:, test: prefixes."

# Deutsch
commit-prompt: "Schreibe eine kurze, deutsche Commit-Nachricht."
```

---

## Wichtige Slash-Commands

### Übersicht aller Commands

| Command | Beschreibung |
|---|---|
| `/add <datei>` | Datei zum Chat-Kontext hinzufügen |
| `/drop <datei>` | Datei aus dem Kontext entfernen |
| `/ls` | Dateien im Kontext auflisten |
| `/run <befehl>` | Shell-Befehl ausführen und Output anzeigen |
| `/test` | Konfiguriertes Test-Command ausführen |
| `/lint` | Konfiguriertes Lint-Command ausführen |
| `/diff` | Diff des letzten Commits anzeigen |
| `/undo` | Letzten Aider-Commit rückgängig machen |
| `/ask <frage>` | Frage stellen ohne Dateien zu bearbeiten |
| `/architect <prompt>` | Architect-Modus für einen einzelnen Prompt |
| `/map` | Repository-Map anzeigen |
| `/map-refresh` | Repository-Map neu generieren |
| `/tokens` | Token-Verbrauch anzeigen |
| `/model <name>` | LLM-Modell wechseln |
| `/settings` | Aktuelle Einstellungen anzeigen |
| `/clear` | Chat-Verlauf löschen |
| `/help` | Hilfe anzeigen |
| `/quit` | Aider beenden |
| `/voice` | Spracheingabe starten |
| `/paste` | Mehrzeiligen Text einfügen |
| `/web <url>` | Webseite abrufen und zum Kontext hinzufügen |
| `/chat-mode <modus>` | Chat-Modus wechseln (code, ask, architect, help) |
| `/copy` | Letzte Antwort in die Zwischenablage kopieren |
| `/reset` | Alle Dateien droppen und Verlauf löschen |
| `/git <befehl>` | Git-Befehl direkt ausführen |

### Praxisbeispiele

```
> /add src/api/routes.py src/models/user.py
> Füge einen neuen Endpunkt für Passwort-Reset hinzu

> /run pytest tests/test_auth.py -v
> Die Tests für test_reset_password schlagen fehl. Bitte behebe den Fehler.

> /diff
> /undo
> Versuche es nochmal mit einem anderen Ansatz

> /lint
> Bitte behebe die Linting-Fehler

> /ask Erkläre mir, wie die Authentifizierung in diesem Projekt funktioniert
```

---

## LLM-Provider Konfiguration

### Unterstützte Provider

| Provider | Env-Variable | Beispiel-Modell |
|---|---|---|
| OpenAI | `OPENAI_API_KEY` | `gpt-5.2`, `gpt-5.1-codex`, `gpt-5.3-codex` |
| Anthropic | `ANTHROPIC_API_KEY` | `claude-sonnet-4.6`, `claude-opus-4.6` |
| Google Gemini | `GEMINI_API_KEY` | `gemini/gemini-2.5-pro`, `gemini/gemini-3.1-pro` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek/deepseek-chat`, `deepseek/deepseek-r1` |
| Ollama (lokal) | – | `ollama/llama4:scout`, `ollama/deepseek-v3.2` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter/anthropic/claude-sonnet-4.6` |
| Azure OpenAI | `AZURE_API_KEY` | Konfigurationsabhängig |
| AWS Bedrock | AWS-Credentials | `bedrock/anthropic.claude-sonnet-4-20250514-v2:0` |

### API-Keys konfigurieren

```bash
# In der Shell-Konfiguration (~/.bashrc, ~/.zshrc)
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
export GEMINI_API_KEY="AIza..."

# Oder per .env-Datei im Projektroot
echo "OPENAI_API_KEY=sk-..." > .env
```

> [!WARNING]
> API-Keys niemals in die `.aider.conf.yml` oder ins Git-Repository schreiben. Verwende Umgebungsvariablen oder eine `.env`-Datei, die in `.gitignore` eingetragen ist.

### Lokale Modelle mit Ollama

```bash
# Ollama installieren und Modell herunterladen
ollama pull llama3.1

# Aider mit Ollama starten
aider --model ollama/llama3.1

# Oder in der Konfiguration
# model: ollama/llama3.1
```

### Modell während der Sitzung wechseln

```
> /model claude-sonnet-4-20250514
Switched to claude-sonnet-4-20250514
```

---

## Vergleich mit anderen CLI Agents

| Feature | Aider | Claude Code | Codex CLI | GitHub Copilot CLI |
|---|---|---|---|---|
| Git-Integration | ✅ Nativ (Auto-Commits) | ⚠️ Manuell | ⚠️ Manuell | ⚠️ Manuell |
| Installation | pip/pipx | npm | npm | VS Code Extension |
| Sandbox | ❌ | ✅ | ✅ | ❌ |
| MCP-Server | ❌ | ✅ | ✅ | ✅ |
| Hook-System | ❌ | ✅ | ❌ | ❌ |
| Lokale LLMs | ✅ (Ollama etc.) | ❌ | ❌ | ❌ |
| Multi-Provider | ✅ | ❌ (nur Anthropic) | ❌ (nur OpenAI) | ✅ |
| Architect-Modus | ✅ | ❌ | ❌ | ❌ |
| Watch-Modus | ✅ | ❌ | ❌ | ❌ |
| Spracheingabe | ✅ | ❌ | ❌ | ❌ |
| Bildeingabe | ✅ | ✅ | ❌ | ✅ |
| AGENTS.md | ✅ | ✅ (CLAUDE.md) | ✅ | ✅ |
| Gewicht | Leicht (~pip) | Mittel (~npm) | Mittel (~npm) | Schwer (VS Code) |

> [!NOTE]
> Aider ist die beste Wahl für Entwickler, die einen leichtgewichtigen, flexiblen Agent suchen, der mit verschiedenen LLM-Providern arbeitet und Git-native ist. Für Sandbox-Sicherheit und MCP-Integration sind Claude Code oder Codex CLI besser geeignet.

---

## Tipps und Best Practices

### Effektiver Kontext

1. **Nur relevante Dateien hinzufügen** – Weniger ist mehr. Füge nur die Dateien hinzu, die für die aktuelle Aufgabe relevant sind
2. **`/add-read` für Referenz-Dateien** – Dateien, die nur als Kontext dienen (z.B. Interfaces, Types), als read-only hinzufügen
3. **Repository-Map nutzen** – Bei großen Projekten hilft die Map der KI, Zusammenhänge zu verstehen

### Token sparen

```yaml
# Kompaktes Map-Format
map-tokens: 512

# Irrelevante Dateien ausschließen
# In .aiderignore:
# node_modules/
# dist/
# *.lock
```

### Git-Workflow

1. **Feature-Branch erstellen** vor dem Aider-Start
2. **`/undo`** sofort nutzen, wenn eine Änderung nicht passt
3. **`/diff`** regelmäßig prüfen, um Änderungen nachzuvollziehen
4. **`auto-lint: true`** aktivieren, um saubere Commits zu gewährleisten

### Prompting-Tipps

- **Spezifisch sein** – „Füge Fehlerbehandlung für den DB-Zugriff in `get_user()` hinzu" statt „Verbessere den Code"
- **Kontext geben** – Relevante Dateien vorher mit `/add` hinzufügen
- **Inkrementell arbeiten** – Kleine, gezielte Änderungen statt großer Refactorings
- **`/ask` vor `/code`** – Erst die KI fragen, dann Code generieren lassen
- **Tests einbeziehen** – „Implementiere Feature X und schreibe Tests dafür"

### Empfohlene Projektkonfiguration

```yaml
# .aider.conf.yml – Empfohlene Standardkonfiguration
model: claude-sonnet-4-20250514
edit-format: diff
architect: false
auto-commits: true
dirty-commits: true
auto-lint: true
auto-test: false
dark-mode: true
map-tokens: 1024
lint-cmd: "ruff check --fix"
test-cmd: "pytest -x"
```

### Shell-Aliase

```bash
# In ~/.bashrc oder ~/.zshrc

# Standard-Start
alias ai="aider"

# Mit spezifischem Modell
alias ai-claude="aider --model claude-sonnet-4-20250514"
alias ai-gpt="aider --model gpt-4o"
alias ai-local="aider --model ollama/llama3.1"

# Architect-Modus
alias ai-arch="aider --architect --model claude-sonnet-4-20250514"

# Nur fragen, nicht editieren
alias ai-ask="aider --chat-mode ask"
```

### Browser-Integration

Aider bietet eine experimentelle Browser-Integration, um Code direkt aus dem Browser-Kontext zu bearbeiten:

```bash
# Browser-Modus starten
aider --browser
```

Dies öffnet ein Web-Interface, das die gleiche Funktionalität wie das Terminal bietet, aber mit einer grafischen Oberfläche.

---

## Weiterführende Links

- [[Custom Instructions für AI Coding Agents]] – Best Practices für CONVENTIONS.md und AGENTS.md
- [[Custom Instructions für AI Coding Agents]] – CLAUDE.md als Alternative/Ergänzung zu CONVENTIONS.md
- [[Claude Code Konfiguration und Hooks]] – Claude Code Konfiguration zum Vergleich
- [[OpenAI Codex CLI Konfiguration]] – Codex CLI Konfiguration zum Vergleich
- [[GitHub Copilot CLI Konfiguration]] – Copilot CLI Konfiguration zum Vergleich
- [[Multi-Agent Workflows]] – Parallele Agent-Ausführung
- [[Was ist ein AI Agent]] – Grundlagen zu AI Agents
