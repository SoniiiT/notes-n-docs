# Feature-Guide: Homelab-Inventar

> [!ABSTRACT]
> Das Homelab-Inventar bildet Hosts, Dienste, Netzwerkbeziehungen, Exposure und Cloud-Anbindung als verknüpfte Markdown-Dateien ab.

---

## Exposition und Implementierung

- Einstieg: `Homelab-Wiki/README.md`
- Hosts: `Homelab-Wiki/servers/`
- Dienste: `Homelab-Wiki/services/`
- Netzwerk: `Homelab-Wiki/network/topology.md`
- Offene Erhebungen: `Homelab-Wiki/todo.md`
- Agentenregeln: `.github/agents/homelab-docs.agent.md`

---

## Abhängigkeiten und Invarianten

Die Beziehung zwischen Host, Dienst, Domain, Port, Exposure, Authentifizierung, Storage, Backup und Netzwerk muss in den betroffenen Dateien zusammenpassen. Öffentliche Dienste und die eingeschränkte Cloud-Anbindung sind besonders reviewbedürftig. Nicht dokumentierte Ports, Peer-Details und genaue VPN-Zwecke sind offen.

---

## Änderungs- und Testpflichten

Bei einer Änderung: Primärdatei aktualisieren, README prüfen, direkte Wiki-Links und Abhängigkeiten suchen, Topologie und Mermaid prüfen, offene Punkte in `todo.md` nachziehen. Es gibt keinen automatisierten Test; die minimal notwendige Prüfung ist eine Pfad-, Link-, Tabellen- und Konsistenzprüfung.

---

## Rollout und Entfernung

Neue Hosts oder wichtige Dienste erst nach Bestätigung des Zielpfads anlegen. Beim Entfernen die Primärdatei, README-Tabellen, Host-/Service-Verweise, Domains, Topologie, Mermaid-Knoten und zugehörige TODOs bereinigen. Vor dem Löschen alle Referenzen per Repository-Suche ermitteln.

