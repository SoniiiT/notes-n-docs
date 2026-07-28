# Entfernung: Homelab-Komponente

> [!WARNING]
> Eine Host- oder Dienstdatei darf erst entfernt werden, wenn alle Bestandsübersichten und Beziehungen bereinigt sind.

---

## Checkliste

1. Betroffene Primärdatei und Komponententyp bestimmen.
2. Alle Verweise per Repository-Suche finden, ohne sensible Werte auszugeben.
3. `Homelab-Wiki/README.md` und gegebenenfalls `todo.md` aktualisieren.
4. Bei Hosts zugehörige Service-, Netzwerk-, Cloud- und Backup-Bezüge prüfen.
5. Bei Diensten Hosttabelle, Domains, Exposure, Auth-, Storage- und Monitoring-Bezüge prüfen.
6. `network/topology.md` und Mermaid-Diagramme bereinigen.
7. Verwaiste Wiki-Links und veraltete Tabellenzeilen prüfen.
8. Erst danach die Datei entfernen und den Kontextindex beim nächsten Refresh aktualisieren.

---

## Rollback und Kompatibilität

Es gibt keine bekannte Datenmigration. Der Rollback ist die Wiederherstellung der gelöschten Markdown-Datei und der konsistenten Referenzen aus Git. Externe Systeme, DNS-Einträge, VPN-Peers und Backups sind vor einer operativen Abschaltung separat zu prüfen; dieses Repository beweist deren tatsächlichen Zustand nicht.

