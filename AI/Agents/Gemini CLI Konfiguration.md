# Gemini CLI — Konfiguration und Nutzung

> [!NOTE] Übersicht
> **Gemini CLI** ist Googles Open-Source AI-Coding-Agent für das Terminal. Er bringt das Gemini-Modell direkt in die Kommandozeile und kann Dateien lesen, bearbeiten, Shell-Befehle ausführen und komplexe Coding-Aufgaben autonom lösen — ähnlich wie Claude Code oder GitHub Copilot CLI.

---

## 1. Was ist Gemini CLI?

Gemini CLI ist ein **agentic coding tool**, das auf Googles Gemini-2.5-Pro-Modell basiert. Es läuft vollständig im Terminal und bietet:

- **Codeverständnis** — analysiert ganze Repositories und beantwortet Fragen zum Code
- **Code-Generierung** — schreibt, refactored und debuggt Code
- **Shell-Integration** — führt Befehle aus und verarbeitet deren Output
- **Multimodale Eingabe** — versteht Text, Code und Bilder
- **Tool Use** — nutzt eingebaute und externe Tools (MCP)
- **Sandboxing** — kann in Docker/Podman-Containern isoliert arbeiten

> [!IMPORTANT] Kostenlos nutzbar
> Gemini CLI bietet mit einem persönlichen Google-Account **1.000 Requests pro Tag** kostenlos über die Gemini API. Für höhere Limits kann ein eigener API-Key oder ein Vertex AI-Zugang verwendet werden.

---

## 2. Installation

### Voraussetzungen

| Anforderung   | Details                  |
| ------------- | ------------------------ |
| **Node.js**   | Version 20 oder höher    |
| **npm**       | Kommt mit Node.js        |
| **Betriebssystem** | Linux, macOS, Windows |

### Globale Installation (empfohlen)

```bash
npm install -g @google/gemini-cli
```

### Ohne Installation (via npx)

```bash
npx @google/gemini-cli
```

### Starten

```bash
gemini
```

> [!TIP] Schnellstart
> Nach dem ersten Start öffnet sich automatisch der Browser zur Authentifizierung. Danach ist Gemini CLI sofort einsatzbereit.

### Update

```bash
npm update -g @google/gemini-cli
```

---

## 3. Authentifizierung

Gemini CLI unterstützt mehrere Authentifizierungsmethoden:

### 3.1 Google Login (Standard)

Beim ersten Start öffnet sich der Browser für eine OAuth-Anmeldung mit dem Google-Account. Das ist die einfachste Methode und bietet das kostenlose Kontingent von 1.000 Requests/Tag.

```bash
gemini
# → Browser öffnet sich automatisch zur Anmeldung
```

### 3.2 API Key

Für höhere Limits oder automatisierte Nutzung kann ein API-Key von [Google AI Studio](https://aistudio.google.com/apikey) verwendet werden:

**Per Umgebungsvariable:**

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
gemini
```

**Per `.env`-Datei (User-Level):**

```bash
# ~/.gemini/.env
GEMINI_API_KEY=YOUR_API_KEY
```

**Per `.env`-Datei (Projekt-Level):**

```bash
# <projekt-root>/.gemini/.env
GEMINI_API_KEY=YOUR_API_KEY
```

### 3.3 Vertex AI (Google Cloud)

Für Enterprise-Nutzung mit Google Cloud:

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
gemini
```

> [!WARNING] API-Key Sicherheit
> Niemals API-Keys in Git committen. Die `.env`-Datei unter `.gemini/` sollte in `.gitignore` eingetragen sein. Gemini CLI fügt automatisch eine `.gitignore` in `.gemini/` hinzu, aber prüfe das sicherheitshalber.

---

## 4. Konfiguration (settings.json)

### 4.1 Konfigurations-Hierarchie

Gemini CLI verwendet ein **dreistufiges Konfigurationssystem** — jede Ebene überschreibt die vorherige:

```
System-Defaults → User-Config → Projekt-Config
                  ~/.gemini/       .gemini/
                  settings.json    settings.json
```

| Ebene       | Pfad                            | Geltungsbereich   |
| ----------- | ------------------------------- | ----------------- |
| **System**  | Eingebaut in Gemini CLI         | Alle Projekte     |
| **User**    | `~/.gemini/settings.json`       | Alle Projekte     |
| **Projekt** | `.gemini/settings.json`         | Nur dieses Projekt |

### 4.2 Wichtige Einstellungen

| Setting            | Typ      | Beschreibung                                                   |
| ------------------ | -------- | -------------------------------------------------------------- |
| `theme`            | string   | Farbschema (`Default`, `Solarized`, etc.)                      |
| `autoAccept`       | boolean  | Dateiänderungen automatisch akzeptieren ohne Bestätigung       |
| `sandbox`          | string   | Sandboxing-Modus: `docker`, `podman` oder `none`               |
| `vimMode`          | boolean  | Vim-Keybindings für die Eingabe aktivieren                     |
| `checkpointing`    | boolean  | Git-basierte Checkpoints bei Dateiänderungen                   |
| `selectedAuthType` | string   | Authentifizierungsmethode (`oauth`, `api-key`, `vertex-ai`)    |
| `fileFiltering`    | object   | Einstellungen für Dateifilter                                  |

### 4.3 Beispiel: settings.json (User-Level)

````json
// ~/.gemini/settings.json
{
  "theme": "Default",
  "autoAccept": false,
  "sandbox": "docker",
  "vimMode": false,
  "checkpointing": true,
  "fileFiltering": {
    "respectGitIgnore": true
  },
  "selectedAuthType": "oauth"
}
````

### 4.4 Beispiel: settings.json (Projekt-Level)

````json
// .gemini/settings.json
{
  "sandbox": "none",
  "autoAccept": true,
  "checkpointing": true
}
````

> [!TIP] Auto-Accept mit Vorsicht
> `autoAccept: true` überspringt die Bestätigungsabfrage bei Dateiänderungen. Nützlich für erfahrene Nutzer, aber in Kombination mit `checkpointing: true` empfohlen, damit Änderungen rückgängig gemacht werden können.

---

## 5. Custom Instructions (GEMINI.md)

### 5.1 GEMINI.md

Ähnlich wie Claude Code (`CLAUDE.md`) unterstützt Gemini CLI eine **GEMINI.md**-Datei, die projektspezifische Anweisungen enthält. Gemini liest diese automatisch beim Start.

**Speicherorte (hierarchisch geladen):**

| Ort                         | Beschreibung                              |
| --------------------------- | ----------------------------------------- |
| `~/.gemini/GEMINI.md`       | Globale Anweisungen (alle Projekte)       |
| `<git-root>/GEMINI.md`      | Repository-weite Anweisungen              |
| `<cwd>/GEMINI.md`           | Verzeichnis-spezifische Anweisungen       |

### 5.2 Beispiel GEMINI.md

````markdown
# Projekt-Konventionen

## Code-Style
- Verwende TypeScript mit striktem Modus
- Schreibe alle Kommentare auf Englisch
- Nutze ESLint und Prettier

## Testing
- Schreibe Unit-Tests mit Vitest
- Teste alle öffentlichen Funktionen
- Mindestens 80% Code-Coverage anstreben

## Git
- Conventional Commits verwenden (feat:, fix:, docs:)
- Immer in Feature-Branches arbeiten

## Architektur
- Services im `src/services/` Ordner
- Typen in `src/types/` definieren
````

### 5.3 AGENTS.md (Cross-Tool Standard)

Gemini CLI liest zusätzlich **AGENTS.md**-Dateien — ein aufkommender Standard, der von mehreren AI-Coding-Tools unterstützt wird. So können Anweisungen tool-übergreifend geteilt werden:

```bash
# Wird von Gemini CLI, Claude Code und anderen gelesen
<git-root>/AGENTS.md
```

> [!NOTE] Hierarchisches Laden
> Wenn sowohl `GEMINI.md` als auch `AGENTS.md` existieren, werden beide geladen. `GEMINI.md` hat bei Konflikten Vorrang, da es Gemini-spezifisch ist.

---

## 6. Modi und Sandbox

### 6.1 Interaktiver Modus (Standard)

```bash
gemini
```

Der Standard-Modus: Man chattet direkt mit Gemini und bestätigt Aktionen interaktiv.

### 6.2 Non-Interaktiver Modus

Für Automatisierung und Scripting:

```bash
echo "Erkläre die main.py Datei" | gemini
```

Oder mit dem `-p` Flag:

```bash
gemini -p "Fasse die README.md zusammen"
```

### 6.3 Sandbox (Docker/Podman)

Gemini CLI kann Shell-Befehle in einem isolierten Container ausführen. Das schützt das Host-System vor unbeabsichtigten Änderungen.

**In settings.json aktivieren:**

```json
{
  "sandbox": "docker"
}
```

**Unterstützte Sandbox-Typen:**

| Typ        | Beschreibung                          |
| ---------- | ------------------------------------- |
| `none`     | Keine Isolation (Standard)            |
| `docker`   | Ausführung in Docker-Container        |
| `podman`   | Ausführung in Podman-Container        |

> [!WARNING] Sandbox-Einschränkungen
> Im Sandbox-Modus hat Gemini CLI eingeschränkten Zugriff auf das Dateisystem. Nur das aktuelle Projektverzeichnis wird in den Container gemountet.

---

## 7. Skills und Erweiterungen

### 7.1 Skills-System

Gemini CLI verfügt über ein **Skills-System**, mit dem die Fähigkeiten des Agents erweitert werden können. Skills sind vorgefertigte Pakete mit Instruktionen und Tools.

**Verfügbare Skills anzeigen:**

```bash
gemini skills list
```

**Skill hinzufügen:**

```bash
npx skills add <skill-name>
```

**Skill entfernen:**

```bash
npx skills remove <skill-name>
```

### 7.2 Custom Themes

Gemini CLI unterstützt benutzerdefinierte Themes:

```json
{
  "theme": "Solarized"
}
```

Verfügbare eingebaute Themes können über die Settings oder den `/theme`-Slash-Command gewechselt werden.

### 7.3 Extensions und Plugins

Das Extension-System ermöglicht es, eigene Tools und Integrationen hinzuzufügen. Extensions werden im `.gemini/`-Verzeichnis konfiguriert.

---

## 8. MCP Server Konfiguration

### 8.1 Was ist MCP?

Das **Model Context Protocol (MCP)** erlaubt es, Gemini CLI mit externen Datenquellen und Tools zu verbinden — z.B. Datenbanken, APIs oder Dateisysteme.

### 8.2 Konfiguration

MCP-Server werden in der `settings.json` konfiguriert:

**User-Level** (`~/.gemini/settings.json`):

````json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
    }
  }
}
````

**Projekt-Level** (`.gemini/settings.json`):

````json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/mydb"
      }
    }
  }
}
````

### 8.3 MCP-Server verwalten

```bash
# Verbundene MCP-Server anzeigen
/tools

# MCP-Server Status prüfen
/mcp
```

> [!TIP] MCP-Server teilen
> Projekt-Level MCP-Server in `.gemini/settings.json` können ins Repository committed werden, damit das ganze Team dieselben Tools nutzt. Sensible Daten wie Tokens gehören dabei in Umgebungsvariablen, nicht in die Config.

---

## 9. Wichtige Slash-Commands

### 9.1 Command-Übersicht

| Command       | Beschreibung                                              |
| ------------- | --------------------------------------------------------- |
| `/help`       | Hilfe und verfügbare Commands anzeigen                    |
| `/tools`      | Liste aller verfügbaren Tools (inkl. MCP)                 |
| `/stats`      | Token-Verbrauch und Session-Statistiken                   |
| `/memory`     | Gespeicherte Erinnerungen verwalten                       |
| `/mcp`        | MCP-Server Status und Verwaltung                          |
| `/theme`      | Theme wechseln                                            |
| `/chat`       | Einen neuen Chat starten (Kontext zurücksetzen)            |
| `/restore`    | Einen vorherigen Checkpoint wiederherstellen               |
| `/quit`       | Gemini CLI beenden                                         |
| `@<datei>`    | Eine Datei explizit in den Kontext laden                   |

### 9.2 Memory-System

Mit `/memory` kann Gemini CLI sich Dinge über das Projekt merken, die über Sessions hinweg bestehen bleiben:

```
/memory add Dieses Projekt verwendet pnpm statt npm
/memory list
/memory clear
```

Memories werden in `~/.gemini/memory.md` oder `.gemini/memory.md` gespeichert.

---

## 10. Keyboard Shortcuts

| Shortcut            | Aktion                                      |
| ------------------- | ------------------------------------------- |
| `Enter`             | Nachricht senden                            |
| `Shift + Enter`     | Neue Zeile (mehrzeiliger Input)             |
| `Tab`               | Autocomplete für Dateipfade                 |
| `Ctrl + C`          | Aktuelle Generierung abbrechen              |
| `Ctrl + D`          | Gemini CLI beenden                          |
| `Esc`               | Aktuelle Aktion abbrechen / Tool-Bestätigung ablehnen |
| `↑` / `↓`           | Durch vorherige Eingaben navigieren         |

> [!NOTE] Vim-Modus
> Mit `"vimMode": true` in der `settings.json` stehen Vim-Keybindings für die Eingabezeile zur Verfügung (Normal/Insert Mode, Bewegungsbefehle etc.).

---

## 11. Vergleich: Gemini CLI vs. Claude Code vs. Copilot CLI

### 11.1 Feature-Vergleich

| Feature                     | Gemini CLI                | Claude Code               | GitHub Copilot CLI          |
| --------------------------- | ------------------------- | ------------------------- | --------------------------- |
| **Hersteller**              | Google                    | Anthropic                 | GitHub / Microsoft          |
| **Modell**                  | Gemini 2.5 Pro            | Claude Sonnet/Opus        | GPT-5.2 / Claude 4.6 / Gemini   |
| **Open Source**             | ✅ Ja (Apache 2.0)        | ❌ Nein (Proprietary)      | ❌ Nein (Proprietary)       |
| **Kostenlos nutzbar**       | ✅ 1.000 Req/Tag           | ❌ Bezahlt (ab ~$20/Mo)   | ❌ Bezahlt (Copilot Sub)    |
| **Installation**            | npm                       | npm                       | GitHub CLI Extension        |
| **Custom Instructions**     | GEMINI.md + AGENTS.md     | CLAUDE.md + AGENTS.md     | copilot-instructions.md    |
| **MCP Support**             | ✅ Ja                      | ✅ Ja                      | ✅ Ja                       |
| **Sandbox**                 | Docker / Podman           | Kein eingebautes          | Kein eingebautes            |
| **Multimodal**              | ✅ Text + Bilder           | ✅ Text + Bilder           | ✅ Text                     |
| **Vim-Modus**               | ✅ Ja                      | ✅ Ja                      | ❌ Nein                     |
| **Git-Checkpointing**       | ✅ Ja                      | ✅ Ja                      | ❌ Nein                     |
| **Skills/Plugins**          | ✅ Skills-System           | ❌ Nein                    | ❌ Nein                     |
| **Offline-Fähig**           | ❌ Nein                    | ❌ Nein                    | ❌ Nein                     |

### 11.2 Wann welches Tool?

| Szenario                              | Empfehlung         |
| ------------------------------------- | ------------------ |
| Kostenlose Nutzung gewünscht          | **Gemini CLI**     |
| Bestes Reasoning für komplexe Tasks   | **Claude Code**    |
| Bereits GitHub Copilot Subscriber     | **Copilot CLI**    |
| Open-Source bevorzugt                 | **Gemini CLI**     |
| Enterprise / Google Cloud Umgebung    | **Gemini CLI**     |
| Multi-Modell-Flexibilität            | **Copilot CLI**    |
| Sandboxed Execution nötig             | **Gemini CLI**     |

---

## 12. Tipps und Troubleshooting

### 12.1 Best Practices

> [!TIP] Effektive Nutzung
> - **GEMINI.md pflegen**: Je besser die Projekt-Instruktionen, desto bessere Ergebnisse
> - **Checkpointing aktivieren**: Ermöglicht einfaches Zurückrollen bei Fehlern
> - **Kontext klein halten**: Arbeite in Unterverzeichnissen für fokussiertere Ergebnisse
> - **`@`-Mentions nutzen**: Lade relevante Dateien explizit mit `@datei.ts` in den Kontext
> - **Skills erkunden**: Prüfe regelmäßig verfügbare Skills mit `gemini skills list`

### 12.2 Häufige Probleme

#### Authentifizierung fehlgeschlagen

```bash
# OAuth-Token zurücksetzen
rm -rf ~/.gemini/auth/
gemini
# → Erneute Anmeldung im Browser
```

#### Node.js Version zu alt

```bash
node --version
# Falls < 20: Node.js aktualisieren
nvm install 20
nvm use 20
```

#### Rate Limit erreicht

> [!WARNING] Rate Limits
> Mit Google Login: **1.000 Requests/Tag**. Bei Überschreitung wird ein Fehler angezeigt. Alternativen:
> - Auf den nächsten Tag warten
> - Eigenen API-Key mit höherem Kontingent verwenden
> - Auf Vertex AI mit Pay-per-Use umsteigen

#### Sandbox startet nicht

```bash
# Docker prüfen
docker info

# Oder auf podman wechseln
# In settings.json: "sandbox": "podman"
```

#### MCP-Server verbindet nicht

```bash
# MCP-Server manuell testen
npx -y @modelcontextprotocol/server-github

# Logs prüfen — Gemini CLI loggt MCP-Fehler in die Konsole
# /tools zeigt den Status aller verbundenen Server
```

### 12.3 Nützliche Umgebungsvariablen

| Variable                  | Beschreibung                                  |
| ------------------------- | --------------------------------------------- |
| `GEMINI_API_KEY`          | API-Key für die Gemini API                    |
| `GOOGLE_CLOUD_PROJECT`    | Google Cloud Projekt-ID (für Vertex AI)       |
| `GOOGLE_CLOUD_LOCATION`   | Google Cloud Region (für Vertex AI)           |
| `GEMINI_SANDBOX`          | Sandbox-Modus überschreiben                   |

### 12.4 Konfiguration zurücksetzen

```bash
# Komplette Gemini CLI Konfiguration zurücksetzen
rm -rf ~/.gemini/

# Nur Settings zurücksetzen
rm ~/.gemini/settings.json

# Nur Memories löschen
rm ~/.gemini/memory.md
```

---

## Weiterführende Links

- [Gemini CLI GitHub Repository](https://github.com/google-gemini/gemini-cli)
- [Google AI Studio — API Key erstellen](https://aistudio.google.com/apikey)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io)
- [[Custom Instructions für AI Coding Agents]]
- [[Claude Code Konfiguration und Hooks]]
- [[GitHub Copilot CLI Konfiguration]]
