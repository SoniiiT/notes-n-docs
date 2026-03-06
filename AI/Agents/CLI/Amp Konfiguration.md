---
tags: [ai, agents, cli, amp, konfiguration]
---

# Amp Konfiguration

> [!ABSTRACT]
> Amp ist ein AI Coding Agent von Sourcegraph (ehemals Cody), der auf tiefe Codebase-Suche und Multi-Repository-Unterstützung spezialisiert ist. Er kombiniert Sourcegraphs Code-Graph-Technologie mit LLM-Modellen für präzise, kontextbewusste Code-Generierung und Refactoring im Terminal und IDE.

---

## Übersicht

| Eigenschaft | Details |
|---|---|
| **Entwickler** | Sourcegraph |
| **Typ** | CLI Agent + IDE Extension |
| **Lizenz** | Proprietär (Free Tier verfügbar) |
| **Kosten** | Free (mit Credits), Teams/Enterprise (Custom) |
| **Plattformen** | macOS, Linux, Windows (WSL) |
| **IDEs** | VS Code, JetBrains, Neovim, Zed |
| **Modelle** | Claude Opus 4.6, GPT-5.3 Codex, u.a. |
| **Custom Instructions** | `AGENT.md` |

---

## Installation

### CLI (empfohlen)

```bash
# Shell-Installer (macOS, Linux, WSL)
curl -fsSL https://ampcode.com/install.sh | bash

# Alternativ via npm
npm install -g @sourcegraph/amp
```

### IDE Extensions

- **VS Code**: Extension „Amp" im Marketplace installieren
- **JetBrains**: Plugin über den JetBrains Marketplace
- **Neovim**: Plugin über den Paketmanager

### Erster Start

```bash
# Amp starten (öffnet Browser zur Authentifizierung)
amp

# Mit API-Key authentifizieren
export AMP_API_KEY="your_key_here"
amp
```

---

## Architektur und Modi

Amp bietet verschiedene Modi, die je nach Aufgabe automatisch oder manuell gewählt werden:

| Modus | Beschreibung | Anwendungsfall |
|---|---|---|
| **Smart** | Ausgewogen zwischen Geschwindigkeit und Qualität | Standard für die meisten Aufgaben |
| **Rush** | Schnell und kostengünstig | Einfache Aufgaben, schnelle Fragen |
| **Deep** | Komplexes Reasoning, mehr Tokens | Schwierige Refactorings, Architekturentscheidungen |

```bash
# Modus wechseln in der CLI
amp --mode deep "Refactore die Auth-Middleware"

# Standard-Modus setzen
amp config set default-mode smart
```

---

## Konfiguration

### AGENT.md

Amp unterstützt `AGENT.md`-Dateien im Repository, um projektspezifische Anweisungen zu definieren:

```markdown
# AGENT.md

## Projekt-Kontext
- TypeScript Monorepo mit Turborepo
- PostgreSQL als Datenbank
- Jest für Tests

## Konventionen
- Verwende `pnpm` statt `npm`
- Alle Funktionen mit JSDoc dokumentieren
- Error Handling immer mit Custom Error Classes
```

### Umgebungsvariablen

| Variable | Beschreibung |
|---|---|
| `AMP_API_KEY` | API-Key für Authentifizierung |
| `AMP_DEFAULT_MODE` | Standard-Modus (smart/rush/deep) |
| `AMP_CONTEXT_LIMIT` | Max Token-Limit für Kontext |

---

## Kernfunktionen

### Code-Graph-Integration

Amp nutzt Sourcegraphs Code-Graph für **präzise Referenzauflösung** über das gesamte Repository und sogar über mehrere Repositories hinweg:

```bash
# Multi-Repo-Suche
amp "Wo wird die UserService Klasse überall verwendet?"

# Cross-Repo-Refactoring
amp "Benenne getUserById in findUserById um, in allen Repos"
```

### Code Review Agent

Amp kann Pull Requests automatisiert reviewen:

```bash
# PR reviewen
amp review --pr 42

# Review mit spezifischem Fokus
amp review --pr 42 --focus "security,performance"
```

### Thread-basierte Zusammenarbeit

Amp speichert Konversationen als **Threads**, die mit dem Team geteilt werden können:

```bash
# Thread erstellen und benennen
amp thread new "auth-refactoring"

# Thread fortsetzen
amp thread resume "auth-refactoring"

# Thread teilen (generiert Link)
amp thread share "auth-refactoring"
```

### Aufgaben-Management

```bash
# TODOs im Projekt verwalten
amp todos list
amp todos add "Rate Limiting implementieren"
amp todos done 3
```

---

## Preismodell

| Plan | Preis | Credits | Features |
|---|---|---|---|
| **Free** | $0/Mo | $10 Free-Credits, dann Pay-as-you-go | CLI, IDE, Docs, Discord |
| **Teams** | Custom | At-cost, Team-Billing | SSO, keine LLM-Datennutzung |
| **Enterprise** | ~$59/user/Mo | Custom | Zero LLM Retention, Compliance, Priority Support |

> [!NOTE]
> Amp verwendet ein **Credit-basiertes** Preismodell. Die Kosten hängen vom genutzten Modell und der Komplexität der Aufgabe ab. Deep-Modus verbraucht mehr Credits als Rush-Modus.

---

## Vergleich mit anderen CLI Agents

| Feature | Amp | Claude Code | GitHub Copilot CLI |
|---|---|---|---|
| **Multi-Repo-Support** | ✅✅ (Kernfeature) | ❌ | ❌ |
| **Code-Graph-Suche** | ✅✅ (Sourcegraph) | ❌ | ❌ |
| **Thread-Sharing** | ✅ | ❌ | ❌ |
| **Custom Instructions** | `AGENT.md` | `CLAUDE.md` | `copilot-instructions.md` |
| **Multi-Modell** | ✅ (3 Modi) | ❌ (nur Claude) | ❌ (nur GPT) |
| **MCP-Support** | ❌ | ✅ | ✅ |
| **Hooks/Lifecycle** | ❌ | ✅✅ | ❌ |

---

## Pro & Contra

| ✅ Vorteile | ❌ Nachteile |
|---|---|
| Beste Multi-Repo-Unterstützung im Markt | Credit-System kann teuer werden |
| Code-Graph für präzise Referenzen | Modellwahl durch Vendor vorgegeben |
| Thread-basierte Team-Zusammenarbeit | Weniger MCP/Hook-Features als Claude Code |
| CLI-first mit IDE-Integration | Kein vollständig lokaler Betrieb |
| Enterprise-Grade Security (SOC 2) | Free Tier relativ begrenzt |

---

## Tipps

> [!TIP]
> - **Rush-Modus** für einfache Fragen nutzen, um Credits zu sparen
> - **`AGENT.md`** im Repo pflegen für konsistente Ergebnisse
> - **Thread-Sharing** für Code Reviews im Team nutzen
> - Multi-Repo-Projekte profitieren am meisten von Amps Sourcegraph-Integration

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Vergleich aller AI Coding Agents
- [[Custom Instructions für AI Coding Agents]] — Instruction-Dateien für verschiedene Agents
- [[Skills]] — Skills-System für wiederverwendbare Workflows
