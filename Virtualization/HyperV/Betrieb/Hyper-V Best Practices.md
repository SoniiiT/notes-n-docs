---
tags:
  - hyper-v
  - best-practices
  - betrieb
  - sicherheit
  - performance
aliases:
  - Hyper-V Empfehlungen
  - Hyper-V Betrieb
---

# Hyper-V Best Practices

> [!ABSTRACT] TL;DR
> Bewährte Konfigurationsempfehlungen für professionell betriebene Hyper-V-Umgebungen — von Host-Härtung und VM-Konfiguration über Storage, Netzwerk und Sicherheit bis zu Updates, Backup und Performance-Monitoring. Gültig für Hyper-V unter Windows Server 2019/2022.

---

## 1. Host-Konfiguration

### Dedizierter Virtualisierungshost

> [!IMPORTANT] Hyper-V-Hosts sind keine Multifunktionsserver
> Ein Hyper-V-Host sollte **ausschließlich** die Hyper-V-Rolle tragen. Zusätzliche Serverrollen (AD, DNS, DHCP, IIS) auf dem Host konkurrieren direkt um CPU und RAM mit den VMs und erhöhen die Angriffsfläche erheblich.

| Rolle | Empfehlung |
|:------|:-----------|
| Hyper-V | ✅ Auf dem Host |
| Active Directory | ❌ Als VM auslagern |
| DNS / DHCP | ❌ Als VM auslagern |
| File Services | ❌ Als VM auslagern |
| Antivirus | ✅ Mit Hyper-V-Ausnahmen konfigurieren |

### Management-NIC — Dedizierte Netzwerkkarte

```powershell
# Dedizierte Management-NIC aus dem externen Switch herausnehmen
# (AllowManagementOS = $false auf dem VM-Traffic-Switch)
New-VMSwitch -Name "VM-Traffic" `
    -NetAdapterName "NIC-VM" -AllowManagementOS $false

# Management-NIC bleibt auf separatem Switch mit AllowManagementOS $true
New-VMSwitch -Name "Management" `
    -NetAdapterName "NIC-Mgmt" -AllowManagementOS $true
```

### NIC-Teaming (LBFO / SET)

> [!TIP] Switch Embedded Teaming (SET) bevorzugen
> SET ist die Microsoft-empfohlene Teaming-Lösung für Hyper-V ab Windows Server 2016. LBFO (Legacy) wird in Windows Server 2022+ nicht mehr weiterentwickelt. SET funktioniert direkt auf dem vSwitch ohne Umweg über LBFO.

```powershell
# SET-Team direkt beim Switch-Erstellen konfigurieren (2 NICs, LACP)
New-VMSwitch -Name "SET-Switch" `
    -NetAdapterName @("NIC1", "NIC2") `
    -EnableEmbeddedTeaming $true `
    -AllowManagementOS $true
```

### Zeitsynchronisation (W32tm)

Hyper-V synchronisiert Gast-Zeit standardmäßig mit dem Host. Für domänenmitglieder VMs ist das zu deaktivieren — sie sollen sich mit dem Domänen-Zeitserver synchronisieren.

```powershell
# W32tm-Status auf dem Host prüfen
w32tm /query /status

# NTP-Quellen konfigurieren
w32tm /config /manualpeerlist:"ntp1.example.local,ntp2.example.local" `
    /syncfromflags:manual /update

# Integration Service "Zeitsynchronisierung" für Domänen-VMs deaktivieren
Disable-VMIntegrationService -VMName "DomänenVM" -Name "Time Synchronization"
```

### Windows Server Core für Hyper-V-Hosts

> [!TIP] Server Core reduziert Angriffsfläche drastisch
> Windows Server Core (ohne GUI) hat eine deutlich kleinere Angriffsfläche, weniger Patches und weniger RAM-Overhead. Verwaltung erfolgt remote per PowerShell, Windows Admin Center oder RSAT. Für neue Hyper-V-Hosts ist Core die erste Wahl.

---

## 2. VM-Konfiguration

### Generation 2 für alle neuen VMs

| Merkmal | Generation 1 | Generation 2 |
|:--------|:------------|:-------------|
| Firmware | BIOS (Legacy) | UEFI |
| Secure Boot | ❌ | ✅ |
| Boot-Optionen | IDE, Legacy-NIC | SCSI, Synthetic NIC |
| PXE-Boot | Legacy NIC nötig | Direkt per Synthetic NIC |
| Performance | Geringer | **Besser** |
| Empfehlung | Nur Legacy-OS | **Standard für alle neuen VMs** |

```powershell
# VM-Generation prüfen
Get-VM -Name "MeineVM" | Select-Object Name, Generation

# Neue VM immer als Generation 2 erstellen
New-VM -Name "NeueVM" -Generation 2 -MemoryStartupBytes 4GB
```

### Dynamic Memory — Wann ja, wann nein

| Workload | Dynamic Memory | Begründung |
|:---------|:--------------|:-----------|
| Web-/App-Server (variabel) | ✅ Empfohlen | Last schwankt, Puffer reicht |
| SQL Server, Exchange | ❌ Fest | Brauchen vorhersehbaren RAM |
| Active Directory | ❌ Fest | Zeitkritisch, kein Ballooning |
| VDI / Desktop-VMs | ✅ Empfohlen | Last stark variabel |
| Linux-VMs | ⚠️ Mit Bedacht | Kernel-Version und Guest-Tools prüfen |

```powershell
# Festen RAM für kritische VMs (z.B. SQL Server)
Set-VMMemory -VMName "SQL-01" -DynamicMemoryEnabled $false -StartupBytes 32GB

# Dynamic Memory für VDI-VMs
Set-VMMemory -VMName "VDI-01" `
    -DynamicMemoryEnabled $true `
    -MinimumBytes 1GB -StartupBytes 2GB -MaximumBytes 8GB -Buffer 20
```

### Integration Services aktuell halten

> [!WARNING] Veraltete Integration Services = Stabilitätsrisiko
> Integration Services (IS) sind für korrekte Zeitsynchronisation, VSS-Backups, Live Migration und Graceful Shutdown zwingend erforderlich. Unter Windows-Gästen werden IS über Windows Update geliefert. Linux-Gäste benötigen aktuelle `linux-tools`-Pakete (Ubuntu) oder `hyperv-daemons` (RHEL/CentOS).

```powershell
# Integration Services aller laufenden VMs prüfen
Get-VM | Where-Object { $_.State -eq 'Running' } |
    Get-VMIntegrationService |
    Where-Object { $_.PrimaryStatusDescription -ne 'OK' -or -not $_.Enabled } |
    Select-Object VMName, Name, Enabled, PrimaryStatusDescription
```

### Keine ISOs dauerhaft gemountet lassen

```powershell
# ISO-Mounts aller VMs prüfen
Get-VM | Get-VMDvdDrive | Where-Object { $_.Path -ne $null } |
    Select-Object VMName, Path

# ISO aushängen nach Installation
Set-VMDvdDrive -VMName "MeineVM" -Path $null
```

### Checkpoint-Disziplin

```powershell
# Checkpoint-Typ auf Production setzen (VSS-konsistent, kein RAM)
Set-VM -VMName "MeineVM" -CheckpointType Production

# Automatische Checkpoints deaktivieren (werden von manchen Backup-Tools störend gesetzt)
Set-VM -VMName "MeineVM" -AutomaticCheckpointsEnabled $false
```

> [!DANGER] Checkpoint-Lawinen verhindern
> Mehr als 3 gestaffelte Checkpoints pro VM erhöhen Merge-Zeiten exponentiell und können den Datastore füllen. Niemals Checkpoints über Wochen laufen lassen.

---

## 3. Storage

### Fixed VHDX für Performance-kritische VMs

| VHDX-Typ | Performance | Platzbedarf | Empfehlung |
|:---------|:-----------|:------------|:-----------|
| **Dynamic** | Mittel (Wachstum kostet I/O) | Sparend | Entwicklung, Test, VDI |
| **Fixed** | **Hoch** (keine Wachstumskosten) | Voll reserviert | Produktion, Datenbanken |
| **Differencing** | Niedrig | Abhängig von Parent | Nur für temporäre Klone |

```powershell
# Fixed VHDX erstellen
New-VHD -Path "E:\Hyper-V\VHDs\SQL-DATA.vhdx" -SizeBytes 500GB -Fixed

# Dynamic VHDX zu Fixed konvertieren (VM muss ausgeschaltet sein)
Convert-VHD -Path "D:\VMs\alt.vhdx" -DestinationPath "E:\VMs\neu-fixed.vhdx" -VHDType Fixed
```

### VHDXs auf separates Volume vom Host-OS

```
Empfohlene Volume-Aufteilung:
┌──────────────────────────────────────────────────────┐
│  C:\  → Windows Server OS + Hyper-V Binaries         │
│  D:\  → VM-Konfigurationsdateien (.vmcx, .vmrs)      │
│  E:\  → VHDX-Dateien (OS-Disks der VMs)              │
│  F:\  → VHDX-Dateien (Daten-Disks der VMs)           │
│  G:\  → Checkpoint-Dateien (.avhdx)                  │
└──────────────────────────────────────────────────────┘
```

### Storage-Auslastung überwachen

> [!WARNING] Datastore nie über 80 % befüllen
> Über 80 % Auslastung riskiert, dass VHDX-Wachstum (Dynamic) oder Checkpoint-Merges den Volume komplett füllen. Vollgelaufene Volumes führen zu sofortigem VM-Stillstand.

```powershell
# Volume-Auslastung aller Volumes prüfen
Get-PSDrive -PSProvider FileSystem |
    Select-Object Name,
        @{N="Gesamt (GB)"; E={[math]::Round($_.Used/1GB + $_.Free/1GB, 0)}},
        @{N="Frei (GB)";   E={[math]::Round($_.Free/1GB, 1)}},
        @{N="Belegt (%)";  E={[math]::Round($_.Used / ($_.Used + $_.Free) * 100, 1)}} |
    Format-Table -AutoSize
```

### VHD-Sets für Cluster Shared Volumes

Bei Failover-Clustern mit Gast-Clustering (z.B. SQL FCI) VHD-Sets (`.vhds`) statt einzelner VHDXs verwenden — sie ermöglichen gemeinsamen Zugriff mehrerer VMs auf einen freigegebenen Datenträger.

---

## 4. Netzwerk

### Netzwerktrennung — Vier-Netz-Prinzip

```
Empfohlene Netzwerksegmentierung:
┌──────────────────────────────────────────────────────────┐
│  VLAN 10 — Management    (RDP, WinRM, Hyper-V Manager)   │
│  VLAN 20 — Live Migration (dedizierter Switch, MTU 9000) │
│  VLAN 30 — VM-Traffic    (Produktionsverkehr der Gäste)  │
│  VLAN 40 — Storage       (iSCSI / SMB, MTU 9000)        │
└──────────────────────────────────────────────────────────┘
```

```powershell
# VLAN-ID auf Management-VMNetworkAdapter setzen
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "Management" `
    -Access -VlanId 10
```

### Jumbo Frames für Storage und Live Migration

```powershell
# MTU auf 9014 setzen (NIC-Ebene)
Set-NetAdapterAdvancedProperty -Name "NIC-Storage" `
    -RegistryKeyword "*JumboPacket" -RegistryValue 9014

# MTU testen (Hyper-V-Host zu Storage-IP)
ping -f -l 8972 <storage-ip>
# -f = Don't Fragment  |  -l 8972 = Payload (+ 28 Byte Header = 9000 Byte gesamt)
```

### VMQ (Virtual Machine Queue) prüfen

> [!BUG] VMQ kann Netzwerkperformance verschlechtern
> VMQ ist auf manchen NIC-Treibern/Chipsets (insbesondere Intel i350 und ältere Broadcom) fehlerhaft implementiert und verursacht sporadische Paketverluste oder hohe Latenzen. Bei unklaren Netzwerkproblemen als erstes deaktivieren und testen.

```powershell
# VMQ-Status aller NICs prüfen
Get-NetAdapterVmq

# VMQ auf einer NIC deaktivieren
Disable-NetAdapterVmq -Name "NIC-VM"

# VMQ auf vSwitch-Adapter-Ebene deaktivieren
Set-VMNetworkAdapter -ManagementOS -VMNetworkAdapterName "vEthernet (VM-Traffic)" `
    -VmqWeight 0
```

---

## 5. Sicherheit

### Hyper-V-Hosts härten

| Maßnahme | Umsetzung |
|:---------|:----------|
| Lokale Admins minimieren | Nur Hyper-V Administrators-Gruppe, keine unnötigen Accounts |
| RDP absichern | NLA erzwingen, RDP-Port ändern oder auf Management-VLAN einschränken |
| Windows Firewall | Nur notwendige Ports öffnen (WinRM: 5985/5986, RDP: 3389, Replica: 443/80) |
| Windows Defender | Aktiviert mit Hyper-V-Ausnahmen (VM-Ordner von Echtzeitscan ausnehmen) |
| WinRM absichern | Nur HTTPS (5986), Zertifikat von interner CA |

### Secure Boot für Generation-2-VMs

```powershell
# Secure Boot aktivieren (Standard bei Gen 2)
Set-VMFirmware -VMName "MeineVM" -EnableSecureBoot On

# Secure Boot Template für Linux-VMs setzen
Set-VMFirmware -VMName "Ubuntu-VM" `
    -EnableSecureBoot On `
    -SecureBootTemplate "MicrosoftUEFICertificateAuthority"

# Secure Boot Template für Windows-VMs (Standard)
Set-VMFirmware -VMName "Windows-VM" `
    -EnableSecureBoot On `
    -SecureBootTemplate "MicrosoftWindows"
```

### Shielded VMs (für hochsichere Umgebungen)

> [!INFO] Shielded VMs schützen vor kompromittierten Admins
> Shielded VMs verschlüsseln die VM-Disk (BitLocker) und verhindern, dass selbst Hyper-V-Admins auf den VM-Inhalt zugreifen können. Voraussetzung: Host Guardian Service (HGS) im Netzwerk. Sinnvoll für mandantenfähige Cloud-Umgebungen und hochsichere Deployments.

---

## 6. Updates & Wartung

### Rolling Updates im Hyper-V-Cluster

> [!IMPORTANT] Cluster-Functional-Level nach Updates anheben
> Nach einem Host-Upgrade auf eine neue Windows-Server-Version bleibt das Cluster-Functional-Level (und damit die VM-Konfigurationsversion) auf dem alten Stand, bis es explizit angehoben wird. Neue Features stehen erst dann zur Verfügung.

```powershell
# Cluster-Functional-Level anzeigen
Get-Cluster | Select-Object ClusterFunctionalLevel

# Functional Level anheben (alle Hosts müssen auf neuer Version laufen!)
Update-ClusterFunctionalLevel -Cluster "MeinCluster"

# VM-Konfigurationsversionen nach Host-Upgrade aktualisieren
# (VM muss ausgeschaltet sein)
Update-VMVersion -VMName "MeineVM" -Force

# Alle VMs auf neuste Konfigurationsversion heben
Get-VM | Where-Object { $_.State -eq 'Off' } | Update-VMVersion -Force
```

### Wartungsreihenfolge für Cluster-Hosts

```
1. Host in Maintenance Mode versetzen (VMs per Live Migration abziehen)
2. Windows Update + Hyper-V-Patches installieren
3. Neustart
4. Hyper-V-Dienst und VM-Start prüfen
5. Host aus Maintenance Mode nehmen
6. Nächsten Host in Maintenance Mode versetzen
```

```powershell
# Host in Drain-Modus (Live Migration aller VMs, dann Maintenance)
Suspend-ClusterNode -Name "hv01" -Drain

# Host nach Update wieder aktivieren
Resume-ClusterNode -Name "hv01" -Failback Immediate
```

---

## 7. Backup & Disaster Recovery

### VSS-Checkpoints für Backup-Konsistenz

> [!IMPORTANT] Backup-Software benötigt Production Checkpoints
> Backup-Tools wie Veeam und Windows Server Backup nutzen VSS im Gast-OS. Dafür müssen Integration Services (insbesondere der VSS-Dienst) im Gast aktiv und aktuell sein. Ohne VSS-Integration gibt es nur absturzkonistente Backups.

| Tool | Stärken | Besonderheit |
|:-----|:--------|:-------------|
| **Veeam Backup & Replication** | Marktführer, CBT, Instant Recovery | Kostenlose Community Edition für ≤ 10 VMs |
| **Windows Server Backup** | Kostenlos, in Windows enthalten | Nur Volume-Level, keine Granularität |
| **Altaro VM Backup** | Günstig, einfach | KMU-fokussiert |
| **Hyper-V Replica** | Nativ, kein Agent | Kein Backup-Ersatz — nur DR |

### Hyper-V Replica als DR-Lösung

```powershell
# Replica-Broker auf Cluster konfigurieren (falls Failover-Cluster)
New-VMReplicationAuthorizationEntry `
    -AllowedPrimaryServer "*" `
    -ReplicaStorageLocation "D:\Replica" `
    -TrustGroup "DefaultTrustGroup"

# Replikationsintervall: 30 Sekunden (nur Windows Server 2012 R2+)
Enable-VMReplication -VMName "MeineVM" `
    -ReplicaServerName "hv-dr.example.local" `
    -ReplicationFrequencySec 30 `
    -AuthenticationType Kerberos
```

### DR-Mindeststandards

| Maßnahme | Mindeststandard |
|:---------|:----------------|
| Backup-Häufigkeit | Täglich — kritische VMs stündlich |
| Aufbewahrung | 30 Tage On-Site, 90 Tage Off-Site |
| Recovery-Test | Mindestens 1× pro Jahr (dokumentiert) |
| Hyper-V Replica RPO | 30 Sekunden bis 15 Minuten (je nach Intervall) |
| Host-Backup | Windows Server Backup für Hyper-V-Host-Konfiguration |

---

## 8. Performance-Monitoring

### Resource Metering aktivieren

```powershell
# Resource Metering für alle VMs aktivieren
Get-VM | Enable-VMResourceMetering

# Ressourcenverbrauch einer VM abfragen
Measure-VM -Name "MeineVM" |
    Select-Object VMName, AverageProcessorUsage, AverageMemoryUsage,
                  TotalDisk, NetworkMeteredTrafficReport
```

### Performance-Schwellwerte (Monitoring-Empfehlungen)

| Metrik | Warnschwelle | Kritisch | Maßnahme |
|:-------|:------------|:---------|:---------|
| Host-CPU-Auslastung | > 70 % | > 85 % | Workload migrieren / Kapazität erweitern |
| Memory Pressure (Host) | > 80 % | > 95 % | RAM-Überbuchung reduzieren / RAM einbauen |
| VHDX-Volume-Auslastung | > 80 % | > 90 % | Kapazität erweitern / VMs migrieren |
| Live Migration-Fehler | 1 pro Tag | 3 pro Tag | Netzwerk und Kerberos prüfen |
| Checkpoint-Alter | > 3 Tage | > 7 Tage | Checkpoint konsolidieren |
| Replica-Gesundheit | Warning | Critical | Netzwerk und RPO-Konfiguration prüfen |

### Hyper-V Performance Counter (Windows Performance Monitor)

```powershell
# Wichtige Hyper-V-Leistungsindikatoren im PowerShell abfragen
Get-Counter `
    "\Hyper-V Hypervisor Logical Processor(_Total)\% Total Run Time",
    "\Hyper-V Dynamic Memory Balancer(*)\Average Pressure",
    "\Hyper-V Virtual Storage Device(*)\Read Bytes/sec",
    "\Hyper-V Virtual Storage Device(*)\Write Bytes/sec" |
    Select-Object -ExpandProperty CounterSamples |
    Select-Object Path, CookedValue | Format-Table -AutoSize
```

> [!TIP] Windows Admin Center für Live-Monitoring nutzen
> Windows Admin Center (kostenlos, kein Agent) bietet eine browserbasierte Echtzeit-Ansicht für CPU, RAM, Disk I/O und Netzwerk aller VMs — deutlich komfortabler als Performance Monitor für den Alltag.

---

## Verwandte Notizen

- [[Was ist Hyper-V]]
- [[Hyper-V VM Verwaltung]]
- [[Hyper-V PowerShell Cheatsheet]]
- [[Hyper-V Troubleshooting]]
- [[Installation Hyper-V (Windows Server 2022)]]