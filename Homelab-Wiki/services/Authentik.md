---
title: Authentik
status: draft
type: service
tags:
  - homelab
---

# Authentik

## Übersicht
| Feld | Wert |
|------|------|
| Host | [[CLOUD-SRV-02]] |
| Zweck | Zentrale SSO-Plattform |
| Port / Domain | `sso.soniiit.net` |
| Exposure | Öffentlich |
| Authentifizierung | Offen |
| Backup | Nein |

---

## Abhängigkeiten
- Benötigt:
  - DNS / Cloudflare
  - Internet-Erreichbarkeit
- Nutzt:
  - `SSD:` 40 GB auf [[CLOUD-SRV-02]]
- Hängt ab von:
  - [[LXC-WIREGUARD]] für Homelab-Anbindung

---

## Betrieb
- Reverse Proxy: Offen
- Storage / Daten: `SSD:` 40 GB auf [[CLOUD-SRV-02]]
- Monitoring: Offen
- Besonderheiten:
  - Details zu angebundenen Diensten und Policies sind noch offen.

---

## Verknüpfte Systeme
- Server:
  - [[CLOUD-SRV-02]]
- Netzwerk:
  - [[topology]]
- Verwandte Dienste:
  - [[Teleport]]

