---
title: WireGuard
status: draft
type: service
tags:
  - homelab
---

# WireGuard

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-WIREGUARD]] |
| Zweck | VPN-Anbindung zwischen Homelab und Cloud |
| Port / Domain | `vpn.soniiit.net` |
| Exposure | Öffentlich |
| Authentifizierung | WireGuard-Peers, Details offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - Internet-Uplink mit DS-Lite
  - Erreichbarkeit von `vpn.soniiit.net`
- Nutzt:
  - `wg0`: `10.0.0.1/24`
  - `wg1`: `10.0.10.1/24`
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Nicht zutreffend
- Storage / Daten: Konfiguration auf [[LXC-WIREGUARD]]
- Monitoring: Offen
- Besonderheiten:
  - Alle Cloud-Server verbinden auf diesen Dienst.
  - Im Homelab ist für die Cloud nur [[LXC-PTERODACTYL-WINGS]] freigegeben.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-WIREGUARD]]
  - [[CLOUD-SRV-01]]
  - [[CLOUD-SRV-02]]
  - [[CLOUD-SRV-03]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Pterodactyl-Wings]]

