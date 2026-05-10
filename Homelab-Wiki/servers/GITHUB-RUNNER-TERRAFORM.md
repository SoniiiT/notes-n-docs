---
title: GITHUB-RUNNER-TERRAFORM
status: draft
type: server
tags:
  - homelab
---

# GITHUB-RUNNER-TERRAFORM

## Übersicht
| Feld                 | Wert                              |
| -------------------- | --------------------------------- |
| Typ                  | LXC                               |
| Standort             | Homelab                           |
| Anbieter / Plattform | Proxmox                           |
| OS                   | Ubuntu Server 24.04               |
| CPU                  | 1 vCPU                            |
| RAM                  | 4 GB                              |
| Disk / Storage       | `M2-Storage:` 32 GB               |
| Erreichbarkeit       | Eingeschränkt, wie [[PVE-SRV-01]] |
| Backup               | Nein                              |

---

## Rolle
GitHub CI/CD Runner für das Terraform-Repository.

---

## Dienste
| Dienst | Port/Domain | Zweck | Exposure |
|--------|-------------|-------|----------|
| GitHub Runner | Offen | Build- und CI/CD-Jobs für Terraform | Intern |

---

## Netzwerk
- IP: `192.168.178.70`
- Subnetz: `192.168.178.0/24`
- VLANs: Keine bekannt
- Netzwerkzone: `eu-central`
- Stadt: Stuttgart
- Land: Deutschland
- Verbindungen zu:
  - GitHub
  - Lokales LAN
- VPN-Bezug: Nicht bekannt

---

## Betrieb
- Reverse Proxy: Nicht zutreffend
- Authentifizierung: GitHub Runner-Registrierung, Details offen
- Storage: `M2-Storage:` 32 GB
- Abhängigkeiten:
  - [[PVE-SRV-01]]
  - GitHub-Konnektivität

---

## Notizen
- Repo-Zuordnung ist bekannt, konkrete Runner-Konfiguration noch offen.

