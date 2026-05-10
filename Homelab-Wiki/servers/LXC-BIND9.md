---
title: LXC-BIND9
status: draft
type: server
tags:
  - homelab
---

# LXC-BIND9

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Ubuntu Server 24.04               |
| CPU                  | 1 vCPU                            |
| RAM                  | 1 GB                              |
| Disk / Storage       | `M2-Storage:` 32 GB               |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
Interner DNS-Server für das Homelab.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Bind9]] | `home.soniiit.net`, `dev.soniiit.net`, `demo.soniiit.net` | Interne DNS-Auflösung | Intern |

---

## Netzwerk
- IP: `192.168.178.3`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - Lokales LAN
  - Homelab-Server
- VPN-Bezug: Nicht bekannt

---

## Betrieb
- Reverse Proxy: Nicht zutreffend
- Authentifizierung: Nicht zutreffend / offen
- Storage: `M2-Storage:` 32 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - Fritzbox 6670

---

## Notizen
- Die CPU basiert auf dem Proxmox-Host.
- RAM-Grösse ist noch offen.

