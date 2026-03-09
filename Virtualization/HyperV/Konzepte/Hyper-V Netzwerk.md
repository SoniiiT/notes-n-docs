---
title: Hyper-V Netzwerk
description: Vollständige Referenz zu virtuellen Switches, NIC Teaming, SR-IOV, VLANs, QoS, VMQ und NAT unter Hyper-V
tags:
  - virtualization
  - hyper-v
  - netzwerk
  - vswitch
  - vlan
  - nic-teaming
date: 2025-01-30
status: published
---

# Hyper-V Netzwerk

> [!ABSTRACT] TL;DR
> Hyper-V virtualisiert Netzwerke über **virtuelle Switches (vSwitches)**, die in drei Typen existieren: External, Internal und Private. Erweiterte Features wie NIC Teaming (LBFO), SR-IOV, VLAN-Tagging, QoS-Bandbreitenlimits und VMQ ermöglichen produktionstaugliche Netzwerkarchitekturen direkt aus PowerShell heraus.

---

## Virtuelle Switches (vSwitches)

Ein virtueller Switch ist das zentrale Netzwerkelement in Hyper-V. Er verbindet VM-Netzwerkadapter untereinander und/oder mit physischen NICs des Hosts. Jeder vSwitch wird im **Hyper-V Virtual Switch Manager** oder per PowerShell verwaltet.

### Die drei vSwitch-Typen

| Typ | VMs ↔ VMs | VMs ↔ Host | VMs ↔ Außennetz |
|-----|:---------:|:----------:|:---------------:|
| **External** | ✅ | ✅ | ✅ |
| **Internal** | ✅ | ✅ | ❌ |
| **Private** | ✅ | ❌ | ❌ |

---

#### External Switch

```
  [Internet / LAN]
        │
  [Physische NIC]
        │
  ┌─────┴──────────────────────────┐
  │     External vSwitch           │
  ├────────┬────────────┬──────────┤
  │  VM 1  │    VM 2   │  Host-OS │
  └────────┴────────────┴──────────┘
```

- Bindet sich an eine **physische NIC** des Hyper-V-Hosts
- VMs erhalten direkten Zugang zum physischen Netzwerk (eigene IP, eigene MAC)
- Das Host-OS bekommt automatisch einen **virtuellen Management-Adapter** (`vEthernet (SwitchName)`)
- **Einsatzszenarien:** Produktions-VMs, die im LAN sichtbar sein sollen; Server-VMs mit öffentlicher IP

```powershell
# External Switch erstellen
New-VMSwitch -Name "External-Switch" `
             -NetAdapterName "Ethernet 1" `
             -AllowManagementOS $true
```

> [!WARNING] Management-Adapter nicht vergessen
> Wenn `-AllowManagementOS $false` gesetzt wird, verliert der Host möglicherweise die Netzwerkverbindung über diese NIC. Standardmäßig sollte `-AllowManagementOS $true` aktiv sein oder eine dedizierte Management-NIC genutzt werden.

---

#### Internal Switch

```
  [Internet / LAN]        ← KEIN Zugang
        ✗
  ┌─────────────────────────────────┐
  │     Internal vSwitch            │
  ├────────┬────────────┬───────────┤
  │  VM 1  │    VM 2   │  Host-OS  │
  └────────┴────────────┴───────────┘
                                ↑
               Host kann als Router/NAT-Gateway fungieren
```

- Isoliertes Netzwerk zwischen VMs und dem **Host-OS**, aber **kein** Zugang zur physischen NIC
- Das Host-OS erhält ebenfalls einen virtuellen Adapter
- **Einsatzszenarien:** Testlabore, NAT-Netzwerke (Host als Gateway), Netzwerke für Containerworkloads

```powershell
# Internal Switch erstellen
New-VMSwitch -Name "Internal-Switch" -SwitchType Internal
```

---

#### Private Switch

```
  ┌─────────────────────────────────┐
  │     Private vSwitch             │
  ├────────┬────────────┬───────────┤
  │  VM 1  │    VM 2   │   VM 3    │
  └────────┴────────────┴───────────┘
             (Host-OS NICHT enthalten)
```

- Vollständige Isolation — **nur VMs** kommunizieren miteinander
- Host-OS hat keinen Adapter an diesem Switch
- **Einsatzszenarien:** Hochsicherheitsnetzwerke, Air-Gapped-Testumgebungen, Malware-Analyse

```powershell
# Private Switch erstellen
New-VMSwitch -Name "Private-Switch" -SwitchType Private
```

---

### VM-Netzwerkadapter mit vSwitch verbinden

```powershell
# Netzwerkadapter einer VM zuweisen
Add-VMNetworkAdapter -VMName "MeineVM" -SwitchName "External-Switch" -Name "NIC-Prod"

# Bestehenden Adapter umhängen
Connect-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod" -SwitchName "Internal-Switch"

# Alle Switches anzeigen
Get-VMSwitch

# Alle VM-Adapter anzeigen
Get-VMNetworkAdapter -VMName "MeineVM"
```

---

## Netzwerkadapter-Typen

Hyper-V bietet zwei grundlegend verschiedene virtuelle NIC-Typen:

| Eigenschaft | **Synthetischer Adapter** | **Legacy-Adapter (emuliert)** |
|-------------|--------------------------|-------------------------------|
| Technologie | Para-virtualisiert (VMBus) | Emuliert Intel 21140 |
| Geschwindigkeit | Sehr hoch (nativ) | Deutlich langsamer |
| Treiber | Integration Services nötig | Kein Treiber nötig |
| PXE-Boot | Nur mit separatem Legacy-NIC | ✅ Nativ möglich |
| Verfügbarkeit | Gen 1 & Gen 2 | **Nur Generation 1** |
| SR-IOV / VMQ | ✅ Unterstützt | ❌ Nicht unterstützt |
| Empfehlung | ✅ Standardwahl | Nur für PXE-Boot / alte OS |

> [!TIP] Immer synthetischen Adapter bevorzugen
> Synthetische Adapter liefern deutlich höheren Durchsatz und geringere CPU-Last. Legacy-Adapter nur verwenden, wenn PXE-Boot ohne UEFI benötigt wird oder das Gast-OS keine Integration Services unterstützt.

---

## NIC Teaming (LBFO)

**Load Balancing and Failover (LBFO)** fasst mehrere physische NICs zu einem logischen Team zusammen. Hyper-V kann dann einen vSwitch auf diesem Team aufbauen.

### Teaming-Modi

| Modus | Beschreibung | Switch-Anforderung |
|-------|-------------|-------------------|
| **Switch Independent** | Jede NIC kann an einem anderen Switch hängen | Keiner |
| **Static Teaming** | Manuell konfiguriertes Link Aggregation | Switch-seitige Konfig nötig |
| **LACP (802.3ad)** | Dynamisches Link Aggregation Control Protocol | Switch mit LACP-Support |

### Lastverteilungsalgorithmen

| Algorithmus | Verteilung nach | Typischer Einsatz |
|-------------|-----------------|-------------------|
| `Dynamic` | Flowlets (adaptiv) | **Empfohlen** — beste Balance |
| `HyperVPort` | VM-Port (MAC) | Hyper-V mit vielen VMs |
| `TransportPorts` | TCP/UDP-Ports | Gleichmäßige Lastverteilung |
| `IPAddresses` | Quell/Ziel-IP | Geringe Verbindungsanzahl |
| `MacAddresses` | MAC-Adressen | Einfache Szenarien |

### NIC Team erstellen und vSwitch darauf aufbauen

```powershell
# NIC Team erstellen (Switch Independent + Dynamic)
New-NetLbfoTeam -Name "NIC-Team-Prod" `
                -TeamMembers "Ethernet 1", "Ethernet 2" `
                -TeamingMode SwitchIndependent `
                -LoadBalancingAlgorithm Dynamic

# External Switch auf dem Team aufbauen
New-VMSwitch -Name "External-Team-Switch" `
             -NetAdapterName "NIC-Team-Prod" `
             -AllowManagementOS $true

# Team-Status prüfen
Get-NetLbfoTeam
Get-NetLbfoTeamMember -Team "NIC-Team-Prod"
```

> [!NOTE] LBFO-Ablösung in neueren Windows-Versionen
> Ab Windows Server 2022 empfiehlt Microsoft **Switch Embedded Teaming (SET)** anstelle von LBFO für Hyper-V-Umgebungen. SET wird direkt im vSwitch konfiguriert und unterstützt RDMA und SR-IOV besser.

### Switch Embedded Teaming (SET)

SET integriert das Teaming direkt in den Hyper-V vSwitch — kein separates NIC-Team-Objekt nötig:

```powershell
# SET-fähigen vSwitch mit zwei NICs erstellen
New-VMSwitch -Name "SET-Switch" `
             -NetAdapterName "Ethernet 1", "Ethernet 2" `
             -EnableEmbeddedTeaming $true `
             -AllowManagementOS $true
```

---

## SR-IOV (Single Root I/O Virtualization)

SR-IOV ermöglicht es, eine physische NIC in mehrere **Virtual Functions (VFs)** aufzuteilen und diese **direkt** an VMs durchzureichen — ohne den vSwitch-Overhead.

```
  [Physische NIC mit SR-IOV]
        │
  ┌─────┴──────────────────────────────┐
  │  Physical Function (PF)            │  → Host-OS / vSwitch-Verwaltung
  ├──────────┬──────────┬──────────────┤
  │   VF 1   │   VF 2   │    VF 3     │  → Direkt in VM gemappt
  └──────────┴──────────┴─────────────┘
      VM 1       VM 2       VM 3
```

### Voraussetzungen

- NIC mit SR-IOV-Unterstützung (im Datenblatt prüfen)
- BIOS/UEFI: **SR-IOV / VT-d** aktiviert
- Physischer Host mit IOMMU-Unterstützung
- Hyper-V-Host: Windows Server 2012 R2 oder neuer
- Gast-OS: Treiber für die spezifische NIC (VF-Treiber)
- vSwitch muss mit `-EnableIov $true` erstellt werden

### Konfiguration

```powershell
# SR-IOV-fähigen vSwitch erstellen
New-VMSwitch -Name "SR-IOV-Switch" `
             -NetAdapterName "Ethernet 1" `
             -EnableIov $true

# SR-IOV für einen VM-Adapter aktivieren
Set-VMNetworkAdapter -VMName "MeineVM" `
                     -Name "NIC-Prod" `
                     -IovWeight 100

# SR-IOV-Status prüfen
Get-VMNetworkAdapter -VMName "MeineVM" | Select-Object Name, IovWeight, IovUsage
```

### Wann SR-IOV sinnvoll ist

> [!TIP] Einsatzszenarien für SR-IOV
> - **Hochdurchsatz-Netzwerke** (10 GbE / 25 GbE+) mit vielen kleinen Paketen
> - **Latenz-kritische Workloads** (z. B. Trading-Systeme, Echtzeit-Datenbanken)
> - **Netzwerk-Funktions-Virtualisierung (NFV)** — virtuelle Firewalls, Router
> - Bei normalen File-Server- oder Web-Workloads bringt SR-IOV kaum Vorteile

> [!WARNING] SR-IOV-Einschränkungen
> - Live Migration ist mit aktiver SR-IOV-VF **eingeschränkt** (Fallback auf vSwitch bei Migration)
> - VLAN-Tagging und QoS-Limits pro Adapter funktionieren nicht mehr über den vSwitch
> - Snapshots können zu Problemen führen

---

## VLAN-Konfiguration

Hyper-V unterstützt IEEE 802.1Q VLAN-Tagging auf VM-Netzwerkadapterebene.

### Access Mode (eine VLAN-ID)

Der VM-Adapter gehört zu genau einer VLAN. Der vSwitch entfernt das Tag beim Eingang zur VM (VM sieht kein Tag).

```powershell
# VM-Adapter auf VLAN 10 setzen (Access Mode)
Set-VMNetworkAdapterVlan -VMName "MeineVM" `
                         -VMNetworkAdapterName "NIC-Prod" `
                         -Access `
                         -VlanId 10

# VLAN-Konfiguration eines Adapters anzeigen
Get-VMNetworkAdapterVlan -VMName "MeineVM"
```

### Trunk Mode (mehrere VLANs)

Der VM-Adapter erhält getaggte Frames — sinnvoll für virtuelle Router, Firewalls oder NAS-VMs, die selbst mehrere VLANs verwalten.

```powershell
# VM-Adapter im Trunk Mode mit nativer VLAN 1 und erlaubten VLANs 10, 20, 30
Set-VMNetworkAdapterVlan -VMName "RouterVM" `
                         -VMNetworkAdapterName "NIC-Trunk" `
                         -Trunk `
                         -NativeVlanId 1 `
                         -AllowedVlanIdList "10,20,30"
```

### VLAN für den Host-Management-Adapter

```powershell
# Management-vNIC des Hosts auf VLAN 99 setzen
Set-VMNetworkAdapterVlan -ManagementOS `
                         -VMNetworkAdapterName "vEthernet (External-Switch)" `
                         -Access `
                         -VlanId 99
```

> [!IMPORTANT] Physischer Switch muss mitspielen
> Der physische Switch-Port, an dem der Hyper-V-Host hängt, muss als **Trunk-Port** konfiguriert sein, der alle benötigten VLANs weiterleitet. Sonst sehen VMs nur ungetaggten Traffic.

---

## QoS / Bandwidth Management

Hyper-V ermöglicht pro VM-Netzwerkadapter die Begrenzung und Garantie von Bandbreite.

### Konzepte

| Parameter | Beschreibung |
|-----------|-------------|
| `MinimumBandwidthWeight` | Relative Gewichtung (1–100) — garantierter Anteil bei Last |
| `MinimumBandwidthAbsolute` | Absolutes Minimum in **Bit/s** |
| `MaximumBandwidth` | Absolutes Maximum in **Bit/s** — Hard Limit |

> [!NOTE] Absolute vs. relative Limits
> Absolute Limits (`...Absolute` / `MaximumBandwidth`) erfordern, dass der vSwitch mit `-MinimumBandwidthMode Absolute` oder `Weight` erstellt wurde. Beide Modi können nicht gemischt werden.

### Konfiguration

```powershell
# vSwitch mit Bandbreiten-Management erstellen (Weight-Modus)
New-VMSwitch -Name "QoS-Switch" `
             -NetAdapterName "Ethernet 1" `
             -MinimumBandwidthMode Weight

# Minimum-Gewichtung setzen (VM bekommt min. 20% der Bandbreite)
Set-VMNetworkAdapter -VMName "MeineVM" `
                     -Name "NIC-Prod" `
                     -MinimumBandwidthWeight 20

# Maximum-Bandbreite auf 1 Gbit/s begrenzen (in Bit/s)
Set-VMNetworkAdapter -VMName "MeineVM" `
                     -Name "NIC-Prod" `
                     -MaximumBandwidth 1000000000

# QoS-Einstellungen anzeigen
Get-VMNetworkAdapter -VMName "MeineVM" | `
    Select-Object Name, MinimumBandwidthWeight, MaximumBandwidth
```

---

## VMQ (Virtual Machine Queue)

**Virtual Machine Queue** ist eine Hardware-Offload-Funktion, bei der die physische NIC eingehende Pakete bereits auf Hardwareebene nach Ziel-VM sortiert und in separate Empfangswarteschlangen einsortiert.

### Vorteile und Einsatz

| Aspekt | Details |
|--------|---------|
| **Vorteil** | Entlastet CPU-Kerne — weniger Interrupts, geringere Latenz |
| **Voraussetzung** | NIC mit VMQ-Unterstützung; Hyper-V-Host mit Multi-Core-CPU |
| **Standard** | Ist auf modernen Systemen **standardmäßig aktiviert** |
| **Problem** | Auf wenig ausgelasteten Systemen / bei wenigen VMs kann VMQ zu höherer Latenz führen (unvollständig gefüllte Queues) |

### VMQ verwalten

```powershell
# VMQ-Status einer VM prüfen
Get-VMNetworkAdapter -VMName "MeineVM" | Select-Object Name, VmqUsage, VmqWeight

# VMQ für einen VM-Adapter deaktivieren (z. B. bei Latenzproblemen)
Set-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod" -VmqWeight 0

# VMQ für einen VM-Adapter aktivieren (Gewicht 1–100)
Set-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod" -VmqWeight 50

# VMQ-Fähigkeit der physischen NIC prüfen
Get-NetAdapterVmq -Name "Ethernet 1"

# VMQ auf physischer NIC deaktivieren (selten nötig)
Disable-NetAdapterVmq -Name "Ethernet 1"
```

> [!BUG] VMQ und 10-GbE-Netzwerke
> Bei einigen 10-GbE-Adaptern (Chelsio, Intel X550) kann VMQ in Kombination mit bestimmten Treibern zu Paketverlust führen. In solchen Fällen VMQ auf der physischen NIC deaktivieren und stattdessen auf RSS (Receive Side Scaling) setzen.

---

## NAT-Netzwerk

Für Testlabore oder isolierte VM-Netzwerke mit Internetzugang bietet Hyper-V zwei NAT-Ansätze.

### Variante 1: WinNAT (empfohlen)

WinNAT ist der native Windows-NAT-Stack — stabiler und konfigurierbarer als ICS.

```powershell
# 1. Internal Switch erstellen (falls nicht vorhanden)
New-VMSwitch -Name "NAT-Switch" -SwitchType Internal

# 2. IP-Adresse für den Host-Management-Adapter setzen (wird Gateway für VMs)
New-NetIPAddress -IPAddress 192.168.100.1 `
                 -PrefixLength 24 `
                 -InterfaceAlias "vEthernet (NAT-Switch)"

# 3. NAT-Objekt erstellen
New-NetNat -Name "HyperV-NAT" `
           -InternalIPInterfaceAddressPrefix "192.168.100.0/24"

# 4. VMs im NAT-Subnetz konfigurieren:
#    IP: 192.168.100.x  |  Gateway: 192.168.100.1  |  DNS: 8.8.8.8

# NAT-Status prüfen
Get-NetNat
Get-NetNatStaticMapping -NatName "HyperV-NAT"
```

### Port-Forwarding mit WinNAT

```powershell
# Eingehende Verbindung auf Port 8080 an VM 192.168.100.10:80 weiterleiten
Add-NetNatStaticMapping -NatName "HyperV-NAT" `
                        -Protocol TCP `
                        -ExternalIPAddress 0.0.0.0 `
                        -ExternalPort 8080 `
                        -InternalIPAddress 192.168.100.10 `
                        -InternalPort 80

# Mapping entfernen
Remove-NetNatStaticMapping -NatName "HyperV-NAT" -StaticMappingID 0
```

### Variante 2: ICS (Internet Connection Sharing)

Einfacher, aber weniger kontrollierbar — wird meist über die GUI aktiviert:

**GUI-Weg:** Netzwerkadapter (externe NIC) → Eigenschaften → Freigabe → *"Anderen Netzwerkbenutzern gestatten, sich über die Internetverbindung dieses Computers zu verbinden"* aktivieren → Ziel-Adapter: Internal Switch auswählen.

> [!TIP] WinNAT vs. ICS
> **WinNAT** ist für Hyper-V die bessere Wahl: vollständig per PowerShell verwaltbar, kein GUI-Eingriff nötig, unterstützt statisches Port-Forwarding und mehrere NAT-Instanzen gleichzeitig. ICS eignet sich nur für schnelle Einmallabore.

---

## PowerShell-Cheatsheet

### vSwitches

```powershell
# Alle Switches auflisten
Get-VMSwitch

# Switch entfernen
Remove-VMSwitch -Name "AlterSwitch" -Force

# Switch umbenennen
Rename-VMSwitch -Name "Alt" -NewName "Neu"

# Alle Eigenschaften eines Switches anzeigen
Get-VMSwitch -Name "External-Switch" | Format-List *
```

### VM-Netzwerkadapter

```powershell
# Adapter hinzufügen
Add-VMNetworkAdapter -VMName "MeineVM" -SwitchName "External-Switch" -Name "NIC-1"

# Adapter entfernen
Remove-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-1"

# Statische MAC-Adresse setzen
Set-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-1" `
                     -StaticMacAddress "00-15-5D-01-02-03"

# Alle Adapter einer VM anzeigen
Get-VMNetworkAdapter -VMName "MeineVM" | Format-Table Name, SwitchName, MacAddress, Status

# Adapter aller VMs im Überblick
Get-VM | Get-VMNetworkAdapter | Select-Object VMName, Name, SwitchName, IPAddresses
```

### VLAN

```powershell
# VLAN-Konfiguration aller VMs anzeigen
Get-VM | Get-VMNetworkAdapter | Get-VMNetworkAdapterVlan

# VLAN-Konfiguration zurücksetzen (kein VLAN)
Set-VMNetworkAdapterVlan -VMName "MeineVM" -VMNetworkAdapterName "NIC-1" -Untagged
```

### Bandbreite & QoS

```powershell
# Bandbreiten-Limits aller VMs anzeigen
Get-VM | Get-VMNetworkAdapter | `
    Select-Object VMName, Name, MinimumBandwidthWeight, MaximumBandwidth

# Maximum-Limit entfernen (0 = unbegrenzt)
Set-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-1" -MaximumBandwidth 0
```

### Diagnose & Monitoring

```powershell
# Netzwerkstatistiken einer VM
Get-VMNetworkAdapterStatistics -VMName "MeineVM"

# IP-Adressen aller VMs abfragen (benötigt Integration Services)
Get-VM | Select-Object Name, @{N="IPs"; E={
    (Get-VMNetworkAdapter -VMName $_.Name).IPAddresses -join ", " }}

# Physische NIC-Informationen
Get-NetAdapter | Select-Object Name, InterfaceDescription, LinkSpeed, Status

# VMQ-Status aller physischen NICs
Get-NetAdapterVmq | Select-Object Name, Enabled, NumberOfReceiveQueues
```

### NAT

```powershell
# Alle NAT-Instanzen anzeigen
Get-NetNat

# NAT entfernen
Remove-NetNat -Name "HyperV-NAT" -Confirm:$false

# Aktive NAT-Verbindungen anzeigen
Get-NetNatSession
```

---

## Verwandte Notizen

- [[Was ist Hyper-V]]
- [[Installation Hyper-V (Windows Server 2022)]]
- [[Hyper-V VM Verwaltung]]
- [[Hyper-V Best Practices]]
- [[Hyper-V Troubleshooting]]