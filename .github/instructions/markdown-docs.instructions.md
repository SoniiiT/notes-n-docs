---
description: "Gemeinsame Regeln für Markdown-Dateien in den Obsidian-Vaults"
applyTo: "**/*.md"
---

# Gemeinsame Markdown- und Obsidian-Regeln

Diese Regeln gelten als gemeinsame Basis für alle Markdown-Dateien in den Obsidian-Vaults dieses Repos.

---

## Sprache und Stil

- Schreibe auf Deutsch.
- Fachbegriffe dürfen Englisch bleiben.
- Schreibe technisch präzise, praxisorientiert und gut scanbar.
- Vermeide Marketing-Sprache und unnötige Füllsätze.
- Wenn Informationen unsicher sind, kennzeichne sie klar statt zu raten.

---

## Benennungskonventionen

- Dateinamen sollen stabil, eindeutig und gut wiederauffindbar sein.
- Für technische Dokumente bevorzugt kurze, sprechende Namen.
- Wenn ein Thema über einen etablierten Produkt- oder Protokollnamen identifizierbar ist, nutze diesen als Dateinamen.
- Keine künstlichen Dubletten wie `Thema Neu`, `Thema Final`, `Thema v2`.
- Bestehende Dateien bevorzugt aktualisieren statt neue Varianten anzulegen.

---

## Grundformatierung

- Genau ein `# H1` pro Datei
- `---` zwischen Hauptabschnitten
- Keine leeren Abschnitte
- Code-Blöcke immer mit Sprachkennung
- Bei verschachtelten Code-Blöcken: äußere Fence mit 4 Backticks
- Tabellen nutzen, wenn Vergleiche oder strukturierte Übersichten sinnvoll sind

---

## Frontmatter

Wenn Frontmatter sinnvoll ist, nutze möglichst diese Felder:

```yaml
---
title: Titel
tags:
  - tag1
  - tag2
status: draft | review | published
type: knowledge | server | service | network
---
```

Regeln:

- `title` entspricht dem sichtbaren Dokumenttitel
- `tags` nur sinnvoll und nicht inflationär nutzen
- `status` für Reifegrad der Notiz
- `type` für spätere Filter, Dataview und Übersichten

---

## Obsidian-spezifische Regeln

- Wiki-Links immer als `[[Dateiname]]`, `[[Dateiname|Alias]]` oder `[[Dateiname#Überschrift]]`
- Keine relativen Pfade wie `../`
- Vor dem Erstellen eines Wiki-Links immer prüfen, ob die Zieldatei existiert
- Embeds als `![[Dateiname]]` oder `![[Dateiname#Überschrift]]`
- Callouts mit Obsidian-Syntax verwenden, z. B. `> [!NOTE]`, `> [!TIP]`, `> [!WARNING]`, `> [!IMPORTANT]`, `> [!DANGER]`, `> [!ABSTRACT]`
- Mermaid für Topologien, Abläufe, Architektur und Beziehungen verwenden, wenn es den Inhalt klarer macht

---

## Tag-Konventionen

Tags sollen nicht blind nach einer festen Liste vergeben werden. Prüfe zuerst, welche Tags in der jeweiligen Vault bereits genutzt werden, und verwende vorhandene Tags bevorzugt weiter, wenn sie fachlich passen.

Bevorzugt flache, fachlich klare Tags wie:

- `#linux`
- `#windows-server`
- `#networking`
- `#security`
- `#container`
- `#virtualization`
- `#microsoft-365`
- `#homelab`

Für Status oder Spezialisierungen sind auch strukturierte Tags sinnvoll, z. B.:

- `#status/draft`
- `#status/review`
- `#status/published`
- `#homelab/service`
- `#homelab/network`

Regeln:

- Vor neuen Tags zuerst bestehende Tags in der Vault prüfen
- Vorhandene Tags bevorzugt wiederverwenden, wenn sie semantisch passen
- Neue Tags nur dann anlegen, wenn kein bestehender Tag das Thema sinnvoll abdeckt
- Neue Tags kurz, fachlich klar und konsistent zur vorhandenen Tag-Landschaft benennen
- Tags selbstständig wählen, damit Inhalte gut auffindbar sind
- So wenig Tags wie sinnvoll verwenden, nicht alles im Dokument künstlich abbilden
- Keine Vollabdeckung per Tags erzwingen
- Keine Tag-Listen aufblasen, nur weil mehrere Themen am Rand erwähnt werden
- Tags sparsam nutzen; gute Dateistruktur und gute Überschriften sind wichtiger als Tag-Spam

---

## Bevorzugte Dokumentationsstruktur

Wenn kein spezialisiertes Template vorgegeben ist, nutze möglichst diese Grundstruktur:

````markdown
# Titel

> [!ABSTRACT]
> Kurze Zusammenfassung des Themas und des praktischen Nutzens.

---

## Was ist das?

Technische Einordnung mit Kontext.

---

## Konfiguration / Umsetzung

Konfiguration, Einstellungen, Beispiele oder Vorgehen.

---

## Verwendung / Praxis

Typische Anwendung, Workflows oder Beispiele.

---

## Best Practices

> [!TIP]
> Bewährte Vorgehensweisen und Empfehlungen.

---

## Troubleshooting

> [!WARNING]
> Häufige Fehler, Symptome und Lösungen.

---

## Verwandte Notizen

- [[Beispiel]]
````

---

## Plugins und Features

Diese Vaults dürfen Obsidian-Features aktiv nutzen, wenn sie fachlich sinnvoll sind:

- Callouts
- Mermaid
- Dataview
- Kanban
- Canvas
- Embeds
- Frontmatter

Nutze Features nicht nur, weil sie möglich sind, sondern wenn sie Struktur, Lesbarkeit oder Wartbarkeit verbessern.

---

## Aktualisierungsregeln

- Bestehende Dateien bevorzugt erweitern oder korrigieren statt neu anzulegen.
- Keine Duplikate erzeugen, wenn das Thema bereits dokumentiert ist.
- Bei Umbenennungen oder strukturellen Änderungen interne Verweise mitziehen.
- Entfernte Systeme und Dienste vollständig aus der Dokumentation entfernen statt sie als Altlast stehenzulassen.
- Veraltete Guides und Wissensnotizen auf den aktuellen Stand bringen statt alte und neue Varianten parallel zu pflegen.
- Wenn eine neue Datei notwendig ist, prüfe vorher, ob bereits eine thematisch passende Datei existiert.

---

## Vault-Abgrenzung

- `Sysadmin-Wiki` enthält allgemeines Wissen über Technologien, Plattformen, Konzepte und Guides zur Zielerreichung.
- `Homelab-Wiki` enthält das konkrete eigene Setup: reale Hosts, Dienste, Netzsegmente, Verbindungen und Abhängigkeiten.
- Allgemeine Theorie, Produktwissen und Best Practices gehören in die `Sysadmin-Wiki`.
- Konkrete Dokumentation des eigenen Setups, der eigenen Infrastruktur und real genutzter Services gehört in die `Homelab-Wiki`.
- `Homelab-Wiki` ist eine technische Ist-Dokumentation, kein Lern- oder Erklärformat.
- In der `Homelab-Wiki` keine allgemeinen Technologie-Erklärungen, keine How-to-Abschnitte und keine didaktischen Ausführungen ergänzen.
- Keine Crosslinks von `Homelab-Wiki` zur `Sysadmin-Wiki`, außer dies wird ausdrücklich gewünscht.
- Wenn eine Sysadmin-Notiz ein Praxisbeispiel aus dem eigenen Setup erwähnt, dieses nur knapp halten und nicht die Homelab-Dokumentation ersetzen.

---

## Qualitätscheckliste

Vor dem Abschließen einer Markdown-Datei prüfen:

- Alle `[[Wiki-Links]]` zeigen auf existierende Dateien
- Es gibt nur einen H1-Titel
- Alle Hauptabschnitte haben Inhalt
- Alle Code-Blöcke haben eine Sprachkennung
- Verschachtelte Fences sind korrekt
- Tabellen sind korrekt formatiert
- Callouts nutzen gültige Obsidian-Syntax
- `## Verwandte Notizen` steht am Ende, wenn interne Verweise sinnvoll sind
- Sprache und Struktur sind konsistent

---

## Definition of Done

Eine Markdown-Datei ist erst fertig, wenn:

- der Inhalt technisch korrekt und verständlich ist
- die Struktur sauber und scanbar ist
- CLI und GUI berücksichtigt sind, wenn beides relevant ist
- alle internen Links gültig sind
- sinnvolle Tags und Frontmatter gesetzt sind, wenn sie Mehrwert bringen
- keine offensichtlichen Dubletten oder veralteten Reste zurückbleiben

---

## Geltung für Agenten

Spezialisierte Agenten in diesem Repo bauen auf diesen Regeln auf. Agenten sollen deshalb ihre Fachlogik beschreiben und gemeinsame Obsidian-, Markdown- und Qualitätsregeln nicht unnötig duplizieren.
