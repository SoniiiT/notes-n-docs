---
title: Teleport
status: draft
type: service
tags:
  - homelab
---

# Teleport

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[CLOUD-SRV-03]] |
| Zweck | Zugriff auf Server und interne WebUIs |
| Port / Domain | `teleport.soniiit.net` |
| Exposure | Öffentlich |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - DNS / Cloudflare
  - Internet-Erreichbarkeit
- Nutzt:
  - [[LXC-TELEPORT-PROXY]] als lokale Brücke ins Homelab
- Hängt ab von:
  - [[LXC-WIREGUARD]]

---

## Betrieb
- Reverse Proxy: Offen
- Storage / Daten: `SSD:` 40 GB auf [[CLOUD-SRV-03]]
- Monitoring: Offen
- Besonderheiten:
  - Soll den Zugriff auf Server und interne WebUIs von überall ermöglichen.

---

## Verknüpfte Systeme
- Server:
  - [[CLOUD-SRV-03]]
  - [[LXC-TELEPORT-PROXY]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Authentik]]

