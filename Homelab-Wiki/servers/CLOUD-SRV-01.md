---
title: CLOUD-SRV-01
status: draft
type: server
tags:
  - homelab
---

# CLOUD-SRV-01

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
VPN-Peer mit öffentlicher IP und Portweiterleitung für externe Zugriffe auf Game-Server und weitere Homelab-Dienste.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| VPN-Peer / Forwarding | `116.203.182.97` | Weiterleitung externer Anfragen ins Homelab über VPN | Öffentlich |

---

## Netzwerk
- IP: `116.203.182.97`
- Subnetz: Öffentliches Cloud-Netz, intern offen
- VLANs: Nicht zutreffend
- Netzwerkzone: `eu-central`
- Stadt: Nürnberg
- Land: Deutschland
- Verbindungen zu:
  - [[LXC-WIREGUARD]]
  - [[LXC-PTERODACTYL-WINGS]]
- VPN-Bezug:
  - Peer am Homelab-WireGuard

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `SSD:` 40 GB
- Abhängigkeiten:
  - [[LXC-WIREGUARD]]
  - Internet-Konnektivität

---

## Notizen
- Kann laut aktuellem Stand auch weitere Ports mit öffentlicher IP auf andere Homelab-Services weiterleiten.
- Region und Absicherung sind noch offen.

