---
title: Bind9
status: draft
type: service
tags:
  - homelab
---

# Bind9

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-BIND9]] |
| Zweck | Interner DNS für Homelab-Server |
| Port / Domain | `home.soniiit.net`, `dev.soniiit.net`, `demo.soniiit.net` |
| Exposure | Intern |
| Authentifizierung | Nicht zutreffend / offen |
| Backup | Ja |

---

## Abhängigkeiten
- Benötigt:
  - Lokales LAN
  - Fritzbox 6670
- Nutzt:
  - Interne DNS-Zonen
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Nicht zutreffend
- Storage / Daten: DNS-Zonen und Konfiguration auf [[LXC-BIND9]]
- Monitoring: Offen
- Besonderheiten:
  - Wird für alle Server im Homelab ausser Cloud als DNS genutzt.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-BIND9]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - Cloudflare

