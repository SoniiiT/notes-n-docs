---
title: LXC-WIREGUARD
status: draft
type: server
tags:
  - homelab
---

# LXC-WIREGUARD

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Debian 13                         |
| CPU                  | 2 vCPU                            |
| RAM                  | 2 GB                              |
| Disk / Storage       | `M2-Storage:` 16 GB               |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
Zentraler WireGuard-Server für die Cloud-Anbindung.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[WireGuard]] | `vpn.soniiit.net` | VPN für Cloud-Server und Homelab-Anbindung | Öffentlich |

---

## Netzwerk
- IP: `192.168.178.45`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - [[CLOUD-SRV-01]]
  - [[CLOUD-SRV-02]]
  - [[CLOUD-SRV-03]]
  - [[LXC-PTERODACTYL-WINGS]]
- VPN-Bezug:
  - `wg0`: `10.0.0.1/24`
  - `wg1`: `10.0.10.1/24`

---

## Betrieb
- Reverse Proxy: Nicht zutreffend
- Authentifizierung: WireGuard-Peers, Details offen
- Storage: `M2-Storage:` 16 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - DNS und Internet für externe Erreichbarkeit

---

## Notizen
- Alle Cloud-Server verbinden auf diesen Host.
- Die Cloud-Seite darf im Homelab nur [[LXC-PTERODACTYL-WINGS]] erreichen.
- Die genaue Nutzung von `wg0` und `wg1` ist noch offen.

