# MCP Server in Claude Code einbinden

> [!IMPORTANT]
> Claude Code ist ein MCP-Host und unterstützt das Einbinden beliebiger MCP-Server über JSON-Konfigurationsdateien. Eingebundene Server erweitern Claude Codes Fähigkeiten um neue Tools, Resources und Prompts.

## Wo wird MCP konfiguriert?

Claude Code liest MCP-Konfigurationen aus zwei Quellen, in dieser Priorität:

```
Priorität (höher = überschreibt niedrigere):
1. Projekt-spezifisch: .claude/settings.json  (im aktuellen Arbeitsverzeichnis)
2. Global:            ~/.claude/settings.json  (für alle Projekte)
```

> [!TIP]
> Projektspezifische Konfigurationen eignen sich für Server, die nur für ein bestimmtes Projekt relevant sind (z.B. ein Server für eine projektspezifische Datenbank). Globale Konfigurationen eignen sich für generelle Server wie Filesystem oder Brave Search.

### Konfigurationsdateien anlegen

```bash
# Globale Konfiguration (für alle Projekte)
mkdir -p ~/.claude
touch ~/.claude/settings.json

# Projektspezifische Konfiguration
mkdir -p .claude
touch .claude/settings.json
```

---

## JSON-Struktur für MCP-Server Konfiguration

Der grundlegende Aufbau der `settings.json`:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "executable",
      "args": ["arg1", "arg2"],
      "env": {
        "ENV_VAR": "value"
      }
    }
  }
}
```

### Vollständige Feldübersicht

| Feld | Typ | Pflicht | Beschreibung |
|------|-----|---------|-------------|
| `command` | string | Ja | Das ausführbare Programm (z.B. `python`, `node`, `npx`) |
| `args` | string[] | Nein | Kommandozeilenargumente |
| `env` | object | Nein | Umgebungsvariablen für den Server-Prozess |
| `type` | string | Nein | `"stdio"` (Standard), `"streamable-http"` (aktuell) oder `"sse"` (deprecated) |
| `url` | string | Nur bei HTTP | URL des Servers (z.B. `https://example.com/mcp`) |

---

## Lokalen stdio-Server einbinden

### Beispiel: Eigener Python-Server

```json
{
  "mcpServers": {
    "mein-python-server": {
      "command": "python3",
      "args": ["/home/user/mcp-servers/mein-server/server.py"],
      "env": {
        "PYTHONPATH": "/home/user/mcp-servers/mein-server",
        "MCP_LOG_LEVEL": "INFO"
      }
    }
  }
}
```

### Beispiel: Mit uv (empfohlen für Python)

```json
{
  "mcpServers": {
    "mein-python-server": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/home/user/mcp-servers/mein-server",
        "python",
        "server.py"
      ]
    }
  }
}
```

> [!TIP]
> Mit `uv run --directory` wird automatisch das richtige virtuelle Environment des Projekts verwendet. Kein manuelles `source .venv/bin/activate` nötig.

### Beispiel: Node.js / TypeScript Server

```json
{
  "mcpServers": {
    "mein-node-server": {
      "command": "node",
      "args": ["/home/user/mcp-servers/mein-node-server/dist/server.js"]
    }
  }
}
```

### Beispiel: Mit npx (für npm-Pakete)

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/documents",
        "/home/user/projects"
      ]
    }
  }
}
```

> [!NOTE]
> Der `-y`-Flag bei `npx` sorgt dafür, dass das Paket ohne Rückfrage installiert wird, falls es noch nicht vorhanden ist.

---

## Remote HTTP-Server einbinden

### Streamable HTTP (aktueller Standard)

Für Server, die über HTTP erreichbar sind (Streamable HTTP – ein einzelner `/mcp`-Endpunkt):

```json
{
  "mcpServers": {
    "remote-server": {
      "type": "streamable-http",
      "url": "https://mein-mcp-server.example.com/mcp"
    }
  }
}
```

### Mit Authentifizierungs-Header

```json
{
  "mcpServers": {
    "remote-server-auth": {
      "type": "streamable-http",
      "url": "https://api.example.com/mcp",
      "env": {
        "MCP_AUTH_TOKEN": "mein-geheimer-token"
      }
    }
  }
}
```

### Legacy: SSE (deprecated)

> [!WARNING]
> Der SSE-Transport (`"type": "sse"`) wurde in der MCP-Spezifikation vom 2025-03-26 als deprecated markiert und durch Streamable HTTP ersetzt. Bestehende SSE-Konfigurationen funktionieren weiterhin, neue Einrichtungen sollten `"type": "streamable-http"` verwenden.

```json
{
  "mcpServers": {
    "remote-server-legacy": {
      "type": "sse",
      "url": "https://mein-mcp-server.example.com/sse"
    }
  }
}
```

---

## Umgebungsvariablen für MCP-Server setzen

Umgebungsvariablen werden unter `env` konfiguriert und sind **nur für den jeweiligen Server-Prozess** sichtbar:

```json
{
  "mcpServers": {
    "github-server": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxxxxxxxxxx"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "BSA_xxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "postgresql://user:password@localhost:5432/mydb"
      }
    }
  }
}
```

> [!WARNING]
> Speichere **niemals** echte Secrets direkt in `.claude/settings.json`, wenn diese Datei in einem Git-Repository liegt! Nutze stattdessen die globale `~/.claude/settings.json` oder Umgebungsvariablen-Referenzen.

### Sicherer Umgang mit Secrets

**Option 1: Globale settings.json (nicht in Git)**
```bash
# Secrets nur in der globalen Konfiguration speichern
~/.claude/settings.json  # Diese Datei liegt nicht im Repo
```

**Option 2: Aus Shell-Umgebung referenzieren**

Manche MCP-Server lesen Secrets automatisch aus der Shell-Umgebung. Setze die Variable in `~/.bashrc` oder `~/.zshrc`:

```bash
# ~/.bashrc oder ~/.zshrc
export GITHUB_PERSONAL_ACCESS_TOKEN="ghp_xxxx"
export BRAVE_API_KEY="BSA_xxxx"
```

Dann in `settings.json` ohne den Wert:
```json
{
  "mcpServers": {
    "github-server": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

---

## Server testen mit Claude Code

### /mcp Kommando

In Claude Code gibt es einen eingebauten Befehl zum Verwalten von MCP-Servern:

```
/mcp
```

Dieser Befehl zeigt:
- Alle konfigurierten Server und deren Status (verbunden / getrennt / Fehler)
- Verfügbare Tools pro Server
- Verbindungsstatus und ggf. Fehlermeldungen

### Typischer Test-Workflow

```
1. Server-Konfiguration in settings.json eintragen
2. Claude Code neu starten (oder neues Gespräch starten)
3. /mcp eingeben → Server sollte als "verbunden" erscheinen
4. Claude Code direkt nach einem Tool fragen:
   "Lies die Datei /home/user/test.txt"
   (wenn filesystem-Server eingebunden ist)
5. Falls Fehler: Troubleshooting (siehe unten)
```

### Direkte Tool-Nutzung testen

```
# Beispiel-Prompts zum Testen verschiedener Server:

Filesystem:
"Liste alle Python-Dateien in /home/user/projects/myapp"

GitHub:
"Zeige die letzten 5 Issues im Repository octocat/Hello-World"

Brave Search:
"Suche nach aktuellen News über das Model Context Protocol"

PostgreSQL:
"Zeige mir alle Tabellen in der Datenbank"

Eigener Server:
"Ruf das Tool 'systeminfo' auf" (oder einfach nach Systeminformationen fragen)
```

---

## Verfügbare Community-Server einbinden

### Filesystem Server

Erlaubt Zugriff auf das lokale Dateisystem:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/documents",
        "/home/user/projects",
        "/tmp"
      ]
    }
  }
}
```

> [!NOTE]
> Alle Verzeichnisse, auf die Claude Code zugreifen soll, müssen explizit als Argumente übergeben werden. Das ist ein Sicherheitsfeature.

**Verfügbare Tools:**
- `read_file` - Datei lesen
- `write_file` - Datei schreiben
- `create_directory` - Verzeichnis erstellen
- `list_directory` - Verzeichnisinhalt auflisten
- `move_file` - Datei verschieben/umbenennen
- `search_files` - Dateien nach Muster suchen
- `get_file_info` - Datei-Metadaten abrufen

### GitHub Server

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxx"
      }
    }
  }
}
```

**Verfügbare Tools:**
- `create_or_update_file` - Datei erstellen/aktualisieren
- `search_repositories` - Repositories suchen
- `create_repository` - Repository erstellen
- `get_file_contents` - Datei lesen
- `push_files` - Mehrere Dateien pushen
- `create_issue` - Issue erstellen
- `create_pull_request` - Pull Request erstellen
- `fork_repository` - Repository forken
- `create_branch` - Branch erstellen
- `list_commits` - Commits auflisten
- `search_code` - Code suchen
- `search_issues` - Issues suchen

**GitHub Token erstellen:**
1. GitHub → Settings → Developer Settings → Personal Access Tokens
2. "Generate new token (classic)"
3. Scopes: `repo`, `read:org` (je nach Bedarf)

### Brave Search Server

```json
{
  "mcpServers": {
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "BSA_xxxx"
      }
    }
  }
}
```

**Verfügbare Tools:**
- `brave_web_search` - Web-Suche
- `brave_local_search` - Lokale Suche (Orte, Restaurants, etc.)

**API-Key erhalten:** https://api.search.brave.com/

### PostgreSQL Server

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://user:password@localhost:5432/mydb"
      ]
    }
  }
}
```

**Verfügbare Tools:**
- `query` - SQL-Abfragen ausführen (nur SELECT)

**Verfügbare Resources:**
- Alle Tabellen als Resources (Schema und Vorschau)

### Fetch Server (HTTP-Anfragen)

```json
{
  "mcpServers": {
    "fetch": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-fetch"]
    }
  }
}
```

**Verfügbare Tools:**
- `fetch` - Beliebige URLs abrufen (GET), konvertiert HTML zu Markdown

### Memory Server (Persistentes Gedächtnis)

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    }
  }
}
```

**Verfügbare Tools:**
- `create_entities` - Entitäten/Fakten speichern
- `search_nodes` - Gespeicherte Informationen suchen
- `read_graph` - Gesamten Wissensgraphen lesen

---

## Vollständige Beispiel-Konfiguration

Hier eine praxisnahe Konfiguration für Entwickler-Workflows:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/projects",
        "/home/user/documents",
        "/tmp"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxxxxxxxxxx"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "BSA_xxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    },
    "fetch": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-fetch"]
    },
    "postgres-dev": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://devuser:devpassword@localhost:5432/development"
      ]
    },
    "mein-custom-server": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/home/user/mcp-servers/mein-server",
        "python",
        "server.py"
      ],
      "env": {
        "MCP_LOG_LEVEL": "WARNING"
      }
    }
  }
}
```

---

## Troubleshooting

### Problem: Server startet nicht

**Symptom:** `/mcp` zeigt den Server als "disconnected" oder mit Fehler.

**Checkliste:**

```bash
# 1. Ist das Programm installiert und im PATH?
which npx        # Node/npm vorhanden?
which python3    # Python vorhanden?
which uv         # uv vorhanden?

# 2. Funktioniert der Befehl manuell?
npx -y @modelcontextprotocol/server-filesystem /tmp
# → Sollte ohne Fehler starten (Ctrl+C zum Beenden)

# 3. Python-Server direkt testen:
python3 /pfad/zu/server.py
# → Sollte starten ohne sofort zu beenden

# 4. Gibt es Syntax-Fehler in settings.json?
python3 -m json.tool ~/.claude/settings.json
# → Gibt "OK" aus wenn JSON valide ist
```

> [!WARNING]
> Der häufigste Fehler: Der Pfad in `args` ist falsch oder existiert nicht. Immer absolute Pfade verwenden und mit `ls /pfad/zu/server.py` prüfen ob die Datei existiert.

### Problem: Tool wird nicht gefunden

**Symptom:** Claude Code sagt, es kennt ein bestimmtes Tool nicht, obwohl der Server läuft.

```bash
# 1. /mcp aufrufen und prüfen ob Server verbunden ist
# 2. Neues Gespräch starten (MCP-Tools werden beim Start geladen)
# 3. Server-Logs prüfen - oft in ~/.claude/logs/ oder per stderr

# Für Python-Server: Fehler in stderr ausgeben lassen
"env": {
  "MCP_LOG_LEVEL": "DEBUG"
}
```

### Problem: Umgebungsvariable nicht verfügbar

**Symptom:** Server startet, aber gibt Fehler "API key not found" oder ähnliches.

```json
// FALSCH: Variable existiert vielleicht nicht
{
  "env": {
    "API_KEY": "$MY_API_KEY"  // Variable-Expansion funktioniert NICHT so
  }
}

// RICHTIG: Direkter Wert
{
  "env": {
    "API_KEY": "echter-wert-hier"
  }
}
```

> [!NOTE]
> Claude Code expandiert **keine** Shell-Variablen in der `env`-Konfiguration. Der Wert wird literal übergeben. Secrets müssen direkt eingetragen werden (in die globale `~/.claude/settings.json`).

### Problem: Berechtigungsfehler

```bash
# Python-Skript ausführbar machen
chmod +x /pfad/zu/server.py

# Oder explizit Python aufrufen (empfohlen)
"command": "python3",
"args": ["/pfad/zu/server.py"]
```

### Problem: Node.js-Pakete werden neu installiert bei jedem Start

```bash
# npx cached packages beschleunigen
# Alternativ: Paket global installieren
npm install -g @modelcontextprotocol/server-filesystem

# Dann in settings.json:
{
  "command": "mcp-server-filesystem",
  "args": ["/home/user/documents"]
}
```

### Problem: Python-Virtualenv wird nicht gefunden

```json
// FALSCH: Aktiviertes venv wird nicht vererbt
{
  "command": "python",
  "args": ["server.py"]
}

// RICHTIG: Explizit Python aus dem venv verwenden
{
  "command": "/home/user/mcp-servers/mein-server/.venv/bin/python",
  "args": ["/home/user/mcp-servers/mein-server/server.py"]
}

// NOCH BESSER: uv run (handhabt venv automatisch)
{
  "command": "uv",
  "args": ["run", "--directory", "/home/user/mcp-servers/mein-server", "python", "server.py"]
}
```

### Debugging mit Logs

```bash
# Claude Code Server-Logs anzeigen (Pfad kann variieren)
ls ~/.claude/logs/

# Oder: Eigenen Log-Handler im Python-Server hinzufügen
import logging
import sys

logging.basicConfig(
    level=logging.DEBUG,
    stream=sys.stderr,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
```

---

## Alternative: `claude mcp add` CLI-Befehl

Statt die `settings.json` manuell zu bearbeiten, kann man MCP-Server auch direkt über die Kommandozeile hinzufügen:

```bash
# Lokalen stdio-Server hinzufügen
claude mcp add mein-server python3 /pfad/zu/server.py

# npm-Paket als Server hinzufügen
claude mcp add filesystem npx -y @modelcontextprotocol/server-filesystem /home/user/documents

# Server mit Umgebungsvariablen hinzufügen
claude mcp add github -e GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxxx npx -y @modelcontextprotocol/server-github

# Konfigurierte Server auflisten
claude mcp list

# Server entfernen
claude mcp remove mein-server
```

> [!TIP]
> `claude mcp add` schreibt automatisch in die richtige `settings.json` und validiert die Eingaben. Das ist oft schneller und weniger fehleranfällig als manuelles JSON-Editing.

---

## Tipps für die Praxis

> [!TIP]
> **Reihenfolge der Einrichtung:**
> 1. Server zuerst manuell in der Kommandozeile testen
> 2. Dann in settings.json eintragen
> 3. Claude Code neu starten / neues Gespräch öffnen
> 4. Mit `/mcp` den Status prüfen
> 5. Mit einem einfachen Prompt das erste Tool testen

> [!TIP]
> **Server-Namen:** Verwende sprechende, eindeutige Namen ohne Leerzeichen. Beispiel: `filesystem-projekte` statt `fs` oder `mein filesystem server`.

> [!TIP]
> **Mehrere Instanzen:** Du kannst denselben Server-Typ mehrfach konfigurieren – z.B. `postgres-dev` und `postgres-prod` für zwei verschiedene Datenbanken.

---

## MCP Server in GitHub Copilot CLI einbinden

GitHub Copilot CLI (das Terminal-basierte Copilot-Tool) unterstützt ebenfalls MCP-Server und bietet dabei einen etwas anderen Ansatz als Claude Code.

### Eingebauter GitHub MCP Server

> [!IMPORTANT]
> GitHub Copilot CLI bringt den **GitHub MCP Server bereits integriert** mit. GitHub-Operationen wie Issues, Pull Requests, Commits, Code Search und Actions sind sofort verfügbar – ohne manuelle Konfiguration oder API-Token-Setup.

Das bedeutet: Für typische GitHub-Workflows (Issues lesen, PRs erstellen, Code suchen) ist **kein zusätzlicher MCP-Server nötig**. Die Tools stehen direkt nach dem Start zur Verfügung.

### Custom MCP Server hinzufügen

Zusätzliche MCP-Server werden über den `/mcp` Slash-Befehl verwaltet:

```
/mcp                      # MCP-Konfiguration öffnen und verwalten
```

Der `/mcp`-Befehl bietet eine interaktive Oberfläche zum:
- Hinzufügen neuer MCP-Server
- Anzeigen konfigurierter Server und deren Status
- Entfernen bestehender Server

Die Konfiguration wird über die CLI-Oberfläche vorgenommen, statt JSON-Dateien manuell zu bearbeiten.

### Vergleich: Claude Code vs. GitHub Copilot CLI

| Aspekt | Claude Code | GitHub Copilot CLI |
|--------|-------------|-------------------|
| **Konfigurationsmethode** | JSON-Dateien (`settings.json`) | `/mcp` Slash-Befehl (interaktiv) |
| **GitHub-Integration** | Manuell per MCP-Server + Token | Eingebaut, kein Setup nötig |
| **Custom Server** | JSON-Einträge in `settings.json` | Über `/mcp` Befehl |
| **Konfigurationsebenen** | Global + Projekt-spezifisch | Über CLI verwaltet |
| **CLI-Verwaltung** | `claude mcp add/remove/list` | `/mcp` im Chat |

> [!TIP]
> Wer primär mit GitHub-Repositories arbeitet, profitiert bei Copilot CLI vom eingebauten GitHub MCP Server – es entfällt das Erstellen eines Personal Access Tokens und die manuelle Server-Konfiguration. Für andere Datenquellen (Datenbanken, Dateisysteme, Web-Suche) müssen Custom MCP Server eingebunden werden, ähnlich wie bei Claude Code.

### Wann welches Tool?

- **GitHub Copilot CLI:** Ideal wenn der Fokus auf GitHub-Workflows liegt und der eingebaute GitHub MCP Server ausreicht.
- **Claude Code:** Flexibler bei der MCP-Konfiguration durch direkte JSON-Bearbeitung und mehrstufige Konfigurationsebenen (global/projekt-spezifisch).

Beide Tools unterstützen das gleiche MCP-Protokoll – ein einmal entwickelter MCP-Server funktioniert mit beiden Hosts.

---

## Verwandte Notizen

- [[Was ist MCP]]
- [[MCP Server erstellen (Python)]]
- [[MCP Tools Resources Prompts]]
