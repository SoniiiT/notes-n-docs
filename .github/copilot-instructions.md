# Copilot Instructions — Obsidian Vault

Dies ist eine **Obsidian Vault** (Knowledge Base), keine klassische Codebase.
Alle Dateien sind Markdown-Dokumentationen in deutscher Sprache.

## Grundregeln

- **Sprache:** Alle Inhalte auf Deutsch schreiben
- **Format:** Obsidian-kompatibles Markdown (Wiki-Links, Callouts, Embeds)
- **Stil:** Technisch präzise, praxisorientiert, mit Beispielen
- **Code-Blöcke:** Immer mit Sprachkennung (```bash, ```python, etc.)
- **Verschachtelte Code-Blöcke:** 4 Backticks (``````) als äußere Fence verwenden wenn innere ``` vorkommen
- **Keine leeren Dateien:** Jede Datei muss substanziellen Inhalt haben

## Wiki-Links

- Verwende `[[Dateiname]]` für interne Verlinkungen
- Verwende `[[Dateiname|Anzeigename]]` für Aliase
- Verwende `[[Dateiname#Überschrift]]` für Heading-Links
- **Prüfe immer**, ob die verlinkte Datei existiert bevor du einen Wiki-Link erstellst
- Keine relativen Pfade (`../`) in Wiki-Links — Obsidian löst Dateinamen global auf

## Vault-Struktur

```
AI/           → KI, LLMs, Agents, Prompting, MCP, RAG, Training, Tools
Applications/ → Software & Anwendungen
Assets/       → Bilder, Diagramme, Medien
Clustering/   → Cluster-Technologien
Container/    → Docker, Podman, Container-Orchestrierung
Homelab/      → Heimserver, Self-Hosting
Linux/        → Linux-Administration, Shell, Systemd
Networking/   → Netzwerk-Konfiguration, Protokolle
Security/     → IT-Sicherheit, Hardening
Templates/    → Obsidian-Vorlagen
Virtualization/ → VMs, Hypervisors
```

## Installierte Plugins

Folgende Plugins sind installiert und können in Dokumentationen genutzt werden:
- **Dataview** — Datenabfragen mit DQL und Inline-Queries
- **Kanban** — Kanban-Boards in Markdown
- **Excalidraw** — Zeichnungen und Diagramme
- **ExcaliBrain** — Visuelle Graph-Ansicht
- **Diagrams.net** — draw.io Integration
- **Table Editor** — Erweiterte Tabellenbearbeitung
- **Codeblock Customizer** — Code-Block-Styling
- **Omnisearch** — Erweiterte Suche
- **Obsidian Git** — Git-Integration
- **Linter** — Markdown-Linting
