---
tags: [ai, agents, cli, open-interpreter, konfiguration]
---

# Open Interpreter Konfiguration

## Was ist Open Interpreter?

Open Interpreter ist ein **local-first AI Code-Ausführungs-Agent** (~62K GitHub Stars), der LLMs befähigt, Code direkt auf deinem Rechner auszuführen. Im Gegensatz zu reinen Coding-Assistenten, die nur Dateien editieren, kann Open Interpreter **beliebigen Code in Python, JavaScript, Shell, R und weiteren Sprachen** tatsächlich ausführen.

| Eigenschaft | Details |
|---|---|
| GitHub | [open-interpreter/open-interpreter](https://github.com/open-interpreter/open-interpreter) |
| Lizenz | AGPL-3.0 |
| Sprache | Python |
| Stars | ~62.000 |
| Fokus | Lokale Code-Ausführung, Privacy-first |
| Philosophie | ChatGPT Code Interpreter – aber lokal und ohne Einschränkungen |

> [!TIP]
> Open Interpreter ist kein reiner Code-Editor oder Chat-Bot. Er kann Dateien erstellen, Daten analysieren, APIs aufrufen, Bilder generieren, GUIs automatisieren und Systeme administrieren – alles durch tatsächliche Code-Ausführung auf deinem Rechner.

### Kernfähigkeiten im Überblick

- **Direkte Code-Ausführung** – Python, JavaScript, Shell, R, AppleScript, PowerShell
- **Datei-Manipulation** – Erstellen, Lesen, Bearbeiten, Konvertieren
- **Datenanalyse & Visualisierung** – Pandas, Matplotlib, Plotly direkt ausführbar
- **Web-Browsing** – Webseiten abrufen und parsen
- **Bildgenerierung & Analyse** – Mit DALL-E oder lokalen Modellen
- **System-Administration** – Pakete installieren, Services verwalten, Logs analysieren
- **Computer Use (GUI)** – Maus und Tastatur automatisieren
- **Offline-Modus** – Vollständig lokal mit lokalen LLMs

---

## Installation

### Via pip (empfohlen)

```bash
pip install open-interpreter
```

### Via pipx (isoliert)

```bash
pipx install open-interpreter
```

### Via Docker

```bash
docker build -t open-interpreter .
docker run -it --rm open-interpreter
```

### Starten

```bash
interpreter                    # Interaktiver Modus
interpreter --model gpt-4o     # Mit spezifischem Modell
interpreter -e "Erstelle ein Balkendiagramm der Top-10 Länder nach BIP"
```

> [!NOTE]
> Beim ersten Start fragt Open Interpreter nach deinem API-Key (standardmäßig OpenAI). Du kannst das mit Umgebungsvariablen oder Profilen vorkonfigurieren.

---

## Konfiguration und Profile

Open Interpreter nutzt YAML-basierte Profile im Verzeichnis `~/.openinterpreter/profiles/`. Profile ermöglichen wiederverwendbare Konfigurationen für verschiedene Anwendungsfälle.

### Profilverzeichnis

```bash
mkdir -p ~/.openinterpreter/profiles
```

### Beispiel-Profil: `default.yaml`

```yaml
llm:
  model: "gpt-4o"
  temperature: 0.0
  api_key: "${OPENAI_API_KEY}"
  max_tokens: 4096
  context_window: 128000

auto_run: false
safe_mode: "off"                     # off, ask, auto
conversation_history: true
max_output: 2000

system_message: |
  Du bist ein hilfreicher Coding-Assistent.
  Antworte auf Deutsch.
  Bevorzuge Python für Datenanalyse.
```

### Profil laden

```bash
interpreter --profile data_science.yaml    # Spezifisches Profil
interpreter                                # Standard-Profil automatisch
```

### Weitere Profile

````yaml
# ~/.openinterpreter/profiles/data_science.yaml
llm:
  model: "gpt-4o"
  temperature: 0.0

auto_run: true
system_message: |
  Du bist ein Data-Science-Experte.
  Nutze pandas, matplotlib und scikit-learn.
  Erstelle immer Visualisierungen wenn möglich.
  Speichere Ergebnisse als CSV oder PNG.
````

````yaml
# ~/.openinterpreter/profiles/sysadmin.yaml
llm:
  model: "gpt-4o"
  temperature: 0.0

safe_mode: "ask"
system_message: |
  Du bist ein Linux-Systemadministrator.
  Erkläre jeden Befehl vor der Ausführung.
  Erstelle Backups vor kritischen Änderungen.
  Nutze bevorzugt Shell-Befehle.
````

---

## LLM-Provider Setup

Open Interpreter unterstützt sowohl Cloud-APIs als auch lokale Modelle. Die Konfiguration erfolgt über CLI-Flags, Umgebungsvariablen oder Profile.

### Cloud-Provider

| Provider | Modell-Beispiel | Umgebungsvariable |
|---|---|---|
| OpenAI | `gpt-4o`, `gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |
| Google | `gemini-1.5-pro` | `GOOGLE_API_KEY` |
| Azure OpenAI | `azure/gpt-4o` | `AZURE_API_KEY` |
| Mistral | `mistral-large-latest` | `MISTRAL_API_KEY` |

```bash
export OPENAI_API_KEY="sk-..."
interpreter --model gpt-4o

export ANTHROPIC_API_KEY="sk-ant-..."
interpreter --model claude-sonnet-4-20250514
```

### Lokale Modelle (Offline-Modus)

```bash
# Ollama (empfohlen)
interpreter --model ollama/llama3.1
interpreter --model ollama/codellama
interpreter --model ollama/deepseek-coder-v2

# LM Studio / OpenAI-kompatible API
interpreter --api_base http://localhost:1234/v1 --model local-model
```

> [!IMPORTANT]
> Lokale Modelle sind ideal für **datenschutzsensible Anwendungen**. Kein Code, keine Daten und keine Prompts verlassen deinen Rechner. Die Code-Qualität hängt jedoch stark vom gewählten Modell ab – größere Modelle (70B+) liefern deutlich bessere Ergebnisse.

### Profil für lokale Nutzung

````yaml
# ~/.openinterpreter/profiles/local.yaml
llm:
  model: "ollama/llama3.1"
  api_base: "http://localhost:11434"
  temperature: 0.0
  context_window: 8192

auto_run: false
system_message: |
  Du bist ein hilfreicher lokaler Coding-Assistent.
  Halte deine Antworten kompakt.
  Bevorzuge einfache, gut lesbare Lösungen.
````

---

## Code-Ausführung

### Unterstützte Sprachen

Open Interpreter kann Code in verschiedenen Sprachen direkt ausführen:

| Sprache | Anwendung | Voraussetzung |
|---|---|---|
| Python | Datenanalyse, Scripting, ML | Python installiert |
| JavaScript | Web-Scripting, APIs | Node.js installiert |
| Shell/Bash | Systemverwaltung, Dateien | Standard auf Linux/macOS |
| R | Statistik, Visualisierung | R installiert |
| AppleScript | macOS-Automation | Nur macOS |
| PowerShell | Windows-Automation | Nur Windows |
| HTML | Web-Previews | Browser verfügbar |

### Ausführungsablauf

```
Nutzer-Prompt → LLM generiert Code → Anzeige des Codes → Bestätigung → Ausführung → Output
```

> [!WARNING]
> Standardmäßig fragt Open Interpreter **vor jeder Code-Ausführung** um Bestätigung. Mit `--auto_run` oder `auto_run: true` wird Code **ohne Rückfrage** ausgeführt. Nutze dies nur in kontrollierten Umgebungen!

### Sandbox-Modus (Docker)

```bash
# Code in Docker-Sandbox ausführen
interpreter --sandbox docker
```

---

## Anwendungsfälle

### Data Science & Analyse

```
>>> Lade verkaufsdaten.csv, erstelle eine Zusammenfassung 
    und ein Balkendiagramm der Top-5 Produkte nach Umsatz
```

### Automation & Scripting

```
>>> Finde alle JPG-Dateien in ~/Bilder größer als 5MB,
    komprimiere sie auf 80% Qualität und sichere die Originale
```

### System-Administration

```
>>> Zeige Top-10 Prozesse nach CPU, prüfe Festplattenspeicher 
    und erstelle einen System-Health-Report als HTML
```

### Datei-Konvertierung

```
>>> Konvertiere alle Markdown-Dateien im Verzeichnis in PDFs
```

---

## Sicherheit

### Safe Mode

Open Interpreter bietet verschiedene Sicherheitsstufen:

| Modus | Flag | Verhalten |
|---|---|---|
| Standard | (ohne Flag) | Bestätigung vor jeder Ausführung |
| Safe Mode | `--safe_mode ask` | Zusätzliche Sicherheitsprüfungen + Bestätigung |
| Auto Safe | `--safe_mode auto` | Automatische Prüfung, blockiert gefährliche Befehle |
| Auto Run | `--auto_run` | Keine Bestätigung (⚠️ Vorsicht!) |

```bash
# Maximale Sicherheit
interpreter --safe_mode ask

# Automatische Sicherheitsprüfung
interpreter --safe_mode auto

# Keine Bestätigung (nur in sicheren Umgebungen!)
interpreter --auto_run
```

### Sicherheits-Best-Practices

> [!CAUTION]
> Open Interpreter kann **jeden Befehl auf deinem System ausführen** – einschließlich destruktiver Operationen. Beachte folgende Regeln:

1. **Niemals `--auto_run` auf Produktionssystemen**
2. **Docker-Sandbox** für experimentelle Aufgaben
3. **Code lesen** bevor du die Ausführung bestätigst
4. **Lokale Modelle** für sensible Daten
5. **Nicht als root/admin** ausführen
6. **System-Message** nutzen, um gefährliche Operationen zu verbieten

````yaml
# Sicherheits-Profil
system_message: |
  NIEMALS folgende Aktionen ausführen:
  - rm -rf auf wichtige Verzeichnisse
  - Systemdateien modifizieren (/etc, /sys, Registry)
  - Netzwerk-Ports öffnen oder Firewall-Regeln ändern
  - Passwörter oder Secrets in Code ausgeben
  Erstelle IMMER Backups vor destruktiven Operationen.
safe_mode: "ask"
auto_run: false
````

---

## Computer Use (GUI Automation)

Open Interpreter unterstützt **Computer Use** – die Fähigkeit, den Desktop zu sehen und GUI-Elemente mit Maus und Tastatur zu steuern.

```bash
# Computer Use Modus starten
interpreter --os
```

### Fähigkeiten

- **Mausbewegung und Klicks** – Buttons, Links, Menüs bedienen
- **Tastatureingaben** – Text tippen, Shortcuts ausführen
- **Screenshot-Analyse** – Bildschirminhalt verstehen und darauf reagieren
- **Anwendungssteuerung** – Programme öffnen, zwischen Fenstern wechseln

> [!NOTE]
> Computer Use erfordert ein Vision-fähiges Modell (z.B. GPT-5.2 oder Claude Sonnet 4.6 mit Vision). Die Genauigkeit hängt stark von der Bildschirmauflösung und dem Modell ab.

---

## API Server Modus

Open Interpreter kann als HTTP-Server gestartet werden und bietet eine REST-API für programmatische Integration.

### Server starten

```bash
interpreter --server               # Standard: Port 8000
interpreter --server --port 3000   # Spezifischer Port
```

### API-Endpunkte

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Erstelle eine Fibonacci-Funktion"}'

curl http://localhost:8000/history         # History abrufen
curl -X POST http://localhost:8000/reset   # Neue Konversation
```

### Integration in eigene Anwendungen

```python
import requests

def ask_interpreter(prompt: str) -> str:
    response = requests.post(
        "http://localhost:8000/chat",
        json={"message": prompt}
    )
    return response.json()["response"]
```

---

## Vergleich: Open Interpreter vs. CLI Coding Agents

| Kriterium | Open Interpreter | Claude Code / Copilot CLI | Cursor / Codex CLI |
|---|---|---|---|
| **Kernfokus** | Code-Ausführung | Code-Editing | Code-Editing |
| **Code ausführen** | ✅ Direkt lokal | ⚠️ Eingeschränkt (Shell) | ⚠️ Sandbox |
| **Sprachen** | Python, JS, Shell, R, ... | Alle (Edit-basiert) | Alle (Edit-basiert) |
| **Offline-fähig** | ✅ Lokale LLMs | ❌ | ❌ |
| **LLM-Auswahl** | Frei wählbar | Claude only | Provider-abhängig |
| **Datenanalyse** | ✅ Exzellent | ⚠️ Umweg über Shell | ⚠️ Umweg über Shell |
| **GUI Automation** | ✅ Computer Use | ❌ | ❌ |
| **Git-Integration** | ⚠️ Via Shell | ✅ Nativ | ✅ Nativ |
| **IDE-Integration** | ❌ | ⚠️ Terminal | ✅ Nativ |
| **Sicherheit** | ⚠️ Manuell/Safe Mode | ✅ Permissions | ✅ Sandbox |
| **Preis** | Eigene API-Keys | Abo + API | Abo + API |

> [!TIP]
> **Open Interpreter** eignet sich besonders für **Datenanalyse, Automation und explorative Aufgaben**, wo tatsächliche Code-Ausführung und iteratives Arbeiten im Vordergrund stehen. **CLI Coding Agents** wie Claude Code oder Copilot CLI sind besser für **Software-Entwicklung**, wo es um Code-Qualität, Refactoring und Git-Workflows geht.

---

## Tipps und Best Practices

### Effektive Prompts

```
# Schlecht – zu vage
>>> Mach was mit den Daten

# Gut – spezifisch mit gewünschtem Output
>>> Lade verkauf.csv mit pandas, berechne den monatlichen Umsatz
    pro Kategorie, erstelle ein Balkendiagramm und speichere als PNG (300 DPI).
```

### Nützliche CLI-Flags

| Flag | Beschreibung |
|---|---|
| `--model` | LLM-Modell auswählen |
| `--auto_run` | Code ohne Bestätigung ausführen |
| `--safe_mode` | Sicherheitsmodus aktivieren |
| `--profile` | Konfigurationsprofil laden |
| `--os` | Computer Use Modus |
| `--server` | API-Server starten |
| `-e "..."` | Einzeiler ausführen |
| `--verbose` | Debug-Ausgabe aktivieren |
| `--vision` | Bildanalyse aktivieren |
| `--sandbox docker` | Code in Docker isolieren |

### Workflow-Empfehlungen

1. **Profile für Szenarien** – Data Science, SysAdmin, Web-Dev als separate Profile
2. **Safe Mode als Standard** – `safe_mode: "ask"` im Default-Profil
3. **System-Messages nutzen** – Sprache, Stil und Einschränkungen vorgeben
4. **Lokale Modelle testen** – Für Datenschutz und Offline-Nutzung
5. **Docker-Sandbox** – Für experimentelle Aufgaben
6. **Ergebnisse prüfen** – Output und generierte Dateien immer kontrollieren

### Umgebungsvariablen

```bash
# In ~/.bashrc oder ~/.zshrc
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
export INTERPRETER_MODEL="gpt-4o"
export INTERPRETER_AUTO_RUN="false"
export INTERPRETER_SAFE_MODE="ask"
```

### Python-SDK Nutzung

```python
from interpreter import interpreter

interpreter.llm.model = "gpt-4o"
interpreter.llm.temperature = 0.0
interpreter.auto_run = False

# Einzelne Anfrage
interpreter.chat("Erstelle einen Scatter-Plot aus data.csv")

# Iterativ mit History
interpreter.chat("Lade die Datei nutzer.csv")
interpreter.chat("Zeige die ersten 10 Zeilen")
interpreter.chat("Erstelle eine Altersverteilung als Histogramm")

# History zurücksetzen
interpreter.messages = []
```

> [!IMPORTANT]
> Open Interpreter ist ein **mächtiges Werkzeug**, das bei verantwortungsvollem Einsatz enorm produktiv ist. Behandle es wie einen Junior-Entwickler mit Root-Zugang: Prüfe den Code, bevor du ihn ausführen lässt, und nutze Sicherheitsmechanismen konsequent.

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Übersicht und Vergleich aller AI Coding Agents
- [[Was ist ein AI Agent]] — Grundlagen und Definition von AI Agents
- [[Sandboxing und Code Execution]] — Sicherheitsmechanismen für Code-Ausführung durch AI Agents
- [[Function Calling und Tool Use]] — Wie LLMs externe Tools und Funktionen aufrufen
