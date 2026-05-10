---
title: Omada
status: draft
type: service
tags:
  - homelab
---

# Omada

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-OMADA]] |
| Zweck | Verwaltung kompatibler TP-Geräte |
| Port / Domain | Offen |
| Exposure | Intern |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - Lokales LAN
  - Erreichbarkeit der Netzwerkgeräte
- Nutzt:
  - `M2-Storage:` 16 GB auf [[LXC-OMADA]]
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Nicht bekannt
- Storage / Daten: `M2-Storage:` 16 GB auf [[LXC-OMADA]]
- Monitoring: Offen
- Besonderheiten:
  - Dient für Konfiguration und Updates von Switches, Router und APs.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-OMADA]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Bind9]]

