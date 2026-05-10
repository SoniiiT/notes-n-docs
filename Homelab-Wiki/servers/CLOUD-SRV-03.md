---
title: CLOUD-SRV-03
status: draft
type: server
tags:
  - homelab
---

# CLOUD-SRV-03

## Übersicht
| Feld                 | Wert                |
| -------------------- | ------------------- |
| Typ                  | Cloud-VM            |
| Standort             | Cloud               |
| Anbieter / Plattform | Hetzner             |
| OS                   | Ubuntu Server 24.04 |
| CPU                  | 2 vCPU              |
| RAM                  | 4 GB                |
| Disk / Storage       | `SSD:` 40 GB        |
| Erreichbarkeit       | 24/7                |
| Backup               | Nein                |

---

## Rolle
Teleport-Server für den Zugriff auf Server und interne WebUIs von überall.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Teleport]] | `teleport.soniiit.net` | Zugriff auf Server und interne WebUIs | Öffentlich |

---

## Netzwerk
- IP: `46.224.192.134`
- Subnetz: Öffentliches Cloud-Netz, intern offen
- VLANs: Nicht zutreffend
- Netzwerkzone: `eu-central`
- Stadt: Nürnberg
- Land: Deutschland
- Verbindungen zu:
  - [[LXC-WIREGUARD]]
  - [[LXC-TELEPORT-PROXY]]
- VPN-Bezug:
  - Peer am Homelab-WireGuard

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `SSD:` 40 GB
- Abhängigkeiten:
  - [[LXC-WIREGUARD]]
  - [[LXC-TELEPORT-PROXY]]

---

## Notizen
- Details zu Rollen, Usern und Absicherung sind noch offen.
- Region ist noch offen.

