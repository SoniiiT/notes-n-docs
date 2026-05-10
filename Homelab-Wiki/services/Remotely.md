---
title: Remotely
status: draft
type: service
tags:
  - homelab
---

# Remotely

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-REMOTELY]] |
| Zweck | Remote-Support |
| Port / Domain | `remote.soniiit.net` |
| Exposure | Öffentlich |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - DNS / Cloudflare
  - Internet-Erreichbarkeit
- Nutzt:
  - Lokale Anwendung auf [[LXC-REMOTELY]]
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Offen
- Storage / Daten: `M2-Storage:` 32 GB auf [[LXC-REMOTELY]]
- Monitoring: Offen
- Besonderheiten:
  - Einsatz ähnlich TeamViewer Quick Support.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-REMOTELY]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Teleport]]

