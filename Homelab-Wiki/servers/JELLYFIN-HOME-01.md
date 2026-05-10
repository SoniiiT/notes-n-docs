---
title: JELLYFIN-HOME-01
status: draft
type: server
tags:
  - homelab
---

# JELLYFIN-HOME-01

## Übersicht
| Feld                 | Wert                               |
| -------------------- | ---------------------------------- |
| Typ                  | VM                                 |
| Standort             | Homelab                            |
| Anbieter / Plattform | Proxmox                            |
| OS                   | Ubuntu Server 24.04                |
| CPU                  | 4 vCPU                             |
| RAM                  | 32 GB                              |
| Disk / Storage       | `Root:` 64 GB, `HDD-Storage:` 5 TB |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]]  |
| Backup               | Nein                               |

---

## Rolle
Media-Server für Serien, Filme, Spiele, Comics und Manga.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| Medienbereitstellung | Offen | Bereitstellung der Medieninhalte | Offen |

---

## Netzwerk
- IP: `192.168.178.98`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - [[PVE-SRV-01]]
  - Lokales LAN
- VPN-Bezug: Nicht bekannt

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage: `Root:` 64 GB, `HDD-Storage:` 5 TB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - Lokales LAN

---

## Notizen
- Öffentliche Domain oder direkter Exposure-Pfad sind bisher nicht dokumentiert.

