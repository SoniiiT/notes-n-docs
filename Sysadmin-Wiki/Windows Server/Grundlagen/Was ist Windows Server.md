---
tags: [windows-server, grundlagen, editionen, lizenzierung, 2022, 2025, core, desktop-experience]
aliases: ["Was ist Windows Server", "Windows Server Editionen", "WinSrv Grundlagen"]
---

# Was ist Windows Server

> [!ABSTRACT]
> Windows Server ist Microsofts Server-Betriebssystem für Unternehmensumgebungen. Es stellt Infrastruktur-Rollen wie Active Directory, DNS, Dateiserver und Virtualisierung bereit. Aktuell sind Windows Server 2022 (LTSC) und Windows Server 2025 (LTSC) relevant — 2025 bringt SMB-over-QUIC, verbesserte AD-Sicherheit und Hot-Patching.

---

## Versionsübersicht

| Version | Release | Support-Ende | Kernel | Bemerkung |
|---------|---------|-------------|--------|-----------|
| **Windows Server 2022** | Okt 2021 | Okt 2031 | Win11 21H2 | Aktueller LTS |
| **Windows Server 2025** | Nov 2024 | Okt 2034 | Win11 24H2 | Neuester LTS, Hot-Patching |
| Windows Server 2019 | Okt 2018 | Jan 2029 | RS5 | Älterer LTS, noch verbreitet |

> [!TIP] LTSC vs. SAC
> Windows Server nutzt ausschließlich **Long-Term Servicing Channel (LTSC)** — kein halbjährlicher Update-Kanal mehr. Langfristige Stabilität ist garantiert.

---

## Editionen

### Standard

- Bis zu **2 Hyper-V-VMs** pro Lizenz (inkl. Host)
- Alle Serverrollen verfügbar
- Geeignet für: mittelgroße Umgebungen, physische Server mit wenigen VMs

### Datacenter

- **Unbegrenzte VMs** pro Lizenz (pro physischem Host)
- Zusätzliche Features: Storage Spaces Direct (S2D), Shielded VMs, Software-Defined Networking
- Geeignet für: Rechenzentren, dichte Virtualisierung, Azure Stack HCI

### Essentials (nur bis 2022)

- Bis 25 Benutzer / 50 Geräte
- Vereinfachte Verwaltung
- ⚠️ Kein eigenständiges Produkt in 2025 mehr

```
Faustregel:
Standard:    < 4 VMs pro Host  → Standard ist günstiger
Datacenter:  ≥ 4 VMs pro Host  → Datacenter rechnet sich
```

---

## Installationsoptionen

### Desktop Experience (GUI)

- Vollständige grafische Oberfläche
- Server Manager, Systemsteuerung, Explorer
- Höherer Ressourcenverbrauch (~4 GB RAM Overhead)
- Empfohlen für: Entwicklung, erste Schritte, Remote Desktop Services

### Server Core (empfohlen für Produktion)

- Keine GUI — nur PowerShell und CMD
- Geringerer Speicherbedarf, kleinere Angriffsfläche
- Alle Serverrollen verfügbar (außer RDS mit GUI)
- Empfohlen für: AD DC, DNS, DHCP, File Server, Hyper-V

### Nano Server

- Extrem minimal, nur als Container-Image
- Keine Serverrollen mehr seit 2019
- Nur für Container-Workloads

```
Empfehlung:
Production DC, DNS, File Server, Hyper-V → Server Core
RDS, IIS (mit GUI-Verwaltung), WAC Host   → Desktop Experience
```

---

## Neue Features in Windows Server 2025

### SMB over QUIC

- SMB ohne VPN über das Internet (UDP 443)
- Für Remote-Clients — kein separates VPN nötig
- Erfordert Zertifikat auf dem File Server

```powershell
# SMB over QUIC aktivieren
Enable-WindowsOptionalFeature -Online -FeatureName SMBoverQUIC
```

### Hot-Patching (Azure Arc)

- Sicherheitspatches ohne Neustart (für Azure-Arc-verbundene Server)
- Reduziert Wartungsfenster erheblich

### Verbesserte AD-Sicherheit

- Kerberos mit AES-256 als Standard
- LDAP-Signing und Channel Binding erzwingen

### Storage Spaces Direct Verbesserungen

- Bis zu 4 PB pro Cluster
- Schnellere Neusynchronisierung

---

## Lizenzierung

### Core-basiertes Lizenzmodell

Windows Server 2022/2025 wird nach **physischen CPU-Kernen** lizenziert:

```
Mindest-Lizenz: 16 Cores pro Server (auch wenn weniger vorhanden)

Beispiel 2-Socket-Server mit je 8 Kernen (16 Kerne gesamt):
→ 1x Windows Server Standard (16-Core-Lizenz) = abgedeckt

Beispiel 2-Socket-Server mit je 12 Kernen (24 Kerne gesamt):
→ 1x Windows Server Standard 16-Core + 1x 8-Core Add-On
```

### Client Access Licenses (CAL)

Für den Zugriff auf Serverrollen (außer IIS öffentlich) braucht jeder User/Gerät eine **CAL**:

| CAL-Typ | Beschreibung |
|---------|-------------|
| Windows Server CAL | Grundlegender Zugriff |
| RDS CAL (User oder Device) | Remote Desktop Services |
| Exchange CAL | Exchange Server |

---

## Hardware-Anforderungen

| Komponente | Minimum | Empfohlen (Produktion) |
|------------|---------|----------------------|
| CPU | 1,4 GHz, 64-bit | 2x Multi-Core |
| RAM | 512 MB (Core) / 2 GB (GUI) | 16 GB+ |
| Festplatte | 32 GB | 100 GB+ (OS-Partition) |
| Netzwerk | 1x GbE | 2x GbE (Teaming) |

---

## Verwandte Notizen

- [[Windows Server Installation]] — Schritt-für-Schritt Installation
- [[Server Manager]] — Rollen und Features hinzufügen
- [[Windows Admin Center]] — Moderne webbasierte Verwaltung
- [[PowerShell für Admins]] — Administration ohne GUI
