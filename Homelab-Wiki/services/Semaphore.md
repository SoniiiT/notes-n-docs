---
title: Semaphore
status: draft
type: service
tags:
  - homelab
---

# Semaphore

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-SEMAPHORE]] |
| Zweck | Ansible-Automatisierung |
| Port / Domain | Offen |
| Exposure | Intern |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - Zielsysteme für Automatisierung
- Nutzt:
  - `M2-Storage:` 32 GB auf [[LXC-SEMAPHORE]]
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Nicht bekannt
- Storage / Daten: `M2-Storage:` 32 GB auf [[LXC-SEMAPHORE]]
- Monitoring: Offen
- Besonderheiten:
  - Dient als Ansible-Automatisierungsserver.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-SEMAPHORE]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Scanopy]]

