# MCP Server erstellen (Python)

> [!IMPORTANT]
> Dieser Guide zeigt, wie man einen vollständigen MCP-Server mit dem offiziellen Python SDK erstellt. Der resultierende Server kann direkt in Claude Code oder Claude Desktop eingebunden werden.

## Voraussetzungen

- **Python 3.10+** (wegen `match`-Statement und moderner Type Hints)
- **uv** (empfohlen) oder **pip**
- Grundkenntnisse in Python und async/await

```bash
# Python-Version prüfen
python3 --version  # >= 3.10 erforderlich

# uv installieren (empfohlen, sehr schnell)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Oder mit pip arbeiten (klassisch)
pip install --upgrade pip
```

---

## MCP Python SDK installieren

### Mit uv (empfohlen)

```bash
# Neues Projekt anlegen
mkdir mein-mcp-server
cd mein-mcp-server

# Virtuelles Environment und Abhängigkeiten
uv init
uv add mcp

# Für HTTP-Anfragen (optional)
uv add httpx

# Für den vollständigen Beispiel-Server
uv add mcp httpx psutil
```

### Mit pip

```bash
mkdir mein-mcp-server
cd mein-mcp-server
python3 -m venv .venv
source .venv/bin/activate

pip install mcp httpx psutil
```

> [!NOTE]
> Das Paket heißt `mcp` auf PyPI. Es enthält sowohl das Server- als auch das Client-Framework sowie alle nötigen Typen und Hilfsfunktionen.

---

## FastMCP – der empfohlene Weg (High-Level API)

> [!IMPORTANT]
> **FastMCP** ist seit 2025 der empfohlene und einfachste Weg, MCP-Server zu erstellen. Es ist Teil des offiziellen `mcp`-SDK und bietet eine deklarative API mit Dekoratoren – vergleichbar mit FastAPI für Webserver.

```python
# server.py - MCP Server mit FastMCP
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("mein-server")

@mcp.tool()
def hallo_welt(name: str) -> str:
    """Gibt eine Begrüßung zurück"""
    return f"Hallo, {name}!"

@mcp.resource("config://app")
def get_config() -> str:
    """Gibt die App-Konfiguration zurück"""
    return "App Config Daten"

if __name__ == "__main__":
    mcp.run()
```

FastMCP übernimmt automatisch:
- **Tool-Registrierung** aus Funktions-Signaturen und Type Hints
- **Input-Schema-Generierung** aus den Parametern
- **Beschreibungen** aus den Docstrings
- **Transport-Handling** (stdio, Streamable HTTP)

```bash
# Server starten (stdio)
python server.py

# Server starten (Streamable HTTP)
python server.py --transport streamable-http

# Oder mit uv
uv run server.py
```

> [!TIP]
> Für die meisten Use Cases ist FastMCP die beste Wahl. Die Low-Level `Server`-API (siehe unten) ist nur nötig, wenn man volle Kontrolle über das Protokoll braucht.

---

## Grundstruktur eines MCP Servers (Low-Level API)

Die Low-Level `Server`-Klasse bietet volle Kontrolle über das MCP-Protokoll, ist aber deutlich aufwändiger als FastMCP:

```python
# server.py - Minimaler MCP Server
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import mcp.types as types

# 1. Server-Instanz erstellen
app = Server("mein-server")

# 2. Tools/Resources/Prompts registrieren
@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="hallo_welt",
            description="Gibt eine Begrüßung zurück",
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {
                        "type": "string",
                        "description": "Name der Person"
                    }
                },
                "required": ["name"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "hallo_welt":
        person = arguments.get("name", "Welt")
        return [TextContent(type="text", text=f"Hallo, {person}!")]
    raise ValueError(f"Unbekanntes Tool: {name}")

# 3. Server starten
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream, app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## Vollständiges Beispiel: Praxisnaher MCP Server

Der folgende Server implementiert drei nützliche Tools:
1. **`datei_lesen`** - Liest eine lokale Datei
2. **`http_anfrage`** - Sendet einen HTTP-GET-Request
3. **`systeminfo`** - Gibt Systeminformationen zurück

```python
#!/usr/bin/env python3
"""
Praxisnaher MCP Server mit drei Tools:
- datei_lesen: Lokale Datei lesen
- http_anfrage: HTTP GET Request senden
- systeminfo: Systeminformationen abrufen

Verwendung:
    python server.py

Einbindung in Claude Code (~/.claude/settings.json):
    {
      "mcpServers": {
        "mein-server": {
          "command": "python",
          "args": ["/absoluter/pfad/zu/server.py"]
        }
      }
    }
"""

import asyncio
import json
import logging
import os
import platform
from pathlib import Path

import httpx
import psutil
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    CallToolResult,
    TextContent,
    Tool,
)

# Logging konfigurieren (WICHTIG: nach stderr, nicht stdout!)
# stdout wird für das MCP-Protokoll verwendet
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[logging.StreamHandler(__import__("sys").stderr)],
)
logger = logging.getLogger("mein-mcp-server")

# Server-Instanz erstellen
app = Server("mein-mcp-server")


# ─────────────────────────────────────────────────────────────────
# Tool-Definitionen (list_tools)
# ─────────────────────────────────────────────────────────────────

@app.list_tools()
async def list_tools() -> list[Tool]:
    """Gibt alle verfügbaren Tools zurück."""
    logger.info("list_tools aufgerufen")
    return [
        Tool(
            name="datei_lesen",
            description=(
                "Liest den Inhalt einer lokalen Datei vom Dateisystem. "
                "Gibt den Dateiinhalt als Text zurück. "
                "Unterstützt alle Textdateien (UTF-8). "
                "Für Binärdateien wird eine Fehlermeldung zurückgegeben."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "pfad": {
                        "type": "string",
                        "description": (
                            "Absoluter oder relativer Pfad zur Datei. "
                            "Beispiel: /home/user/dokument.txt"
                        ),
                    },
                    "max_zeilen": {
                        "type": "integer",
                        "description": (
                            "Maximale Anzahl Zeilen die gelesen werden sollen. "
                            "Standard: 500. Nützlich für große Dateien."
                        ),
                        "default": 500,
                        "minimum": 1,
                        "maximum": 10000,
                    },
                },
                "required": ["pfad"],
            },
        ),
        Tool(
            name="http_anfrage",
            description=(
                "Sendet einen HTTP GET-Request an eine URL und gibt die Antwort zurück. "
                "Nützlich um APIs abzufragen oder Webseiten-Inhalte zu lesen. "
                "Unterstützt nur GET-Requests. Timeout: 30 Sekunden."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "url": {
                        "type": "string",
                        "description": (
                            "Die vollständige URL mit Protokoll. "
                            "Beispiel: https://api.github.com/users/octocat"
                        ),
                    },
                    "headers": {
                        "type": "object",
                        "description": (
                            "Optionale HTTP-Header als Key-Value-Objekt. "
                            "Beispiel: {\"Authorization\": \"Bearer TOKEN\"}"
                        ),
                        "additionalProperties": {"type": "string"},
                    },
                },
                "required": ["url"],
            },
        ),
        Tool(
            name="systeminfo",
            description=(
                "Gibt aktuelle Systeminformationen zurück: "
                "Betriebssystem, CPU-Auslastung, RAM-Nutzung, "
                "Festplattennutzung und Python-Version. "
                "Keine Parameter erforderlich."
            ),
            inputSchema={
                "type": "object",
                "properties": {},
                "required": [],
            },
        ),
    ]


# ─────────────────────────────────────────────────────────────────
# Tool-Handler (call_tool)
# ─────────────────────────────────────────────────────────────────

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Führt das angegebene Tool mit den Argumenten aus."""
    logger.info(f"Tool aufgerufen: {name}, Argumente: {arguments}")

    match name:
        case "datei_lesen":
            return await tool_datei_lesen(arguments)
        case "http_anfrage":
            return await tool_http_anfrage(arguments)
        case "systeminfo":
            return await tool_systeminfo(arguments)
        case _:
            logger.error(f"Unbekanntes Tool: {name}")
            raise ValueError(f"Unbekanntes Tool: '{name}'")


# ─────────────────────────────────────────────────────────────────
# Tool-Implementierungen
# ─────────────────────────────────────────────────────────────────

async def tool_datei_lesen(args: dict) -> list[TextContent]:
    """Implementierung des datei_lesen-Tools."""
    pfad_str = args.get("pfad", "")
    max_zeilen = args.get("max_zeilen", 500)

    if not pfad_str:
        return [TextContent(type="text", text="Fehler: Kein Pfad angegeben.")]

    pfad = Path(pfad_str).expanduser().resolve()

    # Sicherheitscheck: Datei muss existieren
    if not pfad.exists():
        return [TextContent(
            type="text",
            text=f"Fehler: Datei nicht gefunden: {pfad}"
        )]

    if not pfad.is_file():
        return [TextContent(
            type="text",
            text=f"Fehler: Pfad ist kein Datei: {pfad}"
        )]

    # Dateigröße prüfen (max 10 MB)
    groesse = pfad.stat().st_size
    if groesse > 10 * 1024 * 1024:
        return [TextContent(
            type="text",
            text=f"Fehler: Datei zu groß ({groesse / 1024 / 1024:.1f} MB). Maximum: 10 MB."
        )]

    try:
        with open(pfad, "r", encoding="utf-8") as f:
            zeilen = f.readlines()

        gesamt_zeilen = len(zeilen)
        if gesamt_zeilen > max_zeilen:
            inhalt = "".join(zeilen[:max_zeilen])
            hinweis = f"\n\n[Hinweis: Datei hat {gesamt_zeilen} Zeilen, nur die ersten {max_zeilen} werden angezeigt]"
        else:
            inhalt = "".join(zeilen)
            hinweis = ""

        ergebnis = (
            f"Datei: {pfad}\n"
            f"Größe: {groesse} Bytes | Zeilen: {gesamt_zeilen}\n"
            f"{'─' * 60}\n"
            f"{inhalt}"
            f"{hinweis}"
        )
        logger.info(f"Datei gelesen: {pfad} ({groesse} Bytes)")
        return [TextContent(type="text", text=ergebnis)]

    except UnicodeDecodeError:
        return [TextContent(
            type="text",
            text=f"Fehler: Datei ist keine UTF-8-Textdatei: {pfad}"
        )]
    except PermissionError:
        return [TextContent(
            type="text",
            text=f"Fehler: Keine Leseberechtigung für: {pfad}"
        )]
    except Exception as e:
        logger.exception(f"Unerwarteter Fehler beim Lesen von {pfad}")
        return [TextContent(type="text", text=f"Fehler: {type(e).__name__}: {e}")]


async def tool_http_anfrage(args: dict) -> list[TextContent]:
    """Implementierung des http_anfrage-Tools."""
    url = args.get("url", "")
    headers = args.get("headers", {})

    if not url:
        return [TextContent(type="text", text="Fehler: Keine URL angegeben.")]

    if not url.startswith(("http://", "https://")):
        return [TextContent(
            type="text",
            text=f"Fehler: URL muss mit http:// oder https:// beginnen: {url}"
        )]

    try:
        async with httpx.AsyncClient(timeout=30.0, follow_redirects=True) as client:
            logger.info(f"HTTP GET: {url}")
            response = await client.get(url, headers=headers)

        # Content-Type prüfen
        content_type = response.headers.get("content-type", "")

        # JSON formatieren falls möglich
        if "application/json" in content_type:
            try:
                daten = response.json()
                body = json.dumps(daten, indent=2, ensure_ascii=False)
            except Exception:
                body = response.text
        else:
            body = response.text

        # Body auf 50.000 Zeichen begrenzen
        if len(body) > 50000:
            body = body[:50000] + "\n\n[... Antwort abgeschnitten, zu lang ...]"

        ergebnis = (
            f"URL: {url}\n"
            f"Status: {response.status_code} {response.reason_phrase}\n"
            f"Content-Type: {content_type}\n"
            f"{'─' * 60}\n"
            f"{body}"
        )
        logger.info(f"HTTP-Antwort: {response.status_code} von {url}")
        return [TextContent(type="text", text=ergebnis)]

    except httpx.TimeoutException:
        return [TextContent(type="text", text=f"Fehler: Timeout nach 30 Sekunden für {url}")]
    except httpx.ConnectError as e:
        return [TextContent(type="text", text=f"Fehler: Verbindung fehlgeschlagen: {e}")]
    except Exception as e:
        logger.exception(f"Unerwarteter Fehler bei HTTP-Anfrage an {url}")
        return [TextContent(type="text", text=f"Fehler: {type(e).__name__}: {e}")]


async def tool_systeminfo(_args: dict) -> list[TextContent]:
    """Implementierung des systeminfo-Tools."""
    try:
        # CPU
        cpu_prozent = psutil.cpu_percent(interval=1)
        cpu_kerne_physisch = psutil.cpu_count(logical=False)
        cpu_kerne_logisch = psutil.cpu_count(logical=True)

        # RAM
        ram = psutil.virtual_memory()
        ram_gesamt_gb = ram.total / (1024 ** 3)
        ram_belegt_gb = ram.used / (1024 ** 3)
        ram_frei_gb = ram.available / (1024 ** 3)

        # Festplatte (root-Partition)
        disk = psutil.disk_usage("/")
        disk_gesamt_gb = disk.total / (1024 ** 3)
        disk_belegt_gb = disk.used / (1024 ** 3)
        disk_frei_gb = disk.free / (1024 ** 3)

        # Betriebssystem
        os_info = {
            "system": platform.system(),
            "release": platform.release(),
            "version": platform.version(),
            "machine": platform.machine(),
            "processor": platform.processor(),
        }

        # Uptime
        import time
        boot_time = psutil.boot_time()
        uptime_sekunden = int(time.time() - boot_time)
        uptime_stunden = uptime_sekunden // 3600
        uptime_minuten = (uptime_sekunden % 3600) // 60

        ergebnis = f"""System Information
{'═' * 50}

Betriebssystem:
  System:    {os_info['system']} {os_info['release']}
  Maschine:  {os_info['machine']}
  Prozessor: {os_info['processor'] or 'Unbekannt'}
  Uptime:    {uptime_stunden}h {uptime_minuten}min

CPU:
  Auslastung:      {cpu_prozent:.1f}%
  Kerne (physisch): {cpu_kerne_physisch}
  Kerne (logisch):  {cpu_kerne_logisch}

RAM:
  Gesamt:    {ram_gesamt_gb:.1f} GB
  Belegt:    {ram_belegt_gb:.1f} GB ({ram.percent:.1f}%)
  Verfügbar: {ram_frei_gb:.1f} GB

Festplatte (/):
  Gesamt:  {disk_gesamt_gb:.1f} GB
  Belegt:  {disk_belegt_gb:.1f} GB ({disk.percent:.1f}%)
  Frei:    {disk_frei_gb:.1f} GB

Python:
  Version: {platform.python_version()}
  Impl:    {platform.python_implementation()}
"""
        logger.info("systeminfo abgerufen")
        return [TextContent(type="text", text=ergebnis)]

    except Exception as e:
        logger.exception("Fehler beim Abrufen der Systeminformationen")
        return [TextContent(type="text", text=f"Fehler: {type(e).__name__}: {e}")]


# ─────────────────────────────────────────────────────────────────
# Server-Start
# ─────────────────────────────────────────────────────────────────

async def main():
    """Startet den MCP-Server über stdio."""
    logger.info("MCP Server startet...")
    async with stdio_server() as (read_stream, write_stream):
        logger.info("stdio-Verbindung hergestellt, warte auf Verbindung...")
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options(),
        )
    logger.info("MCP Server beendet.")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Projektstruktur

```
mein-mcp-server/
├── server.py          # Haupt-Server-Datei
├── pyproject.toml     # Projektdefinition (bei uv)
├── requirements.txt   # Abhängigkeiten (bei pip)
└── README.md          # Dokumentation
```

### pyproject.toml (bei uv)

```toml
[project]
name = "mein-mcp-server"
version = "0.1.0"
description = "Mein erster MCP Server"
requires-python = ">=3.10"
dependencies = [
    "mcp>=1.0.0",
    "httpx>=0.27.0",
    "psutil>=5.9.0",
]

[project.scripts]
mein-mcp-server = "server:main_sync"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

> [!WARNING]
> Die `main`-Funktion ist `async` und kann nicht direkt als Console-Script-Einstiegspunkt verwendet werden. Definiere einen synchronen Wrapper `main_sync` in `server.py`:
> ```python
> def main_sync():
>     asyncio.run(main())
> ```
> Der Entry Point verweist dann auf `"server:main_sync"` statt `"server:main"`.

```
mcp>=1.0.0
httpx>=0.27.0
psutil>=5.9.0
```

---

## Server testen

### Manueller Test mit dem MCP Inspector

Das MCP SDK enthält ein eingebautes Testtool:

```bash
# Mit uv
uv run mcp dev server.py

# Mit pip (nach Aktivierung des venv)
mcp dev server.py
```

Der Inspector öffnet eine Web-UI unter `http://localhost:5173` wo man Tools interaktiv aufrufen kann.

### Test mit direktem JSON-RPC

```bash
# Server starten und manuell JSON senden (für Debugging)
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | python server.py
```

### Einfacher Smoke-Test

```python
# test_server.py
import asyncio
import subprocess
import json

async def test_server():
    """Testet ob der Server korrekt antwortet."""
    proc = await asyncio.create_subprocess_exec(
        "python", "server.py",
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )

    # Initialize
    init_request = json.dumps({
        "jsonrpc": "2.0",
        "id": 1,
        "method": "initialize",
        "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {"name": "test-client", "version": "0.1.0"}
        }
    }) + "\n"

    proc.stdin.write(init_request.encode())
    await proc.stdin.drain()

    response_line = await proc.stdout.readline()
    response = json.loads(response_line)

    print(f"Server-Name: {response['result']['serverInfo']['name']}")
    print(f"Capabilities: {response['result']['capabilities']}")

    proc.terminate()
    await proc.wait()

asyncio.run(test_server())
```

---

## Server als stdio-Server für Claude Code

So wird der Server in `~/.claude/settings.json` eingebunden:

```json
{
  "mcpServers": {
    "mein-server": {
      "command": "python",
      "args": ["/absoluter/pfad/zu/mein-mcp-server/server.py"],
      "env": {
        "PYTHONPATH": "/absoluter/pfad/zu/mein-mcp-server"
      }
    }
  }
}
```

> [!TIP]
> Bei Verwendung von `uv` kann man den Server noch sauberer einbinden, ohne das venv manuell aktivieren zu müssen:
>
> ```json
> {
>   "mcpServers": {
>     "mein-server": {
>       "command": "uv",
>       "args": ["run", "--project", "/pfad/zu/mein-mcp-server", "python", "server.py"]
>     }
>   }
> }
> ```

---

## Fehlerbehandlung und Logging

### Die wichtigste Regel: stdout ist reserviert

> [!WARNING]
> Alle Ausgaben (print, logging) **müssen** nach **stderr** gehen, nicht nach stdout. Das MCP-Protokoll läuft über stdout. Jede unerwartete Ausgabe auf stdout zerstört das Protokoll und führt zu kryptischen Fehlern.

```python
import logging
import sys

# RICHTIG: Logging nach stderr
logging.basicConfig(
    level=logging.DEBUG,
    stream=sys.stderr,  # <-- kritisch!
    format="%(asctime)s [%(levelname)s] %(message)s"
)

# FALSCH: print() schreibt nach stdout!
# print("Debugging...")  # Niemals in einem MCP-Server!

# RICHTIG: print nach stderr umleiten wenn nötig
print("Debug-Info", file=sys.stderr)
```

### Strukturierte Fehlerantworten

```python
from mcp.types import TextContent

# Tools sollen NIEMALS Exceptions werfen (außer für unbekannte Tool-Namen)
# Stattdessen Fehlermeldung als TextContent zurückgeben:

async def tool_mit_fehlerbehandlung(args: dict) -> list[TextContent]:
    try:
        # ... Logik ...
        return [TextContent(type="text", text="Ergebnis")]
    except FileNotFoundError as e:
        # Verständliche Fehlermeldung für den User / das LLM
        return [TextContent(type="text", text=f"Fehler: Datei nicht gefunden: {e}")]
    except PermissionError as e:
        return [TextContent(type="text", text=f"Fehler: Keine Berechtigung: {e}")]
    except Exception as e:
        # Fallback für unerwartete Fehler
        logger.exception("Unerwarteter Fehler")
        return [TextContent(type="text", text=f"Unerwarteter Fehler: {type(e).__name__}: {e}")]
```

### Log-Level in der Produktion

```python
import os

# Log-Level über Umgebungsvariable steuern
log_level = os.environ.get("MCP_LOG_LEVEL", "INFO").upper()
logging.basicConfig(level=getattr(logging, log_level, logging.INFO), stream=sys.stderr)
```

```json
// In settings.json Log-Level setzen:
{
  "mcpServers": {
    "mein-server": {
      "command": "python",
      "args": ["server.py"],
      "env": {
        "MCP_LOG_LEVEL": "DEBUG"
      }
    }
  }
}
```

---

## Deployment-Optionen

### Option 1: Direkter Python-Aufruf (einfachste)

```json
{
  "command": "python3",
  "args": ["/pfad/zu/server.py"]
}
```

### Option 2: Mit uv run (empfohlen für Entwicklung)

```json
{
  "command": "uv",
  "args": ["run", "--directory", "/pfad/zu/projekt", "python", "server.py"]
}
```

### Option 3: Als installierbares Skript

```toml
# pyproject.toml
[project.scripts]
mein-mcp-server = "server:main_sync"
```

```python
# server.py - Synchroner Einstiegspunkt
def main_sync():
    asyncio.run(main())
```

```bash
pip install -e .
# Dann in settings.json:
# "command": "mein-mcp-server"
```

### Option 4: Streamable HTTP Server (für Remote-Zugriff)

Mit FastMCP ist ein HTTP-Server besonders einfach:

```python
# http_server.py - Streamable HTTP (empfohlen)
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("mein-remote-server")

@mcp.tool()
def hallo(name: str) -> str:
    """Gibt eine Begrüßung zurück"""
    return f"Hallo, {name}!"

if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)
```

```bash
pip install mcp uvicorn
python http_server.py
# Server läuft auf http://localhost:8000/mcp
```

> [!WARNING]
> Der ältere SSE-Transport (`SseServerTransport` mit zwei Endpunkten `/sse` und `/messages`) wurde in der MCP-Spezifikation vom 2025-03-26 als deprecated markiert. Neue Server sollten Streamable HTTP verwenden.

<details>
<summary>Legacy: SSE-Server (deprecated)</summary>

```python
# http_server_sse.py - Legacy SSE Transport
from mcp.server.sse import SseServerTransport
from starlette.applications import Starlette
from starlette.routing import Route, Mount
import uvicorn

# SSE Transport erstellen
sse = SseServerTransport("/messages")

async def handle_sse(request):
    async with sse.connect_sse(
        request.scope, request.receive, request._send
    ) as streams:
        await app.run(streams[0], streams[1], app.create_initialization_options())

# Starlette-App
starlette_app = Starlette(
    routes=[
        Route("/sse", endpoint=handle_sse),
        Mount("/messages", app=sse.handle_post_message),
    ]
)

if __name__ == "__main__":
    uvicorn.run(starlette_app, host="0.0.0.0", port=8000)
```

</details>

---

## Vollständige Verzeichnisstruktur für Produktion

```
mein-mcp-server/
├── pyproject.toml
├── README.md
├── server.py              # Haupt-Einstiegspunkt
├── src/
│   └── mein_server/
│       ├── __init__.py
│       ├── tools/
│       │   ├── __init__.py
│       │   ├── datei_tools.py
│       │   ├── http_tools.py
│       │   └── system_tools.py
│       └── resources/
│           ├── __init__.py
│           └── datei_resources.py
└── tests/
    └── test_tools.py
```

---

## Verwandte Notizen

- [[Was ist MCP]]
- [[MCP Tools Resources Prompts]]
- [[MCP Server in Claude Code einbinden]]
