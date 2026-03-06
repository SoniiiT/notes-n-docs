---
tags: [projekt-ideen, planung, homelab]
aliases: [Projekt Übersicht, Ideen]
---

# 💡 Projektideen

> [!ABSTRACT]
> Sammlung von Projektideen — alles was interessant klingt und irgendwann mal umgesetzt oder ausprobiert werden könnte.

---

## 🤖 AI & Machine Learning

### Self-hosted AI Platform

Aufbau einer lokalen Plattform zur Ausführung von LLMs (z. B. mit Ollama), um interne Dokumente, Logs oder Systeme abfragen zu können — ohne externe Dienste. Ziel: AI-Integration in die eigene Umgebung.

---

## 🔐 Security & Identity

### Zentralisierte Secrets-Verwaltung

Einrichtung eines zentralen Systems zur sicheren Verwaltung und Verteilung von Secrets (z. B. HashiCorp Vault). Integrierbar in Deployments, CI/CD und lokale Skripte.

### Policy & Compliance Scanner

Ein Tool, das Container und Hosts regelmäßig auf definierte Policies prüft (z. B. SSH-Config, offene Ports, Root-Zugänge). Ziel: Sicherheitsstandard automatisch durchsetzen.

### Secrets & Passwörter in HashiCorp Vault migrieren

Alles was aktuell auf GitHub oder anderswo liegt (API-Keys, Passwörter, Secrets) nach HashiCorp Vault verschieben und sauber einrichten. Inklusive VSCode-Integration.

### IDP einrichten (Authentik)

Authentik als zentralen Identity Provider aufsetzen — Single Sign-On für alle Self-Hosted-Dienste, LDAP-Backend, OAuth2/OIDC-Provider.

---

## 🏗️ Infrastructure & Platform

### Automatisiertes Rebuild-System für Container/VMs

Ein System (UI/API), über das Container oder VMs auf Knopfdruck neu erstellt werden können — auf Basis von Snapshots, Templates oder Backups. Inklusive Logging und Statusanzeige.

### Internal Platform API Gateway

Entwicklung eines Gateways, über das Ressourcen wie Deployments, VMs, Logs oder Secrets programmatisch erreichbar sind. Ziel: zentrale Steuerung der internen Plattform über eine einheitliche API.

### Self-Service API Portal für interne Plattformdienste

Weboberfläche und API, über die Entwickler eigene Ressourcen nutzen können (Datenbanken, Deployments, Logs). Mit Authentifizierung, Dokumentation und Zugriffskontrolle.

### Custom Configuration-as-a-Service API

Eigener Konfigurationsdienst: verwaltet, validiert und liefert Config-Dateien (z. B. YAML, JSON) für Deployments und Services — versioniert und CI-fähig.

### API-First Service Mesh

Aufbau einer internen, serviceorientierten Infrastruktur, in der alle Dienste über standardisierte APIs kommunizieren. Mit Auth, Logging und Versionierung.

---

## 📦 Container & CI/CD

### Self-hosted Container Registry

Eigene Container Registry betreiben (z. B. Harbor) zur Speicherung und Verwaltung von internen Docker/K8s-Images — sicher, versioniert und teamfähig.

### Container Build & Deployment Pipeline

Automatisierter Build-Prozess für Container: Build → Test → Scan → Push zur eigenen Registry → optionaler Deploy. Integrierbar in GitOps oder klassische CI/CD-Pipelines.

### Container Build Templates als Code-Generator

Python-Tool oder CLI, das automatisch angepasste Dockerfile- und CI-Pipeline-Vorlagen generiert für verschiedene Technologien (Node.js, Python, Go etc.).

---

## 🧰 HashiCorp Tooling

### Nomad Cluster aufsetzen

Docker-Cluster mit Nomad + Consul erstellen. 5 Docker-Server hochziehen, Nomad als Scheduler installieren und Consul als Koordinations-Backend einrichten.

### OpenTofu → Terraform Migration

Von OpenTofu wieder zurück auf Terraform migrieren und den tfstate in die HashiCorp Cloud verschieben.

### Packer Images erstellen

Aktuelle VM-Images mit Packer bauen: Ubuntu 24.04 und 25.04, Windows Server 2022 und 2025. Optional auch Desktop-Varianten.

### Waypoint anschauen

Schauen ob HashiCorp Waypoint als einheitliches Deployment-Tool interessant wäre.

### Vagrant anschauen

Schauen ob Vagrant für lokale Entwicklungsumgebungen sinnvoll ist — reproduzierbare VMs per Vagrantfile.

### Boundary anschauen

Recherchieren was HashiCorp Boundary ist — Identity-basiertes Access-Management als Ersatz für VPN und SSH-Bastion-Hosts.