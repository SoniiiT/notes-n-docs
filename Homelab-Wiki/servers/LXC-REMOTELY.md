---
title: LXC-REMOTELY
status: draft
type: server
tags:
  - homelab
---

# LXC-REMOTELY

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Ubuntu Server 24.04               |
| CPU                  | 1 vCPU                            |
| RAM                  | 8 GB                              |
| Disk / Storage       | `M2-Storage:` 32 GB               |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
Remote-Support-Dienst, ähnlich zu TeamViewer Quick Support.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Remotely]] | `remote.soniiit.net` | Remote-Support | Öffentlich |

---

## Netzwerk
- IP: `192.168.178.46`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - Lokales LAN
  - [[PVE-SRV-01]]
- VPN-Bezug: Nicht bekannt

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `M2-Storage:` 32 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - DNS und Internet für externe Nutzung

---

## Notizen
- Details zu Port, Auth und Hardening sind noch offen.

