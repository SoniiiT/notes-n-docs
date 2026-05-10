---
title: LXC-SEMAPHORE
status: draft
type: server
tags:
  - homelab
---

# LXC-SEMAPHORE

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Ubuntu Server 24.04               |
| CPU                  | 2 vCPU                            |
| RAM                  | 8 GB                              |
| Disk / Storage       | `M2-Storage:` 32 GB               |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
Ansible-Automatisierungsserver.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| [[Semaphore]] | Offen | Verwaltung und Ausführung von Automatisierungen | Intern |

---

## Netzwerk
- IP: `192.168.178.50`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - Lokales LAN
  - Infrastruktur-Ziele für Automatisierung
- VPN-Bezug: Nicht bekannt

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `M2-Storage:` 32 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - Zielsysteme für Automatisierung

---

## Notizen
- Repo-, Credential- und Job-Details sind noch offen.

