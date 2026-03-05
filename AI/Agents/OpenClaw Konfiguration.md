# OpenClaw Konfiguration

## Was ist OpenClaw?

OpenClaw ist ein quelloffenes, selbst gehostetes AI-Agent-Framework, das als Multi-Channel-Messaging-Gateway fungiert. Es verbindet verschiedene Messaging-Plattformen (Telegram, Discord, WhatsApp, Slack, Signal, iMessage) mit unterschiedlichen KI-Modellen und ermöglicht die Steuerung über ein Web-Dashboard, eine CLI oder direkt per Chat-Nachricht.

> [!NOTE]
> OpenClaw ist **kein Cloud-Dienst**, sondern läuft vollständig auf eigener Hardware. Es gibt keinen Vendor-Lock-in – alle Daten, Konversationen und Konfigurationen bleiben lokal.

**Kernfähigkeiten:**
- Multi-Channel-Messaging (Telegram, Discord, WhatsApp, Slack, Signal, iMessage)
- Multi-Agent-Support (Claude, OpenAI, Google Gemini, lokale LLMs via Ollama)
- Skills/Plugin-System mit eigenem Marketplace (ClawHub)
- Shell-Zugriff und Browser-Automatisierung
- Coding-Agent-Modus mit PTY-Support
- Datei-Operationen und System-Steuerung
- Web-Dashboard und CLI-Interface
- Session-Verwaltung mit persistenter Konversationshistorie

> [!DANGER]
> **Sicherheitswarnung:** OpenClaw gewährt standardmäßig vollen Shell-Zugriff auf das Host-System. Sicherheitsforscher von Netskope haben auf erhebliche Risiken hingewiesen. OpenClaw sollte **ausschließlich auf dedizierter, isolierter Hardware** betrieben werden. Siehe Abschnitt [Sicherheitshinweise](#sicherheitshinweise) für Details.

---

## Geschichte

OpenClaw hat eine bewegte Entstehungsgeschichte mit mehreren Umbenennungen:

| Zeitraum | Name | Beschreibung |
|---|---|---|
| Anfang 2025 | **Clawdbot** | Erster Prototyp als Telegram-Bot mit Claude-Anbindung |
| Mitte 2025 | **MoltBot** | Umbenennung und Erweiterung um Multi-Channel-Support |
| Ende 2025 | **OpenClaw** | Rebranding als vollständiges Open-Source-Framework |

Die Entwicklung begann als persönliches Projekt eines Entwicklers, der einen selbst gehosteten AI-Assistenten für Telegram wollte. Mit wachsender Community wurde das Projekt professionalisiert und als OpenClaw unter einer Open-Source-Lizenz veröffentlicht.

> [!TIP]
> In älteren Dokumentationen, Foren oder GitHub-Issues kann noch von „Clawdbot" oder „MoltBot" die Rede sein – gemeint ist dasselbe Projekt.

---

## Architektur

OpenClaw folgt einer modularen Architektur mit fünf Kernkomponenten:

```
┌──────────────────────────────────────────────────────────┐
│                    Web Dashboard / CLI                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────┐   ┌──────────────┐   ┌──────────────────┐  │
│  │ Gateway  │──▶│ Agent Pipeline│──▶│ Provider Bridge  │  │
│  │ (Kanäle) │   │ (Routing)     │   │ (LLM-Anbindung)  │  │
│  └─────────┘   └──────────────┘   └──────────────────┘  │
│       │               │                    │             │
│       ▼               ▼                    ▼             │
│  ┌─────────┐   ┌──────────────┐   ┌──────────────────┐  │
│  │ Session  │   │ Skills Engine│   │ Tool Executor    │  │
│  │ Store    │   │ (Plugins)    │   │ (Shell, Browser) │  │
│  └─────────┘   └──────────────┘   └──────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Komponenten im Detail

| Komponente | Funktion |
|---|---|
| **Gateway** | Empfängt Nachrichten von allen konfigurierten Kanälen (Telegram, Discord usw.) und normalisiert sie in ein einheitliches internes Format |
| **Agent Pipeline** | Routet Nachrichten zum passenden AI-Provider, wendet System-Prompts und Skills an |
| **Provider Bridge** | Abstraktion über verschiedene LLM-APIs (OpenAI, Anthropic, Google, Ollama) |
| **Skills Engine** | Lädt und verwaltet Skills (Plugins), die als Markdown/YAML-Dateien definiert werden |
| **Session Store** | Persistiert Konversationen, Kontexte und Benutzer-Einstellungen (SQLite oder PostgreSQL) |
| **Tool Executor** | Führt Shell-Befehle, Browser-Aktionen und Datei-Operationen im Auftrag des Agents aus |

---

## Installation

### Voraussetzungen

- Node.js >= 20.x
- npm oder Docker
- Mindestens ein LLM-API-Key (OpenAI, Anthropic, Google) oder lokales Ollama

### Installation per npm

```bash
# Global installieren
npm install -g openclaw@latest

# Version prüfen
openclaw --version

# Onboarding starten (erstellt Konfiguration und installiert Daemon)
openclaw onboard --install-daemon
```

Der `onboard`-Befehl führt durch die Ersteinrichtung:
1. Erstellt die Konfigurationsdatei `~/.openclaw/config.yaml`
2. Fragt nach API-Keys für die gewünschten LLM-Provider
3. Installiert den Daemon als systemd-Service (Linux) oder launchd (macOS)
4. Startet das Web-Dashboard auf `http://localhost:3400`

### Installation per Docker

```bash
# Docker-Image ziehen
docker pull ghcr.io/openclaw/openclaw:latest

# Container starten
docker run -d \
  --name openclaw \
  -p 3400:3400 \
  -v ~/.openclaw:/root/.openclaw \
  -e OPENCLAW_ADMIN_TOKEN=dein-sicheres-token \
  ghcr.io/openclaw/openclaw:latest
```

> [!WARNING]
> Wenn du OpenClaw in Docker mit Shell-Zugriff nutzt, hat der Agent Zugriff auf das **gesamte Container-Dateisystem**. Verwende isolierte Volumes und binde keine sensiblen Host-Verzeichnisse ein.

### Docker Compose

```yaml
version: "3.8"
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw
    restart: unless-stopped
    ports:
      - "3400:3400"
    volumes:
      - ./data:/root/.openclaw
      - ./skills:/root/.openclaw/skills
    environment:
      - OPENCLAW_ADMIN_TOKEN=${ADMIN_TOKEN}
      - OPENCLAW_ANTHROPIC_KEY=${ANTHROPIC_KEY}
      - OPENCLAW_OPENAI_KEY=${OPENAI_KEY}
    # Empfohlen: Sicherheits-Einschränkungen
    security_opt:
      - no-new-privileges:true
    read_only: false
    tmpfs:
      - /tmp
```

### Konfigurationsdatei

Die zentrale Konfiguration liegt unter `~/.openclaw/config.yaml`:

```yaml
server:
  port: 3400
  host: "0.0.0.0"
  admin_token: "dein-sicheres-token"

providers:
  anthropic:
    api_key: "sk-ant-..."
    default_model: "claude-sonnet-4-20250514"
  openai:
    api_key: "sk-..."
    default_model: "gpt-4o"
  ollama:
    endpoint: "http://localhost:11434"
    default_model: "llama3.1"

defaults:
  provider: "anthropic"
  max_tokens: 8192
  temperature: 0.7

session:
  store: "sqlite"            # oder "postgresql"
  database: "~/.openclaw/sessions.db"
  max_history: 100

security:
  shell_access: true
  allowed_commands: []       # leer = alles erlaubt
  sandbox_mode: false
  file_access:
    read: true
    write: true
    root_dir: "~"            # Basis-Verzeichnis für Datei-Zugriff
```

---

## Channel-Konfiguration

Kanäle werden in der Datei `~/.openclaw/channels.yaml` konfiguriert oder über das Web-Dashboard hinzugefügt.

### Telegram

```yaml
channels:
  telegram:
    enabled: true
    bot_token: "123456789:ABCdefGHIjklMNOpqrsTUVwxyz"
    allowed_users:
      - 12345678           # Telegram User-ID
      - 87654321
    provider: "anthropic"  # Provider-Override für diesen Kanal
    system_prompt: |
      Du bist ein hilfreicher Assistent.
      Antworte immer auf Deutsch.
```

> [!TIP]
> Die Telegram User-ID findest du, indem du dem Bot `@userinfobot` eine Nachricht schickst. Verwende **immer** `allowed_users`, um unbefugten Zugriff zu verhindern.

### Discord

```yaml
channels:
  discord:
    enabled: true
    bot_token: "MTIz..."
    application_id: "1234567890"
    allowed_guilds:
      - "9876543210"       # Discord Server-ID
    allowed_roles:
      - "admin"
      - "ai-user"
    command_prefix: "!"
    provider: "openai"
```

### WhatsApp (via WhatsApp Business API)

```yaml
channels:
  whatsapp:
    enabled: true
    phone_number_id: "1234567890"
    access_token: "EAAx..."
    verify_token: "mein-verify-token"
    webhook_url: "https://mein-server.de/webhook/whatsapp"
    allowed_numbers:
      - "+491234567890"
```

### Slack

```yaml
channels:
  slack:
    enabled: true
    bot_token: "xoxb-..."
    app_token: "xapp-..."
    signing_secret: "abc123..."
    allowed_channels:
      - "#ai-assistant"
      - "#development"
```

### Signal

```yaml
channels:
  signal:
    enabled: true
    signal_cli_path: "/usr/local/bin/signal-cli"
    phone_number: "+491234567890"
    allowed_contacts:
      - "+490987654321"
```

### Alle Kanäle auf einen Blick

| Kanal | Auth-Methode | Besonderheiten |
|---|---|---|
| Telegram | Bot-Token + User-IDs | Inline-Keyboards, Datei-Upload/-Download |
| Discord | Bot-Token + Rollen | Slash-Commands, Thread-Support |
| WhatsApp | Business API | Webhook-basiert, Template-Messages |
| Slack | Bot + App Token | Socket Mode, Slash-Commands |
| Signal | signal-cli | Ende-zu-Ende-verschlüsselt |
| iMessage | AppleScript Bridge | Nur auf macOS, experimentell |

---

## Skills und Plugin-System

Skills sind das Herzstück der Erweiterbarkeit von OpenClaw. Ein Skill ist eine Markdown- oder YAML-Datei, die Anweisungen, Werkzeuge und Kontext für den Agent definiert.

### Skill-Struktur

Ein Skill besteht aus einer einzelnen Datei unter `~/.openclaw/skills/`:

````yaml
# ~/.openclaw/skills/web-search.skill.yaml
name: "web-search"
version: "1.2.0"
description: "Websuche via DuckDuckGo oder Google"
author: "community"

triggers:
  - "suche nach"
  - "google mal"
  - "finde im web"

tools:
  - name: "web_search"
    type: "shell"
    command: "curl -s 'https://api.duckduckgo.com/?q={{query}}&format=json'"
    parameters:
      query:
        type: "string"
        required: true
        description: "Suchbegriff"

system_prompt: |
  Du hast Zugriff auf eine Websuche. Nutze das Tool `web_search`,
  um aktuelle Informationen zu finden. Fasse die Ergebnisse
  übersichtlich zusammen.
````

### Skill als Markdown

Skills können auch als Markdown geschrieben werden:

````markdown
---
name: home-automation
version: 1.0.0
triggers:
  - "licht"
  - "heizung"
  - "smarthome"
---

# Home Automation Skill

Du steuerst ein Smart Home über die Home Assistant API.

## Verfügbare Tools

### `ha_call_service`
Ruft einen Home Assistant Service auf.

```yaml
type: shell
command: |
  curl -s -X POST \
    -H "Authorization: Bearer {{ha_token}}" \
    -H "Content-Type: application/json" \
    -d '{"entity_id": "{{entity}}"}' \
    "http://homeassistant.local:8123/api/services/{{domain}}/{{service}}"
parameters:
  domain: { type: string, required: true }
  service: { type: string, required: true }
  entity: { type: string, required: true }
```

## Anweisungen

- Frage immer nach, bevor du Geräte ausschaltest
- Gib Temperaturwerte in Celsius an
- Bestätige jede Aktion mit einer Zusammenfassung
````

### ClawHub – Skill-Marketplace

ClawHub ist der zentrale Marketplace für Community-Skills:

```bash
# Skills suchen
openclaw skills search "home automation"

# Skill installieren
openclaw skills install clawhub/home-automation

# Installierte Skills auflisten
openclaw skills list

# Skill aktualisieren
openclaw skills update clawhub/home-automation

# Skill entfernen
openclaw skills remove clawhub/home-automation
```

> [!WARNING]
> Skills von Drittanbietern können beliebige Shell-Befehle enthalten. **Prüfe den Inhalt jedes Skills vor der Installation.** Installiere nur Skills von vertrauenswürdigen Autoren.

---

## Coding Agent Modus

OpenClaw enthält einen vollwertigen Coding-Agent-Modus mit PTY-Support (Pseudo-Terminal), der ähnlich wie Claude Code oder GitHub Copilot CLI funktioniert.

### Aktivierung

```bash
# Coding-Modus direkt starten
openclaw code

# Mit spezifischem Provider
openclaw code --provider anthropic --model claude-sonnet-4-20250514

# In einem bestimmten Verzeichnis
openclaw code --cwd /pfad/zum/projekt
```

### Funktionen im Coding-Modus

| Funktion | Beschreibung |
|---|---|
| **Datei lesen/schreiben** | Liest und bearbeitet Dateien im Projektverzeichnis |
| **Shell-Befehle** | Führt Build-, Test- und Lint-Befehle aus |
| **PTY-Support** | Volles Terminal mit interaktiver Ein-/Ausgabe |
| **Git-Integration** | Commits, Branches, Diffs und Pull Requests |
| **Multi-Datei-Edits** | Bearbeitet mehrere Dateien in einer Sitzung |
| **Kontext-Erkennung** | Erkennt Projekttyp, Sprache und Build-System automatisch |

### Coding-Agent über Chat-Kanal

Der Coding-Modus kann auch über jeden konfigurierten Chat-Kanal genutzt werden:

```
Benutzer (Telegram): /code Schreibe einen Unit-Test für die UserService-Klasse

OpenClaw: Ich analysiere das Projekt und erstelle den Test...
           ✅ test/UserService.test.ts erstellt (47 Zeilen)
           ✅ npm test – alle 23 Tests bestanden
```

---

## Multi-Agent Support

OpenClaw unterstützt die gleichzeitige Nutzung mehrerer LLM-Provider und kann je nach Kanal, Skill oder Aufgabe den passenden Agent auswählen.

### Provider-Konfiguration

```yaml
providers:
  anthropic:
    api_key: "sk-ant-..."
    default_model: "claude-sonnet-4-20250514"
    max_tokens: 8192
  openai:
    api_key: "sk-..."
    default_model: "gpt-4o"
    max_tokens: 4096
  google:
    api_key: "AIza..."
    default_model: "gemini-2.5-pro"
  ollama:
    endpoint: "http://localhost:11434"
    default_model: "llama3.1:70b"
    # Kein API-Key nötig – läuft lokal
```

### Routing-Strategien

```yaml
routing:
  # Standard-Provider für alle Kanäle
  default: "anthropic"

  # Kanal-spezifisches Routing
  channel_overrides:
    telegram: "anthropic"
    discord: "openai"
    slack: "google"

  # Skill-basiertes Routing
  skill_overrides:
    code-review: "anthropic"
    creative-writing: "openai"
    translation: "google"

  # Fallback-Kette bei Fehler/Rate-Limit
  fallback_chain:
    - "anthropic"
    - "openai"
    - "ollama"
```

### Lokale Modelle mit Ollama

```bash
# Ollama installieren (falls noch nicht vorhanden)
curl -fsSL https://ollama.ai/install.sh | sh

# Modell herunterladen
ollama pull llama3.1:70b

# In OpenClaw als Provider konfigurieren
openclaw config set providers.ollama.endpoint "http://localhost:11434"
openclaw config set providers.ollama.default_model "llama3.1:70b"
```

> [!TIP]
> Für maximale Privatsphäre kann OpenClaw vollständig offline mit Ollama betrieben werden – keine Daten verlassen den eigenen Server.

---

## Sicherheitshinweise

> [!DANGER]
> ### ⚠️ Kritische Sicherheitswarnung
>
> OpenClaw gewährt dem AI-Agent **vollen Shell-Zugriff** auf das Host-System. Sicherheitsforscher der Firma **Netskope** haben auf erhebliche Risiken hingewiesen:
>
> - **Arbitrary Code Execution:** Der Agent kann beliebige Befehle ausführen
> - **Datenexfiltration:** Zugriff auf alle Dateien im konfigurierten Basis-Verzeichnis
> - **Privilege Escalation:** Bei Root-Betrieb hat der Agent volle Systemkontrolle
> - **Supply-Chain-Angriffe:** Bösartige Skills können als Einfallstor dienen
> - **Prompt Injection:** Über Chat-Nachrichten eingeschleuste Befehle
>
> **OpenClaw sollte NIEMALS auf Produktivsystemen oder Systemen mit sensiblen Daten betrieben werden.**

### Empfohlene Sicherheitsmaßnahmen

| Maßnahme | Priorität | Beschreibung |
|---|---|---|
| Dedizierte Hardware | 🔴 Kritisch | OpenClaw nur auf isolierter Hardware oder VMs betreiben |
| Docker mit Einschränkungen | 🔴 Kritisch | Container ohne `--privileged`, mit `no-new-privileges` |
| Nicht als Root | 🔴 Kritisch | Einen dedizierten Benutzer ohne sudo-Rechte verwenden |
| `allowed_users` setzen | 🟡 Wichtig | Zugriff auf bekannte User-IDs beschränken |
| `allowed_commands` nutzen | 🟡 Wichtig | Erlaubte Shell-Befehle explizit whitelisten |
| Skills prüfen | 🟡 Wichtig | Jeden Skill-Quellcode vor Installation lesen |
| Netzwerk isolieren | 🟠 Empfohlen | Firewall-Regeln: nur nötige Ports und Ziele erlauben |
| Regelmäßige Updates | 🟠 Empfohlen | `npm update -g openclaw@latest` regelmäßig ausführen |
| Logging aktivieren | 🟠 Empfohlen | Alle Agent-Aktionen in Logdateien protokollieren |

### Sandbox-Modus aktivieren

```yaml
security:
  sandbox_mode: true
  allowed_commands:
    - "ls"
    - "cat"
    - "grep"
    - "find"
    - "git"
    - "npm"
    - "node"
    - "python3"
  denied_commands:
    - "rm -rf"
    - "sudo"
    - "chmod"
    - "curl"       # Verhindert Datenexfiltration
    - "wget"
  file_access:
    read: true
    write: true
    root_dir: "/home/openclaw/workspace"  # Eingeschränktes Verzeichnis
  network:
    allow_outbound: false   # Kein Internetzugriff für den Agent
```

### Docker-Härtung

```bash
docker run -d \
  --name openclaw \
  --security-opt no-new-privileges:true \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --memory 2g \
  --cpus 1.5 \
  --network openclaw-net \
  -p 3400:3400 \
  -v ./data:/root/.openclaw:rw \
  ghcr.io/openclaw/openclaw:latest
```

---

## Vergleich: OpenClaw vs. klassische CLI Agents

| Eigenschaft | OpenClaw | Claude Code | GitHub Copilot CLI | Codex CLI |
|---|---|---|---|---|
| **Typ** | Multi-Channel Gateway | Terminal Agent | Terminal Agent | Terminal Agent |
| **Self-Hosted** | ✅ Vollständig | ❌ Cloud | ❌ Cloud | ❌ Cloud |
| **Messaging-Kanäle** | 6+ (Telegram, Discord, …) | ❌ Nur Terminal | ❌ Nur Terminal | ❌ Nur Terminal |
| **Multi-Provider** | ✅ Claude, GPT, Gemini, Ollama | ❌ Nur Claude | ✅ Claude + GPT | ❌ Nur OpenAI |
| **Lokale LLMs** | ✅ Via Ollama | ❌ | ❌ | ❌ |
| **Plugin-System** | ✅ ClawHub | ❌ | ❌ | ❌ |
| **Coding-Agent** | ✅ | ✅ | ✅ | ✅ |
| **Web-Dashboard** | ✅ | ❌ | ❌ | ❌ |
| **Kosten** | Kostenlos (+ API-Kosten) | API-Kosten | GitHub-Lizenz | API-Kosten |
| **Sicherheitsrisiko** | 🔴 Hoch (Shell-Zugriff) | 🟡 Mittel | 🟡 Mittel | 🟡 Mittel |
| **Offline-Betrieb** | ✅ Mit Ollama | ❌ | ❌ | ❌ |

### Wann OpenClaw wählen?

- Du willst einen AI-Assistenten über **Telegram, Discord oder WhatsApp** nutzen
- Du brauchst **volle Kontrolle** über Daten und Infrastruktur
- Du möchtest **verschiedene LLM-Provider** kombinieren
- Du willst **offline** mit lokalen Modellen arbeiten
- Du brauchst ein **Plugin-System** für wiederkehrende Aufgaben

### Wann klassische CLI Agents bevorzugen?

- Du arbeitest ausschließlich im **Terminal**
- Du willst eine **einfache Einrichtung** ohne Server-Betrieb
- **Sicherheit** hat höchste Priorität (weniger Angriffsfläche)
- Du brauchst keinen Multi-Channel-Support

---

## Tipps und Best Practices

### 1. Sicheres Setup von Anfang an

```bash
# Dedizierten Benutzer erstellen
sudo useradd -m -s /bin/bash openclaw-user
sudo su - openclaw-user

# Installation als unprivilegierter Benutzer
npm install -g openclaw@latest
openclaw onboard --install-daemon

# Workspace-Verzeichnis anlegen
mkdir -p ~/workspace
```

### 2. Systemd-Service härten

```ini
# /etc/systemd/system/openclaw.service
[Unit]
Description=OpenClaw AI Agent
After=network.target

[Service]
Type=simple
User=openclaw-user
Group=openclaw-user
WorkingDirectory=/home/openclaw-user
ExecStart=/home/openclaw-user/.npm-global/bin/openclaw serve
Restart=on-failure
RestartSec=10

# Sicherheits-Härtung
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/openclaw-user/.openclaw
PrivateTmp=true
ProtectKernelTunables=true
ProtectControlGroups=true

[Install]
WantedBy=multi-user.target
```

### 3. Nützliche CLI-Befehle

```bash
# Status prüfen
openclaw status

# Logs anzeigen (live)
openclaw logs --follow

# Konfiguration validieren
openclaw config check

# Einzelnen Kanal testen
openclaw test-channel telegram

# Skill debuggen
openclaw skills test web-search --input "Wetter in Berlin"

# Alle Sessions anzeigen
openclaw sessions list

# Bestimmte Session fortsetzen
openclaw sessions resume <session-id>

# Provider-Verbindung testen
openclaw providers ping anthropic
```

### 4. Monitoring und Logging

```yaml
logging:
  level: "info"              # debug, info, warn, error
  file: "~/.openclaw/logs/openclaw.log"
  max_size: "50mb"
  max_files: 10
  log_agent_actions: true    # Alle Agent-Aktionen protokollieren
  log_messages: false        # Chat-Nachrichten NICHT loggen (Privatsphäre)

monitoring:
  metrics_endpoint: "/metrics"  # Prometheus-kompatibel
  health_check: "/health"
```

### 5. Backup-Strategie

```bash
# Tägliches Backup der Konfiguration und Sessions
#!/bin/bash
BACKUP_DIR="/backup/openclaw/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

# Konfiguration sichern
cp -r ~/.openclaw/config.yaml "$BACKUP_DIR/"
cp -r ~/.openclaw/channels.yaml "$BACKUP_DIR/"
cp -r ~/.openclaw/skills/ "$BACKUP_DIR/skills/"

# Session-Datenbank sichern
sqlite3 ~/.openclaw/sessions.db ".backup '$BACKUP_DIR/sessions.db'"

echo "Backup erstellt: $BACKUP_DIR"
```

### 6. Performance-Tipps

> [!TIP]
> - Verwende **SQLite** für Einzelbenutzer-Setups und **PostgreSQL** ab 3+ gleichzeitigen Kanälen
> - Setze `max_history` auf einen sinnvollen Wert (50–100), um den Kontext nicht zu überladen
> - Nutze die **Fallback-Kette**, um bei Rate-Limits automatisch auf alternative Provider zu wechseln
> - Lokale Modelle über Ollama sind langsamer, verursachen aber keine API-Kosten

### 7. Skill-Entwicklung

Eigene Skills schreiben:

1. Erstelle eine `.skill.yaml`-Datei unter `~/.openclaw/skills/`
2. Definiere `name`, `triggers` und `tools`
3. Teste mit `openclaw skills test <name>`
4. Optional: Veröffentliche auf ClawHub mit `openclaw skills publish`

```bash
# Neuen Skill scaffolden
openclaw skills create mein-skill

# Ergebnis: ~/.openclaw/skills/mein-skill.skill.yaml
```

---

> [!ABSTRACT] Zusammenfassung
> OpenClaw ist ein mächtiges, selbst gehostetes AI-Agent-Framework mit Multi-Channel-Support und Plugin-System. Die Stärken liegen in der Flexibilität und dem Local-First-Ansatz. Die größte Schwäche ist das inhärente Sicherheitsrisiko durch den vollen Shell-Zugriff. Ein sorgfältiges Setup mit Isolation, Berechtigungseinschränkungen und regelmäßigen Updates ist **zwingend erforderlich**.
