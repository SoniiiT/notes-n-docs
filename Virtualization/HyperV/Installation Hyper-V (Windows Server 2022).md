---
tags: [virtualization, hyper-v, installation, windows-server]
---

# Installation Hyper-V – Windows Server 2022

## Voraussetzungen

### Hardware-Anforderungen

| Komponente | Minimum | Empfohlen |
| ---------- | ------- | --------- |
| CPU        | 64-bit mit Intel VT-x / AMD-V | Intel VT-x + VT-d / AMD-Vi |
| RAM        | 4 GB | 32 GB+ |
| Speicher   | 32 GB (OS) | 500 GB+ (für VMs) |
| Netzwerk   | 1x NIC | 2x NIC (Management + VM-Traffic) |

### BIOS/UEFI Einstellungen

Vor der Installation sicherstellen, dass folgende Optionen im BIOS/UEFI aktiviert sind:

- **Intel:** `Intel Virtualization Technology (VT-x)` und `VT-d`
- **AMD:** `AMD-V (SVM Mode)` und `AMD-Vi (IOMMU)`
- **DEP/NX:** `Data Execution Prevention` aktiviert

> [!NOTE]
> Die Einstellungen sind je nach Hersteller unter unterschiedlichen Bezeichnungen zu finden (z.B. "Virtualization", "Virtualization Technology", "SVM Mode").

---

## Installation der Hyper-V Rolle

### Option A – Server Manager (GUI)

1. **Server Manager** öffnen → `Manage` → `Add Roles and Features`
2. Installation type: `Role-based or feature-based installation`
3. Server auswählen → `Next`
4. Unter **Server Roles** → `Hyper-V` anklicken → `Add Features` bestätigen
5. Bei **Virtual Switches**: Netzwerkadapter für den externen Switch wählen (kann auch später gemacht werden)
6. Bei **Virtual Machine Migration**: nach Bedarf konfigurieren
7. **Default Stores**: Pfade für VMs und VHDs festlegen
8. `Install` klicken → Server neu starten

### Option B – PowerShell

```powershell
# Hyper-V Rolle installieren
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart

# Prüfen ob Installation erfolgreich war (nach Neustart)
Get-WindowsFeature -Name Hyper-V
```

> [!TIP]
> `-Restart` startet den Server automatisch neu. Weglassen wenn der Neustart manuell durchgeführt werden soll.

---

## Virtuellen Switch einrichten

Hyper-V kennt drei Switch-Typen:

| Typ | Beschreibung | Verwendung |
| --- | ------------ | ---------- |
| **External** | Verbindet VMs mit dem physischen Netzwerk | Produktions-VMs, Internet-Zugang |
| **Internal** | Kommunikation zwischen VMs + Host | Test-Umgebungen mit Host-Zugang |
| **Private** | Nur Kommunikation zwischen VMs | Isolierte Lab-Umgebungen |

### External Switch erstellen

#### Via Hyper-V Manager (GUI)

1. **Hyper-V Manager** öffnen → rechts `Virtual Switch Manager`
2. `New virtual network switch` → `External` → `Create Virtual Switch`
3. Name vergeben (z.B. `vSwitch-External`)
4. Physischen Netzwerkadapter auswählen
5. `Allow management operating system to share this network adapter` aktivieren (empfohlen)
6. `Apply` → `OK`

#### Via PowerShell

```powershell
# Verfügbare Netzwerkadapter anzeigen
Get-NetAdapter

# External Switch erstellen (NIC-Name anpassen)
New-VMSwitch -Name "vSwitch-External" -NetAdapterName "Ethernet" -AllowManagementOS $true

# Internal Switch erstellen
New-VMSwitch -Name "vSwitch-Internal" -SwitchType Internal

# Private Switch erstellen
New-VMSwitch -Name "vSwitch-Private" -SwitchType Private

# Alle Switches anzeigen
Get-VMSwitch
```

---

## Standardpfade konfigurieren

```powershell
# Standard-Speicherpfade setzen
Set-VMHost -VirtualHardDiskPath "D:\Hyper-V\VHDs"
Set-VMHost -VirtualMachinePath "D:\Hyper-V\VMs"

# Aktuelle Einstellungen prüfen
Get-VMHost | Select-Object VirtualHardDiskPath, VirtualMachinePath
```

---

## PowerShell Referenz

### Host & Dienste

```powershell
# Hyper-V Host Informationen
Get-VMHost

# Hyper-V Dienst Status prüfen
Get-Service -Name vmms

# Hyper-V Dienst neustarten
Restart-Service -Name vmms
```

### VMs verwalten

```powershell
# Alle VMs anzeigen
Get-VM

# VM starten / stoppen / neustarten
Start-VM -Name "VM-Name"
Stop-VM -Name "VM-Name"
Restart-VM -Name "VM-Name"

# VM Status prüfen
Get-VM -Name "VM-Name" | Select-Object Name, State, CPUUsage, MemoryAssigned

# Neue VM erstellen
New-VM -Name "VM-Name" `
       -MemoryStartupBytes 4GB `
       -Generation 2 `
       -SwitchName "vSwitch-External" `
       -NewVHDPath "D:\Hyper-V\VHDs\VM-Name.vhdx" `
       -NewVHDSizeBytes 60GB

# VM löschen (inkl. VHD)
Remove-VM -Name "VM-Name" -Force
```

### Snapshots / Checkpoints

```powershell
# Checkpoint erstellen
Checkpoint-VM -Name "VM-Name" -SnapshotName "Vor-Update"

# Checkpoints anzeigen
Get-VMCheckpoint -VMName "VM-Name"

# Checkpoint wiederherstellen
Restore-VMCheckpoint -Name "Vor-Update" -VMName "VM-Name"

# Checkpoint löschen
Remove-VMCheckpoint -VMName "VM-Name" -Name "Vor-Update"
```

### Netzwerk

```powershell
# Alle virtuellen Switches anzeigen
Get-VMSwitch

# Netzwerkadapter einer VM anzeigen
Get-VMNetworkAdapter -VMName "VM-Name"

# Netzwerkadapter mit Switch verbinden
Connect-VMNetworkAdapter -VMName "VM-Name" -SwitchName "vSwitch-External"
```

### Storage

```powershell
# VHDs einer VM anzeigen
Get-VMHardDiskDrive -VMName "VM-Name"

# Neue VHD erstellen und anhängen
New-VHD -Path "D:\Hyper-V\VHDs\VM-Name-Data.vhdx" -SizeBytes 100GB -Dynamic
Add-VMHardDiskDrive -VMName "VM-Name" -Path "D:\Hyper-V\VHDs\VM-Name-Data.vhdx"

# VHD-Größe anzeigen
Get-VHD -Path "D:\Hyper-V\VHDs\VM-Name.vhdx"
```

---

## Nützliche Links

- [Hyper-V on Windows Server 2022 – Microsoft Docs](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/hyper-v-on-windows-server)
- [Hyper-V PowerShell Module Reference](https://learn.microsoft.com/en-us/powershell/module/hyper-v/)

---

## Verwandte Notizen

- [[Hyper-V Failover Cluster]] — Hochverfügbarkeit mit Hyper-V Failover Clustering
