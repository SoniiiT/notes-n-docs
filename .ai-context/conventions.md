# Beobachtete Konventionen

> [!ABSTRACT]
> Die Konventionen stammen aus `.github/instructions/markdown-docs.instructions.md`, den beiden Agent-Definitionen und den vorhandenen Vault-Dateien. Unsichere oder nur abgeleitete Regeln sind als solche gekennzeichnet.

---

## Sprache und Dokumentformat

- Dokumentation wird auf Deutsch geschrieben; etablierte Fachbegriffe dürfen Englisch bleiben.
- Jede Markdown-Datei hat genau eine H1-Überschrift.
- Hauptabschnitte werden mit `---` getrennt; leere Abschnitte werden vermieden.
- Code-Blöcke tragen eine Sprachkennung.
- Tabellen, Callouts und Mermaid werden nur eingesetzt, wenn sie die Lesbarkeit oder Beziehungen verbessern.
- Obsidian-Wiki-Links zeigen auf existierende Dateien; relative Pfade wie `../` sind nicht vorgesehen.

---

## Frontmatter und Benennung

- Frontmatter wird für Titel, Status, Tags und fachliche Typen verwendet, wenn es für Suche, Dataview oder Inventar nützlich ist.
- Im Homelab sind `type: server`, `type: service` und `type: network` die vorhandenen Zieltypen.
- Homelab-Serverdateien folgen `<hostname>.md`, Servicedateien `<dienstname>.md`; die Netzwerktopologie liegt unter `network/topology.md`.
- Bestehende Dateien werden bevorzugt aktualisiert; künstliche Varianten wie `-neu`, `-final` oder `-v2` sind nicht vorgesehen.
- Tags sollen sparsam und konsistent mit den bereits verwendeten Vault-Tags eingesetzt werden.

---

## Fachliche Trennung

- `Sysadmin-Wiki`: allgemeines Wissen, Erklärungen, CLI-/GUI-Wege, Best Practices und Troubleshooting.
- `Homelab-Wiki`: nüchterne Ist-Dokumentation des konkreten Setups, ohne allgemeine Theorie oder Lern-How-tos.
- Homelab: ein Host pro Datei, ein wichtiger Dienst pro Datei, eine zentrale Netzwerktopologie und ein README als Einstiegspunkt.
- Unsichere Informationen werden als offen gekennzeichnet; Agenten sollen keine fehlenden Infrastrukturdetails erraten.

---

## Konsistenz und Qualität

Bei Homelab-Änderungen müssen README, Hostdateien, Servicedateien, Wiki-Links und Topologiediagramme zusammenpassen. Öffentlich exponierte Dienste müssen in den betroffenen Übersichten konsistent markiert sein. Bei einer Entfernung sind alle Tabellen, Diagramme und Referenzen zu bereinigen.

Es existieren keine automatisierten Test- oder Linkcheck-Suiten. Die aktuelle Qualitätssicherung ist deshalb eine manuelle bzw. agentengestützte Prüfung von Markdown-Struktur, JSON-Syntax, Pfaden und Obsidian-Links.

---

## Sicherheit und sensible Inhalte

Die Vault enthält Infrastrukturdetails und Suchtreffer zu Begriffen wie `password`, `secret`, `token` oder `api-key`. Beim Indexieren werden keine Werte aus solchen Feldern persistiert. Neue Kontextdateien enthalten nur Struktur, Rollen, dokumentierte Beziehungen und den Hinweis auf offene Punkte; Geheimnisse, Tokens und private Schlüssel gehören nicht in `.ai-context`.

