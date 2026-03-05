# Warp Konfiguration

> [!ABSTRACT]
> Warp ist ein **AI-nativer Terminal-Emulator**, der die klassische Kommandozeile mit modernen IDE-Features und KI-Integration verbindet. Statt wie andere AI Coding Agents Code zu generieren, modernisiert Warp das Terminal selbst — mit Block-basierter Ausgabe, AI-Befehlsgenerierung und Team-Collaboration-Features.

---

## Übersicht

| Eigenschaft | Details |
|---|---|
| **Entwickler** | Warp, Inc. |
| **Typ** | AI Terminal Emulator |
| **Lizenz** | Proprietär (Free Tier verfügbar) |
| **Kosten** | Free ($0/Mo), Build ($18/Mo), Max ($180/Mo), Business ($45/user/Mo) |
| **Plattformen** | macOS, Windows, Linux |
| **Modelle** | OpenAI, Anthropic, Gemini (BYOK auf Build+) |
| **Shell-Support** | Bash, Zsh, Fish, PowerShell |

> [!IMPORTANT]
> Warp ist **kein AI Coding Agent** im klassischen Sinne wie Claude Code oder Aider. Es ist ein Terminal-Emulator mit KI-Features. Es ersetzt nicht deinen Coding-Agent, sondern das Terminal, in dem du ihn ausführst.

---

## Installation

### macOS

```bash
# Via Homebrew
brew install --cask warp
```

### Windows & Linux

Download von [warp.dev](https://www.warp.dev) oder über den jeweiligen Paketmanager.

### Erster Start

1. Warp öffnen
2. Account erstellen (erforderlich für AI-Features)
3. Shell auswählen (Bash, Zsh, Fish, PowerShell)
4. Optional: Theme und Keybindings anpassen

---

## Block-basierte Ausgabe

Das revolutionäre Feature von Warp: Jeder Befehl und seine Ausgabe bilden einen **Block**:

```
┌─────────────────────────────────────────────┐
│ $ git log --oneline -5                      │  ← Input
├─────────────────────────────────────────────┤
│ a1b2c3d Fix authentication bug              │
│ d4e5f6g Add user profile page               │  ← Output
│ h7i8j9k Update dependencies                 │
│ ...                                         │
└─────────────────────────────────────────────┘
```

### Block-Aktionen

| Aktion | Beschreibung |
|---|---|
| **Kopieren** | Einzelnen Block-Output kopieren statt manuell selektieren |
| **Teilen** | Permalink zu einem Block für Debugging-Hilfe generieren |
| **Bearbeiten** | Vorherigen Befehl direkt im Block editieren und erneut ausführen |
| **Suchen** | Durch Block-Historie filtern und navigieren |

---

## AI-Features

### Befehlsgenerierung

```bash
# Natürliche Sprache → Shell-Befehl
# Eingabe: "Finde alle Dateien größer als 50MB"
# Warp generiert: find . -type f -size +50M

# Eingabe: "Zeige den Speicherverbrauch sortiert nach Größe"
# Warp generiert: du -sh * | sort -rh
```

### Error-Erklärung

Wenn ein Befehl fehlschlägt, kann Warp den Fehler erklären und einen Fix vorschlagen:

```bash
$ docker build -t myapp .
# Error: failed to solve: dockerfile parse error

# Warp AI erklärt:
# "Der Fehler liegt in Zeile 12 deines Dockerfiles.
#  'RUN apt install' braucht das '-y' Flag für
#  nicht-interaktive Installation."
```

### Agent Mode

Der Agent Mode versteht Projektkontext und hilft bei komplexeren Aufgaben:

```bash
# Projekt-basierte Hilfe
warp agent "Wie deploye ich diese Node.js App auf AWS?"

# Workflow-Automatisierung
warp agent "Erstelle ein Shell-Skript für das tägliche Backup"
```

---

## Warp Drive — Team-Collaboration

Warp Drive ist eine Cloud-basierte Plattform für Team-Zusammenarbeit:

| Feature | Beschreibung |
|---|---|
| **Shared Commands** | Häufig genutzte Befehle im Team teilen |
| **Workflows** | Parametrisierte Befehlssequenzen speichern und teilen |
| **Environments** | Umgebungskonfigurationen zentral verwalten |
| **Live Sessions** | Terminal-Sessions in Echtzeit teilen |

### Workflow-Beispiel

```yaml
# Warp Workflow: deploy-staging
name: Deploy to Staging
command: |
  git checkout {{branch}}
  npm run build
  docker build -t myapp:{{tag}} .
  docker push myapp:{{tag}}
  kubectl set image deployment/myapp myapp=myapp:{{tag}}
parameters:
  branch:
    description: "Branch to deploy"
    default: "develop"
  tag:
    description: "Docker image tag"
    default: "staging-latest"
```

---

## Konfiguration

### Themes und Keybindings

- **Themes**: Umfangreiche Theme-Unterstützung (ähnlich VS Code)
- **Keybindings**: Vim-Mode, Emacs-Mode oder Custom Mappings
- **Schriftarten**: Nerd Fonts und Ligature-Support

### AI-Provider (BYOK)

Ab dem Build-Plan können eigene API-Keys hinterlegt werden:

```
Settings → AI → Bring Your Own Key
→ OpenAI API Key
→ Anthropic API Key
→ Google Gemini API Key
```

### Privacy-Einstellungen

| Einstellung | Beschreibung |
|---|---|
| **Telemetry** | Ein/Aus schaltbar |
| **AI Data** | Opt-out für AI-Training möglich |
| **Session History** | Lokal oder Cloud-gespeichert |

---

## Preismodell

| Plan | Preis | AI Credits/Mo | Codebases | Workflows | Collaboration |
|---|---|---|---|---|---|
| **Free** | $0/Mo | 75 | 3 (3k Files) | 10 | Basic |
| **Build** | $18/Mo | 1.500 | 40 (100k Files) | Unbegrenzt | Unbegrenzt |
| **Max** | $180/Mo | 18.000 | 40 | Unbegrenzt | Unbegrenzt |
| **Business** | $45/user/Mo | 1.500/user | 40 | Unbegrenzt | Team-Admin |
| **Enterprise** | Custom | Custom | Unbegrenzt | Unbegrenzt | SAML, RBAC |

> [!NOTE]
> Neue Accounts erhalten in den ersten 2 Monaten **300 Credits/Mo** statt 75 im Free Plan.

---

## Pro & Contra

| ✅ Vorteile | ❌ Nachteile |
|---|---|
| Revolutionäres Block-basiertes Terminal | Account + Internet für AI-Features nötig |
| GPU-beschleunigte UI, extrem schnell | Privacy-Bedenken durch Cloud-AI |
| Team-Collaboration mit Warp Drive | Kein AI Coding Agent (generiert keinen Code in Dateien) |
| AI-Befehlsgenerierung spart Zeit | Advanced Features nur in Bezahl-Plänen |
| Cross-Platform (macOS, Windows, Linux) | Noch nicht so verbreitet wie iTerm2/Alacritty |
| Multi-Cursor, Syntax Highlighting im Terminal | — |

---

## Warp + AI Coding Agent: Die Kombination

Warp ergänzt AI Coding Agents optimal:

```bash
# Warp als Terminal + Claude Code als Agent
warp                          # Terminal starten
claude                        # Claude Code darin ausführen

# Warp AI für Shell-Befehle
# Claude Code für Code-Generierung
# = Beste Kombination beider Welten
```

> [!TIP]
> Warp ist **kein Ersatz** für Claude Code, Gemini CLI oder Copilot CLI, sondern eine **Ergänzung**. Nutze Warp als moderneres Terminal, in dem du deine AI Coding Agents ausführst. Die AI-Features von Warp helfen bei Shell-Befehlen und DevOps-Tasks.

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Vergleich aller AI Coding Agents
- [[Claude Code Konfiguration und Hooks]] — AI Agent, der gut in Warp läuft
- [[Gemini CLI Konfiguration]] — Weiterer CLI Agent
