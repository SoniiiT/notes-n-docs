---
title: TODO
status: draft
tags:
  - homelab
---

# TODO

> [!ABSTRACT]
> Offene Punkte für die weitere Dokumentation des Homelabs und der Cloud-Systeme.

---

## Offene Dienste

| Thema | Status | Notiz |
|------|------|------|
| Home Assistant | Offen | Öffentliche Erreichbarkeit bekannt: `homeassistant.soniiit.net` |
| N8N | Offen | Öffentliche Erreichbarkeit bekannt: `n8n.soniiit.net` |
| Uptime Kuma im Homelab | Offen | Öffentliche Erreichbarkeit bekannt: `uptime.soniiit.net` |
| Uptime Kuma in der Cloud | Offen | Öffentliche Erreichbarkeit bekannt: `status.soniiit.net` |
| Authentik | Offen | Öffentliche Erreichbarkeit bekannt: `sso.soniiit.net` |
| Teleport | Offen | Öffentliche Erreichbarkeit bekannt: `teleport.soniiit.net` |
| Harbor | Offen | Öffentliche Erreichbarkeit bekannt: `reg.soniiit.net` |
| Remotely | Offen | Öffentliche Erreichbarkeit bekannt: `remote.soniiit.net` |
| Pterodactyl Panel | Offen | Öffentliche Erreichbarkeit bekannt: `panel.soniiit.net` |
| VPN | Offen | Öffentliche Erreichbarkeit bekannt: `vpn.soniiit.net` |
| Weitere noch nicht genannte Dienste | Offen | Später separat erfassen |

---

## Offene Auth- und Zugangsfragen

| Thema | Status | Notiz |
|------|------|------|
| Zentrale Authentifizierung | Offen | Authentik ist bekannt, Details zu angebundenen Diensten fehlen |
| Lokale Accounts | Offen | Noch nicht dokumentiert |
| Teleport-Zugriffsmodell | Offen | Rollen, User und geschützte Ziele fehlen |
| Zugriff auf öffentliche Dienste | Offen | Pro Dienst Auth-Verfahren noch offen |
| Absicherung der Cloud-Server | Offen | Hardening, Firewall oder ähnliche Schutzmassnahmen noch offen |

---

## Offene Netzwerk- und Exposure-Details

| Thema | Status | Notiz |
|------|------|------|
| Port-Zuordnungen der öffentlichen Dienste | Offen | Domains sind bekannt, Ports fehlen weitgehend |
| Exponierungsweg pro Dienst | Offen | Direkt, Tunnel oder sonstige Weiterleitung noch nicht vollständig dokumentiert |
| WireGuard-Zweck von `wg0` | Offen | `10.0.0.1/24` bekannt, Nutzung noch offen |
| WireGuard-Zweck von `wg1` | Offen | `10.0.10.1/24` bekannt, Nutzung noch offen |
| DNS-Details | Teilweise | Fritzbox lokal, Bind9 für Server und Cloudflare als DNS-Provider bekannt |

---

## Offene Betriebsdetails

| Thema | Status | Notiz |
|------|------|------|
| Region der Cloud-Server | Offen | Anbieter Hetzner bekannt, Region fehlt |
| Monitoring-Konzept | Offen | Nur Teile über Uptime Kuma bekannt |
| Backup-Strategie gesamt | Offen | Einzelne Systeme bekannt, Gesamtüberblick fehlt |
| Storage-Zuordnung pro Dienst | Offen | Teilweise pro Host bekannt, nicht pro Dienst |
| Abhängigkeiten zwischen Diensten | Offen | Müssen später pro Dienst dokumentiert werden |

---

## Notizen

- Homelab-LAN: `192.168.178.0/24`
- WireGuard:
  - `wg0`: `10.0.0.1/24`
  - `wg1`: `10.0.10.1/24`
- Internet-Uplink: DS-Lite
- Alle Cloud-Server verbinden auf `LXC-WIREGUARD`.
- Cloud-Server dürfen im Homelab nur `LXC-PTERODACTYL-WINGS` erreichen.
- DNS-Provider für `soniiit.net`: Cloudflare

