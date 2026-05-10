---
tags: [ai, agents, cli, goose, konfiguration]
---

# Goose — AI Coding Agent von Block

## Überblick

**Goose** ist ein autonomer AI-Coding-Agent, entwickelt von **Block** (ehemals Square) und komplett in **Rust** geschrieben. Mit über **~30.000 GitHub-Stars** (`block/goose`) gehört Goose zu den populärsten Open-Source-Agenten im AI-Coding-Bereich. Goose arbeitet direkt im Terminal, kann eigenständig Code schreiben, Dateien bearbeiten, Shell-Befehle ausführen und komplexe Infrastruktur-Änderungen durchführen — alles innerhalb einer sicheren Sandbox-Umgebung.

> [!info] Enterprise-Backed Open Source
> Goose wird von Block (dem Unternehmen hinter Square, Cash App und TBD) entwickelt. Das bedeutet langfristige Stabilität, professionelle Code-Qualität und aktive Weiterentwicklung.

> [!tip] Warum Rust?
> Die Implementierung in Rust sorgt für extrem schnelle Startzeiten, minimalen Speicherverbrauch und hohe Zuverlässigkeit — deutlich performanter als Python- oder Node.js-basierte Agenten.

---

## 1. Was ist Goose?

Goose ist ein **terminal-basierter AI-Agent** mit TUI (Terminal User Interface), der als autonomer Coding-Assistent agiert:

- **Code autonom schreiben und refactoren** — über mehrere Dateien hinweg
- **Shell-Befehle ausführen** — Build-Prozesse, Tests, Deployments
- **Dateien lesen, erstellen und bearbeiten** — mit vollem Dateisystem-Zugriff
- **Mehrere LLM-Provider nutzen** — OpenAI, Anthropic, Google, lokale Modelle
- **MCP-Server nativ integrieren** — Model Context Protocol für erweiterte Fähigkeiten
- **Modular erweiterbar** — durch Extensions und Toolkits

### Architektur

```
┌─────────────────────────────────────┐
│           Goose TUI / CLI           │
├──────────┬──────────┬───────────────┤
│ Profiles │Extensions│  MCP Servers  │
├──────────┴──────────┴───────────────┤
│        Sandbox / Sicherheit         │
├─────────────────────────────────────┤
│   LLM Provider (Multi-Provider)    │
│  OpenAI │ Anthropic │ Google │ ... │
└─────────────────────────────────────┘
```

---

## 2. Installation

### Homebrew (empfohlen)

````bash
brew install goose
goose --version
````

### Cargo (Rust Toolchain)

````bash
cargo install goose-cli

# Oder aus Source bauen
git clone https://github.com/block/goose.git
cd goose && cargo build --release
````

### Binary Download

````bash
# Release-Binaries: https://github.com/block/goose/releases
curl -L https://github.com/block/goose/releases/latest/download/goose-linux-x86_64 -o goose
chmod +x goose && sudo mv goose /usr/local/bin/
````

### Installations-Script

````bash
curl -fsSL https://github.com/block/goose/releases/latest/download/install.sh | bash
````

> [!warning] PATH-Konfiguration
> Nach der Installation sicherstellen, dass das Binary im `$PATH` liegt. Bei Cargo-Installation wird `~/.cargo/bin/` automatisch verwendet.

---

## 3. Konfiguration

Goose wird über eine YAML-Datei unter `~/.config/goose/config.yaml` konfiguriert.

### Grundkonfiguration

````yaml
# ~/.config/goose/config.yaml
GOOSE_PROVIDER: openai
GOOSE_MODEL: gpt-4o
````

### Unterstützte Provider

| Provider       | Bezeichnung   | Beispiel-Modelle                        | API-Key Variable        |
| -------------- | ------------- | --------------------------------------- | ----------------------- |
| **OpenAI**     | `openai`      | `gpt-4o`, `gpt-4o-mini`, `o3`          | `OPENAI_API_KEY`        |
| **Anthropic**  | `anthropic`   | `claude-sonnet-4-20250514`, `claude-haiku-4-20250514` | `ANTHROPIC_API_KEY`     |
| **Google**     | `google`      | `gemini-2.5-pro`, `gemini-2.5-flash`   | `GOOGLE_API_KEY`        |
| **Ollama**     | `ollama`      | `llama3`, `codellama`, `mistral`        | — (lokal)               |
| **OpenRouter** | `openrouter`  | Diverse Modelle über Routing            | `OPENROUTER_API_KEY`    |

### API-Keys setzen

````bash
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."

# Oder interaktiv
goose configure
````

### Vollständiges Beispiel

````yaml
GOOSE_PROVIDER: anthropic
GOOSE_MODEL: claude-sonnet-4-20250514

extensions:
  developer:
    enabled: true
    requires:
      - name: shell
      - name: file_edit
  github:
    enabled: true
    requires:
      - name: github_api
        env:
          GITHUB_TOKEN: "${GITHUB_TOKEN}"
````

---

## 4. Profile und Sessions

### Profile

Profile ermöglichen verschiedene Konfigurationen für unterschiedliche Anwendungsfälle.

````yaml
profiles:
  frontend:
    provider: anthropic
    model: claude-sonnet-4-20250514
    extensions: [developer, browser]

  devops:
    provider: openai
    model: gpt-4o
    extensions: [developer, kubernetes, docker]

  lightweight:
    provider: ollama
    model: llama3
    extensions: [developer]
````

````bash
goose session --profile frontend
goose session --profile devops
````

### Sessions

Sessions sind persistente Konversations-Kontexte über mehrere Interaktionen hinweg.

````bash
goose session                              # Neue Session starten
goose session --name "refactor-auth"       # Benannte Session
goose session --resume                     # Letzte Session fortsetzen
goose session --resume --name "refactor-auth"  # Bestimmte Session fortsetzen
goose session list                         # Sessions auflisten
````

> [!tip] Session-Management
> Sessions speichern den gesamten Konversationsverlauf. Komplexe Aufgaben können so über mehrere Sitzungen bearbeitet werden, ohne den Kontext zu verlieren.

---

## 5. Extensions und Toolkits

Extensions erweitern Goose durch modulare Tool-Pakete.

### Eingebaute Extensions

| Extension       | Beschreibung                          | Tools                             |
| --------------- | ------------------------------------- | --------------------------------- |
| **developer**   | Kern-Entwicklungswerkzeuge            | Shell, File Edit, Code Search     |
| **github**      | GitHub-Integration                    | PRs, Issues, Reviews, Actions     |
| **browser**     | Web-Browsing                          | URL Fetch, Screenshots            |
| **docker**      | Container-Management                  | Build, Run, Compose, Logs         |
| **kubernetes**  | Kubernetes-Operationen                | Kubectl, Helm, Cluster-Mgmt      |
| **memory**      | Persistenter Speicher zwischen Sessions | Remember, Recall, Forget        |
| **computeruse** | Desktop-Automatisierung               | Screenshot, Click, Type           |

### Extensions aktivieren

````yaml
extensions:
  developer: { enabled: true }
  github: { enabled: true }
  memory: { enabled: true }
````

### Eigene Extensions (als MCP-Server)

````yaml
extensions:
  my-custom-tool:
    type: stdio
    command: python
    args: ["/path/to/my_mcp_server.py"]
    env:
      MY_API_KEY: "${MY_API_KEY}"
````

---

## 6. MCP Integration

Goose hat **native MCP-Unterstützung** (Model Context Protocol). MCP-Server werden direkt als Extensions eingebunden.

````yaml
extensions:
  # Stdio-basierter MCP-Server
  filesystem:
    type: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/erlaubter/pfad"]

  # SSE-basierter MCP-Server
  remote-server:
    type: sse
    uri: "http://localhost:8080/sse"

  # Python MCP-Server
  database:
    type: stdio
    command: python
    args: ["-m", "mcp_server_sqlite", "--db", "/pfad/zur/db.sqlite"]
````

> [!success] MCP als First-Class-Feature
> - Kein separater Proxy oder Wrapper nötig
> - Automatische Tool-Discovery aus MCP-Servern
> - Unterstützung für `stdio` und `sse` Transport
> - Environment-Variablen pro Server konfigurierbar

---

## 7. Sandbox und Sicherheit

Goose implementiert mehrere Sicherheitsschichten gegen unkontrollierte Aktionen.

```
┌──────────────────────────────────┐
│     Benutzer-Bestätigung         │  ← Kritische Aktionen erfordern OK
├──────────────────────────────────┤
│   Sandbox-Ausführungsumgebung    │  ← Isolierte Befehlsausführung
├──────────────────────────────────┤
│   Dateisystem-Einschränkungen    │  ← Nur erlaubte Pfade
└──────────────────────────────────┘
```

````yaml
sandbox:
  allowed_paths:
    - "/home/user/projects"
    - "/tmp/goose-workspace"
  auto_approve:
    - "ls"
    - "cat"
    - "grep"
    - "git status"
    - "git diff"
````

> [!danger] Sicherheitshinweis
> Goose kann Shell-Befehle ausführen und Dateien ändern. Bei kritischen Operationen (`rm -rf`, `git push --force`) wird eine Bestätigung eingeholt. In Produktionsumgebungen mit Vorsicht einsetzen.

---

## 8. Wichtige Commands

### Basis-Befehle

````bash
goose session                    # Interaktive Session starten
goose run "Schreibe Tests"       # Einmaligen Task ausführen
goose configure                  # Konfiguration bearbeiten
goose --version                  # Version anzeigen
````

### Innerhalb einer Session

| Befehl / Shortcut    | Beschreibung                       |
| -------------------- | ---------------------------------- |
| `/help`              | Hilfe anzeigen                     |
| `/exit` / `Ctrl+C`   | Session beenden                    |
| `/clear`             | Konversationsverlauf löschen       |
| `/undo`              | Letzte Aktion rückgängig machen    |
| `/retry`             | Letzte Anfrage wiederholen         |
| `/extensions`        | Aktive Extensions anzeigen         |
| `/model`             | Modell anzeigen/wechseln           |

### Pipe und Automation

````bash
echo "Erkläre diese Datei" | goose run --file src/main.rs
goose run "Finde Bugs" --file src/handler.rs
goose run --non-interactive "Führe Tests aus und fixe Fehler"
````

---

## 9. Vergleich mit anderen CLI Agents

| Feature                | **Goose**        | Claude Code     | Gemini CLI      | Codex CLI       |
| ---------------------- | ---------------- | --------------- | --------------- | --------------- |
| **Sprache**            | Rust             | TypeScript      | Python/TS       | TypeScript      |
| **Entwickler**         | Block (Square)   | Anthropic       | Google          | OpenAI          |
| **Open Source**        | ✅ Apache 2.0    | ✅               | ✅               | ✅               |
| **MCP Support**        | ✅ Nativ         | ✅               | ✅               | ✅               |
| **Multi-Provider**     | ✅               | ❌ Nur Claude    | ❌ Nur Gemini    | ❌ Nur OpenAI    |
| **Performance**        | ⚡ Sehr schnell  | Schnell         | Schnell         | Schnell         |
| **Profile**            | ✅               | ❌               | ❌               | ❌               |
| **Lokale Modelle**     | ✅ (Ollama)      | ❌               | ❌               | ❌               |
| **GitHub Stars**       | ~30K             | ~40K            | ~50K            | ~20K            |

> [!note] Stärken von Goose
> - **Multi-Provider**: Flexibel zwischen LLM-Anbietern wechseln
> - **Rust-Performance**: Schnellster Agent im Vergleich
> - **Enterprise-Backing**: Langfristige Wartung durch Block
> - **Modulares Design**: Extensions für maßgeschneiderte Workflows
> - **Profile**: Verschiedene Konfigurationen für verschiedene Projekte

---

## 10. Tipps und Best Practices

### Projekt-Setup mit .goosehints

````bash
# .goosehints im Projekt-Root — ähnlich wie CLAUDE.md oder AGENTS.md
cat > .goosehints << 'EOF'
Dieses Projekt verwendet:
- Rust mit Cargo
- PostgreSQL als Datenbank
- Docker für lokale Entwicklung
- Tests: `cargo test` / Linting: `cargo clippy`
EOF
````

### Effektive Prompts

````bash
# Kontext mitgeben
goose run "Refactore die Auth-Logik in src/auth/ — nutze JWT statt Sessions"

# Schritt für Schritt
goose run "1. Lies die API-Struktur 2. Erstelle OpenAPI-Docs 3. Generiere Types"

# Mit Einschränkungen
goose run "Fixe den Bug in parser.rs — ändere nur parse_token()"
````

### Profile für verschiedene Aufgaben

````yaml
profiles:
  quick:    # Schnell und günstig
    provider: openai
    model: gpt-4o-mini
  complex:  # Best-in-Class für schwierige Aufgaben
    provider: anthropic
    model: claude-sonnet-4-20250514
````

### Memory nutzen

In einer Goose-Session kann man den Agenten bitten, sich projektspezifische Konventionen zu merken — diese bleiben über Sessions hinweg erhalten.

---

## Weiterführende Links

- **GitHub**: [block/goose](https://github.com/block/goose)
- **Dokumentation**: [block.github.io/goose](https://block.github.io/goose/)
- **Block**: [block.xyz](https://block.xyz)
- **MCP Spezifikation**: [modelcontextprotocol.io](https://modelcontextprotocol.io)

> [!abstract] Zusammenfassung
> Goose ist ein leistungsstarker, in Rust geschriebener AI-Coding-Agent von Block mit nativer MCP-Unterstützung, Multi-Provider-Flexibilität und modularem Extensions-System. Die Kombination aus Enterprise-Backing, Rust-Performance und Profil-System macht Goose zu einer exzellenten Wahl für Entwickler, die einen schnellen und flexiblen Terminal-Agenten suchen.

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Übersicht und Vergleich aller AI Coding Agents
- [[Was ist ein AI Agent]] — Grundlagen und Definition von AI Agents
- [[Custom Instructions für AI Coding Agents]] — Best Practices für Instruktionsdateien und Konfiguration
- [[Skills]] — Fähigkeiten und Skill-Systeme von AI Agents
