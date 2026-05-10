---
tags: [virtualization, esxi, vsphere, vmware, powercli, automation, powershell, scripting]
aliases: ["PowerCLI", "VMware PowerCLI", "vSphere Automation", "VMware PowerShell"]
---

# PowerCLI

> [!ABSTRACT]
> VMware PowerCLI ist das offizielle PowerShell-Modul für die Automatisierung von VMware-Infrastruktur. Es bietet Cmdlets für ESXi, vCenter, vSAN, NSX, Horizon und weitere VMware-Produkte und läuft auf Windows, Linux und macOS.

---

## Was ist PowerCLI?

PowerCLI ist ein PowerShell-basiertes Automatisierungswerkzeug von VMware (seit 2024 Broadcom). Es wurde ursprünglich 2006 als VI Toolkit für ESX 3.x eingeführt und heißt seit Version 5.0 (2011) „PowerCLI".

**Kernmerkmale:**
- Über 800 Cmdlets für vSphere, vSAN, NSX, Cloud Director, Horizon, vROps
- Läuft auf Windows PowerShell 5.1 und PowerShell 7.x (Cross-Platform)
- Verbindung zu mehreren vCenter-Servern gleichzeitig möglich
- Vollständige Skript-Automatisierung für Reporting, Bulk-Operationen, Lifecycle-Management

| Version | PowerShell-Anforderung | Besonderheit |
|---|---|---|
| PowerCLI 12.x | PS 5.1 oder PS 7.x | Letzte Version mit kostenlosem Download von my.vmware.com |
| PowerCLI 13.x | PS 5.1 oder PS 7.x | Broadcom-Ära, via PowerShell Gallery |
| PowerCLI 14.x+ | PS 7.x empfohlen | Neueste Cmdlets |

---

## Installation

### Aus der PowerShell Gallery (empfohlen)

```powershell
# Installation (aktuellste Version)
Install-Module -Name VMware.PowerCLI -Scope CurrentUser

# Oder für alle Benutzer (als Administrator)
Install-Module -Name VMware.PowerCLI -Scope AllUsers

# Bestimmte Version installieren
Install-Module -Name VMware.PowerCLI -RequiredVersion 13.3.0 -Scope CurrentUser

# Bereits installierte Version prüfen
Get-Module -Name VMware.PowerCLI -ListAvailable

# Update
Update-Module -Name VMware.PowerCLI
```

### Zertifikat- und CEIP-Warnungen konfigurieren

```powershell
# SSL-Zertifikat-Warnung unterdrücken (nicht empfohlen für Produktion)
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false

# CEIP (Customer Experience Improvement Program) deaktivieren
Set-PowerCLIConfiguration -ParticipateInCEIP $false -Confirm:$false

# Konfiguration anzeigen
Get-PowerCLIConfiguration
```

---

## Verbindungen

```powershell
# Mit vCenter verbinden
Connect-VIServer -Server vcenter.domain.local -User administrator@vsphere.local -Password 'P@ssw0rd'

# Mit mehreren vCentern gleichzeitig
Connect-VIServer -Server vcenter01.domain.local, vcenter02.domain.local -User administrator@vsphere.local -Password 'P@ssw0rd'

# Direkt mit ESXi-Host verbinden
Connect-VIServer -Server esxi01.domain.local -User root -Password 'P@ssw0rd'

# Verbindung anzeigen
$global:DefaultVIServers

# Verbindung trennen
Disconnect-VIServer -Server * -Confirm:$false
```

---

## VM-Verwaltung

### Basis-Operationen

```powershell
# Alle VMs auflisten
Get-VM

# VMs filtern
Get-VM -Name "Web*"
Get-VM | Where-Object { $_.PowerState -eq "PoweredOn" }
Get-VM | Where-Object { $_.MemoryGB -gt 16 }

# VM starten / stoppen / neu starten
Start-VM -VM "WebServer01"
Stop-VM -VM "WebServer01" -Confirm:$false
Restart-VM -VM "WebServer01" -Confirm:$false

# Graceful Shutdown (über VMware Tools)
Stop-VMGuest -VM "WebServer01" -Confirm:$false
Restart-VMGuest -VM "WebServer01" -Confirm:$false
```

### VM erstellen und konfigurieren

```powershell
# VM erstellen
New-VM -Name "TestVM" `
       -ResourcePool (Get-ResourcePool "Production") `
       -Datastore (Get-Datastore "vSAN-Datastore") `
       -DiskGB 100 `
       -MemoryGB 8 `
       -NumCpu 4 `
       -GuestId "windows2022srv_64Guest" `
       -NetworkName "VM-Network"

# VM-Ressourcen ändern
Set-VM -VM "TestVM" -MemoryGB 16 -NumCpu 8 -Confirm:$false

# VM löschen (inkl. Dateien)
Remove-VM -VM "TestVM" -DeletePermanently -Confirm:$false

# VM klonen
New-VM -VM "TemplateVM" -Name "Klon01" -Datastore "vSAN-Datastore" -VMHost "esxi01.domain.local"
```

---

## Snapshots

```powershell
# Snapshot erstellen
New-Snapshot -VM "WebServer01" -Name "Vor-Update-$(Get-Date -Format 'yyyy-MM-dd')" -Memory $false -Quiesce $false

# Snapshots anzeigen
Get-Snapshot -VM "WebServer01"

# Snapshot wiederherstellen (auf letzten Snapshot)
Set-VM -VM "WebServer01" -Snapshot (Get-Snapshot -VM "WebServer01" | Sort-Object Created | Select-Object -Last 1) -Confirm:$false

# Einzelnen Snapshot löschen
Remove-Snapshot -Snapshot (Get-Snapshot -VM "WebServer01" -Name "Vor-Update*") -Confirm:$false

# Alle Snapshots einer VM löschen
Get-VM "WebServer01" | Get-Snapshot | Remove-Snapshot -Confirm:$false

# Snapshots konsolidieren
(Get-VM "WebServer01").ExtensionData.ConsolidateVMDisks()
```

> [!WARNING]
> Alte Snapshots verlangsamen VM-Performance und verbrauchen Speicher. Regelmäßig prüfen und löschen.

---

## Host-Verwaltung

```powershell
# Alle ESXi-Hosts auflisten
Get-VMHost

# Host-Details
Get-VMHost "esxi01.domain.local" | Select-Object Name, Version, ConnectionState, PowerState

# Wartungsmodus aktivieren/deaktivieren
Set-VMHost -VMHost "esxi01.domain.local" -State Maintenance
Set-VMHost -VMHost "esxi01.domain.local" -State Connected

# Host-Dienste verwalten
Get-VMHostService -VMHost "esxi01.domain.local"
Start-VMHostService -VMHostService (Get-VMHostService -VMHost "esxi01.domain.local" | Where-Object { $_.Key -eq "ssh" })
Stop-VMHostService -VMHostService (Get-VMHostService -VMHost "esxi01.domain.local" | Where-Object { $_.Key -eq "ssh" })

# Host neu starten
Restart-VMHost -VMHost "esxi01.domain.local" -Confirm:$false
```

---

## Netzwerk

```powershell
# vSwitches anzeigen
Get-VirtualSwitch -VMHost "esxi01.domain.local"

# Neuen vSwitch erstellen
New-VirtualSwitch -VMHost "esxi01.domain.local" -Name "vSwitch1" -NumPorts 120

# Portgroup erstellen
New-VirtualPortGroup -VirtualSwitch (Get-VirtualSwitch -Name "vSwitch1") -Name "VLAN100" -VLanId 100

# Netzwerkadapter anzeigen
Get-VMHostNetworkAdapter -VMHost "esxi01.domain.local"

# VMkernel-Adapter konfigurieren
New-VMHostNetworkAdapter -VMHost "esxi01.domain.local" `
                          -PortGroup "Management Network" `
                          -IP 192.168.1.100 `
                          -SubnetMask 255.255.255.0 `
                          -VirtualSwitch (Get-VirtualSwitch -Name "vSwitch0")
```

---

## Storage

```powershell
# Datastores anzeigen
Get-Datastore

# Datastore-Details
Get-Datastore | Select-Object Name, CapacityGB, FreeSpaceGB, @{N="Used%";E={[math]::Round((1-($_.FreeSpaceGB/$_.CapacityGB))*100,1)}}

# NFS-Datastore hinzufügen
New-Datastore -VMHost "esxi01.domain.local" -Name "NAS-Share" -Nfs -NfsHost 192.168.1.50 -Path "/export/vmdata"

# Festplatten einer VM anzeigen
Get-HardDisk -VM "WebServer01"

# Festplatte erweitern
Get-HardDisk -VM "WebServer01" | Where-Object { $_.Name -eq "Hard disk 1" } | Set-HardDisk -CapacityGB 200 -Confirm:$false
```

---

## Praktische Reporting-Scripts

### VM-Inventar exportieren

```powershell
# Alle VMs mit Details in CSV exportieren
Get-VM | Select-Object Name, PowerState, NumCpu, MemoryGB,
    @{N="Datastore";E={(Get-Datastore -VM $_).Name -join ", "}},
    @{N="Host";E={$_.VMHost.Name}},
    @{N="IP";E={@($_.Guest.IPAddress)[0]}} |
Export-Csv -Path "C:\Reports\VM-Inventar.csv" -NoTypeInformation -Encoding UTF8
```

### Alte Snapshots finden

```powershell
# Snapshots älter als 7 Tage
$cutoff = (Get-Date).AddDays(-7)
Get-VM | Get-Snapshot | Where-Object { $_.Created -lt $cutoff } |
    Select-Object VM, Name, Created, SizeGB |
    Sort-Object Created |
    Format-Table -AutoSize
```

### Datastore-Auslastung

```powershell
# Datastores mit mehr als 80% Auslastung
Get-Datastore | Where-Object { ($_.FreeSpaceGB / $_.CapacityGB) -lt 0.2 } |
    Select-Object Name, CapacityGB, FreeSpaceGB,
    @{N="Auslastung%";E={[math]::Round((1-($_.FreeSpaceGB/$_.CapacityGB))*100,1)}} |
    Sort-Object "Auslastung%" -Descending |
    Format-Table -AutoSize
```

### Alle VMs eines Clusters neu starten

```powershell
# VMs in einem Cluster geordnet neu starten (mit Bestätigung)
$cluster = Get-Cluster "Prod-Cluster"
$vms = Get-VM -Location $cluster | Where-Object { $_.PowerState -eq "PoweredOn" }
foreach ($vm in $vms) {
    Write-Host "Neustart: $($vm.Name)"
    Restart-VMGuest -VM $vm -Confirm:$false
    Start-Sleep -Seconds 30
}
```

---

## Fehlerbehandlung und Logging

```powershell
# Fehlerbehandlung mit try/catch
try {
    Connect-VIServer -Server vcenter.domain.local -User admin@vsphere.local -Password 'P@ss' -ErrorAction Stop
    $vms = Get-VM -ErrorAction Stop
} catch {
    Write-Error "Fehler: $($_.Exception.Message)"
    # Logging in Datei
    "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - FEHLER: $($_.Exception.Message)" | 
        Add-Content -Path "C:\Logs\powercli.log"
} finally {
    Disconnect-VIServer -Server * -Confirm:$false -ErrorAction SilentlyContinue
}
```

---

## Verwandte Notizen

- [[ESXi Grundlagen]]
- [[vCenter]]
- [[ESXi Installation]]
- [[vSphere Lifecycle Manager]]
