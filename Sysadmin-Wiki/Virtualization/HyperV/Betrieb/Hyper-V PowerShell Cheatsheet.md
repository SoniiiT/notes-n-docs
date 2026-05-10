---
tags:
  - hyper-v
  - powershell
  - cheatsheet
  - referenz
  - cli
aliases:
  - Hyper-V PowerShell
  - Hyper-V PS Cheatsheet
---

# Hyper-V PowerShell Cheatsheet

> [!ABSTRACT] TL;DR
> Vollständige PowerShell-Schnellreferenz für Hyper-V — Host-Verwaltung, VM-Lifecycle, Hardware-Konfiguration, Checkpoints, Netzwerk, Live Migration, Replica und Reporting-Einzeiler. Modul `Hyper-V` muss auf dem Host oder Remote-System installiert sein.

---

## Modul & Remote-Verbindung

```powershell
# Hyper-V-Modul laden (auf Host oder Windows-Admin-PC)
Import-Module Hyper-V

# Alle verfügbaren Hyper-V-Cmdlets auflisten
Get-Command -Module Hyper-V

# Remote-Host verbinden (Credential-Prompt)
$s = New-PSSession -ComputerName hv01.example.local -Credential (Get-Credential)
Enter-PSSession $s

# Alle nachfolgenden Befehle funktionieren auch remote via -ComputerName
Get-VM -ComputerName hv01.example.local
```

> [!TIP] RSAT installieren
> Auf einem Windows-Admin-PC die Hyper-V-PowerShell-Tools via RSAT installieren:
> `Add-WindowsFeature RSAT-Hyper-V-Tools` (Server) oder unter *Optionale Features → RSAT* (Client).

---

## Host-Befehle

### Host-Informationen & Status

```powershell
# Hyper-V-Hostkonfiguration anzeigen
Get-VMHost

# Einzelne Eigenschaften gezielt abfragen
Get-VMHost | Select-Object `
    ComputerName,
    LogicalProcessorCount,
    MemoryCapacity,
    VirtualMachinePath,
    VirtualHardDiskPath,
    EnableEnhancedSessionMode

# Hyper-V-Dienst-Status prüfen
Get-Service -Name vmms | Select-Object Name, Status, StartType

# Hyper-V-Dienst starten / stoppen / neu starten
Start-Service   -Name vmms
Stop-Service    -Name vmms
Restart-Service -Name vmms
```

### Standardpfade setzen

```powershell
# Standard-Ablageort für VM-Konfigurationen setzen
Set-VMHost -VirtualMachinePath  "D:\Hyper-V\VMs"

# Standard-Ablageort für VHDX-Dateien setzen
Set-VMHost -VirtualHardDiskPath "E:\Hyper-V\VHDs"

# Enhanced Session Mode aktivieren (RDP-ähnlicher Vollzugriff per VM Connect)
Set-VMHost -EnableEnhancedSessionMode $true
```

### NUMA & Ressourcen-Übersicht

```powershell
# NUMA-Knoten des Hosts anzeigen
Get-VMHostNumaNode

# Host-Ressourcenauslastung (CPU, RAM)
Get-VMHostNumaNodeStatus

# Hyper-V-Event-Log-Einträge (letzte 50 Einträge)
Get-WinEvent -ProviderName 'Microsoft-Windows-Hyper-V-VMMS' `
    -MaxEvents 50 | Select-Object TimeCreated, Id, LevelDisplayName, Message
```

---

## VM-Verwaltung

### VMs auflisten & Status

```powershell
# Alle VMs auf dem lokalen Host
Get-VM

# VMs mit Name, State, CPU-Nutzung, RAM und Laufzeit
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime

# Nur laufende VMs
Get-VM | Where-Object { $_.State -eq 'Running' }

# VM nach Name suchen (Wildcards möglich)
Get-VM -Name "Win2022-*"
```

### Power-Management

```powershell
Start-VM    -Name "MeineVM"           # Starten
Stop-VM     -Name "MeineVM"           # Graceful Shutdown (benötigt Integration Services)
Stop-VM     -Name "MeineVM" -Force    # Hart ausschalten (entspricht Strom trennen)
Restart-VM  -Name "MeineVM"           # Neustart (Graceful)
Restart-VM  -Name "MeineVM" -Force    # Neustart erzwingen
Suspend-VM  -Name "MeineVM"           # Anhalten (Pause)
Resume-VM   -Name "MeineVM"           # Fortsetzen nach Suspend
Save-VM     -Name "MeineVM"           # Saved State (Hibernation)
```

> [!WARNING] `Stop-VM -Force`
> Entspricht dem Herausziehen des Stromkabels — Datenverlust und Filesystem-Korruption im Gast-OS sind möglich. Nur einsetzen wenn Graceful Shutdown nicht reagiert.

### VM erstellen

```powershell
# Neue Generation-2-VM erstellen (ohne Disk)
New-VM `
    -Name               "SRV-TEST-01" `
    -Generation         2 `
    -MemoryStartupBytes 4GB `
    -SwitchName         "External Switch" `
    -Path               "D:\Hyper-V\VMs"

# Neue VM mit bestehender VHDX
New-VM `
    -Name               "SRV-TEST-02" `
    -Generation         2 `
    -MemoryStartupBytes 4GB `
    -VHDPath            "E:\Hyper-V\VHDs\SRV-TEST-02.vhdx" `
    -SwitchName         "External Switch"

# Neue VHDX erstellen und an VM anhängen
New-VHD -Path "E:\Hyper-V\VHDs\SRV-TEST-01-OS.vhdx" -SizeBytes 80GB -Dynamic
Add-VMHardDiskDrive -VMName "SRV-TEST-01" `
    -Path "E:\Hyper-V\VHDs\SRV-TEST-01-OS.vhdx"
```

### VM entfernen

```powershell
# VM entfernen — Konfiguration und Disk bleiben auf dem Volume!
Remove-VM -Name "MeineVM" -Force

# VM inklusive aller VHDXs vollständig löschen
$vm    = Get-VM -Name "MeineVM"
$disks = $vm | Get-VMHardDiskDrive
Remove-VM -Name "MeineVM" -Force
$disks | ForEach-Object { Remove-Item -Path $_.Path -Force }
```

---

## Hardware-Konfiguration

### CPU

```powershell
# CPU-Konfiguration anzeigen
Get-VMProcessor -VMName "MeineVM"

# vCPU-Anzahl setzen (VM muss ausgeschaltet sein)
Set-VMProcessor -VMName "MeineVM" -Count 4

# NUMA-Spanning deaktivieren (Performance-Optimierung für große VMs)
Set-VMProcessor -VMName "MeineVM" `
    -MaximumCountPerNumaNode   4 `
    -MaximumCountPerNumaSocket 4

# CPU-Kompatibilitätsmodus für Live Migration aktivieren
Set-VMProcessor -VMName "MeineVM" -CompatibilityForMigrationEnabled $true
```

### RAM

```powershell
# RAM-Konfiguration anzeigen
Get-VMMemory -VMName "MeineVM"

# Festen RAM setzen (Dynamic Memory deaktiviert)
Set-VMMemory -VMName "MeineVM" -DynamicMemoryEnabled $false -StartupBytes 8GB

# Dynamic Memory konfigurieren
Set-VMMemory -VMName "MeineVM" `
    -DynamicMemoryEnabled $true `
    -MinimumBytes          2GB  `
    -StartupBytes          4GB  `
    -MaximumBytes         16GB  `
    -Buffer                20     # Puffer in Prozent
```

> [!TIP] Dynamic Memory mit Bedacht einsetzen
> Dynamic Memory spart RAM bei geringer Last, kann aber bei plötzlichem Mehrbedarf zu Ballooning-Effekten führen. Für Datenbanken und zeitkritische Dienste festen RAM bevorzugen.

### Netzwerk

```powershell
# Netzwerkadapter anzeigen
Get-VMNetworkAdapter -VMName "MeineVM"

# Netzwerkadapter hinzufügen
Add-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod" -SwitchName "External Switch"

# Adapter mit Switch verbinden
Connect-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod" -SwitchName "External Switch"

# Adapter trennen (ohne zu entfernen)
Disconnect-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod"

# Adapter entfernen
Remove-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod"

# VLAN für einen Adapter setzen (Access Port)
Set-VMNetworkAdapterVlan -VMName "MeineVM" -VMNetworkAdapterName "NIC-Prod" `
    -Access -VlanId 10

# Statische MAC-Adresse setzen
Set-VMNetworkAdapter -VMName "MeineVM" -Name "NIC-Prod" `
    -StaticMacAddress "00-15-5D-AA-BB-CC"
```

### Festplatten & VHDXs

```powershell
# Alle Disks einer VM anzeigen
Get-VMHardDiskDrive -VMName "MeineVM"

# Neue VHDX erstellen — Dynamic (dünn, platzsparend)
New-VHD -Path "E:\Hyper-V\VHDs\Daten.vhdx" -SizeBytes 200GB -Dynamic

# Neue VHDX erstellen — Fixed (volle Kapazität reserviert, schneller)
New-VHD -Path "E:\Hyper-V\VHDs\Daten-Fixed.vhdx" -SizeBytes 200GB -Fixed

# VHDX an VM anhängen
Add-VMHardDiskDrive -VMName "MeineVM" -Path "E:\Hyper-V\VHDs\Daten.vhdx"

# VHDX-Größe ändern (VM muss ausgeschaltet sein — nur Vergrößerung möglich)
Resize-VHD -Path "E:\Hyper-V\VHDs\Daten.vhdx" -SizeBytes 500GB

# VHDX-Metadaten anzeigen (Typ, Größe, belegter Platz)
Get-VHD -Path "E:\Hyper-V\VHDs\Daten.vhdx"

# VHDX komprimieren — ungenutzten Platz freigeben (VM muss ausgeschaltet sein)
Optimize-VHD -Path "E:\Hyper-V\VHDs\Daten.vhdx" -Mode Full
```

---

## Checkpoints (Snapshots)

```powershell
# Checkpoint erstellen
Checkpoint-VM -VMName "MeineVM" -SnapshotName "Vor-Patch-2025-01"

# Alle Checkpoints einer VM anzeigen
Get-VMCheckpoint -VMName "MeineVM"

# Checkpoint wiederherstellen (VM wird gestoppt!)
Restore-VMCheckpoint -VMName "MeineVM" -Name "Vor-Patch-2025-01" -Confirm:$false

# Via Pipeline wiederherstellen
Get-VMCheckpoint -VMName "MeineVM" -Name "Vor-Patch-2025-01" |
    Restore-VMCheckpoint -Confirm:$false

# Einzelnen Checkpoint löschen
Remove-VMCheckpoint -VMName "MeineVM" -Name "Vor-Patch-2025-01"

# Alle Checkpoints einer VM löschen
Get-VMCheckpoint -VMName "MeineVM" | Remove-VMCheckpoint -Confirm:$false

# Checkpoint-Typ umstellen (Production = VSS-basiert, kein RAM-Abbild)
Set-VM -VMName "MeineVM" -CheckpointType Production
```

> [!WARNING] Checkpoint-Typen
> **Standard Checkpoints** sichern RAM + Disk (schnell, aber nicht applikationskonsistent). **Production Checkpoints** nutzen VSS im Gast-OS — konsistent für Datenbanken und Active Directory, kein RAM-Abbild.

---

## Netzwerk (Switches)

```powershell
# Alle virtuellen Switches anzeigen
Get-VMSwitch

# Externen Switch erstellen (mit physischer NIC, Host hat Zugriff)
New-VMSwitch -Name "External Switch" `
    -NetAdapterName "Ethernet 1" -AllowManagementOS $true

# Internen Switch erstellen (Host + VMs kommunizieren, kein physischer Uplink)
New-VMSwitch -Name "Internal Switch" -SwitchType Internal

# Privaten Switch erstellen (nur VMs untereinander, Host isoliert)
New-VMSwitch -Name "Private Switch" -SwitchType Private

# Switch entfernen
Remove-VMSwitch -Name "Private Switch" -Force

# VLAN-Konfiguration auf Adapter-Ebene anzeigen
Get-VMNetworkAdapterVlan -VMName "MeineVM"

# Trunk-Port konfigurieren (mehrere VLANs — für Router-VMs, pfSense, OPNsense)
Set-VMNetworkAdapterVlan -VMName "RouterVM" -Trunk `
    -AllowedVlanIdList "10,20,30,100" `
    -NativeVlanId 1
```

---

## Live Migration

```powershell
# Live Migration einer laufenden VM auf anderen Host
Move-VM -Name "MeineVM" -DestinationHost "hv02.example.local"

# Live Migration mit Storage (VM + alle Disks zusammen migrieren)
Move-VM -Name "MeineVM" `
    -DestinationHost        "hv02.example.local" `
    -DestinationStoragePath "D:\Hyper-V\VMs"

# Nur Storage migrieren (VM läuft weiter auf demselben Host)
Move-VMStorage -VMName "MeineVM" -DestinationStoragePath "F:\Hyper-V\VMs"

# Einzelne VHDX auf anderen Pfad migrieren
Move-VMStorage -VMName "MeineVM" -VHDs @(
    @{
        SourceFilePath      = "E:\Hyper-V\VHDs\MeineVM.vhdx"
        DestinationFilePath = "F:\Hyper-V\VHDs\MeineVM.vhdx"
    }
)

# Live Migration auf Host-Ebene konfigurieren
Set-VMHost -VirtualMachineMigrationEnabled   $true
Set-VMHost -MaximumVirtualMachineMigrations  4   # Parallele VM-Migrationen
Set-VMHost -MaximumStorageMigrations         4   # Parallele Storage-Migrationen
```

> [!IMPORTANT] Voraussetzungen Live Migration
> Beide Hosts müssen Domänenmitglieder sein — Kerberos-Auth ist Pflicht. Bei unterschiedlichen CPU-Generationen `CompatibilityForMigrationEnabled $true` setzen. Live Migration-Netzwerk auf dediziertem VLAN mit Jumbo Frames (MTU 9000) betreiben.

---

## Hyper-V Replica

```powershell
# Replikation für eine VM aktivieren
Enable-VMReplication `
    -VMName                  "MeineVM" `
    -ReplicaServerName       "hv-replica.example.local" `
    -ReplicaServerPort       80 `
    -AuthenticationType      "Kerberos" `
    -ReplicationFrequencySec 300       # 5-Minuten-Intervall (300 / 900)

# Initiale Replikation starten
Start-VMInitialReplication -VMName "MeineVM"

# Replikationsstatus einer VM anzeigen
Get-VMReplication -VMName "MeineVM"

# Replikationsstatus aller VMs (Übersicht)
Get-VMReplication |
    Select-Object VMName, State, Health, LastReplicationTime, FrequencySec |
    Format-Table -AutoSize

# Replikation pausieren / fortsetzen
Suspend-VMReplication -VMName "MeineVM"
Resume-VMReplication  -VMName "MeineVM"

# Test-Failover durchführen (Replikat-VM starten — Original läuft weiter)
Start-VMFailover -VMName "MeineVM" `
    -ComputerName "hv-replica.example.local" -AsTest

# Test-Failover beenden und Replikat-VM entfernen
Stop-VMFailover -VMName "MeineVM" -ComputerName "hv-replica.example.local"

# Geplanten Failover (Primary → Replica, geplante Wartung)
Start-VMFailover -VMName "MeineVM" -Confirm:$false

# Replikation deaktivieren
Disable-VMReplication -VMName "MeineVM"
```

---

## Reporting & Diagnose-Einzeiler

```powershell
# Alle VMs mit CPU-Auslastung, RAM-Zuweisung und Laufzeit
Get-VM | Select-Object `
    Name, State, CPUUsage,
    @{N="RAM zugewiesen (GB)"; E={[math]::Round($_.MemoryAssigned / 1GB, 1)}},
    @{N="RAM Bedarf (GB)";     E={[math]::Round($_.MemoryDemand   / 1GB, 1)}},
    Uptime | Format-Table -AutoSize

# Alle Checkpoints des Hosts mit Alter (älteste zuerst)
Get-VM | Get-VMCheckpoint | Select-Object `
    VMName, Name, CreationTime,
    @{N="Alter (Tage)"; E={(New-TimeSpan -Start $_.CreationTime).Days}} |
    Sort-Object "Alter (Tage)" -Descending | Format-Table -AutoSize

# Checkpoints älter als 7 Tage — zur sofortigen Bereinigung
Get-VM | Get-VMCheckpoint |
    Where-Object { $_.CreationTime -lt (Get-Date).AddDays(-7) } |
    Select-Object VMName, Name, CreationTime | Format-Table -AutoSize

# VMs mit problematischen Integration Services
Get-VM | Where-Object { $_.State -eq 'Running' } | ForEach-Object {
    $is = Get-VMIntegrationService -VMName $_.Name
    [PSCustomObject]@{
        VM       = $_.Name
        Probleme = ($is | Where-Object {
            -not $_.Enabled -or $_.PrimaryStatusDescription -ne 'OK'
        } | Measure-Object).Count
    }
} | Where-Object { $_.Probleme -gt 0 } | Format-Table -AutoSize

# VHDX-Bestandsaufnahme: alle Disks mit Typ, Größe und belegtem Platz
Get-VM | Get-VMHardDiskDrive | ForEach-Object {
    $vhd = Get-VHD -Path $_.Path -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        VM            = $_.VMName
        Pfad          = $_.Path
        Typ           = $vhd.VhdType
        "Größe (GB)"  = [math]::Round($vhd.Size     / 1GB, 1)
        "Belegt (GB)" = [math]::Round($vhd.FileSize / 1GB, 1)
    }
} | Format-Table -AutoSize

# Resource Metering aktivieren und VM-Verbrauch abfragen
Enable-VMResourceMetering -VMName "MeineVM"
Measure-VM -Name "MeineVM" |
    Select-Object VMName, AverageProcessorUsage, AverageMemoryUsage,
                  TotalDisk, NetworkMeteredTrafficReport
```

---

## Verwandte Notizen

- [[Was ist Hyper-V]]
- [[Hyper-V VM Verwaltung]]
- [[Hyper-V Best Practices]]
- [[Hyper-V Troubleshooting]]
- [[Installation Hyper-V (Windows Server 2022)]]