# CLAUDE.md schreiben

## Was ist eine CLAUDE.md?

Eine `CLAUDE.md` ist eine Markdown-Datei, die **Projekt-spezifische Instruktionen, Konventionen und Kontext** für Claude Code enthält. Claude liest diese Datei automatisch beim Start einer Session und berücksichtigt ihren Inhalt bei allen Aktionen.

Sie ist das Äquivalent eines "Onboarding-Dokuments" – nur für KI statt für neue Entwickler.

> [!IMPORTANT]
> CLAUDE.md ist kein System-Prompt im klassischen Sinne, sondern eine **Gedächtnis-Erweiterung**. Claude kombiniert den Inhalt der CLAUDE.md mit den eigentlichen User-Anfragen. Je präziser und relevanter die CLAUDE.md, desto besser die Ergebnisse.

---

## Speicherorte und Geltungsbereiche

Claude Code liest CLAUDE.md-Dateien aus mehreren Speicherorten und kombiniert sie:

| Datei | Geltungsbereich | Beschreibung |
|---|---|---|
| `~/.claude/CLAUDE.md` | Global | Gilt für alle Projekte des Nutzers |
| `./CLAUDE.md` | Projekt-Root | Gilt für das gesamte Projekt |
| `./src/CLAUDE.md` | Unterverzeichnis | Gilt nur wenn in diesem Dir gearbeitet wird |

### Ladereihenfolge:

```
1. ~/.claude/CLAUDE.md (global, immer geladen)
2. /pfad/zum/projekt/CLAUDE.md (Projekt-Root)
3. /pfad/zum/projekt/subdir/CLAUDE.md (wenn im Subdir gearbeitet wird)
```

Alle gefundenen CLAUDE.md-Dateien werden **kombiniert** in den Kontext geladen.

> [!NOTE]
> Unterverzeichnis-CLAUDE.md-Dateien werden nur geladen, wenn Claude Code tatsächlich in diesem Verzeichnis arbeitet oder explizit auf Dateien dort zugreift. Sie erlauben modul-spezifische Anweisungen in großen Monorepos.

### Globale vs. Projekt-CLAUDE.md:

**`~/.claude/CLAUDE.md` (global)** – Persönliche Präferenzen und Standards:
```markdown
# Meine globalen Claude-Einstellungen

## Allgemeine Präferenzen
- Antworte immer auf Deutsch, es sei denn, der Kontext erfordert Englisch
- Bevorzuge typsichere Lösungen
- Erkläre nie offensichtliche Dinge – ich bin Senior-Entwickler
- Beim Refactoring: Verändere nie das Verhalten, nur die Struktur
```

**`./CLAUDE.md` (Projekt)** – Projekt-spezifischer Kontext:
```markdown
# Projektname: payment-service

## Stack
- Node.js 20 + TypeScript 5
- PostgreSQL 15 via Prisma ORM
- Jest für Tests
```

---

## Struktur einer CLAUDE.md

Eine gut strukturierte CLAUDE.md folgt diesem Aufbau:

```markdown
# [Projektname]

## Projektübersicht
[1-3 Sätze: Was macht dieses Projekt, welches Problem löst es]

## Technologie-Stack
[Liste aller relevanten Technologien mit Versionen]

## Projektstruktur
[Verzeichnisbaum oder Erklärung der wichtigsten Verzeichnisse]

## Entwicklungsumgebung
[Wie man das Projekt startet, wichtige Befehle]

## Konventionen und Standards
[Code-Style, Naming-Conventions, Architektur-Prinzipien]

## Häufige Aufgaben
[Wie spezifische, wiederkehrende Tasks erledigt werden]

## Verbotene Aktionen
[Was Claude NICHT tun darf]

## Wichtige Hinweise
[Fallstricke, bekannte Probleme, Besonderheiten]
```

---

## Was sollte rein?

### Projektkontext

```markdown
## Projektübersicht
payment-service ist der zentrale Zahlungsabwicklungs-Microservice der XY-Plattform.
Er verarbeitet Stripe-Webhooks, verwaltet Abonnements und kommuniziert mit dem
user-service über gRPC.

## Wichtiger Kontext
- Dieser Service ist PCI-DSS-konform – keine Kartendaten jemals loggen!
- Wir sind in einer laufenden Migration von REST zu gRPC (50% abgeschlossen)
- Der Branch `legacy-api` wird noch 3 Monate parallel betrieben
```

### Technologie-Stack

```markdown
## Stack
- **Runtime:** Node.js 20.x (LTS)
- **Sprache:** TypeScript 5.4 (strict mode aktiviert)
- **Framework:** Fastify 4.x (NICHT Express!)
- **ORM:** Prisma 5.x mit PostgreSQL 15
- **Tests:** Vitest (Unit), Playwright (E2E)
- **Deployment:** Kubernetes via Helm Charts
- **CI/CD:** GitHub Actions
- **Monitoring:** Datadog APM + strukturiertes JSON-Logging
```

### Konventionen

```markdown
## Code-Konventionen
- Alle Funktionen müssen explizite Return-Types haben
- `async/await` statt `.then()/.catch()`
- Fehler immer mit `AppError`-Klasse werfen (nicht mit `Error` direkt)
- Keine `any` in TypeScript – bei Bedarf `unknown` nutzen
- Dateinamen: kebab-case (payment-processor.ts, user-repository.ts)
- Test-Dateien: `*.test.ts` im gleichen Verzeichnis wie die Quelle

## Git-Konventionen
- Conventional Commits: feat:, fix:, chore:, docs:, refactor:
- Branch-Namen: feat/TICKET-123-kurze-beschreibung
- Kein direkter Push auf `main` – immer PR
```

### Verbotene Aktionen

```markdown
## VERBOTENE AKTIONEN
- NIEMALS `npm install --save` ohne explizite Aufforderung ausführen
- NIEMALS direkt auf die Produktions-DB schreiben (nur READ auf prod)
- NIEMALS Migrations direkt ausführen – immer nur generieren und prüfen
- NIEMALS Secrets oder API-Keys in Code-Dateien schreiben
- NIEMALS `console.log` in Produktions-Code – immer `logger.info()` etc.
- NIEMALS `force push` auf main/develop
```

### Häufige Aufgaben

````markdown
## Häufige Aufgaben

### Neue Datenbank-Migration erstellen
```bash
npx prisma migrate dev --name beschreibung-der-aenderung
```

### Tests ausführen
```bash
npm run test          # alle Unit-Tests
npm run test:watch    # im Watch-Modus
npm run test:e2e      # End-to-End Tests (benötigt laufende DB)
```

### Lokale Entwicklung starten
```bash
docker-compose up -d postgres redis   # Infrastruktur starten
npm run dev                           # Service starten (Port 3000)
```
````

---

## Was sollte NICHT rein?

> [!WARNING]
> Folgende Inhalte gehören NICHT in CLAUDE.md:

| Nicht rein | Warum | Alternative |
|---|---|---|
| API-Keys, Tokens, Passwörter | Sicherheitsrisiko – CLAUDE.md landet oft in Git | `.env`-Datei, Secret Manager |
| Sehr langer Text (>500 Zeilen) | Verdünnt den Kontext, wichtige Infos gehen unter | Mehrere CLAUDE.md in Unterverzeichnissen |
| Komplette Dokumentationen | Zu viel Rauschen, irrelevante Details | Als separate Dateien, per Verweis |
| Veraltete Informationen | Verwirrt Claude mehr als es hilft | Regelmäßig aktualisieren |
| Selbstverständliche Best Practices | Unnötiger Token-Verbrauch | Weglassen – Claude kennt sie bereits |
| Spezifische Datei-Inhalte | Besser zur Laufzeit lesen lassen | Tool-Aufruf `read_file` |

---

## Vollständiges Beispiel: CLAUDE.md für ein DevOps-Projekt

````markdown
# infra-platform – Infrastructure Management Service

## Projektübersicht
infra-platform ist das zentrale Infrastruktur-Verwaltungssystem für alle
Cloud-Ressourcen der Firma. Es orchestriert Terraform-Deployments, verwaltet
Kubernetes-Cluster via ArgoCD und stellt ein internes Self-Service-Portal bereit.

**Wichtig:** Alle Änderungen an Produktions-Infrastruktur müssen durch den
Approval-Workflow in Jira (Projekt: OPS) genehmigt werden.

## Technologie-Stack
- **Sprache:** Python 3.12 + FastAPI 0.111
- **IaC:** Terraform 1.8.x (AWS + GCP)
- **Container:** Docker + Kubernetes 1.29
- **GitOps:** ArgoCD 2.10
- **CI/CD:** GitHub Actions
- **Tests:** pytest 8.x + moto (AWS Mocking)
- **Linting:** ruff + mypy (strict)
- **Dokumentation:** MkDocs Material

## Projektstruktur
```
infra-platform/
├── api/                    # FastAPI Application
│   ├── routers/            # Route-Handler nach Domäne
│   ├── services/           # Business Logic
│   ├── models/             # Pydantic Models
│   └── dependencies.py     # DI Container
├── terraform/
│   ├── modules/            # Wiederverwendbare TF-Module
│   ├── environments/       # dev/, staging/, prod/
│   └── shared/             # Gemeinsame Ressourcen
├── k8s/
│   ├── base/               # Kustomize Base Configs
│   └── overlays/           # Umgebungs-spezifische Overrides
├── scripts/                # Utility Scripts (Python)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
└── docs/                   # MkDocs Source
```

## Entwicklungsumgebung

### Voraussetzungen
- Python 3.12 via pyenv
- Docker Desktop oder Colima
- kubectl + helm + terraform CLI

### Setup
```bash
make dev-setup          # Virtualenv + Dependencies installieren
make dev-start          # Lokale Services starten (docker-compose)
make dev-migrate        # DB-Migrationen ausführen
```

### Wichtige Make-Targets
```bash
make test               # Unit + Integration Tests
make lint               # ruff + mypy
make fmt                # ruff format (auto-fix)
make tf-plan ENV=dev    # Terraform Plan für dev
make tf-apply ENV=dev   # Terraform Apply (nur nach Review!)
make docs-serve         # Lokale Doku auf Port 8000
```

## Code-Konventionen

### Python
- Type hints sind Pflicht (mypy strict)
- Pydantic v2 für alle Datenmodelle
- Repository Pattern für alle DB-Zugriffe
- Services dürfen nicht direkt auf DB zugreifen – nur über Repositories
- Alle Exceptions von `InfraException` ableiten
- Strukturiertes Logging mit `structlog`, nie `print()`

### Terraform
- Alle Ressourcen müssen Tags haben: `project`, `environment`, `managed_by`
- Remote State in S3 + DynamoDB Locking (nie local state auf prod)
- Module immer mit `source = "../../modules/..."` referenzieren
- Variablen mit Beschreibung und Validierung

### Git & Branching
- Conventional Commits: feat:, fix:, chore:, docs:, refactor:, infra:
- Feature Branches: `feat/OPS-123-kurze-beschreibung`
- Infra Branches: `infra/OPS-456-terraform-beschreibung`
- Squash Merge auf main

## Häufige Aufgaben

### Neuen AWS-Account onboarden
1. Terraform-Konfiguration in `terraform/environments/new-account/` erstellen
2. `make tf-plan ENV=new-account` ausführen und Review
3. PR erstellen mit Label `terraform-plan`
4. Nach Approval: `make tf-apply ENV=new-account`
5. ArgoCD-App-Manifest in `k8s/overlays/new-account/` erstellen

### Kubernetes-Deployment debuggen
```bash
# Standard-Debugging-Reihenfolge:
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
```

### Neuen API-Endpoint erstellen
1. Pydantic Models in `api/models/<domäne>.py`
2. Repository in `api/repositories/<domäne>_repository.py`
3. Service in `api/services/<domäne>_service.py`
4. Router in `api/routers/<domäne>.py`
5. Router in `api/main.py` registrieren
6. Tests in `tests/unit/` + `tests/integration/`

## Umgebungen

| Umgebung | Zweck | Zugang |
|---|---|---|
| local | Entwicklung | docker-compose lokal |
| dev | Feature-Tests | Automatisch via CI bei PR |
| staging | Pre-Production | Manuelles Deploy nach PR-Merge |
| prod | Produktion | Nur via Approval-Workflow |

**Staging-URL:** https://infra-platform.staging.internal
**Prod-URL:** https://infra-platform.internal

## VERBOTENE AKTIONEN
- NIEMALS `terraform apply` direkt auf prod ausführen – immer CI/CD
- NIEMALS AWS-Credentials in Code, Commits oder Logs
- NIEMALS DB-Passwörter im Klartext – immer AWS Secrets Manager
- NIEMALS `kubectl delete` auf Produktions-Namespace ohne Backup
- NIEMALS Dependencies direkt hinzufügen – immer pyproject.toml + `make lock`
- NIEMALS `--force` Flag bei git push

## Bekannte Fallstricke
- Die `terraform/modules/eks/` hat einen Known Bug mit IAM Roles bei Node Groups
  → Workaround: immer `lifecycle { ignore_changes = [node_groups] }` setzen
- Integration Tests benötigen laufenden LocalStack (Port 4566)
  → `make dev-start` startet LocalStack automatisch
- ArgoCD Sync kann bis zu 3 Minuten dauern – nicht ungeduldig werden
- pytest-asyncio Version muss exakt 0.23.x sein – höhere Versionen brechen Tests
````

---

## Tipps für effektive CLAUDE.md Dateien

> [!TIP]
> Die wichtigsten Prinzipien für eine wirkungsvolle CLAUDE.md:

**1. Konkret statt abstrakt**
```markdown
# Schlecht:
Verwende gute Coding-Practices.

# Gut:
Nutze immer explizite Return-Types. Keine `any`-Types.
Fehlerbehandlung ausschließlich via `AppError`-Klasse.
```

**2. Verbote mit Begründung**
```markdown
# Schlecht:
Führe keine Migrations aus.

# Gut:
NIEMALS Migrations direkt ausführen (npx prisma migrate deploy).
Nur generieren (migrate dev). Grund: Produktions-DB hat separate Deployment-Pipeline.
```

**3. Häufige Aufgaben als ausführbare Snippets**
```markdown
# Schlecht:
Tests laufen mit npm.

# Gut:
### Tests ausführen
npm run test:unit          # Unit-Tests (schnell, ~30s)
npm run test:integration   # Integration (benötigt Docker, ~2min)
npm run test -- --watch    # Watch-Modus für Entwicklung
```

**4. Regelmäßig aktualisieren**
- CLAUDE.md bei jedem größeren Technologie-Wechsel updaten
- Veraltete Infos löschen – sie verwirren mehr als sie helfen
- Nach schwierigen Debugging-Sessions: Fallstricke dokumentieren

**5. Länge im Rahmen halten**
- Optimal: 100-300 Zeilen
- Maximum: ~500 Zeilen (danach besser aufteilen)
- Unterverzeichnis-CLAUDE.md für modul-spezifische Details nutzen

---

## Wie Claude Code die Datei verwendet

Claude Code lädt CLAUDE.md automatisch beim Start der Session:

```
Session-Start
    │
    ▼
Suche CLAUDE.md:
    ├── ~/.claude/CLAUDE.md  → geladen
    ├── ./CLAUDE.md          → geladen
    └── ./subdir/CLAUDE.md   → geladen (wenn im Subdir)
    │
    ▼
Alle CLAUDE.md-Inhalte werden in den System-Kontext eingefügt
    │
    ▼
Jede User-Anfrage wird mit diesem Kontext kombiniert
    │
    ▼
Claude berücksichtigt Konventionen, Verbote und Hinweise
bei JEDEM Tool-Aufruf und jeder Antwort
```

> [!NOTE]
> Die CLAUDE.md im Projekt-Root und die globale `~/.claude/CLAUDE.md` werden beim Start der Session in den Kontext geladen. Unterverzeichnis-CLAUDE.md-Dateien werden **dynamisch** geladen, wenn Claude Code in das jeweilige Verzeichnis navigiert oder dort Dateien bearbeitet. Änderungen an der Root-CLAUDE.md wirken sich erst bei der nächsten Session aus (außer man fordert Claude explizit auf, die Datei neu zu lesen).

### Manuelle Anweisung zum Neuladen:

```
"Bitte lies die CLAUDE.md in diesem Projekt neu und bestätige mir,
welche Konventionen du verwenden sollst."
```

---

## Verwandte Notizen

- [[Was ist ein AI Agent]]
- [[Claude Code Konfiguration und Hooks]]
- [[Multi-Agent Workflows]]
