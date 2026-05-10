---
title: LXC-TELEPORT-PROXY
status: draft
type: server
tags:
  - homelab
---

# LXC-TELEPORT-PROXY

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Ubuntu Server 24.04               |
| CPU                  | 4 vCPU                            |
| RAM                  | 32 GB                             |
| Disk / Storage       | `M2-Storage:` 32 GB               |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
Lokaler Teleport-Proxy als Brücke zwischen dem Cloud-Teleport-Server und dem Homelab.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| Teleport Proxy | Offen | Zugriff auf Server und interne WebUIs aus dem Homelab heraus bereitstellen | Intern |

---

## Netzwerk
- IP: `192.168.178.48`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - [[CLOUD-SRV-03]]
  - Lokales LAN
- VPN-Bezug:
  - Funktional mit dem Teleport-Server gekoppelt

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Teleport, Details offen
- Storage: `M2-Storage:` 32 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - [[CLOUD-SRV-03]]

---

## Notizen
- Portbelegung und konkrete Anbindung an Teleport sind noch offen.

