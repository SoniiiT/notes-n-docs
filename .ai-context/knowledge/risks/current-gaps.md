# Risiken und offene Punkte

> [!ABSTRACT]
> Diese Punkte sind aus dem aktuellen Dateibestand und den vorhandenen TODOs ersichtlich. Sie sind keine erfundenen Betriebsannahmen.

---

## Dokumentationsrisiken

- Es gibt keine automatisierte Prüfung für Wiki-Links, Tabellenkonsistenz oder Mermaid-Referenzen.
- Viele Dokumente tragen `status: draft`; der Reifegrad ist deshalb nicht überall produktionsreif.
- Die Sysadmin-Wiki hat keinen erkannten zentralen README-Einstiegspunkt wie die Homelab-Wiki.
- Homelab-TODOs nennen offene Port-, Exposure-, Auth-, Backup-, Monitoring- und Abhängigkeitsdetails.
- Bei den WireGuard-Segmenten `wg0` und `wg1` ist der genaue Zweck in der Quelle noch offen.
- Die Quelle enthält Infrastruktur- und öffentliche Erreichbarkeitsdetails. Neue Indizes dürfen daraus keine geheimen Zugangswerte ableiten.

---

## Review-Auslöser

Diesen Kontext aktualisieren, wenn sich Vault-Struktur, Host-/Dienstinventar, Agentenregeln, Obsidian-Plugins oder die Cloud-/Netzwerktopologie wesentlich ändern. Bei Änderungen an vorhandenen `.smart-env`-Daten (falls solche Dateien vorhanden sind) oder geheimnisnahen Inhalten nur Metadaten und Rollen indexieren, niemals Werte.

