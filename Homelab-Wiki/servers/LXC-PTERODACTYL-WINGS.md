---
title: LXC-PTERODACTYL-WINGS
status: draft
type: server
tags:
  - homelab
---

# LXC-PTERODACTYL-WINGS

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Ubuntu Server 24.04               |
| CPU                  | 6 vCPU                            |
| RAM                  | 32 GB                             |
| Disk / Storage       | `HDD-Storage:` 512 GB             |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
Pterodactyl-Node für den Betrieb von Game-Servern.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Pterodactyl-Wings]] | Offen | Ausführung der Game-Server | VPN-only / Intern |

---

## Netzwerk
- IP: `192.168.178.41`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - [[LXC-PTERODACTYL-PANEL]]
  - [[LXC-WIREGUARD]]
  - Cloud-Server über VPN
- VPN-Bezug:
  - Einziger explizit freigegebener Homelab-Zielhost für Cloud-Server

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `HDD-Storage:` 512 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - [[LXC-PTERODACTYL-PANEL]]
  - [[LXC-WIREGUARD]]

---

## Notizen
- Die öffentlich exponierten Spiel-Ports werden aktuell über [[CLOUD-SRV-01]] weitergeleitet.

