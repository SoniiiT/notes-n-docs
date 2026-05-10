---
title: Harbor
status: draft
type: service
tags:
  - homelab
---

# Harbor

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[LXC-HARBOR]] |
| Zweck | Container-Registry |
| Port / Domain | `reg.soniiit.net` |
| Exposure | Öffentlich |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - DNS / Cloudflare
  - Internet-Erreichbarkeit
- Nutzt:
  - Lokale Registry-Daten auf [[LXC-HARBOR]]
- Hängt ab von:
  - [[PVE-SRV-01]]

---

## Betrieb
- Reverse Proxy: Offen
- Storage / Daten: `M2-Storage:` 32 GB auf [[LXC-HARBOR]]
- Monitoring: Offen
- Besonderheiten:
  - Weitere Registry-Details sind noch offen.

---

## Verknüpfte Systeme
- Server:
  - [[LXC-HARBOR]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Bind9]]

