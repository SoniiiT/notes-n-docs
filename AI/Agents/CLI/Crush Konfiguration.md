# Crush Konfiguration

> [!ABSTRACT]
> Crush ist ein Open-Source AI Coding Agent von **Charmbracelet**, der auf die Unix-Philosophie setzt: leichtgewichtig, composable und terminal-nativ. Geschrieben in Go, bietet Crush Multi-Modell-Support, LSP-Integration für präzises Code-Verständnis und MCP-Erweiterbarkeit — alles in einer eleganten TUI (Terminal User Interface).

---

## Übersicht

| Eigenschaft | Details |
|---|---|
| **Entwickler** | Charmbracelet |
| **Typ** | CLI Agent (TUI) |
| **Lizenz** | Open Source |
| **Kosten** | Kostenlos (BYOK für LLM-Provider) |
| **Plattformen** | macOS, Linux, Windows, BSD |
| **Modelle** | OpenAI, Anthropic, Gemini, Groq, Bedrock, Ollama, u.a. |
| **Sprache** | Go |

---

## Installation

### macOS / Linux (Homebrew)

```bash
brew install charmbracelet/tap/crush
```

### Windows

```powershell
# Winget
winget install charmbracelet.crush

# Scoop
scoop bucket add charm https://github.com/charmbracelet/scoop-bucket.git
scoop install crush
```

### npm

```bash
npm install -g @charmland/crush
```

### Linux (Debian/Ubuntu)

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://repo.charm.sh/apt/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/charm.gpg
echo "deb [signed-by=/etc/apt/keyrings/charm.gpg] https://repo.charm.sh/apt/ * *" | \
  sudo tee /etc/apt/sources.list.d/charm.list
sudo apt update && sudo apt install crush
```

### Linux (Fedora/RHEL)

```bash
echo '[charm]
name=Charm
baseurl=https://repo.charm.sh/yum/
enabled=1
gpgcheck=1
gpgkey=https://repo.charm.sh/yum/gpg.key' | sudo tee /etc/yum.repos.d/charm.repo
sudo yum install crush
```

---

## Konfiguration

### API-Keys setzen

```bash
# Anthropic (Claude)
export ANTHROPIC_API_KEY="sk-ant-..."

# OpenAI
export OPENAI_API_KEY="sk-..."

# Google Gemini
export GOOGLE_API_KEY="..."

# Lokale Modelle (Ollama)
# Kein Key nötig, Ollama muss laufen
```

### Konfigurationsdatei

Crush liest Konfiguration aus `~/.config/crush/crush.json` oder projektlokal aus `crush.json`:

```json
{
  "$schema": "https://charm.land/crush.json",
  "default_model": "claude-sonnet-4-6",
  "providers": {
    "anthropic": {
      "api_key": "${ANTHROPIC_API_KEY}"
    },
    "ollama": {
      "base_url": "http://localhost:11434"
    }
  },
  "permissions": {
    "allow_file_write": true,
    "allow_shell_exec": false
  }
}
```

---

## Kernfunktionen

### LSP-Integration

Crush startet automatisch **Language Server** für das aktuelle Projekt:

| Sprache | LSP Server |
|---|---|
| Go | gopls |
| Rust | rust-analyzer |
| Python | pyright |
| TypeScript | typescript-language-server |
| Java | jdtls |

Dies ermöglicht **echtes Code-Verständnis** — nicht nur Text-Matching:
- Symbol-Definitionen und Referenzen
- Typ-Informationen
- Diagnostics und Fehler
- Auto-Imports

### Session-Management

```bash
# Neue Session starten
crush

# Session benennen
crush --session "auth-refactoring"

# Session fortsetzen
crush --resume "auth-refactoring"

# Sessions auflisten
crush sessions list
```

Jede Session behält ihren eigenen Kontext und Verlauf.

### Modell-Wechsel zur Laufzeit

```bash
# In einer laufenden Crush-Session:
/model claude-opus-4-6    # Wechsel zu Claude Opus
/model gpt-5.2            # Wechsel zu GPT
/model ollama:llama3      # Wechsel zu lokalem Modell
```

### MCP-Erweiterungen

Crush unterstützt MCP-Server für externe Tool-Integration:

```json
{
  "mcp_servers": {
    "github": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    },
    "filesystem": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem"]
    }
  }
}
```

---

## Permission-System

Crush fragt vor jeder potenziell gefährlichen Aktion um Erlaubnis:

```
Crush möchte folgenden Befehl ausführen:
  rm -rf node_modules && npm install
[A]kzeptieren  [V]erweigern  [I]mmer erlauben
```

Berechtigungen können in der Konfiguration vordefiniert werden:

```json
{
  "permissions": {
    "allow_file_write": true,
    "allow_file_delete": false,
    "allow_shell_exec": true,
    "denied_commands": ["rm -rf /", "sudo rm"]
  }
}
```

---

## Vergleich mit anderen CLI Agents

| Feature | Crush | Claude Code | Aider | OpenCode |
|---|---|---|---|---|
| **Open Source** | ✅ | ❌ | ✅ | ✅ |
| **Kosten** | Kostenlos (BYOK) | Subscription/API | Kostenlos (BYOK) | Kostenlos (BYOK) |
| **LSP-Integration** | ✅✅ (nativ) | ❌ | ❌ | ❌ |
| **Multi-Modell** | ✅✅✅ | ❌ | ✅✅ | ✅✅ |
| **MCP-Support** | ✅ | ✅ | ❌ | ❌ |
| **TUI** | ✅✅ (Charmbracelet) | CLI | CLI | ✅ (TUI) |
| **Sessions** | ✅ (benannt, fortsetzbar) | ✅ | ❌ | ✅ |
| **Geschrieben in** | Go | TypeScript | Python | Go |

---

## Pro & Contra

| ✅ Vorteile | ❌ Nachteile |
|---|---|
| Open Source, kostenlos | Noch relativ jung, kleinere Community |
| Elegante TUI (Charmbracelet-Qualität) | Keine IDE-Integration |
| LSP-Integration für echtes Code-Verständnis | Kein eigenes Modell (BYOK nötig) |
| Extrem schnell und leichtgewichtig (Go) | Weniger Dokumentation als etablierte Tools |
| Multi-Modell inkl. lokale Modelle | Kein Hooks/Lifecycle-System |
| MCP-erweiterbar | — |

---

## Tipps

> [!TIP]
> - **LSP-Integration** macht Crush besonders stark bei typisierten Sprachen (Go, Rust, TypeScript)
> - **Sessions** nutzen, um verschiedene Aufgaben sauber zu trennen
> - **Lokale Modelle** via Ollama für sensible Projekte einsetzen
> - Crush kombiniert sich hervorragend mit **Warp** als Terminal-Emulator

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Vergleich aller AI Coding Agents
- [[OpenCode Konfiguration]] — Ähnlicher Open-Source CLI Agent
- [[Aider Konfiguration]] — Weiterer Open-Source CLI Agent
- [[Was ist MCP]] — Model Context Protocol für Erweiterungen
