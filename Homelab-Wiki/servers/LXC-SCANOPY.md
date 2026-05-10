---
title: LXC-SCANOPY
status: draft
type: server
tags:
  - homelab
---

# LXC-SCANOPY

## Übersicht
| Feld | Wert |
|------|------|
| Typ | LXC |
| Standort | Homelab |
| Anbieter / Plattform | Proxmox |
| OS | Debian 13 |
| CPU | 2 vCPU |
| RAM | 4 GB |
| Disk / Storage | `M2-Storage:` 16 GB |
| Erreichbarkeit | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup | Nein |

---

## Rolle
Automatische Netzwerkdokumentation.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Scanopy]] | Offen | Erfassung und Dokumentation des Netzwerks | Intern |

---

## Netzwerk
- IP: `192.168.178.52`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - Lokales LAN
  - Netzwerkgeräte und Ziele für Inventarisierung
- VPN-Bezug: Nicht bekannt

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `M2-Storage:` 16 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - Lokales Netzwerk

---

## Notizen
- Datenquellen und Monitoring-Anbindung sind noch offen.

