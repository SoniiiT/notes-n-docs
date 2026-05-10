---
title: Pterodactyl-Wings
status: draft
type: service
tags:
  - homelab
---

# Pterodactyl-Wings

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-PTERODACTYL-WINGS]] |
| Zweck | Ausführung der Game-Server |
| Port / Domain | Offen |
| Exposure | VPN-only / Intern |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - [[WireGuard]]
  - [[Pterodactyl-Panel]]
- Nutzt:
  - `HDD-Storage:` 512 GB auf [[LXC-PTERODACTYL-WINGS]]
- Hängt ab von:
  - [[PVE-SRV-01]]
  - [[CLOUD-SRV-01]] für externe Portweiterleitungen

---

## Betrieb
- Reverse Proxy: Nicht bekannt
- Storage / Daten: `HDD-Storage:` 512 GB auf [[LXC-PTERODACTYL-WINGS]]
- Monitoring: Offen
- Besonderheiten:
  - Einziger bekannter Homelab-Host, den die Cloud-Server direkt erreichen dürfen.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-PTERODACTYL-WINGS]]
  - [[LXC-PTERODACTYL-PANEL]]
  - [[CLOUD-SRV-01]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[WireGuard]]
  - [[Pterodactyl-Panel]]

