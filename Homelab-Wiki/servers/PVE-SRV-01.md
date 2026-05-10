---
title: PVE-SRV-01
status: draft
type: server
tags:
  - homelab
---

# PVE-SRV-01

## Übersicht
| Feld                 | Wert                                                                  |
| -------------------- | --------------------------------------------------------------------- |
| Typ                  | Physisch                                                              |
| Standort             | Homelab                                                               |
| Anbieter / Plattform | Proxmox                                                               |
| OS                   | Proxmox VE 9                                                          |
| CPU                  | `Cores:` 4, `Threads:` 8                                              |
| RAM                  | 128 GB                                                                |
| Disk / Storage       | `local`: 256 GB M2, `M2-Storage`: 1 TB, `HDD-Storage`: 10 TB          |
| Erreichbarkeit       | Eingeschränkt, Sommer 05:00-07:00 offline, Winter 05:00-06:00 offline |
| Backup               | nein                                                                  |

---

## Rolle
Zentraler Hypervisor für die Homelab-VMs und -LXCs.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| Proxmox VE | Offen | Verwaltung des Hypervisors und der Gäste | Intern |

---

## Netzwerk
- IP: `192.168.178.6`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - Lokales LAN
  - Gehostete VMs und LXCs
- VPN-Bezug: Kein direkter VPN-Endpunkt bekannt

---

## Betrieb
- Reverse Proxy: Keiner bekannt
- Authentifizierung: Offen
- Storage:
  - `local`: 256 GB M2
  - `M2-Storage`: 1 TB MS
  - `HDD-Storage`: 10 TB
- Abhängigkeiten:
  - Strom, LAN und Fritzbox 6670

---

## Notizen
- Viele Homelab-Services sind indirekt an die Offline-Fenster dieses Hosts gekoppelt.
