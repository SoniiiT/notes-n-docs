---
tags: [ai, agents, konzept, custom-instructions]
---

# Custom Instructions für AI Coding Agents

## Einleitung

**Custom Instructions** (auch: System-Instruktionen, Coding-Richtlinien) sind projektspezifische Anweisungen, die AI Coding Agents automatisch beim Start einer Session laden. Sie definieren Konventionen, Verbote, bevorzugte Technologien und wiederkehrende Workflows – damit der Agent von Anfang an so arbeitet, als hätte er das Projekt-Onboarding durchlaufen.

Während jeder Agent sein eigenes Dateiformat mitbringt, konvergiert das Ökosystem zunehmend auf gemeinsame Standards. Dieses Dokument erklärt, welche Instruction-Dateien es gibt, wie sie zusammenwirken und wie man ein Setup aufbaut, das **alle gängigen Agents gleichzeitig** bedient.

> [!IMPORTANT]
> Custom Instructions sind kein Ersatz für gute Dokumentation. Sie sind eine **Brücke** zwischen der Projektdokumentation und dem Agent. Der Agent liest sie bei jedem Start – daher sollten sie kurz, präzise und aktuell sein.

---

## Übersicht: Instruction-Dateien pro Agent

Jeder AI Coding Agent hat eigene Konventionen, wo und wie Instruktionen abgelegt werden:

| Agent | Datei(en) | Speicherorte | Format |
|---|---|---|---|
| **Claude Code** | `CLAUDE.md` | `~/.claude/`, Projekt-Root, Subdirectories | Markdown |
| **GitHub Copilot** | `copilot-instructions.md`, `AGENTS.md`, `*.instructions.md` | `.github/`, Projekt-Root, `$HOME/.copilot/` | Markdown |
| **Gemini** | `GEMINI.md` | Projekt-Root, `cwd` | Markdown |
| **Cursor** | `.cursorrules`, `.cursor/rules/*.mdc` | Projekt-Root | Markdown / MDC |
| **Windsurf** | `.windsurfrules` | Projekt-Root | Markdown |
| **Aider** | `.aider.conf.yml`, `CONVENTIONS.md` | Projekt-Root, `$HOME` | YAML / Markdown |

### Claude Code

Claude Code nutzt ein **hierarchisches System** mit drei Ebenen:

```
~/.claude/CLAUDE.md          → Global (alle Projekte)
./CLAUDE.md                  → Projekt-Root (gesamtes Projekt)
./src/backend/CLAUDE.md      → Subdirectory (modul-spezifisch)
```

Alle gefundenen `CLAUDE.md`-Dateien werden **kombiniert** in den Kontext geladen. Subdirectory-Dateien werden dynamisch geladen, wenn Claude in dem jeweiligen Verzeichnis arbeitet.

### GitHub Copilot (CLI & Editor)

Copilot liest Instruktionen aus **mehreren Quellen** gleichzeitig:

```
$HOME/.copilot/copilot-instructions.md          → Global (alle Projekte)
.github/copilot-instructions.md                 → Repo-Level
AGENTS.md                                       → Universal (Git-Root & cwd)
CLAUDE.md                                       → Kompatibilität (Git-Root & cwd)
GEMINI.md                                       → Kompatibilität (Git-Root & cwd)
.github/instructions/**/*.instructions.md       → Scoped Instructions (Glob-Patterns)
```

Zusätzlich können weitere Verzeichnisse über die Umgebungsvariable `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` eingebunden werden.

> [!NOTE]
> GitHub Copilot liest auch `CLAUDE.md` und `GEMINI.md` – man kann also mit einer einzigen `AGENTS.md` oder den agent-spezifischen Dateien mehrere Tools gleichzeitig bedienen.

### Gemini

Gemini liest `GEMINI.md` im Git-Root und im aktuellen Arbeitsverzeichnis. Das Verhalten ist vergleichbar mit Claude Code, allerdings ohne das hierarchische Subdirectory-System.

### Cursor

Cursor unterstützt zwei Formate:

- **`.cursorrules`** – Eine einzelne Datei im Projekt-Root (Legacy-Format)
- **`.cursor/rules/*.mdc`** – Mehrere Regeldateien mit Frontmatter-Metadaten (neues Format)

Das MDC-Format erlaubt es, Regeln mit Glob-Patterns an bestimmte Dateitypen zu binden:

````markdown
---
description: Python-Konventionen
globs: ["**/*.py"]
---

- Verwende Type Hints für alle Funktionen
- Nutze `ruff` statt `flake8` für Linting
- Pydantic v2 für alle Datenmodelle
````

### Windsurf

Windsurf liest eine einzelne `.windsurfrules`-Datei im Projekt-Root. Das Format ist einfaches Markdown ohne Metadaten.

### Aider

Aider verwendet zwei Mechanismen:

- **`.aider.conf.yml`** – Konfigurationsdatei (technische Einstellungen)
- **`CONVENTIONS.md`** – Coding-Konventionen im Markdown-Format (wird per `read`-Direktive geladen)

```yaml
# .aider.conf.yml
read:
  - CONVENTIONS.md
  - docs/architecture.md
```

---

## AGENTS.md — Der universelle Standard

`AGENTS.md` entwickelt sich zum **plattformübergreifenden Standard** für Custom Instructions. GitHub Copilot liest diese Datei nativ, und andere Tools können sie ebenfalls unterstützen.

### Warum AGENTS.md?

| Eigenschaft | Vorteil |
|---|---|
| Agent-agnostisch | Funktioniert mit Copilot und potenziell weiteren Tools |
| Einfach zu pflegen | Eine Datei statt vieler agent-spezifischer Dateien |
| Versioniert in Git | Teil des Repos, für das ganze Team sichtbar |
| Markdown-Standard | Keine proprietären Formate oder Frontmatter nötig |

### Empfohlene Struktur

````markdown
# Project Instructions

## Overview
[Kurze Projektbeschreibung – Was macht das Projekt, welchen Stack nutzt es]

## Architecture
[Wichtige Architekturentscheidungen, die der Agent kennen muss]

## Conventions
- [Code-Style, Naming, Patterns]
- [Test-Konventionen]
- [Git-Workflow]

## Commands
```bash
npm run build       # Projekt bauen
npm run test        # Tests ausführen
npm run lint        # Linting
```

## Rules
- NIEMALS Secrets in Code-Dateien schreiben
- NIEMALS Dependencies ohne explizite Aufforderung installieren
- Alle Funktionen müssen explizite Return-Types haben
````

> [!TIP]
> Verwende `AGENTS.md` als **gemeinsame Basis** und lege nur agent-spezifische Abweichungen in `CLAUDE.md`, `GEMINI.md` oder `.cursorrules` ab. So vermeidest du Redundanz.

### Platzierung

`AGENTS.md` wird im **Git-Root** und/oder im **aktuellen Arbeitsverzeichnis** gelesen. Für Monorepos kann man mehrere `AGENTS.md`-Dateien in Subpackages ablegen:

```
monorepo/
├── AGENTS.md                  # Globale Instruktionen
├── packages/
│   ├── frontend/
│   │   └── AGENTS.md          # Frontend-spezifisch
│   └── backend/
│       └── AGENTS.md          # Backend-spezifisch
```

---

## .github/instructions/ — Scoped Instructions

GitHub Copilot unterstützt **dateitypspezifische Instruktionen** über das Verzeichnis `.github/instructions/`. Dateien in diesem Verzeichnis müssen dem Pattern `*.instructions.md` folgen und können per Frontmatter auf bestimmte Dateien eingeschränkt werden.

### Funktionsweise

```
.github/instructions/
├── python.instructions.md         # Gilt für *.py
├── typescript.instructions.md     # Gilt für *.ts, *.tsx
├── terraform.instructions.md      # Gilt für *.tf
└── testing.instructions.md        # Gilt für *.test.*, *.spec.*
```

### Format mit Glob-Pattern

Jede Instruction-Datei kann ein `applyTo`-Frontmatter-Feld enthalten, das bestimmt, für welche Dateien die Instruktionen gelten:

````markdown
---
applyTo: "**/*.py"
---

# Python-Konventionen

- Type Hints sind Pflicht (mypy strict)
- Pydantic v2 für alle Datenmodelle
- `ruff` für Linting und Formatierung
- Docstrings im Google-Format
- Strukturiertes Logging mit `structlog`, nie `print()`
````

### Beispiel: TypeScript-Instruktionen

````markdown
---
applyTo: "**/*.{ts,tsx}"
---

# TypeScript-Konventionen

- Strict Mode ist aktiviert – kein `any` verwenden
- Bevorzuge `interface` über `type` für Objekttypen
- React-Komponenten als Function Components mit expliziten Props-Types
- Barrel-Exports (`index.ts`) für jedes Feature-Modul
- Alle async-Funktionen mit Error-Boundary wrappen
````

### Beispiel: Test-Instruktionen

````markdown
---
applyTo: "**/*.{test,spec}.{ts,tsx,js}"
---

# Test-Konventionen

- Verwende `describe` / `it` Blocks (kein `test()`)
- AAA-Pattern: Arrange, Act, Assert
- Mocks immer in `beforeEach` aufräumen
- Keine Magic Numbers – Testdaten als Konstanten definieren
- Integration-Tests nur in `tests/integration/`
````

> [!IMPORTANT]
> Scoped Instructions werden **zusätzlich** zu den globalen Instruktionen geladen, nicht stattdessen. Sie ergänzen die Instruktionen aus `AGENTS.md`, `copilot-instructions.md` etc. mit dateityp-spezifischem Kontext.

---

## copilot-instructions.md

Die `copilot-instructions.md` ist die **Copilot-native** Instruction-Datei, die auf zwei Ebenen existiert:

### Global (alle Projekte)

```
$HOME/.copilot/copilot-instructions.md
```

Hier gehören persönliche Präferenzen rein, die für alle Projekte gelten:

```markdown
# Globale Copilot-Instruktionen

## Sprache
- Antworte auf Deutsch, es sei denn, der Code-Kontext ist Englisch
- Kommentare im Code immer auf Englisch

## Allgemeine Regeln
- Bevorzuge typsichere Lösungen
- Erkläre nicht offensichtliche Dinge – ich bin Senior-Entwickler
- Minimale Änderungen: Ändere nur, was nötig ist
```

### Repository-Level

```
.github/copilot-instructions.md
```

Hier gehören projekt-spezifische Instruktionen rein, die das gesamte Team betreffen:

```markdown
# Copilot-Instruktionen für dieses Repository

## Stack
- Python 3.12, FastAPI, SQLAlchemy 2.0
- PostgreSQL 16, Redis 7
- pytest für Tests, ruff für Linting

## Konventionen
- Repository Pattern für alle DB-Zugriffe
- Pydantic v2 Models in `app/schemas/`
- Alembic für Migrations
```

> [!NOTE]
> Die `copilot-instructions.md` im `.github/`-Verzeichnis wird oft auch von GitHub Copilot im Editor (VS Code, JetBrains) gelesen – nicht nur von der CLI. Damit ist sie der beste Ort für teamweite Instruktionen.

---

## Umgebungsvariable: COPILOT_CUSTOM_INSTRUCTIONS_DIRS

Für Teams, die Instruktionen an einem zentralen Ort pflegen oder mehrere Repos mit denselben Regeln versorgen wollen, bietet Copilot die Umgebungsvariable `COPILOT_CUSTOM_INSTRUCTIONS_DIRS`:

```bash
# In ~/.bashrc oder ~/.zshrc
export COPILOT_CUSTOM_INSTRUCTIONS_DIRS="$HOME/.config/team-instructions:$HOME/.config/my-rules"
```

Copilot liest dann alle Instruction-Dateien aus den angegebenen Verzeichnissen zusätzlich zu den Standardquellen.

---

## Claude Code: CLAUDE.md Praxis-Guide

Die `CLAUDE.md` ist das mächtigste Instruction-Format im Ökosystem — dank hierarchischem Laden und Subdirectory-Support. Dieser Abschnitt zeigt, wie man sie optimal strukturiert.

### Empfohlene Struktur

````markdown
# [Projektname]

## Projektübersicht
[1-3 Sätze: Was macht dieses Projekt, welches Problem löst es]

## Technologie-Stack
[Liste aller relevanten Technologien mit Versionen]

## Projektstruktur
[Verzeichnisbaum der wichtigsten Verzeichnisse]

## Entwicklungsumgebung
[Setup-Befehle, wichtige Make-Targets]

## Konventionen und Standards
[Code-Style, Naming, Architektur-Prinzipien]

## Häufige Aufgaben
[Wiederkehrende Tasks als ausführbare Snippets]

## Verbotene Aktionen
[Was Claude NICHT tun darf – mit Begründung]
````

### Was sollte rein / Was nicht?

| ✅ Sollte rein | ❌ Sollte NICHT rein |
|---|---|
| Stack mit Versionen (Node.js 20, TypeScript 5.4) | API-Keys, Tokens, Passwörter |
| Projekt-spezifische Konventionen | Generische Best Practices (kennt Claude bereits) |
| Verbotene Aktionen mit Begründung | Komplette Dokumentationen (zu viel Rauschen) |
| Häufige Aufgaben als Bash-Snippets | Veraltete Informationen |
| Fallstricke und Known Bugs | Mehr als ~500 Zeilen (lieber aufteilen) |

### Vollständiges Beispiel: DevOps-Projekt

````markdown
# infra-platform – Infrastructure Management Service

## Projektübersicht
infra-platform ist das zentrale Infrastruktur-Verwaltungssystem für alle
Cloud-Ressourcen. Es orchestriert Terraform-Deployments, verwaltet
Kubernetes-Cluster via ArgoCD und stellt ein Self-Service-Portal bereit.

## Technologie-Stack
- **Sprache:** Python 3.12 + FastAPI 0.111
- **IaC:** Terraform 1.8.x (AWS + GCP)
- **Container:** Docker + Kubernetes 1.29
- **GitOps:** ArgoCD 2.10
- **Tests:** pytest 8.x + moto (AWS Mocking)
- **Linting:** ruff + mypy (strict)

## Projektstruktur
```
infra-platform/
├── api/                    # FastAPI Application
│   ├── routers/            # Route-Handler nach Domäne
│   ├── services/           # Business Logic
│   └── models/             # Pydantic Models
├── terraform/
│   ├── modules/            # Wiederverwendbare TF-Module
│   └── environments/       # dev/, staging/, prod/
├── k8s/
│   ├── base/               # Kustomize Base Configs
│   └── overlays/           # Umgebungs-spezifische Overrides
└── tests/
    ├── unit/
    └── integration/
```

## Wichtige Befehle
```bash
make dev-setup          # Virtualenv + Dependencies
make test               # Unit + Integration Tests
make lint               # ruff + mypy
make tf-plan ENV=dev    # Terraform Plan
```

## Code-Konventionen
- Type hints Pflicht (mypy strict), Pydantic v2 für Datenmodelle
- Repository Pattern für alle DB-Zugriffe
- Strukturiertes Logging mit `structlog`, nie `print()`
- Conventional Commits: feat:, fix:, chore:, infra:

## VERBOTENE AKTIONEN
- NIEMALS `terraform apply` direkt auf prod – immer CI/CD
- NIEMALS AWS-Credentials in Code oder Logs
- NIEMALS `kubectl delete` auf Prod ohne Backup
- NIEMALS Dependencies direkt hinzufügen – immer `make lock`

## Bekannte Fallstricke
- `terraform/modules/eks/` hat Known Bug mit IAM Roles bei Node Groups
  → Workaround: `lifecycle { ignore_changes = [node_groups] }`
- pytest-asyncio muss exakt 0.23.x sein – höhere Versionen brechen Tests
````

> [!TIP]
> **Optimal: 100–300 Zeilen.** Unterverzeichnis-CLAUDE.md für modul-spezifische Details nutzen. Regelmäßig aktualisieren — veraltete Infos verwirren mehr als sie helfen.

---

## Praktisches Beispiel: Multi-Agent-kompatibles Setup

Das folgende Setup funktioniert mit **allen gängigen AI Coding Agents gleichzeitig**:

```
project/
├── AGENTS.md                        # Universal (Copilot, potenziell weitere)
├── CLAUDE.md                        # Claude Code spezifisch
├── GEMINI.md                        # Gemini spezifisch
├── .cursorrules                     # Cursor spezifisch
├── .windsurfrules                   # Windsurf spezifisch
├── .github/
│   ├── copilot-instructions.md      # Copilot repo-level
│   └── instructions/
│       ├── python.instructions.md   # Copilot: nur für *.py
│       └── typescript.instructions.md # Copilot: nur für *.ts
├── .aider.conf.yml                  # Aider Konfiguration
├── src/
│   ├── backend/
│   │   └── CLAUDE.md                # Claude: Backend-Kontext
│   └── frontend/
│       └── CLAUDE.md                # Claude: Frontend-Kontext
└── CONVENTIONS.md                   # Aider (per read-Direktive)
```

### Strategie: Gemeinsame Basis + Agent-spezifisch

**`AGENTS.md`** enthält die universellen Regeln (wird von Copilot direkt gelesen, und kann per Symlink oder Copy für andere Agents genutzt werden):

````markdown
# Project: payment-platform

## Stack
- Python 3.12, FastAPI, SQLAlchemy 2.0, PostgreSQL 16

## Conventions
- Type Hints sind Pflicht
- Repository Pattern für DB-Zugriffe
- Conventional Commits (feat:, fix:, chore:)
- Tests in `tests/` mit pytest

## Commands
```bash
make dev          # Lokale Entwicklung starten
make test         # Alle Tests ausführen
make lint         # ruff + mypy
```

## Rules
- NIEMALS `print()` verwenden – immer `structlog`
- NIEMALS raw SQL – immer SQLAlchemy ORM
- NIEMALS Secrets committen
````

**`CLAUDE.md`** enthält nur das, was Claude zusätzlich wissen muss:

```markdown
# payment-platform – Claude-spezifisch

## Zusätzlicher Kontext
- Verwende `/compact` regelmäßig bei langen Sessions
- Bei Refactoring: Immer zuerst Tests schreiben, dann refactoren
- Laufende Migration von REST zu gRPC (API-Module in `src/api/`)
```

**`.cursorrules`** enthält nur Cursor-spezifische Ergänzungen:

```markdown
# Cursor-Regeln

- Verwende immer die @-Referenzen für Kontext
- Bei Code-Generierung: Generiere immer auch den zugehörigen Test
- Bevorzuge Inline-Suggestions über Chat-Antworten
```

---

## Best Practices für plattformübergreifende Instructions

### 1. DRY-Prinzip anwenden

> [!WARNING]
> Dupliziere keine Instruktionen über mehrere Dateien. Abweichungen zwischen `CLAUDE.md`, `AGENTS.md` und `.cursorrules` führen zu inkonsistentem Agent-Verhalten und sind schwer zu pflegen.

**Empfohlene Aufteilung:**

| Inhalt | Wo ablegen | Warum |
|---|---|---|
| Stack, Konventionen, Verbote | `AGENTS.md` | Universell, für alle Agents |
| Build/Test-Befehle | `AGENTS.md` | Jeder Agent braucht sie |
| Agent-spezifisches Verhalten | `CLAUDE.md` / `.cursorrules` etc. | Nur für diesen Agent relevant |
| Persönliche Präferenzen | `~/.copilot/copilot-instructions.md` / `~/.claude/CLAUDE.md` | Nur für dich, nicht fürs Team |
| Dateityp-spezifische Regeln | `.github/instructions/` | Nur relevant bei bestimmten Dateitypen |

### 2. Konkret statt abstrakt

```markdown
# ❌ Schlecht:
Schreibe sauberen Code.

# ✅ Gut:
- Funktionen maximal 30 Zeilen
- Explizite Return-Types für alle public Functions
- Keine verschachtelten Ternary-Operatoren
```

### 3. Verbote mit Begründung

```markdown
# ❌ Schlecht:
Führe keine Migrations aus.

# ✅ Gut:
NIEMALS `alembic upgrade head` direkt ausführen.
Nur `alembic revision --autogenerate` zum Generieren.
Grund: Produktion hat separate Deployment-Pipeline.
```

### 4. Regelmäßig aktualisieren

- Bei jedem Stack-Wechsel die Instruction-Dateien anpassen
- Veraltete Regeln entfernen – sie verwirren den Agent
- Nach schwierigen Debugging-Sessions: Fallstricke dokumentieren

### 5. Länge im Rahmen halten

- Optimal: **100–300 Zeilen** pro Instruction-Datei
- Maximum: ~500 Zeilen (danach aufteilen)
- Lieber mehrere fokussierte Dateien als eine riesige

> [!TIP]
> Teste deine Instructions regelmäßig, indem du den Agent eine typische Aufgabe erledigen lässt. Wenn er Konventionen ignoriert, sind die Instructions zu lang, zu vage oder widersprüchlich.

---

## Vergleichstabelle: Features pro Agent

| Feature | Claude Code | GitHub Copilot | Gemini | Cursor | Windsurf | Aider |
|---|---|---|---|---|---|---|
| **Hierarchisch** (Global → Projekt → Subdir) | ✅ | ✅ | ❌ | Teilweise | ❌ | ❌ |
| **Scoped** (per Dateityp/Glob) | ❌ | ✅ | ❌ | ✅ (MDC) | ❌ | ❌ |
| **Glob-Patterns** | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **Globale User-Config** | `~/.claude/` | `$HOME/.copilot/` | ❌ | ❌ | ❌ | `$HOME/.aider.conf.yml` |
| **Repo-Level** | `./CLAUDE.md` | `.github/copilot-instructions.md` | `./GEMINI.md` | `.cursorrules` | `.windsurfrules` | `.aider.conf.yml` |
| **Universelle Datei** (`AGENTS.md`) | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Liest andere Formate** | Nein | `CLAUDE.md`, `GEMINI.md` | Nein | Nein | Nein | Per `read:` |
| **Env-Variable für Extra-Dirs** | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Subdirectory-Dateien** | ✅ | Via `AGENTS.md` | ❌ | Via `.cursor/rules/` | ❌ | ❌ |
| **Frontmatter/Metadaten** | ❌ | `applyTo` | ❌ | MDC Frontmatter | ❌ | ❌ |

### Legende

- ✅ = Nativ unterstützt
- ❌ = Nicht unterstützt
- Teilweise = Eingeschränkt oder über Workarounds möglich

---

## Ladereihenfolge bei GitHub Copilot

Copilot lädt Instructions aus allen konfigurierten Quellen und kombiniert sie:

```
Session-Start
    │
    ▼
1. $HOME/.copilot/copilot-instructions.md     (global, immer)
2. COPILOT_CUSTOM_INSTRUCTIONS_DIRS           (env-var, immer)
    │
    ▼
3. .github/copilot-instructions.md            (repo-level)
4. AGENTS.md                                  (git-root & cwd)
5. CLAUDE.md                                  (git-root & cwd)
6. GEMINI.md                                  (git-root & cwd)
    │
    ▼
7. .github/instructions/**/*.instructions.md  (scoped, kontextabhängig)
    │
    ▼
Alle Instruktionen werden in den Kontext geladen
und bei jeder Anfrage berücksichtigt.
```

> [!NOTE]
> Die Scoped Instructions (`.github/instructions/`) werden nur geladen, wenn die `applyTo`-Patterns auf Dateien im aktuellen Kontext zutreffen. Arbeitet man z.B. nur an Python-Dateien, werden die TypeScript-Instructions nicht geladen.

---

## Verwandte Notizen

- [[Skills]] — Modulare, wiederverwendbare Wissenspakete für AI Agents
- [[Claude Code Konfiguration und Hooks]]
- [[Was ist ein AI Agent]]
- [[Multi-Agent Workflows]]
