---
title: LXC-OMADA
status: draft
type: server
tags:
  - homelab
---

# LXC-OMADA

## Übersicht
| Feld | Wert |
|------|------|
| Typ | LXC |
| Standort | Homelab |
| Anbieter / Plattform | Proxmox |
| OS | Debian 12 |
| CPU | 2 vCPU |
| RAM | 4 GB |
| Disk / Storage | `M2-Storage:` 16 GB |
| Erreichbarkeit | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup | Nein |

---

## Rolle
Omada-Controller für kompatible TP-Geräte.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Omada]] | Offen | Verwaltung und Updates für Netzwerkgeräte | Intern |

---

## Netzwerk
- IP: `192.168.178.51`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - Lokales LAN
  - TP-Geräte
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
- Konkrete Port- und Domain-Details sind noch offen.

