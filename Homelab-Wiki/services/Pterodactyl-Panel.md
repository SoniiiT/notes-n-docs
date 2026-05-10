---
title: Pterodactyl-Panel
status: draft
type: service
tags:
  - homelab
---

# Pterodactyl-Panel

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-PTERODACTYL-PANEL]] |
| Zweck | Verwaltung der Pterodactyl-Nodes und Game-Server |
| Port / Domain | `panel.soniiit.net` |
| Exposure | Öffentlich |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - DNS / Cloudflare
  - Internet-Erreichbarkeit
- Nutzt:
  - [[Pterodactyl-Wings]]
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Offen
- Storage / Daten: `M2-Storage:` 32 GB auf [[LXC-PTERODACTYL-PANEL]]
- Monitoring: Offen
- Besonderheiten:
  - Zentraler Einstiegspunkt für die Verwaltung der Game-Server.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-PTERODACTYL-PANEL]]
  - [[LXC-PTERODACTYL-WINGS]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Pterodactyl-Wings]]

