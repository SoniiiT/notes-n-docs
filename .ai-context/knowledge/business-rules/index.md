# Geschäfts- und Konsistenzregeln

> [!ABSTRACT]
> Die Regeln betreffen hier die Dokumentationsdomäne, nicht eine Anwendung mit Geschäftslogik.

---

## Bestätigte Regeln

1. Ein Homelab-Host in `servers/` soll in `Homelab-Wiki/README.md` erscheinen.
2. Ein wichtiger Dienst in `services/` soll im passenden Host und in der README konsistent auftauchen.
3. Öffentliche Exposure muss in Service-, Host- und Übersichtsdateien gleich gekennzeichnet sein.
4. VPN-, Reverse-Proxy- und Cloud-Verbindungen müssen in `network/topology.md` sichtbar sein.
5. Entfernte Hosts oder Dienste müssen aus Tabellen, Wiki-Links und Mermaid-Diagrammen entfernt werden.
6. Fehlende oder unsichere Informationen werden als offen markiert und nicht erfunden.
7. Homelab-Cloud-Verbindungen sind laut vorhandener Doku auf den freigegebenen Pterodactyl-Wings-Zugriff beschränkt; Details vor Änderungen an der Netzfreigabe gegen die aktuelle Quelle prüfen.

**Quellenstatus:** Die ersten sechs Regeln sind explizit in `.github/agents/homelab-docs.agent.md` festgelegt. Die siebte Regel ist in den Homelab-Notizen als dokumentierter Ist-Zustand beschrieben.

