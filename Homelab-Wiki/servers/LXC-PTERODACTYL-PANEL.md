---
title: LXC-PTERODACTYL-PANEL
status: draft
type: server
tags:
  - homelab
---

# LXC-PTERODACTYL-PANEL

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Ubuntu Server 24.04               |
| CPU                  | 2 vCPU                            |
| RAM                  | 32 GB                             |
| Disk / Storage       | `M2-Storage:` 32 GB               |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
Weboberfläche und zentrale Verwaltung für Pterodactyl-Nodes.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Pterodactyl-Panel]] | `panel.soniiit.net` | Verwaltung der Game-Server | Öffentlich |

---

## Netzwerk
- IP: `192.168.178.40`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - [[LXC-PTERODACTYL-WINGS]]
  - Lokales LAN
- VPN-Bezug: Nicht direkt bekannt

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `M2-Storage:` 32 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - [[LXC-PTERODACTYL-WINGS]]

---

## Notizen
- Die genaue Auth-Integration ist noch offen.

