---
tags: [ai, agents, cli, claude-code, konfiguration]
---

# Claude Code Konfiguration und Hooks

## settings.json – Übersicht und Speicherort

Claude Code liest Konfiguration aus einer `settings.json`-Datei. Ähnlich wie bei CLAUDE.md gibt es mehrere Ebenen mit unterschiedlichem Geltungsbereich:

| Speicherort | Geltungsbereich | Beschreibung |
|---|---|---|
| `~/.claude/settings.json` | Global | Gilt für alle Projekte des Nutzers |
| `./.claude/settings.json` | Projekt | Gilt nur für dieses Projekt |
| `./.claude/settings.local.json` | Projekt (lokal) | Wie Projekt, aber in .gitignore |

> [!IMPORTANT]
> Projekt-Einstellungen überschreiben globale Einstellungen. `settings.local.json` eignet sich für persönliche Anpassungen, die nicht ins Git-Repository sollen (z.B. abweichende Tool-Permissions für eigene Entwicklungsumgebung).

### Speicherort prüfen und anlegen:

```bash
# Globale Einstellungen
ls ~/.claude/settings.json

# Projekt-Einstellungen
ls .claude/settings.json

# Verzeichnis anlegen falls nötig
mkdir -p ~/.claude
mkdir -p .claude
```

---

## Vollständige settings.json – Struktur und alle Felder

```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(npm:*)",
      "Bash(python:*)",
      "Read(*)",
      "Write(src/**)",
      "Edit(src/**)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(curl:*)",
      "Bash(wget:*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"[HOOK] Bash-Befehl: $CLAUDE_TOOL_INPUT\" >> ~/.claude/audit.log"
          }
        ]
      }
    ],
    "PostToolUse": [],
    "Notification": [],
    "Stop": []
  },
  "env": {
    "NODE_ENV": "development",
    "LOG_LEVEL": "debug"
  },
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"],
      "env": {}
    }
  },
  "cleanupPeriodDays": 30
}
```

---

## Wichtige Einstellungen im Detail

### model

Das Standard-Modell für alle Claude Code Sessions.

```json
{
  "model": "claude-sonnet-4-6"
}
```

Verfügbare Modelle (Stand 2025):
- `claude-opus-4-6` – Leistungsstärkstes Modell, langsamer, teurer
- `claude-sonnet-4-6` – Ausgewogen (empfohlen für die meisten Aufgaben)
- `claude-haiku-4-5` – Schnellstes, günstigstes Modell

> [!TIP]
> `claude-sonnet-4-6` ist für die meisten Entwicklungsaufgaben der beste Kompromiss. `claude-opus-4-6` lohnt sich für sehr komplexe Architektur- oder Refactoring-Aufgaben.

### permissions

Granulare Kontrolle darüber, welche Tools und Befehle Claude ausführen darf.

```json
{
  "permissions": {
    "allow": [
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git status:*)",
      "Bash(npm test:*)",
      "Bash(npm run:*)",
      "Read(**)",
      "Write(src/**)",
      "Write(tests/**)",
      "Edit(**)"
    ],
    "deny": [
      "Bash(git push:*)",
      "Bash(git reset --hard:*)",
      "Bash(rm -rf:*)",
      "Bash(sudo:*)",
      "Bash(curl:*)",
      "Bash(wget:*)"
    ]
  }
}
```

**Syntax der Permission-Rules:**

```
"ToolName(befehl:argument)"

Beispiele:
"Bash(*)"              → Alle Bash-Befehle erlaubt
"Bash(git:*)"          → git mit beliebigen Argumenten
"Bash(git log:*)"      → nur 'git log', alle Flags erlaubt
"Bash(npm run:build)"  → nur 'npm run build'
"Read(**)"             → alle Dateien lesen
"Write(src/**)"        → nur in src/ schreiben
"Write(*.md)"          → nur Markdown-Dateien schreiben
```

### env

Umgebungsvariablen, die Claude Code für alle Tool-Aufrufe setzt.

```json
{
  "env": {
    "NODE_ENV": "development",
    "DATABASE_URL": "postgresql://localhost:5432/dev_db",
    "LOG_LEVEL": "debug",
    "PYTHONPATH": "/home/user/projects/myapp"
  }
}
```

---

## Was sind Hooks?

Hooks sind **Shell-Befehle oder Skripte, die Claude Code automatisch vor oder nach bestimmten Ereignissen ausführt**. Sie erlauben:

- Automatisierung von Routineaufgaben (git pull vor Dateioperationen)
- Audit-Logging aller Tool-Aufrufe
- Sicherheits-Prüfungen vor riskanten Operationen
- Benachrichtigungen (z.B. bei langen Tasks)
- Integration mit externen Systemen

> [!NOTE]
> Hooks laufen als Shell-Prozesse auf dem lokalen System. Sie haben Zugriff auf die gleichen Umgebungsvariablen wie Claude Code und können über Environment-Variablen auf Kontext-Informationen zugreifen.

---

## Hook-Typen

### PreToolUse

Wird ausgeführt **bevor** Claude ein Tool aufruft. Kann den Tool-Aufruf blockieren (Exit Code != 0).

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/check-dangerous-command.sh"
          }
        ]
      }
    ]
  }
}
```

**Verfügbare Umgebungsvariablen in PreToolUse:**

| Variable | Beschreibung | Beispiel |
|---|---|---|
| `CLAUDE_TOOL_NAME` | Name des Tools | `Bash`, `Write`, `Edit` |
| `CLAUDE_TOOL_INPUT` | JSON-kodierter Input | `{"command":"git status"}` |
| `CLAUDE_SESSION_ID` | Aktuelle Session-ID | `sess_abc123` |
| `CLAUDE_WORKING_DIR` | Arbeitsverzeichnis | `/home/user/project` |

### PostToolUse

Wird ausgeführt **nachdem** ein Tool-Aufruf abgeschlossen wurde.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_OUTPUT_FILE\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

**Zusätzliche Umgebungsvariablen in PostToolUse:**

| Variable | Beschreibung |
|---|---|
| `CLAUDE_TOOL_OUTPUT` | JSON-kodierter Output des Tools |
| `CLAUDE_TOOL_SUCCESS` | `true` oder `false` |
| `CLAUDE_TOOL_OUTPUT_FILE` | Pfad zur erstellten/geänderten Datei (Write/Edit) |

### Notification

Wird ausgeführt, wenn Claude eine Benachrichtigung an den User sendet.

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' \"$CLAUDE_NOTIFICATION_MESSAGE\""
          }
        ]
      }
    ]
  }
}
```

**Umgebungsvariablen:**

| Variable | Beschreibung |
|---|---|
| `CLAUDE_NOTIFICATION_MESSAGE` | Text der Benachrichtigung |
| `CLAUDE_NOTIFICATION_TYPE` | Typ (info, warning, error) |

### Stop

Wird ausgeführt, wenn Claude eine Task abschließt.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Session beendet: $(date)\" >> ~/.claude/sessions.log"
          }
        ]
      }
    ]
  }
}
```

---

## Hook-Konfiguration – vollständige JSON-Struktur

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "ToolName",
        "hooks": [
          {
            "type": "command",
            "command": "shell-befehl oder /pfad/zum/skript.sh",
            "timeout": 10000
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "ToolName",
        "hooks": [
          {
            "type": "command",
            "command": "shell-befehl"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "shell-befehl"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "shell-befehl"
          }
        ]
      }
    ]
  }
}
```

**Matcher-Optionen:**

| Matcher | Bedeutung |
|---|---|
| `"Bash"` | Nur Bash-Tool-Aufrufe |
| `"Write"` | Nur Write-Tool-Aufrufe |
| `"Edit"` | Nur Edit-Tool-Aufrufe |
| `"Read"` | Nur Read-Tool-Aufrufe |
| `"*"` oder kein Matcher | Alle Tools |

**Hook-Felder:**

| Feld | Pflicht | Beschreibung |
|---|---|---|
| `type` | Ja | Immer `"command"` |
| `command` | Ja | Shell-Befehl oder Skript-Pfad |
| `timeout` | Nein | Timeout in ms (Standard: 60000) |

---

## Beispiel-Hook 1: Automatisch git pull vor Dateioperationen

Stellt sicher, dass Claude immer auf dem aktuellen Stand des Repos arbeitet.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/.claude/hooks/git-pull-if-needed.sh",
            "timeout": 30000
          }
        ]
      }
    ]
  }
}
```

Das zugehörige Skript `~/.claude/hooks/git-pull-if-needed.sh`:

```bash
#!/bin/bash
# Führt git pull aus wenn im git-Repository und nicht auf main

set -e

# Prüfen ob im git-Repo
if ! git -C "$CLAUDE_WORKING_DIR" rev-parse --git-dir > /dev/null 2>&1; then
    exit 0  # Kein git-Repo, nichts tun
fi

# Aktuellen Branch ermitteln
BRANCH=$(git -C "$CLAUDE_WORKING_DIR" branch --show-current)

# Nur auf Feature-Branches automatisch pullen
if [[ "$BRANCH" == feat/* ]] || [[ "$BRANCH" == fix/* ]]; then
    echo "[Hook] git pull auf Branch: $BRANCH"

    # Prüfen ob Remote vorhanden
    if git -C "$CLAUDE_WORKING_DIR" remote get-url origin > /dev/null 2>&1; then
        git -C "$CLAUDE_WORKING_DIR" pull --ff-only origin "$BRANCH" 2>&1 || {
            echo "[Hook] git pull fehlgeschlagen (möglicherweise divergiert) – übersprungen"
        }
    fi
fi

exit 0  # Immer erfolgreich – Pull-Fehler sollen Tool-Aufruf nicht blockieren
```

```bash
chmod +x ~/.claude/hooks/git-pull-if-needed.sh
```

---

## Beispiel-Hook 2: Logging aller Tool-Aufrufe

Erstellt ein vollständiges Audit-Log aller Claude-Aktionen.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/audit-log.sh pre"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/audit-log.sh post"
          }
        ]
      }
    ]
  }
}
```

Das Skript `~/.claude/hooks/audit-log.sh`:

```bash
#!/bin/bash
# Audit-Logging für alle Claude Code Tool-Aufrufe

PHASE="$1"  # "pre" oder "post"
LOG_FILE="${CLAUDE_AUDIT_LOG:-$HOME/.claude/audit.log}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Log-Verzeichnis sicherstellen
mkdir -p "$(dirname "$LOG_FILE")"

# JSON-Log-Eintrag schreiben
cat >> "$LOG_FILE" << EOF
{
  "timestamp": "$TIMESTAMP",
  "phase": "$PHASE",
  "session_id": "${CLAUDE_SESSION_ID:-unknown}",
  "tool": "${CLAUDE_TOOL_NAME:-unknown}",
  "working_dir": "${CLAUDE_WORKING_DIR:-unknown}",
  "input": $(echo "${CLAUDE_TOOL_INPUT:-{}}" | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(json.dumps(d))" 2>/dev/null || echo '{}'),
  "success": "${CLAUDE_TOOL_SUCCESS:-unknown}"
}
EOF

# Sicherheits-Alert: Riskante Befehle erkennen
if [[ "$PHASE" == "pre" ]] && [[ "$CLAUDE_TOOL_NAME" == "Bash" ]]; then
    COMMAND=$(echo "${CLAUDE_TOOL_INPUT:-{}}" | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(d.get('command',''))" 2>/dev/null)

    DANGEROUS_PATTERNS=("rm -rf" "git push --force" "DROP TABLE" "format" "mkfs" "dd if=")

    for pattern in "${DANGEROUS_PATTERNS[@]}"; do
        if echo "$COMMAND" | grep -qi "$pattern"; then
            echo "[SECURITY ALERT] Potenziell gefährlicher Befehl erkannt: $pattern" >&2
            echo "[SECURITY ALERT] Befehl: $COMMAND" >> "${LOG_FILE}.alerts"
            # Exit 0: Warnung loggen, aber nicht blockieren
            # Exit 1: Befehl blockieren
        fi
    done
fi

exit 0
```

```bash
chmod +x ~/.claude/hooks/audit-log.sh
```

### Log-Datei analysieren:

```bash
# Alle Tool-Aufrufe der letzten Session
cat ~/.claude/audit.log | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if line in ['{', '}', '']:
        continue
    try:
        # Einfache Anzeige der wichtigsten Felder
        print(line)
    except:
        pass
" | grep -E '"tool"|"phase"|"timestamp"'

# Nur Bash-Aufrufe zählen
grep '"tool": "Bash"' ~/.claude/audit.log | wc -l

# Write-Operationen auflisten
grep '"tool": "Write"' ~/.claude/audit.log
```

---

## Berechtigungsmodi

Claude Code hat vier Berechtigungsmodi, die das Interaktionsverhalten steuern:

### default (Standard)

Claude fragt bei jeder potenziell riskanten Aktion nach Bestätigung.

```
Claude möchte ausführen: git push origin main
[y/n]? _
```

**Geeignet für:** Normaler Arbeitsalltag, wenn Kontrolle wichtig ist.

### acceptEdits

Claude darf Dateien lesen und bearbeiten, ohne für jede Änderung zu fragen. Bash-Befehle erfordern weiterhin Bestätigung.

```json
{
  "permissions": {
    "mode": "acceptEdits"
  }
}
```

**Geeignet für:** Refactoring-Sessions, wo viele Dateiänderungen erwartet werden.

### bypassPermissions

Claude darf alles ohne Bestätigung ausführen – auch riskante Bash-Befehle.

```json
{
  "permissions": {
    "mode": "bypassPermissions"
  }
}
```

> [!WARNING]
> `bypassPermissions` ist nur für vollständig automatisierte Pipelines gedacht, wo kein Mensch zur Interaktion verfügbar ist (z.B. CI/CD). Im normalen Arbeitsalltag NIEMALS verwenden – Claude kann destructive Aktionen ohne Rückfrage ausführen.

**Geeignet für:** Vollautomatisierte CI/CD-Pipelines, Docker-Container ohne User-Interaktion.

### plan

Claude kann keine Aktionen ausführen – nur planen und beschreiben was es tun würde.

```bash
claude --plan "Refactore die gesamte API auf async/await"
```

**Geeignet für:** Vorab-Review eines komplexen Refactoring, Sicherheitsüberprüfung vor kritischen Änderungen.

---

## Erlaubte und verbotene Tools konfigurieren

### Alle Tools einer Kategorie erlauben/verbieten:

```json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Bash(git:*)",
      "Bash(npm:*)",
      "Bash(python3:*)",
      "Bash(pytest:*)"
    ],
    "deny": [
      "Bash(sudo:*)",
      "Bash(curl:*)",
      "Bash(wget:*)",
      "Bash(ssh:*)",
      "Write(/etc/**)",
      "Write(/usr/**)",
      "Write(~/.ssh/**)"
    ]
  }
}
```

### Praxisbeispiel: Sichere Konfiguration für ein Web-Projekt:

```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": [
      "Read(**)",
      "Write(src/**)",
      "Write(tests/**)",
      "Write(docs/**)",
      "Edit(src/**)",
      "Edit(tests/**)",
      "Edit(docs/**)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(npm install:*)",
      "Bash(npm run:*)",
      "Bash(npm test:*)",
      "Bash(npx:*)",
      "Bash(node:*)"
    ],
    "deny": [
      "Bash(git push:*)",
      "Bash(git reset:*)",
      "Bash(git rebase:*)",
      "Bash(rm -rf:*)",
      "Bash(sudo:*)",
      "Bash(curl:*)",
      "Bash(wget:*)",
      "Write(.env:*)",
      "Write(.env.*:*)",
      "Write(**/*.pem:*)",
      "Write(**/*.key:*)"
    ]
  }
}
```

---

## MCP-Server in settings.json einbinden

MCP (Model Context Protocol) Server erweitern Claude Code mit zusätzlichen Tools und Ressourcen. Sie werden als externe Prozesse gestartet, die Claude Code über das MCP-Protokoll kommuniziert.

### MCP-Konfiguration in settings.json:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/user/projects",
        "/home/user/documents"
      ],
      "env": {}
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "postgresql://user:pass@localhost:5432/mydb"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "${BRAVE_API_KEY}"
      }
    },
    "custom-internal-api": {
      "command": "python3",
      "args": ["/home/user/.claude/mcp-servers/internal-api-server.py"],
      "env": {
        "API_BASE_URL": "https://api.internal.company.com",
        "API_KEY": "${INTERNAL_API_KEY}"
      }
    }
  }
}
```

### Verfügbare offizielle MCP-Server:

| Server | Package | Funktion |
|---|---|---|
| Filesystem | `@modelcontextprotocol/server-filesystem` | Erweiterte Dateioperationen |
| GitHub | `@modelcontextprotocol/server-github` | GitHub API (Issues, PRs, Repos) |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | Direkte DB-Abfragen |
| SQLite | `@modelcontextprotocol/server-sqlite` | SQLite-Datenbanken |
| Brave Search | `@modelcontextprotocol/server-brave-search` | Websuche via Brave |
| Slack | `@modelcontextprotocol/server-slack` | Slack API |
| Puppeteer | `@modelcontextprotocol/server-puppeteer` | Browser-Automatisierung |
| Memory | `@modelcontextprotocol/server-memory` | Persistentes Key-Value-Memory |

### Eigenen MCP-Server schreiben (minimal):

```python
#!/usr/bin/env python3
# ~/.claude/mcp-servers/my-custom-server.py

import asyncio
import json
import sys
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("my-custom-server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="get_deployment_status",
            description="Gibt den aktuellen Deployment-Status einer Umgebung zurück",
            inputSchema={
                "type": "object",
                "properties": {
                    "environment": {
                        "type": "string",
                        "enum": ["dev", "staging", "prod"],
                        "description": "Die Zielumgebung"
                    }
                },
                "required": ["environment"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "get_deployment_status":
        env = arguments.get("environment")
        # Hier echte API-Abfrage implementieren
        status = {
            "environment": env,
            "status": "healthy",
            "version": "v2.3.1",
            "last_deploy": "2024-01-15T14:30:00Z",
            "pods_running": 3
        }
        return [TextContent(type="text", text=json.dumps(status, indent=2))]

    raise ValueError(f"Unbekanntes Tool: {name}")

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

MCP-Server installieren:
```bash
pip install mcp
chmod +x ~/.claude/mcp-servers/my-custom-server.py
```

Konfiguration hinzufügen:
```json
{
  "mcpServers": {
    "deployment": {
      "command": "python3",
      "args": ["/home/user/.claude/mcp-servers/my-custom-server.py"]
    }
  }
}
```

---

## Vollständige Beispiel-Konfiguration (Produktiv)

```json
{
  "model": "claude-sonnet-4-6",

  "permissions": {
    "allow": [
      "Read(**)",
      "Write(src/**)",
      "Write(tests/**)",
      "Write(docs/**)",
      "Edit(**)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git branch:*)",
      "Bash(git checkout:*)",
      "Bash(npm:*)",
      "Bash(node:*)",
      "Bash(npx:*)",
      "Bash(python3:*)",
      "Bash(pytest:*)",
      "Bash(docker ps:*)",
      "Bash(docker logs:*)",
      "Bash(kubectl get:*)",
      "Bash(kubectl describe:*)",
      "Bash(kubectl logs:*)"
    ],
    "deny": [
      "Bash(git push --force:*)",
      "Bash(git reset --hard:*)",
      "Bash(rm -rf:*)",
      "Bash(sudo:*)",
      "Bash(kubectl delete:*)",
      "Bash(kubectl apply:*)",
      "Bash(docker rm:*)",
      "Write(.env:*)",
      "Write(**/*.pem:*)",
      "Write(**/*.key:*)",
      "Write(/etc/**)"
    ]
  },

  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/audit-log.sh pre"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/security-check.sh",
            "timeout": 5000
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/audit-log.sh post"
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/auto-format.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Session beendet: ${CLAUDE_SESSION_ID}\" >> ~/.claude/sessions.log"
          }
        ]
      }
    ]
  },

  "env": {
    "NODE_ENV": "development",
    "LOG_LEVEL": "debug"
  },

  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"],
      "env": {}
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  },

  "cleanupPeriodDays": 30
}
```

---

## Häufige Probleme und Lösungen

> [!WARNING]
> Häufige Fehlerquellen bei der Hook-Konfiguration:

**Hook-Skript wird nicht ausgeführt:**
```bash
# Prüfen: Ist das Skript ausführbar?
ls -la ~/.claude/hooks/
chmod +x ~/.claude/hooks/*.sh

# Prüfen: Korrekte Shebang-Zeile?
head -1 ~/.claude/hooks/mein-hook.sh
# Sollte sein: #!/bin/bash
```

**Hook blockiert Tool-Aufrufe unbeabsichtigt:**
```bash
# PreToolUse-Hooks mit Exit Code != 0 blockieren den Tool-Aufruf
# Immer mit 'exit 0' enden, es sei denn Blockierung ist beabsichtigt
echo "Hook ausgeführt" >> /tmp/hook.log
exit 0  # Explizit!
```

**MCP-Server startet nicht:**
```bash
# Manuell testen
npx -y @modelcontextprotocol/server-filesystem /home/user/projects

# Logs prüfen
cat ~/.claude/mcp-logs/*.log 2>/dev/null || echo "Keine MCP-Logs gefunden"
```

**Permissions greifen nicht:**
```json
// Falsch (Leerzeichen im Matcher):
"Bash(git push : *)"

// Richtig:
"Bash(git push:*)"
```

---

## Verwandte Notizen

- [[Skills]] — Skills-System für wiederverwendbare Workflows
- [[Was ist ein AI Agent]]
- [[Custom Instructions für AI Coding Agents]]
- [[Multi-Agent Workflows]]
