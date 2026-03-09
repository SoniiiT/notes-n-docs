---
tags:
  - virtualization
  - hyper-v
  - storage
  - vhd
  - vhdx
  - iscsi
---

# Hyper-V Storage

> [!ABSTRACT] TL;DR
> Hyper-V unterstützt virtuelle Festplatten (VHD/VHDX), Pass-through Disks, SMB 3.x, iSCSI und Storage Spaces Direct. VHDX ist der moderne Standard — immer VHDX statt VHD verwenden. Fixed-Disks für Produktion, Dynamic für Entwicklung, Differencing für Test-Umgebungen.

---

## VHD vs. VHDX

VHDX ist das seit Windows Server 2012 eingeführte, modernisierte Format für virtuelle Festplatten. Es löst nahezu alle Limitierungen des alten VHD-Formats und sollte in jeder neuen Umgebung ausschließlich verwendet werden.

| Eigenschaft | VHD | VHDX |
|---|---|---|
| **Max. Größe** | 2 TB | 64 TB |
| **Block-Größe** | 512 KB (Fixed), 2 MB (Dynamic) | Konfigurierbar: 1 MB–256 MB |
| **Sektor-Größe** | 512 Byte (emuliert) | 512e und native 4K-Sektoren |
| **Journal (Resilienz)** | Nein | Ja — schützt bei Stromausfall |
| **Alignment** | 512 Byte | 1 MB — optimal für moderne Speicher |
| **Metadaten-Struktur** | Einfach, fehleranfällig | Verbessert, robuster |
| **Mindest-Host-OS** | Windows Server 2003+ | Windows Server 2012+ |
| **Max. Snapshots** | 50 | 50 |
| **Große Sektorgrößen (4K)** | Nein | Ja |
| **PowerShell-Cmdlets** | `New-VHD -Path *.vhd` | `New-VHD -Path *.vhdx` |

> [!TIP] Empfehlung
> Immer **VHDX** verwenden. VHD nur noch für Kompatibilität mit älteren Hyper-V-Hosts (vor WS 2012) oder für Generation-1-VMs, die zwingend ein legacy-Format benötigen. Bestehende VHDs können mit `Convert-VHD` migriert werden.

> [!INFO] 4K-Sektor-Support
> Moderne SSDs und NVMe-Laufwerke nutzen native 4K-Sektoren. VHDX mit `SectorSizeBytes 4096` vermeidet den 512e-Emulations-Overhead und liefert bessere Performance auf solcher Hardware.

---

## Disk-Typen

Hyper-V kennt drei Typen für virtuelle Festplatten, die sich in Speicherverhalten und Performance grundlegend unterscheiden.

| Typ | Beschreibung | Speicherverbrauch | Performance | Typischer Einsatz |
|---|---|---|---|---|
| **Fixed** | Belegt sofort die volle Größe auf dem Host | Sofort = VHD-Größe | ★★★★★ Beste | Produktions-VMs, Datenbanken |
| **Dynamic** | Wächst bei Bedarf bis zur max. Größe | Anfangs minimal, wächst | ★★★☆☆ Mittel | Entwicklung, Test, wenig kritisch |
| **Differencing** | Speichert nur Deltas zur Parent-VHDX | Minimal (nur Änderungen) | ★★☆☆☆ Schlechter | Testlabore, Massendeployments |

### Fixed Disk

Die virtuelle Festplatte reserviert beim Erstellen sofort den gesamten Speicherplatz auf dem Host. Der VHDX-Header gibt der VM immer dieselbe konsistente Blockstruktur — keine Fragmentierung durch Wachstum, keine Meta-Operationen zur Laufzeit.

> [!TIP] Produktionsempfehlung
> Fixed Disks sind der **Standard für Produktions-Workloads**. Der höhere initiale Speicherverbrauch ist der Preis für deterministische, maximale I/O-Performance ohne versteckte Latenzen.

### Dynamic Disk

Der tatsächlich belegte Speicherplatz wächst automatisch, wenn die VM Daten schreibt. Das spart Speicher, kostet aber Performance durch die zusätzlichen Metadaten-Operationen beim Erweitern. Auf einem vollgelaufenen Host-Dateisystem kann eine Dynamic Disk nicht mehr wachsen — die VM hängt oder crasht.

> [!WARNING] Speicher-Overcommit
> Dynamic Disks verleiten zum Overcommit: Wenn zehn VMs je 500 GB dynamic haben, aber der Host nur 2 TB frei hat, funktioniert das — bis alle VMs gleichzeitig vollschreiben. Im schlimmsten Fall frieren VMs ein oder korrumpieren Daten.

### Differencing Disk

Eine Differencing Disk hat eine **Parent-VHDX**, die schreibgeschützt bleibt. Alle Schreiboperationen landen in der Child-VHDX (Delta). Das ist das technische Fundament für Hyper-V Checkpoints.

**Einsatzszenarien:**
- Laborumgebungen: ein goldenes Parent-Image, viele Child-VMs
- Schnelles Zurücksetzen: Child löschen → VM ist wieder frisch
- Platzsparende Massenbereitstellung identischer OS-Instanzen

> [!WARNING] Differencing in Produktion
> Differencing Disks **nicht** in Produktionsumgebungen verwenden. Die Parent-VHDX darf nie verändert werden, solange Children existieren — sonst korrumpieren alle Children. Performance nimmt mit Tiefe der Chain ab.

---

## VHDX-Operationen mit PowerShell

### Erstellen

```powershell
# Fixed Disk — 100 GB, sofort belegt
New-VHD -Path "D:\VMs\Server01\OS.vhdx" `
        -SizeBytes 100GB `
        -Fixed

# Dynamic Disk — max. 500 GB, wächst bei Bedarf
New-VHD -Path "D:\VMs\Server01\Data.vhdx" `
        -SizeBytes 500GB `
        -Dynamic

# Dynamic mit angepasster Block-Größe (32 MB für sequentielle Workloads)
New-VHD -Path "D:\VMs\Server01\Logs.vhdx" `
        -SizeBytes 200GB `
        -Dynamic `
        -BlockSizeBytes 32MB

# Differencing Disk — Parent bleibt schreibgeschützt
New-VHD -Path "D:\VMs\TestVM01\Delta.vhdx" `
        -ParentPath "D:\Templates\WS2025-Base.vhdx" `
        -Differencing
```

### Vergrößern

```powershell
# VHDX auf 200 GB erweitern
Resize-VHD -Path "D:\VMs\Server01\Data.vhdx" -SizeBytes 200GB
```

> [!WARNING] Checkpoints vor Resize entfernen
> `Resize-VHD` schlägt fehl, wenn die VM aktive Checkpoints (Snapshots) hat. Vorher alle Checkpoints zusammenführen:
> ```powershell
> Get-VMSnapshot -VMName "Server01" | Remove-VMSnapshot -IncludeAllChildSnapshots
> ```
> Anschließend nach dem Resize im Gast-OS die Partition erweitern (Disk Management oder `Resize-Partition`).

### Verkleinern (Compact)

```powershell
# Schritt 1: Im Gast-OS — Laufwerk defragmentieren und Nullen schreiben
# (auf Windows-Gast als Admin ausführen)
Optimize-Volume -DriveLetter D -ReTrim -Verbose

# Schritt 2: VM herunterfahren, dann auf dem Host komprimieren
Optimize-VHD -Path "D:\VMs\Server01\Data.vhdx" -Mode Full

# Schritt 3: Eigentliche Größenreduktion
Compact-VHD -Path "D:\VMs\Server01\Data.vhdx"

# Danach Größe prüfen
Get-VHD -Path "D:\VMs\Server01\Data.vhdx" | Select-Object FileSize, Size
```

> [!INFO] Compact nur bei Dynamic Disks
> `Compact-VHD` wirkt ausschließlich auf **Dynamic Disks**. Fixed Disks belegen immer ihre volle deklarierte Größe — eine Verkleinerung erfordert ein Neuerstellen mit `Convert-VHD`.

### Konvertieren

```powershell
# Dynamic → Fixed (für Produktionsmigration)
Convert-VHD -Path "D:\VMs\Server01\OS.vhdx" `
            -DestinationPath "D:\VMs\Server01\OS-Fixed.vhdx" `
            -VHDType Fixed

# VHD → VHDX (Legacy-Migration)
Convert-VHD -Path "D:\VMs\LegacyVM\OS.vhd" `
            -DestinationPath "D:\VMs\LegacyVM\OS.vhdx" `
            -VHDFormat VHDX

# Fixed → Dynamic (Speichersparen im Lab)
Convert-VHD -Path "D:\VMs\Lab01\OS.vhdx" `
            -DestinationPath "D:\VMs\Lab01\OS-Dynamic.vhdx" `
            -VHDType Dynamic
```

### Informationen abrufen

```powershell
# Einzelne VHDX inspizieren
Get-VHD -Path "D:\VMs\Server01\OS.vhdx"

# Alle VHDXs einer VM anzeigen
Get-VMHardDiskDrive -VMName "Server01" |
    ForEach-Object { Get-VHD -Path $_.Path }

# Übersichtstabelle aller VMs mit Disk-Größen
Get-VM | Get-VMHardDiskDrive |
    Select-Object VMName, Path,
        @{N="FileSizeGB"; E={ [math]::Round((Get-VHD $_.Path).FileSize/1GB, 1) }},
        @{N="MaxSizeGB";  E={ [math]::Round((Get-VHD $_.Path).Size/1GB, 1) }}
```

### Mounten und Unmounten auf dem Host

```powershell
# VHDX auf dem Host einhängen (ohne laufende VM)
Mount-VHD -Path "D:\VMs\Server01\Data.vhdx" -ReadOnly

# Laufwerksbuchstaben anzeigen
Get-VHD -Path "D:\VMs\Server01\Data.vhdx" | Select-Object DiskNumber

# Aushängen
Dismount-VHD -Path "D:\VMs\Server01\Data.vhdx"

# Alternativ per DiskNumber
Dismount-VHD -DiskNumber 3
```

> [!TIP] Offline-Datenzugriff
> `Mount-VHD -ReadOnly` erlaubt den direkten Zugriff auf VHDX-Inhalte vom Host aus — nützlich für Dateiwiederherstellung ohne VM-Start. Nach dem Zugriff immer `Dismount-VHD` aufrufen, sonst kann Hyper-V die Disk nicht an die VM übergeben.

---

## Shared VHDX (Cluster Shared Volumes)

Shared VHDX ermöglicht mehreren VMs gleichzeitigen Zugriff auf dieselbe VHDX-Datei — die Grundlage für Windows Server Failover Cluster innerhalb von VMs (z. B. SQL Server AlwaysOn Availability Groups, Scale-Out File Server).

### Einsatzszenarien

| Szenario | Beschreibung |
|---|---|
| **Failover Cluster** | Zwei VMs teilen sich eine VHDX als Cluster Shared Volume |
| **SQL AlwaysOn** | Shared Disk für Quorum oder Datenbankdateien |
| **Scale-Out File Server** | Mehrere Knoten greifen auf gemeinsamen Storage zu |
| **Oracle RAC** | Shared Storage für verteilte Datenbankinstanzen |

### Einschränkungen

> [!WARNING] Shared VHDX — Bekannte Limitierungen
> - Nur **Fixed Disks** unterstützt — kein Dynamic, kein Differencing
> - **Keine Checkpoints** möglich, solange Shared VHDX angehängt ist
> - **Kein Live Migration** mit angehängten Shared VHDXs (außer SMB-Storage)
> - Nur auf **SCSI-Controller** (nicht IDE) verwendbar
> - Erfordert **SMB 3.0+** oder **CSV** als Speicherort

### PowerShell

```powershell
# Shared VHDX erstellen (Fixed, da Pflicht)
New-VHD -Path "\\FileServer\VMs\Cluster-Quorum.vhdx" `
        -SizeBytes 10GB `
        -Fixed

# Shared VHDX an VM1 anhängen
Add-VMSharedVHD -VMName "ClusterNode01" `
                -ControllerType SCSI `
                -ControllerNumber 0 `
                -ControllerLocation 1 `
                -Path "\\FileServer\VMs\Cluster-Quorum.vhdx"

# An VM2 anhängen (dieselbe Datei)
Add-VMSharedVHD -VMName "ClusterNode02" `
                -ControllerType SCSI `
                -ControllerNumber 0 `
                -ControllerLocation 1 `
                -Path "\\FileServer\VMs\Cluster-Quorum.vhdx"

# Shared VHDXs einer VM anzeigen
Get-VMSharedVHD -VMName "ClusterNode01"
```

---

## Pass-through Disks

Eine Pass-through Disk verbindet eine **physische Festplatte oder LUN** direkt mit einer VM — ohne Umweg über eine VHDX-Datei. Der Hypervisor-Layer entfällt für den Datenpfad.

### Wann sinnvoll?

| Szenario | Begründung |
|---|---|
| Datenbankserver mit extremen IOPS-Anforderungen | Kein VHDX-Overhead im I/O-Pfad |
| Sehr große Volumes (>64 TB — über VHDX-Limit) | VHDX-Maximum wird umgangen |
| Migrationsszenarien | Physische Disk direkt in VM übernehmen |
| Spezieller Treiber-Support im Gast nötig | Direkter Hardware-Zugriff |

### Nachteile

> [!WARNING] Pass-through Einschränkungen
> - **Kein Live Migration / vMotion** — physische Disk ist an den Host gebunden
> - **Keine Checkpoints** — der Zustand einer physischen Disk lässt sich nicht snapshotten
> - **Keine einfache Portabilität** — Disk muss physisch bewegt oder neu zugewiesen werden
> - Die Disk muss auf dem Host als **Offline** markiert sein (nicht durch Host-OS verwendet)

### PowerShell

```powershell
# Disk auf dem Host offline setzen (Voraussetzung!)
Set-Disk -Number 2 -IsOffline $true
Set-Disk -Number 2 -IsReadOnly $false

# Pass-through Disk an VM anhängen
Add-VMHardDiskDrive -VMName "DBServer01" `
                    -ControllerType SCSI `
                    -ControllerNumber 0 `
                    -DiskNumber 2

# Übersicht: Welche physischen Disks sind als Pass-through genutzt?
Get-VMHardDiskDrive -VMName "DBServer01" |
    Where-Object { $_.DiskNumber -ne $null }

# Pass-through Disk entfernen und Host-Disk wieder online
Remove-VMHardDiskDrive -VMName "DBServer01" `
                       -ControllerType SCSI `
                       -ControllerNumber 0 `
                       -ControllerLocation 1
Set-Disk -Number 2 -IsOffline $false
```

---

## Storage Spaces und Storage Spaces Direct

### Storage Spaces (lokal)

Storage Spaces fasst mehrere physische Disks zu einem virtuellen Pool zusammen und bietet Resilienz (Mirror, Parity) auf Software-Ebene. Hyper-V VMs können direkt auf Storage-Spaces-Volumes liegen.

```powershell
# Storage Pool aus vier Disks erstellen
$Disks = Get-PhysicalDisk -CanPool $true
New-StoragePool -FriendlyName "VMPool" `
                -StorageSubSystemFriendlyName "Windows Storage*" `
                -PhysicalDisks $Disks

# Mirror-Volume für VMs (2-Wege-Spiegel)
New-VirtualDisk -StoragePoolFriendlyName "VMPool" `
                -FriendlyName "VM-Mirror" `
                -Size 2TB `
                -ResiliencySettingName Mirror `
                -ProvisioningType Fixed

# Volume initialisieren und formatieren
Get-VirtualDisk -FriendlyName "VM-Mirror" |
    Get-Disk | Initialize-Disk -PartitionStyle GPT
New-Volume -DiskFriendlyName "VM-Mirror" `
           -FileSystem ReFS `
           -FriendlyName "VMs" `
           -DriveLetter V
```

> [!TIP] ReFS für VHDX
> **ReFS** (Resilient File System) statt NTFS für VM-Storage verwenden. ReFS bietet nativen Copy-on-Write für VHDX-Checkpoints (Block Cloning) — Checkpoint-Operationen sind dadurch extrem schnell und verbrauchen kaum zusätzlichen Speicherplatz.

### Storage Spaces Direct (S2D)

S2D ist die hyperkonvergente Variante: Lokale Disks aller Cluster-Knoten werden zu einem gemeinsamen Pool zusammengefasst. Ideal für Hyper-V Failover Cluster ohne dedizierte SAN/NAS-Hardware.

```powershell
# S2D auf Cluster aktivieren (auf einem Knoten ausführen)
Enable-ClusterStorageSpacesDirect -PoolFriendlyName "S2D-Pool"

# CSV (Cluster Shared Volume) für VMs erstellen
New-Volume -StoragePoolFriendlyName "S2D*" `
           -FriendlyName "CSV-VMs" `
           -FileSystem CSVFS_ReFS `
           -Size 5TB

# Tiered Storage: SSD-Tier + HDD-Tier kombinieren
New-Volume -StoragePoolFriendlyName "S2D*" `
           -FriendlyName "CSV-Tiered" `
           -StorageTierFriendlyNames "Performance", "Capacity" `
           -StorageTierSizes 500GB, 4TB `
           -FileSystem CSVFS_ReFS
```

---

## Netzwerk-Storage

### SMB 3.x für VM-Dateien

Hyper-V unterstützt nativ das Speichern von VM-Dateien (VHDX, VM-Konfiguration) auf SMB 3.x Shares. Das ist der empfohlene Weg für Scale-Out File Server (SoFS) und NAS-Systeme.

**Voraussetzungen:**
- SMB 3.0 oder neuer auf dem File Server
- SMB Signing oder SMB Encryption (empfohlen)
- Dienstkonto des Hyper-V Hosts hat Vollzugriff auf den Share

```powershell
# Standard-Pfade für VMs auf SMB umbiegen
Set-VMHost -VirtualHardDiskPath "\\FileServer\VMStorage\VHDs" `
           -VirtualMachinePath "\\FileServer\VMStorage\VMs"

# Einzelne VM auf SMB-Share erstellen
New-VM -Name "WebServer01" `
       -MemoryStartupBytes 4GB `
       -SwitchName "Production" `
       -NewVHDPath "\\FileServer\VMStorage\VHDs\WebServer01-OS.vhdx" `
       -NewVHDSizeBytes 80GB `
       -Generation 2

# SMB-Bandbreitenlimit für Live Migration setzen
Set-SmbBandwidthLimit -Category LiveMigration -BytesPerSecond 500MB
```

### iSCSI

iSCSI verbindet den Hyper-V Host mit einem iSCSI-Target (NAS, SAN) über das Netzwerk. Die LUN erscheint dem Host als lokale Disk und kann direkt mit VMs genutzt werden.

```powershell
# iSCSI-Initiator starten und automatisch starten
Start-Service msiscsi
Set-Service msiscsi -StartupType Automatic

# Target entdecken (Discovery)
New-IscsiTargetPortal -TargetPortalAddress "192.168.10.50"

# Verfügbare Targets anzeigen
Get-IscsiTarget

# Verbindung zum Target herstellen
Connect-IscsiTarget -NodeAddress "iqn.2024-01.com.storage:target01" `
                    -IsPersistent $true `
                    -IsMultipathEnabled $true

# Verbundene Sessions anzeigen
Get-IscsiSession

# Disk initialisieren (nach Verbindung)
Get-Disk | Where-Object PartitionStyle -eq RAW | Initialize-Disk -PartitionStyle GPT
```

> [!INFO] iSCSI Netzwerktrennung
> iSCSI-Traffic sollte auf einem **dedizierten VLAN oder Netzwerk-Interface** laufen — getrennt vom VM-Traffic und Management-Netz. Empfohlen: 10 GbE oder schneller, Jumbo Frames (MTU 9000) für optimalen Durchsatz.

### Vergleich: Storage-Typen

| Kriterium | Lokal (Direct Attach) | SMB 3.x (NAS/SoFS) | iSCSI | Fibre Channel |
|---|---|---|---|---|
| **Performance** | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★★★ |
| **Kosten** | Niedrig | Mittel | Mittel | Hoch |
| **Skalierbarkeit** | Begrenzt | Gut | Gut | Sehr gut |
| **Live Migration** | Nur mit CSV | ✅ Ja | ✅ Ja (CSV) | ✅ Ja (CSV) |
| **Shared VHDX** | ✅ Mit CSV | ✅ Ja | ✅ Ja | ✅ Ja |
| **Einrichtung** | Einfach | Einfach | Mittel | Komplex |
| **Redundanz** | Manuell / S2D | File-Server HA | Multipath (MPIO) | Zoning, MPIO |
| **Protokoll-Overhead** | Keiner | Minimal | Gering | Keiner |
| **Empfehlung** | Einzelner Host, S2D | SMB-Cluster, NAS | SAN-Umgebungen | Enterprise SAN |

---

## Storage QoS

Storage Quality of Service schützt kritische VMs vor "noisy neighbours" — VMs, die den gemeinsamen Speicher mit exzessiven I/Os monopolisieren. Policies definieren IOPS-Grenzen pro VM oder Disk.

### Policies erstellen und zuweisen

```powershell
# QoS-Policy für Datenbank-VMs (hohe Garantie, hohes Limit)
New-StorageQosPolicy -Name "DatabasePolicy" `
                     -PolicyType Dedicated `
                     -MinimumIops 2000 `
                     -MaximumIops 8000

# QoS-Policy für Standard-VMs
New-StorageQosPolicy -Name "StandardPolicy" `
                     -PolicyType Dedicated `
                     -MinimumIops 500 `
                     -MaximumIops 2000

# Policy-ID für Zuweisung auslesen
$PolicyId = (Get-StorageQosPolicy -Name "DatabasePolicy").PolicyId

# Policy einer VM-Disk zuweisen
Set-VMHardDiskDrive -VMName "DBServer01" `
                    -ControllerType SCSI `
                    -ControllerNumber 0 `
                    -ControllerLocation 0 `
                    -QoSPolicyID $PolicyId

# Alternativ: Direkt beim Erstellen der Disk zuweisen
Add-VMHardDiskDrive -VMName "AppServer01" `
                    -Path "D:\VMs\AppServer01\Data.vhdx" `
                    -QoSPolicyID (Get-StorageQosPolicy -Name "StandardPolicy").PolicyId
```

### Monitoring

```powershell
# Aktuelle IOPS-Flows aller VMs anzeigen
Get-StorageQosFlow | Select-Object VMName, VHDPath, InitiatorIOPS, `
                                    LimitIOPS, ReservationIOPS, `
                                    @{N="ThrottledMs"; E={$_.FilledBucketCount}}

# Flows sortiert nach aktuellen IOPS (Top-Consumer)
Get-StorageQosFlow | Sort-Object InitiatorIOPS -Descending |
    Select-Object -First 10 VMName, VHDPath, InitiatorIOPS, LimitIOPS

# Alle Policies anzeigen
Get-StorageQosPolicy | Select-Object Name, PolicyType, MinimumIops, MaximumIops

# Policy einer VM prüfen
Get-VMHardDiskDrive -VMName "DBServer01" | Select-Object VMName, Path, QoSPolicyID
```

> [!TIP] Aggregated vs. Dedicated Policies
> - **Dedicated**: Jede VHDX, der die Policy zugewiesen ist, bekommt individuell Min/Max IOPS
> - **Aggregated**: Alle VHDXs mit dieser Policy teilen sich gemeinsam die definierten Min/Max IOPS — nützlich für Multi-Disk-VMs, bei denen das Gesamt-IOPS-Budget begrenzt werden soll

> [!INFO] Storage QoS Voraussetzungen
> Storage QoS für mehrere Hyper-V Hosts erfordert einen **Scale-Out File Server** als zentralen Storage-Dienst. Lokal auf einem einzelnen Host funktioniert QoS ohne zusätzliche Infrastruktur.

---

## PowerShell Kurzreferenz

| Cmdlet | Beschreibung |
|---|---|
| `New-VHD` | Neue VHDX/VHD erstellen (Fixed, Dynamic, Differencing) |
| `Get-VHD` | Informationen zu einer VHDX abrufen |
| `Resize-VHD` | VHDX vergrößern (Checkpoints vorher entfernen) |
| `Optimize-VHD` | Nullblöcke vor Compact-Operation entfernen |
| `Compact-VHD` | Dynamic VHDX physisch verkleinern |
| `Convert-VHD` | Format oder Typ konvertieren (Dynamic↔Fixed, VHD→VHDX) |
| `Mount-VHD` | VHDX auf dem Host einhängen (auch ReadOnly) |
| `Dismount-VHD` | VHDX vom Host aushängen |
| `Add-VMHardDiskDrive` | Disk an VM anhängen (VHDX oder Pass-through) |
| `Remove-VMHardDiskDrive` | Disk von VM trennen |
| `Get-VMHardDiskDrive` | Disks einer VM anzeigen |
| `Add-VMSharedVHD` | Shared VHDX an VM anhängen |
| `Get-VMSharedVHD` | Shared VHDXs einer VM anzeigen |
| `Set-VMHost` | Standard-Pfade für VHDs und VMs setzen |
| `New-StorageQosPolicy` | QoS-Policy erstellen |
| `Get-StorageQosPolicy` | QoS-Policies anzeigen |
| `Set-VMHardDiskDrive` | Disk-Eigenschaften ändern (inkl. QoS-Policy) |
| `Get-StorageQosFlow` | Aktuelle IOPS-Flows überwachen |
| `Enable-ClusterStorageSpacesDirect` | S2D auf Cluster aktivieren |
| `New-StoragePool` | Storage-Pool aus physischen Disks erstellen |
| `New-VirtualDisk` | Virtuelles Volume im Pool erstellen |
| `New-IscsiTargetPortal` | iSCSI Discovery-Portal hinzufügen |
| `Connect-IscsiTarget` | Verbindung zu iSCSI-Target aufbauen |
| `Get-IscsiSession` | Aktive iSCSI-Sessions anzeigen |
| `Set-Disk` | Disk online/offline/readonly schalten |

---

## Verwandte Notizen

- [[Hyper-V VM Verwaltung]]
- [[VM Generationen und Konfigurationsversionen]]
- [[Hyper-V Best Practices]]