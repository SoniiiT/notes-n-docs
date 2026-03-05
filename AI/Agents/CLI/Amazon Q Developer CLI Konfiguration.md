# Amazon Q Developer CLI Konfiguration

> [!ABSTRACT]
> Amazon Q Developer CLI ist AWS' AI-powered Coding Agent, der auf Cloud-native Entwicklung spezialisiert ist. Er bietet agentic Coding, Infrastructure-as-Code-Generierung, Security Scanning und tiefe AWS-Service-Integration direkt im Terminal und in IDEs.

---

## Übersicht

| Eigenschaft | Details |
|---|---|
| **Entwickler** | Amazon Web Services (AWS) |
| **Typ** | CLI Agent + IDE Extension |
| **Lizenz** | Proprietär (Free Tier verfügbar) |
| **Kosten** | Free ($0/Mo), Pro ($19/user/Mo) |
| **Plattformen** | macOS, Windows, Linux |
| **IDEs** | VS Code, JetBrains, Visual Studio, Eclipse |
| **Modelle** | Amazon eigene Modelle |
| **Custom Instructions** | Über AWS-Konfiguration |
| **Vorgänger** | AWS CodeWhisperer |

---

## Installation

### macOS

```bash
# Via Homebrew
brew install --cask amazon-q

# Oder .dmg von der AWS-Website herunterladen
```

### Linux

```bash
# AppImage (Ubuntu/Debian)
sudo apt install libfuse2
curl --proto '=https' --tlsv1.2 -sSf \
  https://desktop-release.q.us-east-1.amazonaws.com/latest/amazon-q.appimage \
  -o amazon-q.appimage
chmod +x amazon-q.appimage
./amazon-q.appimage
```

### Windows

Download des Installers von der [AWS Q Developer Seite](https://aws.amazon.com/q/developer/).

### Authentifizierung

```bash
# Mit AWS Builder ID (kostenlos)
q login

# Mit AWS IAM Identity Center (Enterprise)
q login --sso
```

### Health Check

```bash
# Installation prüfen
q doctor

# Probleme melden
q issue
```

---

## Kernfunktionen

### Agentic Coding

Amazon Q kann mehrstufige Entwicklungsaufgaben autonom ausführen:

```bash
# Feature implementieren
q "Erstelle eine Lambda-Funktion, die S3-Events verarbeitet
   und die Metadaten in DynamoDB speichert"

# Code erklären
q "Erkläre diese CloudFormation-Template"

# Debugging
q "Warum schlägt mein ECS-Deployment fehl?"
```

### AWS CLI Autocomplete

Amazon Q vervollständigt AWS CLI-Befehle intelligent:

```bash
# Natürliche Sprache → AWS CLI
q "Liste alle laufenden EC2-Instanzen in us-east-1"
# → aws ec2 describe-instances --region us-east-1 --filters "Name=instance-state-name,Values=running"

# Komplexe Abfragen
q "Zeige alle S3-Buckets, die öffentlich zugänglich sind"
```

### Infrastructure as Code

```bash
# CDK-Code generieren
q "Erstelle ein CDK-Stack mit API Gateway, Lambda und DynamoDB"

# CloudFormation Templates
q "Konvertiere dieses Terraform-File zu CloudFormation"

# SAM Templates
q "Erstelle ein SAM-Template für eine Serverless REST API"
```

### Security Scanning

Amazon Q scannt Code auf Sicherheitslücken:

```bash
# Security Scan ausführen
q scan --path ./src

# OWASP Top 10 Prüfung
q "Prüfe mein Projekt auf SQL Injection Schwachstellen"
```

### Code-Transformation

Automatisierte Code-Upgrades für große Projekte:

| Transformation | Beschreibung |
|---|---|
| **Java 8 → 17** | Automatische Migration inkl. API-Änderungen |
| **.NET Framework → .NET 6+** | Modernisierung von Legacy-Code |
| **Language Upgrades** | Framework-Version-Upgrades |

---

## Preismodell

| Plan | Preis | Agentic Requests | Code-Transformation | Security Scan |
|---|---|---|---|---|
| **Free** | $0/Mo | 50/Mo | 1.000 LOC/Mo | Basic |
| **Pro** | $19/user/Mo | 1.000/Mo | 4.000 LOC/Mo (+$0,003/LOC) | Erweitert + Admin |

> [!NOTE]
> Amazon Q Developer ist besonders attraktiv für AWS-zentrierte Teams. Der Free Tier bietet genug für individuelle Entwickler, während Pro für Teams mit intensiver AWS-Nutzung konzipiert ist.

---

## Integration mit AWS-Services

Amazon Q versteht nativ alle AWS-Services:

| Service-Bereich | Unterstützung |
|---|---|
| **Compute** | EC2, Lambda, ECS, EKS, Fargate |
| **Storage** | S3, EFS, EBS |
| **Database** | DynamoDB, RDS, Aurora, ElastiCache |
| **Networking** | VPC, Route 53, CloudFront, ALB |
| **Security** | IAM, KMS, Secrets Manager, WAF |
| **DevOps** | CodePipeline, CodeBuild, CloudFormation, CDK |
| **Monitoring** | CloudWatch, X-Ray, CloudTrail |

### Praxis-Beispiel: Debugging

```bash
# AWS Console Fehler diagnostizieren
q "Mein API Gateway gibt 502 Bad Gateway zurück.
   Die Lambda-Funktion heißt user-service-prod."

# Q analysiert:
# 1. Lambda-Logs in CloudWatch
# 2. API Gateway Konfiguration
# 3. IAM-Permissions
# 4. VPC/Subnet-Konfiguration
```

---

## Vergleich mit anderen CLI Agents

| Feature | Amazon Q | Claude Code | Gemini CLI |
|---|---|---|---|
| **AWS-Integration** | ✅✅✅ (Kernfeature) | Via MCP | Via MCP |
| **IaC-Generierung** | ✅✅ (CDK, CFN, SAM) | Generisch | Generisch |
| **Security Scanning** | ✅✅ (nativ) | Manuell | Manuell |
| **Code-Transformation** | ✅✅ (Java, .NET) | ❌ | ❌ |
| **Multi-Cloud** | ❌ (nur AWS) | ✅ | ✅ |
| **Lokale Modelle** | ❌ | ❌ | ❌ |
| **Preis (Pro)** | $19/user/Mo | Subscription (Pro $20/Mo, Max $100/Mo) oder API | Subscription (Plus €7,99/Mo, Pro €21,99/Mo) |

---

## Pro & Contra

| ✅ Vorteile | ❌ Nachteile |
|---|---|
| Beste AWS-Integration im Markt | Wenig Mehrwert außerhalb von AWS |
| Starker Free Tier (50 Requests/Mo) | Proprietäre Modelle, kein BYOK |
| Security Scanning integriert | Weniger flexibel als Multi-Modell-Agents |
| Code-Transformation für Legacy-Projekte | Agentic Requests begrenzt |
| Multi-IDE-Support | Kein MCP-Support |

---

## Tipps

> [!TIP]
> - **AWS Builder ID** ist kostenlos — reicht für den Free Tier vollständig aus
> - **`q doctor`** bei Problemen ausführen für automatische Diagnose
> - Ideal als **Ergänzung** zu Claude Code oder Gemini CLI für AWS-spezifische Aufgaben
> - **Code-Transformation** spart enorm viel Zeit bei Java/. NET-Migrationen

---

## Verwandte Notizen

- [[AI Coding Agents Vergleich]] — Vergleich aller AI Coding Agents
- [[Kiro Konfiguration]] — AWS' IDE-Agent mit Spec-driven Development
- [[Custom Instructions für AI Coding Agents]] — Instruction-Dateien für verschiedene Agents
