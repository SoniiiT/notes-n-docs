# Projektkarte

> [!ABSTRACT]
> Dieses Repository ist eine deutschsprachige Obsidian-Dokumentation mit einer allgemeinen Sysadmin-Wiki und einer konkreten Homelab-Wiki. `.ai-context/` ist der gemeinsame, kompakte Kontext für Agenten; der Quelltext der Vaults und die expliziten Projektregeln bleiben maßgeblich.

---

## Projektgrenze

Die Repository-Grenze liegt im Root dieses Git-Repositories. Es gibt keine Anwendung mit Runtime-Entry-Point, kein Package-Manifest und keine erkennbaren Build-, Test- oder Deployment-Pipelines. Die wesentlichen Inhalte sind Markdown, Obsidian-Konfiguration, Assets und gespeicherte Wissensdaten.

---

## Wichtige Bereiche

| Pfad | Zweck | Navigation |
|---|---|---|
| `Homelab-Wiki/` | Ist-Dokumentation des eigenen Homelabs und der Cloud-Systeme | `README.md`, `servers/`, `services/`, `network/topology.md`, `todo.md` |
| `Homelab-Wiki/servers/` | Ein Host pro Datei, aktuell 17 Markdown-Dateien | Über `Homelab-Wiki/README.md` und Wiki-Links |
| `Homelab-Wiki/services/` | Wichtige Dienste, aktuell 11 Markdown-Dateien | Über README, Host-Dateien und Service-Abhängigkeiten |
| `Homelab-Wiki/network/` | Zentrale Netzwerktopologie | `topology.md` |
| `Sysadmin-Wiki/` | Allgemeines Technologie-, Plattform- und Best-Practice-Wissen | Kategorien und verschachtelte Themenpfade |
| `Sysadmin-Wiki/AI/` | AI-Wissen, Agents, LLMs, MCP, Prompting, RAG, Tools und Training | `AI Index.md`, `AI Best Practices.md`, `AI Troubleshooting.md` |
| `Sysadmin-Wiki/Container/` | Docker, LXC/LXD und Podman | Themen- und Betriebsunterordner |
| `Sysadmin-Wiki/Microsoft 365/` | Azure- und Microsoft-365-Dokumentation | Azure-, Administration-, Security- und Service-Pfade |
| `Sysadmin-Wiki/Clustering/` | Cluster- und Docker-Swarm-Wissen | Themen- und Betriebsunterordner |
| `Sysadmin-Wiki/Virtualization/` | ESXi und Virtualisierung | Themen- und Betriebsunterordner |
| `Sysadmin-Wiki/Windows Server/` | Windows-Server- und Active-Directory-Wissen | Themen- und Betriebsunterordner |
| `.github/` | Agent-Definitionen und gemeinsame Markdown-Regeln | `agents/`, `instructions/` |
| `*.obsidian/` | Vault-spezifische Obsidian-Einstellungen und Plugins | Nicht als fachliche Quelle behandeln |

---

## Einstiegspunkte

1. Für den konkreten Bestand: `Homelab-Wiki/README.md`.
2. Für Netzwerkbeziehungen und Cloud-Anbindung: `Homelab-Wiki/network/topology.md`.
3. Für einen Host: `Homelab-Wiki/servers/<hostname>.md`.
4. Für einen wichtigen Dienst: `Homelab-Wiki/services/<dienst>.md`.
5. Für allgemeines Admin-Wissen: den passenden Themenpfad unter `Sysadmin-Wiki/`.
6. Für Agentenregeln: `.github/instructions/markdown-docs.instructions.md` und `.github/agents/`.
7. Für den kompakten maschinenlesbaren Kontext: `.ai-context/manifest.json`, `architecture.md` und die Indizes.

---

## Bereichsgrenzen

`Homelab-Wiki` dokumentiert reale Hosts, Dienste, Netzsegmente, Exposure und Abhängigkeiten. `Sysadmin-Wiki` dokumentiert allgemeine Technologie, Konzepte, Best Practices und lernorientierte Guides. Crosslinks von Homelab zu Sysadmin sind laut den vorhandenen Regeln nicht vorgesehen, außer sie werden ausdrücklich verlangt.

