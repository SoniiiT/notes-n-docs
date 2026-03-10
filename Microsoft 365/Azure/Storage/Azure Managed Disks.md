---
tags:
  - azure
  - storage
  - managed-disks
  - premium-ssd
  - ultra-disk
  - vm-storage
aliases:
  - Azure Disks
  - Managed Disks Azure
  - Azure VM Disk
---

# Azure Managed Disks

> [!ABSTRACT]
> Azure Managed Disks sind verwaltete virtuelle Festplatten für Azure VMs — ohne eigenen Storage Account. Microsoft übernimmt Redundanz, Replikation und Skalierung. Von Standard HDD bis Ultra Disk decken sie alle Workload-Anforderungen von Dev/Test bis SAP HANA ab.

---

## Was sind Managed Disks?

Managed Disks sind **Block-Storage-Volumes**, die an Azure VMs angehängt werden. Im Gegensatz zu unmanaged Disks (Legacy VHDs in Storage Accounts) verwaltet Azure automatisch:

- Redundanz (LRS oder ZRS)
- Speicher-Zuweisung und Isolation
- Snapshot-Management
- Backup-Integration

**Vorteile gegenüber unmanaged Disks:**
- Kein manueller Storage Account nötig
- Einfachere VM-Migration zwischen Availability Zones
- Native Azure Backup und Site Recovery Integration
- ZRS-Unterstützung für zonenredundante Disks

---

## Disk-Typen im Überblick

| Typ | Kürzel | Speicher | Max IOPS | Max Throughput | Latenz | Anwendungsfall | Preis-Tier |
|-----|--------|----------|----------|----------------|--------|----------------|-----------|
| **Standard HDD** | S | HDD | 500 IOPS | 60 MiB/s | ms–Bereich | Dev/Test, Archiv, selten genutzte Daten | Niedrig |
| **Standard SSD** | E | SSD | 6.000 IOPS | 750 MiB/s | < 10 ms | Webserver, Low-Traffic Anwendungen | Niedrig-Mittel |
| **Premium SSD v1** | P | NVMe SSD | 20.000 IOPS | 900 MiB/s | < 5 ms | Datenbanken, ERP, Prod-Workloads | Mittel-Hoch |
| **Premium SSD v2** | — | NVMe SSD | 80.000 IOPS | 1.200 MiB/s | < 2 ms | High-Performance DB, SAP, Oracle | Hoch |
| **Ultra Disk** | — | NVMe SSD | 400.000 IOPS | 10.000 MiB/s | < 1 ms | SAP HANA, HPC, Trading | Sehr hoch |

> [!NOTE]
> Premium SSD v2 und Ultra Disk haben keine festen Größenstufen — IOPS und Throughput werden **unabhängig** von der Größe konfiguriert. Das ermöglicht präzises Right-Sizing.

---

## OS Disk vs. Data Disk vs. Temp Disk

| Disk-Typ | Persistenz | Verschlüsselung | Größe | Beschreibung |
|----------|-----------|-----------------|-------|-------------|
| **OS Disk** | Ja | SSE Standard | Bis 4 TiB | Betriebssystem, Boot-Partition |
| **Data Disk** | Ja | SSE Standard | Bis 32 TiB (Ultra: 64 TiB) | Anwendungsdaten, Datenbanken |
| **Temp Disk** | **Nein** | Nicht verschlüsselt | VM-abhängig | Temporärer Speicher (Swap/Page) |

> [!DANGER]
> Die **Temp Disk** (D: auf Windows, /dev/sdb1 auf Linux) verliert alle Daten bei VM-Neustart, Deallokation oder Host-Migration. Niemals persistente Daten auf der Temp Disk speichern!

---

## Premium SSD v1 — Größen und IOPS

| Disk-Größe | Kürzel | Kapazität | IOPS | Throughput | Preis ($/Monat) |
|-----------|--------|-----------|------|------------|-----------------|
| P4 | — | 32 GiB | 120 | 25 MiB/s | ~4,80 $ |
| P6 | — | 64 GiB | 240 | 50 MiB/s | ~9,60 $ |
| P10 | — | 128 GiB | 500 | 100 MiB/s | ~19,71 $ |
| P20 | — | 512 GiB | 2.300 | 150 MiB/s | ~66,56 $ |
| P30 | — | 1 TiB | 5.000 | 200 MiB/s | ~122,88 $ |
| P40 | — | 2 TiB | 7.500 | 250 MiB/s | ~245,76 $ |
| P50 | — | 4 TiB | 7.500 | 250 MiB/s | ~491,52 $ |
| P60 | — | 8 TiB | 16.000 | 500 MiB/s | ~1.228,80 $ |
| P80 | — | 32 TiB | 20.000 | 900 MiB/s | ~6.553,60 $ |

---

## Disk Bursting

Temporäres Überschreiten der Basis-IOPS:

| Typ | Burst-Methode | Max Burst IOPS | Max Burst Throughput | Anwendungsfall |
|-----|--------------|----------------|----------------------|----------------|
| **Premium SSD v1 (P1–P20)** | Credit-basiert (30 Min) | 3.500 IOPS | 170 MiB/s | Gelegentliche Lastspitzen |
| **Premium SSD v1 (P30+)** | On-Demand Bursting | Doppelte Basis-IOPS | Doppelter Basis-Throughput | Häufigere Peaks |
| **Premium SSD v2** | On-Demand | 80.000 IOPS | 1.200 MiB/s | Konfigurierbar |
| **Ultra Disk** | On-Demand | 400.000 IOPS | 10.000 MiB/s | Maximale Performance |

```powershell
# On-Demand Bursting für Premium SSD v1 (P30+) aktivieren
$disk = Get-AzDisk -ResourceGroupName "myRG" -DiskName "myDisk"
$disk.BurstingEnabled = $true
Update-AzDisk -ResourceGroupName "myRG" -DiskName "myDisk" -Disk $disk
```

---

## Snapshots

```powershell
# Inkrementellen Snapshot erstellen (empfohlen — günstiger)
$diskId = (Get-AzDisk -ResourceGroupName "myRG" -DiskName "myDataDisk").Id

$snapshotConfig = New-AzSnapshotConfig `
    -SourceUri $diskId `
    -Location "westeurope" `
    -CreateOption Copy `
    -Incremental $true

$snapshot = New-AzSnapshot `
    -ResourceGroupName "myRG" `
    -SnapshotName "myDataDisk-snap-$(Get-Date -Format 'yyyyMMdd-HHmm')" `
    -Snapshot $snapshotConfig

Write-Host "Snapshot erstellt: $($snapshot.Id)"

# Neuen Disk aus Snapshot erstellen
$snapshotId = $snapshot.Id
$newDiskConfig = New-AzDiskConfig `
    -SourceResourceId $snapshotId `
    -Location "westeurope" `
    -CreateOption Copy `
    -SkuName "Premium_LRS"

$newDisk = New-AzDisk `
    -ResourceGroupName "myRG" `
    -DiskName "myDataDisk-restored" `
    -Disk $newDiskConfig
```

> [!TIP]
> **Inkrementelle Snapshots** (empfohlen): Speichern nur die Änderungen seit dem letzten Snapshot. Deutlich günstiger als Full Snapshots und unterstützen Cross-Region Copy für DR.

---

## Disk-Verschlüsselung

| Methode | Schlüssel | Wo verschlüsselt | Transparenz |
|---------|-----------|-----------------|------------|
| **SSE-PMK** (Platform Managed Key) | Microsoft | Server-seitig (Azure) | Vollständig automatisch |
| **SSE-CMK** (Customer Managed Key) | Azure Key Vault | Server-seitig (Azure) | Kunde verwaltet Schlüssel |
| **Azure Disk Encryption (ADE)** | Key Vault + Entra ID | Gast-OS (BitLocker/dm-crypt) | In-Guest Verschlüsselung |
| **Double Encryption** | CMK + PMK | Beide Ebenen | Maximale Compliance |

```powershell
# Disk Encryption Set (CMK) erstellen
$keyVault = Get-AzKeyVault -ResourceGroupName "myRG" -VaultName "myKeyVault"
$key = Add-AzKeyVaultKey -VaultName "myKeyVault" -Name "diskEncryptionKey" -Destination "Software"

$desConfig = New-AzDiskEncryptionSetConfig `
    -Location "westeurope" `
    -SourceVaultId $keyVault.ResourceId `
    -KeyUrl $key.Key.Kid `
    -IdentityType SystemAssigned

$des = New-AzDiskEncryptionSet `
    -ResourceGroupName "myRG" `
    -Name "myDiskEncryptionSet" `
    -InputObject $desConfig

# Key Vault Zugriff für DES erlauben
Set-AzKeyVaultAccessPolicy `
    -VaultName "myKeyVault" `
    -ObjectId $des.Identity.PrincipalId `
    -PermissionsToKeys "WrapKey","UnwrapKey","Get"

# Neuen Disk mit CMK erstellen
$diskConfig = New-AzDiskConfig `
    -Location "westeurope" `
    -CreateOption Empty `
    -DiskSizeGB 256 `
    -SkuName "Premium_LRS" `
    -DiskEncryptionSetId $des.Id

New-AzDisk -ResourceGroupName "myRG" -DiskName "myEncryptedDisk" -Disk $diskConfig
```

---

## Shared Disks

Mehrere VMs können denselben Managed Disk gleichzeitig verwenden (für Cluster):

| Disk-Typ | Max Shared VMs | Anwendungsfall |
|----------|----------------|----------------|
| **Premium SSD v1** | 5 VMs | Windows Server FCI, SAP ASCS |
| **Premium SSD v2** | 10 VMs | Größere Cluster |
| **Ultra Disk** | 5 VMs | High-Performance Cluster |

```powershell
# Shared Disk erstellen (maxShares = 2)
$diskConfig = New-AzDiskConfig `
    -Location "westeurope" `
    -CreateOption Empty `
    -DiskSizeGB 1024 `
    -SkuName "Premium_LRS" `
    -MaxSharesCount 2

$sharedDisk = New-AzDisk `
    -ResourceGroupName "myRG" `
    -DiskName "mySharedDisk" `
    -Disk $diskConfig
```

> [!WARNING]
> Shared Disks benötigen **cluster-aware Software** (Windows Server Failover Clustering, Pacemaker). Ohne Cluster-Software entstehen Datenkorruptionen durch simultane Schreibzugriffe.

---

## Performance Tier ändern

Premium SSD v1 ermöglicht temporäres oder dauerhaftes Upgrade des Performance-Tiers:

```powershell
# Performance Tier von P10 auf P30 hochsetzen (ohne Größenänderung)
$disk = Get-AzDisk -ResourceGroupName "myRG" -DiskName "myDisk"
$disk.Tier = "P30"
Update-AzDisk -ResourceGroupName "myRG" -DiskName "myDisk" -Disk $disk
```

> [!NOTE]
> Der Performance Tier kann jederzeit hochgesetzt werden, aber Downgrade ist erst nach 12 Stunden möglich. Nützlich für temporäre Lastspitzen (z. B. Datenmigration, Monatsbericht).

---

## ZRS-Disks

Zone-redundante Managed Disks (verfügbar für Premium SSD v1 und Standard SSD):

```powershell
# ZRS Disk erstellen
$diskConfig = New-AzDiskConfig `
    -Location "westeurope" `
    -CreateOption Empty `
    -DiskSizeGB 512 `
    -SkuName "Premium_ZRS"  # ZRS statt LRS

New-AzDisk -ResourceGroupName "myRG" -DiskName "myZrsDisk" -Disk $diskConfig
```

---

## Azure Backup für Managed Disks

```powershell
# Backup Vault erstellen (neu für Disk Backup)
New-AzDataProtectionBackupVault `
    -ResourceGroupName "myRG" `
    -VaultName "myBackupVault" `
    -Location "westeurope" `
    -StorageSetting @{
        DatastoreType = "VaultStore"
        Type = "LocallyRedundant"
    }

# Backup Policy für Managed Disks erstellen
$schedule = New-AzDataProtectionPolicyTriggerScheduleClientObject -ScheduleExpression "R/2024-01-01T02:00:00+00:00/P1D"
$retentionRule = New-AzDataProtectionRetentionLifeCycleClientObject -SourceDataStore "OperationalStore" -SourceRetentionDurationType "Days" -SourceRetentionDurationCount 30

# Disk Backup aktivieren
Initialize-AzDataProtectionBackupInstance `
    -DatasourceType "AzureDisk" `
    -DatasourceLocation "westeurope" `
    -DatasourceId (Get-AzDisk -ResourceGroupName "myRG" -DiskName "myDisk").Id `
    -PolicyId $policy.Id
```

---

## Disk Export und Import

```powershell
# SAS-URL für Disk-Download generieren (Export)
$sasConfig = New-AzDiskGrantAccessConfig -DurationInSecond 3600 -Access "Read"
$sas = Grant-AzDiskAccess `
    -ResourceGroupName "myRG" `
    -DiskName "myDisk" `
    -Access $sasConfig

# VHD herunterladen (AzCopy mit SAS)
azcopy copy $sas.AccessSAS "/lokaler/pfad/disk.vhd"

# Zugriff widerrufen
Revoke-AzDiskAccess -ResourceGroupName "myRG" -DiskName "myDisk"

# On-Prem VHD hochladen (Import)
$diskConfig = New-AzDiskConfig `
    -Location "westeurope" `
    -CreateOption Upload `
    -DiskSizeGB 128 `
    -SkuName "Standard_LRS" `
    -UploadSizeBytes 137438953472  # 128 GiB in Bytes + 512 für Footer

$disk = New-AzDisk -ResourceGroupName "myRG" -DiskName "importedDisk" -Disk $diskConfig
$uploadSas = Grant-AzDiskAccess -ResourceGroupName "myRG" -DiskName "importedDisk" -Access "Write" -DurationInSecond 86400

azcopy copy "/lokaler/pfad/disk.vhd" $uploadSas.AccessSAS --blob-type PageBlob
Revoke-AzDiskAccess -ResourceGroupName "myRG" -DiskName "importedDisk"
```

---

## Bicep-Beispiel: Data Disk an VM anhängen

```bicep
param location string = resourceGroup().location
param vmName string
param dataDiskName string = '${vmName}-datadisk-01'

resource dataDisk 'Microsoft.Compute/disks@2023-10-02' = {
  name: dataDiskName
  location: location
  sku: {
    name: 'Premium_LRS'
  }
  properties: {
    diskSizeGB: 512
    creationData: {
      createOption: 'Empty'
    }
    tier: 'P20'
    burstingEnabled: true
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2023-09-01' existing = {
  name: vmName
}

resource vmDataDiskAttachment 'Microsoft.Compute/virtualMachines@2023-09-01' = {
  name: vmName
  location: location
  properties: {
    storageProfile: {
      dataDisks: [
        {
          lun: 0
          name: dataDisk.name
          createOption: 'Attach'
          managedDisk: {
            id: dataDisk.id
          }
          caching: 'ReadOnly'    // Empfohlen für Datenbank-Daten
        }
      ]
    }
  }
}

output dataDiskId string = dataDisk.id
```

---

## Verwandte Notizen

- [[Azure Storage Grundlagen]]
- [[Azure Virtual Machines]]
- [[Azure NetApp Files]]
- [[Azure RBAC]]
