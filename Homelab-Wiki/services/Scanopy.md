---
title: Scanopy
status: draft
type: service
tags:
  - homelab
---

# Scanopy

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-SCANOPY]] |
| Zweck | Automatische Netzwerkdokumentation |
| Port / Domain | Offen |
| Exposure | Intern |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - Zugriff auf Netzwerkdaten
- Nutzt:
  - `M2-Storage:` 16 GB auf [[LXC-SCANOPY]]
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Nicht bekannt
- Storage / Daten: `M2-Storage:` 16 GB auf [[LXC-SCANOPY]]
- Monitoring: Offen
- Besonderheiten:
  - Dient der automatischen Netzwerkdokumentation.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-SCANOPY]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Semaphore]]

