# Abhängigkeiten

> [!ABSTRACT]
> Das Projekt hat keine erkennbare Runtime-Abhängigkeit im Sinn einer Anwendung. Es hängt fachlich von Obsidian, Git und den in der Homelab-Dokumentation beschriebenen Infrastrukturkomponenten ab.

---

## Repository- und Editorabhängigkeiten

| Abhängigkeit | Rolle | Auswirkung |
|---|---|---|
| Git | Versionierung und Synchronisation der Markdown-Vaults | Änderungen müssen als konsistente Dateisätze nachvollziehbar bleiben |
| Obsidian | Editor und Wissensnavigation | Wiki-Links, Frontmatter, Callouts, Mermaid und Vault-Konfiguration müssen kompatibel bleiben |
| `obsidian-git` | In der Sysadmin-Vault konfigurierte Synchronisation | Veröffentlichen oder Synchronisieren kann vom Plugin abhängen |
| `dataview`, `omnisearch`, `homepage`, `table-editor-obsidian` | Suche, Übersichten und Bearbeitung | Frontmatter- und Tabellenkonventionen sind für Nutzbarkeit relevant |
| Excalidraw-/Diagramm-/Mindmap-Plugins | Visuelle Wissensdarstellung | Zugehörige Assets und Plugin-Konfiguration nicht ohne Prüfung entfernen |
| Linter und Formatierungs-Plugins | Editorseitige Qualitätsunterstützung | Nicht als Ersatz für fachliche oder Link-Konsistenzprüfung behandeln |

Die vollständige Plugin-Liste steht in `Sysadmin-Wiki/.obsidian/community-plugins.json`. Die konkrete Verfügbarkeit oder Version der Plugins wird in diesem Index nicht vorausgesetzt.

---

## Dokumentierte Infrastrukturabhängigkeiten

Die Homelab-Vault beschreibt als externe bzw. betriebliche Bezugspunkte unter anderem Proxmox, Debian-/Ubuntu-Systeme, WireGuard, Bind9, Fritzbox, Cloudflare, Cloud-Server und Pterodactyl. Diese sind keine installierten Repository-Pakete, sondern Systeme, deren Zustand in der Doku abgebildet wird. Änderungen an ihnen müssen die betroffenen Homelab-Dateien und die Topologie berücksichtigen.

---

## Nicht vorhanden oder nicht nachgewiesen

- Kein `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml` oder vergleichbares Runtime-Manifest.
- Keine Lockfiles oder Dependency-Installation im Repository.
- Keine Datenbankmigrationen, API-Spezifikation, Queue-Konfiguration oder Worker-Definition.
- Keine CI-, Build- oder Deployment-Datei im inventarisierten Root.
- Keine Netzwerkverbindung ist für die Kontextinitialisierung erforderlich.

