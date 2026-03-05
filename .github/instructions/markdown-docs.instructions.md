---
description: "Regeln für alle Markdown-Dateien in der Obsidian Vault"
applyTo: "**/*.md"
---

# Markdown-Dokumentation Regeln

## Formatierung
- Sprache: Deutsch (Fachbegriffe dürfen Englisch bleiben)
- Nur ein `# H1` pro Datei (der Dokumenttitel)
- `---` (horizontale Linie) zwischen Hauptabschnitten
- Code-Blöcke immer mit Sprachkennung: ` ```bash `, ` ```python `, etc.
- Bei verschachtelten Code-Blöcken: 4 Backticks als äußere Fence

## Obsidian-spezifisch
- Wiki-Links: `[[Dateiname]]` — kein Pfad nötig, Obsidian löst global auf
- Callouts: `> [!NOTE]`, `> [!TIP]`, `> [!WARNING]`, `> [!IMPORTANT]`, `> [!DANGER]`
- Embeds: `![[Dateiname]]` oder `![[bild.png|400]]`
- Tags: `#tag` oder im Frontmatter als `tags: [tag1, tag2]`

## Struktur jeder Dokumentation
1. `# Titel` — Dokumenttitel
2. `> [!ABSTRACT]` — Kurzzusammenfassung (optional)
3. Hauptinhalt mit `## Abschnitten`
4. `## Verwandte Notizen` — Wiki-Links am Ende

## Qualität
- Alle Wiki-Links müssen auf existierende Dateien zeigen
- Keine leeren Abschnitte (jede Überschrift braucht Inhalt)
- Tabellen für Vergleiche, Mermaid für Abläufe
- Zielgröße: 300–700 Zeilen pro Dokument
