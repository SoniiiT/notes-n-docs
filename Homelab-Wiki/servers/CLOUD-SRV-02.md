---
title: CLOUD-SRV-02
status: draft
type: server
tags:
  - homelab
---

# CLOUD-SRV-02

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
Host für Authentik und externes Uptime-Monitoring.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Authentik]] | `sso.soniiit.net` | Zentrale SSO-Plattform | Öffentlich |
| Uptime Kuma | `status.soniiit.net` | Externes Monitoring | Öffentlich |

---

## Netzwerk
- IP: `46.224.184.186`
- Subnetz: Öffentliches Cloud-Netz, intern offen
- VLANs: Nicht zutreffend
- Netzwerkzone: `eu-central`
- Stadt: Nürnberg
- Land: Deutschland
- Verbindungen zu:
  - [[LXC-WIREGUARD]]
  - Öffentliche Internet-Dienste
- VPN-Bezug:
  - Peer am Homelab-WireGuard

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `SSD:` 40 GB
- Abhängigkeiten:
  - [[LXC-WIREGUARD]]
  - DNS / Cloudflare

---

## Notizen
- Details zur Absicherung und Region sind noch offen.
- Für Uptime Kuma ist die Instanzbezeichnung in der Doku noch nicht weiter unterschieden.
