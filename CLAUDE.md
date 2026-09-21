# OpenWolf

This project uses OpenWolf for context management. The always-on rules live in `.claude/rules/openwolf.md`; the hooks handle bookkeeping (anatomy index, memory log, read tracking) automatically.

For the full operating protocol (session handoff, memory discipline, bug logging), load the `openwolf` skill, or read `.wolf/OPENWOLF.md`. Regenerate the session handoff with `/handoff`.


# Claude-Kontext

Der kanonische gemeinsame Projektkontext liegt unter `.ai-context/`. Lies bei Aufgaben zur Repository-Struktur zuerst `.ai-context/manifest.json`, `.ai-context/project-map.md` und die passenden Dateien unter `.ai-context/knowledge/`.

Die Zusammenfassungen sind Orientierung, nicht die fachliche Quelle. Maßgeblich bleiben der aktuelle Vault-Inhalt und explizite Projektregeln. Vor Markdown-Änderungen `.github/instructions/markdown-docs.instructions.md` und die passende Agent-Definition unter `.github/agents/` prüfen. Keine Geheimnisse, Tokens oder privaten Schlüssel in Kontextdateien persistieren.

