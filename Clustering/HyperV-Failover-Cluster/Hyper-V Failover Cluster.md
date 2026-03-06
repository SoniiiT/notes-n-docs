---
tags: [clustering, hyper-v, failover-cluster, windows-server]
---

# Hyper-V Failover Cluster – Windows Server 2022

## Überblick

Ein Hyper-V Failover Cluster besteht aus mehreren Windows Servern (Nodes), die gemeinsam auf einen Shared Storage zugreifen. Fällt ein Node aus, übernehmen die verbleibenden Nodes automatisch die VMs — ohne (oder mit minimalem) Downtime.

```
┌─────────────────────────────────────────────────────┐
│                  Failover Cluster                   │
│                                                     │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│   │  Node 1  │    │  Node 2  │    │  Node N  │      │
│   │ Hyper-V  │    │ Hyper-V  │    │ Hyper-V  │      │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘      │
│        │               │               │            │
│        └───────────────┴───────────────┘            │
│                        │                            │
│               ┌────────┴────────┐                   │
│               │  Shared Storage │                   │
│               │ (iSCSI / FC)    │                   │
│               └─────────────────┘                   │
└─────────────────────────────────────────────────────┘
```

---

## Voraussetzungen

### Software

- Windows Server 2022 (Standard oder Datacenter) auf allen Nodes
- Alle Nodes müssen **Mitglied derselben Active Directory Domain** sein
- DNS muss für alle Nodes korrekt auflösen (Forward + Reverse)
- Gleiche Windows Server Version + Patchstand auf allen Nodes (empfohlen)

> [!WARNING]
> Ein Failover Cluster **erfordert** eine Active Directory Domain. Workgroup-Cluster sind mit erheblichen Einschränkungen verbunden und nicht empfohlen.

### Hardware

| Komponente | Anforderung |
| ---------- | ----------- |
| CPU | 64-bit, kompatible CPU-Familien auf allen Nodes (für Live Migration) |
| RAM | Mind. 8 GB pro Node |
| NICs (Hyper-V) | Mind. 4 Ethernet NICs pro Node (Management, Cluster/CSV, Live Migration, VM Network) |
| Storage-Adapter | Zusätzlich: iSCSI-Initiator (Software-NIC) oder FC HBA (Hardware, PCIe) |

> [!TIP]
> Für Live Migration ohne Kompatibilitätsmodus sollten alle Nodes CPUs **derselben Generation und Herstellers** (Intel oder AMD) haben.

### Netzwerkplanung

Ein Hyper-V Failover Cluster benötigt **4 dedizierte Netzwerke** für den Clusterbetrieb. Storage-Adapter kommen **zusätzlich** dazu und zählen nicht zu den 4 Basis-NICs.

#### Hyper-V Cluster Basis-NICs (mind. 4 pro Node)

| Netzwerk | Zweck | Empfehlung |
| -------- | ----- | ---------- |
| Management | RDP, Server-Verwaltung, WinRM, AD, DNS | 1 GbE, dediziertes VLAN |
| Cluster / CSV | Node-zu-Node Heartbeat + CSV-Redirected-Mode Traffic | 1–10 GbE, dediziertes VLAN, nicht routbar |
| Live Migration | VM-Speicher-Transfer zwischen Nodes | 10 GbE, dediziertes VLAN |
| VM Network | Externer Traffic der VMs (vSwitch External) | 10 GbE, Produktions-VLAN |

> [!NOTE]
> Cluster- und CSV-Traffic laufen über **dasselbe** Netzwerk. Ein separater CSV-NIC ist nicht erforderlich — der Cluster wählt automatisch das beste verfügbare Netzwerk für CSV-Kommunikation.

> [!IMPORTANT]
> **Live Migration und Cluster/CSV müssen jeweils in einem eigenen, dedizierten VLAN liegen.** Beide Netzwerke erzeugen hohen Traffic der andere Verbindungen saturieren kann — Live Migration während einer Migration, CSV im Redirected-Mode. Teilen sie sich ein VLAN mit anderen Netzwerken (z.B. Management), können Heartbeat-Pakete verzögert werden, was dazu führt dass der Cluster einen Node als ausgefallen markiert und VMs unnötig failovern. Die VLANs sollten außerdem **nicht routbar** sein — kein Default-Gateway, keine DNS-Registrierung.

#### Storage-Adapter (zusätzlich zu den 4 Basis-NICs)

| Storage-Methode | Adapter | Mindestanzahl | Hinweis |
| --------------- | ------- | ------------- | ------- |
| iSCSI | Ethernet NIC | 2x (für MPIO) | Dediziertes, nicht routbares VLAN. Kein NIC-Teaming via Hersteller-Software |
| Fiber Channel | FC HBA (PCIe) | 1x (direkt) oder 2x (redundant) | Direkt zum Storage oder via FC Switch. Zoning am FC Switch erforderlich |
| FC direkt (Direct Attach) | FC HBA (PCIe) | 1x pro Node | Direkte Kabelverbindung zur Storage-Einheit, kein FC Switch nötig |

> [!NOTE]
> FC HBAs sind **keine** Ethernet-NICs — sie sind separate PCIe-Karten und erscheinen nicht in der Windows NIC-Liste. Bei Fiber Channel bleiben die 4 Ethernet NICs unverändert; der HBA kommt als zusätzlicher PCIe-Adapter hinzu.

### Jumbo Frames (MTU 9000)

Jumbo Frames müssen für Live Migration, Cluster/CSV und iSCSI konfiguriert werden. Sie reduzieren den CPU-Overhead und erhöhen den Durchsatz um bis zu **~27%**.

> [!WARNING]
> Jumbo Frames müssen **end-to-end** konsistent konfiguriert sein — auf den NICs **aller** Nodes **und** auf allen Switch-Ports, die diese Netzwerke tragen. Eine inkonsistente MTU führt zu Fragmentierung oder Verbindungsabbrüchen.

| Netzwerk | Jumbo Frames | MTU |
| -------- | ------------ | --- |
| Live Migration | **Ja** | 9000 oder 9014 |
| Cluster / CSV | **Ja** | 9000 oder 9014 |
| iSCSI Storage | **Ja** | 9000 oder 9014 |
| Management | Nein | 1500 (Standard) |
| VM Network | Nein | 1500 (Standard) |

> [!NOTE]
> Der MTU-Wert (9000 oder 9014) hängt vom Hersteller ab. Wichtig ist, dass **alle** Geräte im Pfad denselben Wert verwenden. 9014 ist der häufigste Wert bei Intel NICs und berücksichtigt den Ethernet-Frame-Header.

#### NICs konfigurieren (PowerShell)

Auf **allen** Nodes ausführen — für jeden betroffenen Adapter:

```powershell
# Adapter anzeigen
Get-NetAdapter

# Jumbo Frames auf einem Adapter setzen (AdapterName anpassen)
Set-NetAdapterAdvancedProperty -Name "LM-NIC" -RegistryKeyword "*JumboPacket" -RegistryValue 9014
Set-NetAdapterAdvancedProperty -Name "CSV-NIC" -RegistryKeyword "*JumboPacket" -RegistryValue 9014
Set-NetAdapterAdvancedProperty -Name "iSCSI-NIC-1" -RegistryKeyword "*JumboPacket" -RegistryValue 9014
Set-NetAdapterAdvancedProperty -Name "iSCSI-NIC-2" -RegistryKeyword "*JumboPacket" -RegistryValue 9014

# Konfiguration prüfen
Get-NetAdapterAdvancedProperty -Name "LM-NIC" -RegistryKeyword "*JumboPacket"

# MTU-Wert über NetIPInterface prüfen
Get-NetIPInterface | Where-Object InterfaceAlias -like "*LM*" | Select-Object InterfaceAlias, NlMtu
```

#### Switch-Konfiguration

Der Switch muss auf **allen Ports**, die Live Migration, CSV und iSCSI-VLANs tragen, ebenfalls auf MTU 9000/9014 konfiguriert werden. Dies ist **herstellerabhängig**:

| Hersteller | Befehl (Beispiel) |
| ---------- | ----------------- |
| Cisco IOS | `system mtu jumbo 9000` |
| Cisco Nexus | `policy-map type network-qos jumbo` / `mtu 9216` |
| HPE / Aruba | `jumbo` auf dem Interface oder VLAN |
| Mikrotik | `/interface ethernet set mtu=9000` |

> [!WARNING]
> Jumbo Frames **nicht** auf dem VM Network oder Management-Netzwerk aktivieren, es sei denn die gesamte Infrastruktur inkl. aller Client-Geräte unterstützt MTU 9000. Im Zweifel Management und VM Network auf Standard-MTU (1500) belassen.

#### Jumbo Frames testen

Nach der Konfiguration auf allen Nodes und am Switch testen:

```powershell
# Ping mit Jumbo-Frame-Größe zwischen zwei Nodes (IP anpassen)
# -f = Don't Fragment, -l = Paketgröße (8972 = 9000 MTU - 28 Byte Header)
ping -f -l 8972 192.168.20.2

# Wenn der Ping erfolgreich ist, ist Jumbo Frames end-to-end korrekt konfiguriert.
# Wenn er fehlschlägt, ist ein Gerät im Pfad nicht korrekt konfiguriert.
```

---

## Shared Storage einrichten

### Option A – iSCSI

#### iSCSI Initiator auf jedem Node konfigurieren

```powershell
# iSCSI Initiator Dienst starten und auf automatisch setzen
Start-Service -Name MSiSCSI
Set-Service -Name MSiSCSI -StartupType Automatic

# iSCSI Target verbinden (IP des Storage-Systems anpassen)
New-IscsiTargetPortal -TargetPortalAddress "192.168.10.100"

# Verfügbare Targets anzeigen
Get-IscsiTarget

# Mit Target verbinden (TargetNodeAddress aus Get-IscsiTarget)
Connect-IscsiTarget -NodeAddress "iqn.2024-01.com.storage:target01" -IsPersistent $true

# Verbindung prüfen
Get-IscsiSession
```

#### MPIO (Multipath I/O) für iSCSI Redundanz

```powershell
# MPIO Feature installieren (auf allen Nodes)
Add-WindowsFeature -Name Multipath-IO -IncludeManagementTools

# iSCSI für MPIO registrieren
Enable-MSDSMAutomaticClaim -BusType iSCSI

# MPIO Load-Balancing Policy setzen
Set-MSDSMGlobalDefaultLoadBalancePolicy -Policy RR  # Round Robin

# MPIO Pfade prüfen
Get-MSDSMDevice | Select-Object -ExpandProperty DeviceInstanceId
```

> [!NOTE]
> MPIO benötigt mindestens zwei separate iSCSI-Pfade (zwei NICs oder zwei Ports am Storage). Nach der Installation muss neu gestartet werden.

### Option B – Fiber Channel (FC)

Fiber Channel ist hardwareabhängig und erfordert FC HBAs in allen Nodes sowie einen FC Switch (oder Direct Attach).

```powershell
# FC Adapter anzeigen
Get-InitiatorPort | Where-Object { $_.ConnectionType -eq "Fibre Channel" }

# FC-Zonen müssen am FC Switch konfiguriert werden (herstellerabhängig)
# Nach Zoning: LUNs erscheinen automatisch als Disk im Disk Management
```

> [!NOTE]
> Das Zoning am FC Switch (z.B. Brocade, Cisco MDS) ist herstellerspezifisch und muss separat konfiguriert werden. Jeder Node benötigt Zugriff auf dieselben LUNs.

#### Disks initialisieren und formatieren (gilt für iSCSI und FC)

Nach dem Verbinden der Shared Storage LUNs müssen diese auf **einem** Node initialisiert werden — die anderen Nodes sehen die Disks danach automatisch.

```powershell
# Neue Disks anzeigen (Offline-Status erwartet)
Get-Disk | Where-Object PartitionStyle -eq "RAW"

# Disk online bringen, initialisieren und formatieren
# Disk-Nummer anpassen (Get-Disk zeigt die Nummer)
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter
Format-Volume -DriveLetter E -FileSystem NTFS -NewFileSystemLabel "Cluster-CSV-01" -Confirm:$false
```

---

## Failover Clustering Feature installieren

Auf **allen Nodes** ausführen:

```powershell
# Failover Clustering + Management Tools installieren
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools

# Hyper-V Rolle sicherstellen (falls noch nicht installiert)
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools

# Neustart erforderlich
Restart-Computer
```

---

## Cluster Validierung

Vor der Cluster-Erstellung muss eine Validierung durchgeführt werden. Fehler im Validierungsbericht können zu einem nicht unterstützten Cluster führen.

```powershell
# Validierungstest auf einem zukünftigen Cluster-Node ausführen
# Alle Node-Namen als Array angeben
Test-Cluster -Node "Node1", "Node2", "Node3"

# Nur bestimmte Tests ausführen (optional, für schnellere Validierung)
Test-Cluster -Node "Node1", "Node2" -Include "Storage", "Network"
```

> [!WARNING]
> Der Validierungsbericht wird als HTML-Datei gespeichert. Alle **Fehler** müssen vor der Cluster-Erstellung behoben werden. **Warnungen** können je nach Umgebung akzeptiert werden.

Via GUI: **Failover Cluster Manager** → `Validate Configuration` → Nodes hinzufügen → alle Tests ausführen.

---

## Cluster erstellen

### Via PowerShell

```powershell
# Cluster erstellen
# -Name: Cluster-Name (wird als Computerobjekt in AD angelegt)
# -Node: alle Nodes
# -StaticAddress: Cluster-IP (oder weglassen für DHCP)
New-Cluster -Name "HV-Cluster-01" `
            -Node "Node1", "Node2", "Node3" `
            -StaticAddress "192.168.1.50"

# Cluster-Status prüfen
Get-Cluster
Get-ClusterNode
```

### Via GUI (Failover Cluster Manager)

1. **Failover Cluster Manager** öffnen
2. `Create Cluster` → Nodes hinzufügen
3. Cluster-Name und IP-Adresse vergeben
4. Validierung bestätigen
5. `Finish`

---

## Cluster Shared Volumes (CSV)

CSV ermöglicht allen Nodes simultanen Zugriff auf dieselbe Disk — notwendig für Live Migration.

```powershell
# Disk zum Cluster hinzufügen (Disk muss bereits initialisiert sein)
Add-ClusterDisk -InputObject (Get-Disk -Number 1)

# Alle Cluster-Disks anzeigen
Get-ClusterAvailableDisk
Get-ClusterResource | Where-Object ResourceType -eq "Physical Disk"

# Disk als CSV aktivieren
Add-ClusterSharedVolume -Name "Cluster Disk 1"

# CSVs anzeigen
Get-ClusterSharedVolume

# CSV Pfad: C:\ClusterStorage\Volume1\ (auf allen Nodes verfügbar)
```

---

## Quorum konfigurieren

Das Quorum bestimmt, wie viele Stimmen für einen funktionsfähigen Cluster benötigt werden. Jeder Node hat 1 Stimme, ein Witness hat ebenfalls 1 Stimme.

### Empfehlung nach Node-Anzahl

| Nodes | Empfohlener Quorum-Typ | Begründung |
| ----- | ---------------------- | ---------- |
| 2 | Node + File Share Witness oder Cloud Witness | Verhindert Split-Brain |
| 3 | Node Majority | Toleriert 1 Node-Ausfall ohne Witness |
| 4 | Node + Disk Witness | Gerade Anzahl → Witness für Tiebreaker |
| 5+ | Node Majority | Toleriert automatisch mehrere Ausfälle |

### Quorum-Typen

#### 1. Node Majority (kein Witness)

Geeignet ab 3 Nodes (ungerade Anzahl). Cluster bleibt online solange Mehrheit der Nodes erreichbar.

```powershell
Set-ClusterQuorum -NodeMajority
```

#### 2. Disk Witness

Ein kleines Shared Disk (mind. 512 MB) dient als Witness. Empfohlen bei gerader Node-Anzahl mit Zugriff auf Shared Storage.

```powershell
# Witness-Disk muss im Cluster als Resource vorhanden sein
Set-ClusterQuorum -DiskWitness "Cluster Disk Witness"
```

#### 3. File Share Witness

Eine SMB-Freigabe auf einem separaten Server (z.B. DC oder NAS) dient als Witness. Ideal für 2-Node Cluster.

```powershell
# Freigabe muss existieren und der Cluster-Account braucht Schreibrechte
Set-ClusterQuorum -FileShareWitness "\\DC01\ClusterWitness"
```

**Freigabe vorbereiten:**
```powershell
# Auf dem Witness-Server ausführen
New-Item -Path "C:\ClusterWitness" -ItemType Directory
New-SmbShare -Name "ClusterWitness" -Path "C:\ClusterWitness" -FullAccess "DOMAIN\Cluster-Name$"
```

#### 4. Cloud Witness (Azure Blob Storage)

Kein lokaler Witness-Server nötig. Empfohlen wenn kein separater Server verfügbar ist oder für Multi-Site Cluster.

```powershell
# Azure Storage Account Name und Access Key benötigt
Set-ClusterQuorum -CloudWitness `
    -AccountName "mystorageaccount" `
    -AccessKey "Base64EncodedAccessKey=="
```

> [!TIP]
> Cloud Witness braucht ausgehende HTTPS-Verbindung (Port 443) zu Azure. Für reine Homelab-Umgebungen ohne Internet-Zugang ist File Share Witness die bessere Wahl.

### Aktuellen Quorum anzeigen

```powershell
Get-ClusterQuorum
Get-ClusterNode | Select-Object Name, NodeWeight, State
```

---

## Hyper-V VMs als Cluster-Ressource einrichten

```powershell
# Neue Highly Available VM erstellen (direkt auf CSV-Pfad)
New-VM -Name "VM-HA-01" `
       -Path "C:\ClusterStorage\Volume1\VMs" `
       -MemoryStartupBytes 4GB `
       -Generation 2 `
       -SwitchName "vSwitch-External" `
       -NewVHDPath "C:\ClusterStorage\Volume1\VHDs\VM-HA-01.vhdx" `
       -NewVHDSizeBytes 60GB

# VM zum Cluster hinzufügen
Add-ClusterVirtualMachineRole -VirtualMachine "VM-HA-01"

# Bestehende VM zum Cluster hinzufügen
Add-ClusterVirtualMachineRole -VMId (Get-VM -Name "VM-HA-01").VMId

# Alle Cluster-VMs anzeigen
Get-ClusterGroup | Where-Object GroupType -eq "VirtualMachine"

# VM manuell auf anderen Node verschieben (Live Migration)
Move-ClusterVirtualMachineRole -Name "VM-HA-01" -Node "Node2"
```

---

## Live Migration konfigurieren

```powershell
# Live Migration aktivieren (auf allen Nodes)
Enable-VMMigration

# Erlaubte Netzwerke für Live Migration festlegen
Add-VMMigrationNetwork -Subnet "192.168.20.0/24" -Priority 1

# Maximale parallele Migrationen festlegen
Set-VMHost -MaximumVirtualMachineMigrations 2

# Live Migration Authentifizierungsmethode
Set-VMHost -VirtualMachineMigrationAuthenticationType Kerberos

# Live Migration testen
Move-ClusterVirtualMachineRole -Name "VM-HA-01" -Node "Node2"
```

---

## PowerShell Referenz

### Cluster Status

```powershell
# Allgemeiner Cluster-Status
Get-Cluster
Get-ClusterNode
Get-ClusterResource
Get-ClusterGroup

# Cluster-Events anzeigen
Get-ClusterLog -Destination "C:\Temp\ClusterLog"
```

### Failover simulieren

```powershell
# Node in Maintenance Mode versetzen (VMs werden migriert)
Suspend-ClusterNode -Name "Node1" -Drain

# Node wieder aktivieren
Resume-ClusterNode -Name "Node1"

# Node aus Cluster entfernen (für Wartung)
Stop-ClusterNode -Name "Node1"

# Node wieder hinzufügen
Start-ClusterNode -Name "Node1"
```

### Cluster-Netzwerke

```powershell
# Cluster-Netzwerke anzeigen
Get-ClusterNetwork

# Netzwerk-Rolle festlegen
# 0 = None, 1 = Cluster only, 3 = Cluster + Client
(Get-ClusterNetwork -Name "Cluster Heartbeat").Role = 1
(Get-ClusterNetwork -Name "Management").Role = 3
```

### Troubleshooting

```powershell
# Cluster-Validierung erneut ausführen
Test-Cluster -Node "Node1", "Node2"

# Cluster-Logs generieren
Get-ClusterLog -Destination "C:\Temp" -TimeSpan 60  # Letzte 60 Minuten

# Event Log nach Cluster-Fehlern durchsuchen
Get-WinEvent -LogName "Microsoft-Windows-FailoverClustering/Operational" |
    Where-Object LevelDisplayName -eq "Error" |
    Select-Object TimeCreated, Message -First 20

# Cluster-Dienst neustarten (nur wenn nötig!)
Restart-Service -Name ClusSvc
```

---

## Nützliche Links

- [Failover Clustering in Windows Server – Microsoft Docs](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview)
- [Hyper-V Live Migration – Microsoft Docs](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/manage/live-migration-overview)
- [Configure and Manage Quorum – Microsoft Docs](https://learn.microsoft.com/en-us/windows-server/failover-clustering/manage-cluster-quorum)

---

## Verwandte Notizen

- [[Installation Hyper-V (Windows Server 2022)]] — Hyper-V Rolle installieren und konfigurieren
