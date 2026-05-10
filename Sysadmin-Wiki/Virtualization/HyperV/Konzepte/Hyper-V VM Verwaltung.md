---
title: Hyper-V VM Verwaltung
description: Vollständige Referenz zur Erstellung, Konfiguration und Verwaltung von Hyper-V VMs — Dynamic Memory, Checkpoints, Disk-Erweiterung, Export/Import, Templates und Lifecycle-Operationen
tags:
  - virtualization
  - hyper-v
  - vm
  - verwaltung
  - checkpoints
  - vhdx
date: 2025-01-30
status: published
---

# Hyper-V VM Verwaltung

> [!ABSTRACT] TL;DR
> Eine Hyper-V VM durchläuft ihren Lebenszyklus von der Erstellung über die laufende Verwaltung bis hin zu Export und Archivierung. Kernthemen sind die Wahl zwischen GUI und PowerShell, der korrekte Einsatz von **Dynamic Memory**, das sichere Arbeiten mit **Checkpoints (Snapshots)**, das Erweitern von VHDX-Disks im laufenden Betrieb sowie das Klonen von VMs über **Templates und Differencing Disks**.

---

## VM erstellen

### GUI — Schritt für Schritt (Hyper-V Manager)

Der **Assistent für neue virtuelle Maschinen** führt durch alle Pflichtschritte. Zu öffnen über: `Hyper-V Manager → Aktion → Neu → Virtuelle Maschine…`

| Schritt | Einstellung | Empfehlung |
|--------:|-------------|------------|
| **1** | Name und Pfad | Aussagekräftiger Name, Pfad auf dedizierten VM-Datenträger |
| **2** | Generation wählen | **Generation 2** für alle modernen Windows/Linux-Gäste (UEFI, Secure Boot) |
| **3** | Startspeicher | Initialen RAM festlegen; Dynamic Memory separat konfigurieren |
| **4** | Netzwerkkonfiguration | Vorhandenen vSwitch auswählen oder vorerst „Nicht verbunden" |
| **5** | Virtuelle Festplatte | Neue VHDX erstellen — Größe großzügig wählen (dynamisch erweiternd) |
| **6** | Installationsoptionen | ISO-Datei angeben oder später über DVD-Laufwerk mounten |
| **7** | Zusammenfassung | Alle Einstellungen prüfen, Häkchen bei „VM nach Erstellung starten" optional |
| **8** | Fertigstellen | VM wird erstellt und erscheint in der Verwaltungskonsole |

> [!TIP] Nach der Erstellung anpassen
> Der Assistent bietet nur Grundeinstellungen. Prozessoranzahl, Secure Boot, Integration Services und erweiterte Netzwerkoptionen werden **nach** der Erstellung in den VM-Einstellungen konfiguriert.

---

### PowerShell — Vollständige VM-Erstellung

```powershell
# 1. Neue VM anlegen (Generation 2, 4 GB RAM, neue VHDX)
New-VM `
    -Name "SRV-WEB01" `
    -Generation 2 `
    -MemoryStartupBytes 4GB `
    -NewVHDPath "D:\VMs\SRV-WEB01\SRV-WEB01-OS.vhdx" `
    -NewVHDSizeBytes 80GB `
    -SwitchName "vSwitch-External"

# 2. Prozessoren konfigurieren
Set-VMProcessor "SRV-WEB01" `
    -Count 4 `
    -Reserve 10 `
    -Maximum 100 `
    -RelativeWeight 100

# 3. DVD-Laufwerk hinzufügen und ISO mounten
Add-VMDvdDrive -VMName "SRV-WEB01" -Path "E:\ISOs\Windows_Server_2022.iso"

# 4. Boot-Reihenfolge setzen (Generation 2: DVD zuerst)
$dvd = Get-VMDvdDrive -VMName "SRV-WEB01"
Set-VMFirmware "SRV-WEB01" -FirstBootDevice $dvd

# 5. Secure Boot für Linux anpassen (Microsoft UEFI CA)
Set-VMFirmware "SRV-WEB01" `
    -SecureBootTemplate "MicrosoftUEFICertificateAuthority"

# 6. VM starten
Start-VM -Name "SRV-WEB01"

# 7. Konsole öffnen
VMConnect.exe localhost "SRV-WEB01"
```

> [!NOTE] Secure Boot und Linux
> Bei Linux-Gästen muss die Secure-Boot-Vorlage auf **MicrosoftUEFICertificateAuthority** geändert werden. Mit der Standard-Vorlage `MicrosoftWindows` bootet Linux nicht.

---

## Dynamischer Arbeitsspeicher (Dynamic Memory)

Dynamic Memory erlaubt Hyper-V, den RAM einer laufenden VM innerhalb definierter Grenzen automatisch anzupassen — nach oben wenn die VM mehr benötigt, nach unten wenn andere VMs RAM brauchen.

### Parameter-Tabelle

| Parameter | PowerShell-Param | Beschreibung | Typischer Wert |
|-----------|-----------------|--------------|---------------|
| **Startspeicher** | `-StartupBytes` | RAM beim VM-Start | = Minimalspeicher |
| **Minimalspeicher** | `-MinimumBytes` | Untergrenze — nie unterschritten | 512 MB – 2 GB |
| **Maximalspeicher** | `-MaximumBytes` | Obergrenze — nie überschritten | 4× Startspeicher |
| **Speicherpuffer** | `-Buffer` | % freier RAM, den Hyper-V vorhalten soll | 20 % |
| **Gewichtung** | `-Priority` | Priorität bei RAM-Engpass (0–100) | 50 (normal) |

### Wann aktivieren / deaktivieren

| Szenario | Empfehlung | Begründung |
|----------|-----------|------------|
| Web-/App-Server (variante Last) | ✅ Aktivieren | RAM-Nutzung schwankt stark, Einsparung möglich |
| Entwicklungs-/Test-VMs | ✅ Aktivieren | Viele VMs, geringe Gleichzeitigkeit |
| Datenbankserver (SQL, Oracle) | ❌ Deaktivieren | DB-Engines reservieren RAM vorab, Dynamic Memory kollidiert |
| In-Memory-Cache (Redis, Memcached) | ❌ Deaktivieren | Cache-Größe muss stabil bleiben |
| VMs mit eigenem Memory-Ballooning | ❌ Deaktivieren | Doppeltes Ballooning führt zu unvorhersehbarem Verhalten |
| Produktions-VMs mit fester SLA | ⚠️ Prüfen | Feste Zuweisung vermeidet Latenz-Spitzen durch RAM-Anpassung |

> [!DANGER] SQL Server und Dynamic Memory
> Microsoft empfiehlt ausdrücklich, Dynamic Memory für SQL Server VMs zu **deaktivieren**. SQL Server verwaltet seinen Buffer Pool eigenständig und reagiert nicht korrekt auf RAM-Entzug durch den Hypervisor. Die Folge können schwere Performance-Einbrüche, Query-Timeouts und im schlimmsten Fall Datenbankfehler sein. Stattdessen: **fixen RAM** zuweisen und den `max server memory`-Parameter in SQL Server entsprechend setzen.

### PowerShell — Dynamic Memory konfigurieren

```powershell
# Dynamic Memory aktivieren und Grenzen setzen
Set-VMMemory "SRV-WEB01" `
    -DynamicMemoryEnabled $true `
    -StartupBytes 2GB `
    -MinimumBytes 512MB `
    -MaximumBytes 8GB `
    -Buffer 20 `
    -Priority 50

# Dynamic Memory deaktivieren (fixer RAM)
Set-VMMemory "SRV-DB01" `
    -DynamicMemoryEnabled $false `
    -StartupBytes 16GB

# Aktuellen Speicherstatus prüfen
Get-VMMemory -VMName "SRV-WEB01" | Format-List *

# RAM aller VMs im Überblick
Get-VM | Select-Object Name,
    @{N="RAM (GB)"; E={[math]::Round($_.MemoryAssigned/1GB,1)}},
    @{N="DynMemory"; E={$_.DynamicMemoryEnabled}},
    @{N="Min (MB)"; E={$_.MemoryMinimum/1MB}},
    @{N="Max (GB)"; E={$_.MemoryMaximum/1GB}}
```

---

## Checkpoints (Snapshots)

Ein Checkpoint ist ein gespeicherter Zustand einer VM zu einem bestimmten Zeitpunkt. Er besteht aus einer **AVHDX-Datei** (Delta-Disk), die Änderungen gegenüber dem Parent aufzeichnet, sowie einer Konfigurationsdatei.

### Standard-Checkpoint vs. Produktions-Checkpoint

| Merkmal | Standard-Checkpoint | Produktions-Checkpoint |
|---------|--------------------|-----------------------|
| **Speichert RAM-Inhalt** | ✅ Ja | ❌ Nein |
| **Wiederherstellung** | Exakt zum Zeitpunkt (inkl. laufender Prozesse) | Sauberer Boot-Zustand |
| **Anwendungskonsistenz** | ❌ Nein (crash-consistent) | ✅ Ja (via VSS / fsfreeze) |
| **Geeignet für** | Test/Entwicklung, schnelle Sicherung vor Änderungen | Produktionsserver, Datenbanken |
| **Gast-Betriebssystem** | Beliebig | Windows (VSS) oder Linux (fsfreeze) |
| **Standardeinstellung** | Hyper-V wählt automatisch | Muss explizit gesetzt werden |

> [!NOTE] Automatische Auswahl
> Ist der Checkpoint-Typ auf **Produktion** gesetzt, fällt Hyper-V automatisch auf einen Standard-Checkpoint zurück, wenn der Gast VSS/fsfreeze nicht unterstützt — z. B. bei älteren Linux-Systemen ohne Hyper-V Integration Services.

### AVHDX-Deltaketten-Diagramm

```
Checkpoint-Kette (vereinfacht):

  ┌─────────────────────────────────┐
  │  SRV-WEB01-OS.vhdx              │  ← Basis-Disk (Parent, Read-Only
  │  [Alle Daten bis CP1]           │    sobald Checkpoint existiert)
  └────────────┬────────────────────┘
               │ referenziert
  ┌────────────▼────────────────────┐
  │  SRV-WEB01-OS_CP1.avhdx         │  ← Delta nach Checkpoint 1
  │  [Änderungen seit CP1]          │    (nur geänderte Blöcke)
  └────────────┬────────────────────┘
               │ referenziert
  ┌────────────▼────────────────────┐
  │  SRV-WEB01-OS_CP2.avhdx         │  ← Delta nach Checkpoint 2
  │  [Änderungen seit CP2]          │    (aktiver Schreibziel)
  └─────────────────────────────────┘

Beim Lesen: Hyper-V mergt alle Deltas on-the-fly.
Beim Löschen: Merge-Vorgang schreibt Deltas in Parent zurück.
```

> [!WARNING] Lange Ketten vermeiden
> Deltaketten mit mehr als 3–4 Ebenen senken die I/O-Performance spürbar, da Hyper-V bei jedem Lesezugriff alle Ebenen durchsucht. Checkpoints regelmäßig zusammenführen (Merge) oder löschen.

### PowerShell — Checkpoint-Verwaltung

```powershell
# Checkpoint erstellen
Checkpoint-VM -Name "SRV-WEB01" -SnapshotName "Vor Update KB5034441"

# Produktions-Checkpoint erzwingen
Checkpoint-VM -Name "SRV-DB01" -SnapshotName "Vor Datenbankupgrade" `
    -SnapshotType Production

# Alle Checkpoints einer VM auflisten
Get-VMCheckpoint -VMName "SRV-WEB01" | Format-Table Name, CreationTime, ParentCheckpointName

# Checkpoint wiederherstellen (VM muss ausgeschaltet sein)
$cp = Get-VMCheckpoint -VMName "SRV-WEB01" -Name "Vor Update KB5034441"
Restore-VMCheckpoint -VMCheckpoint $cp

# Checkpoint löschen (löst Merge aus — kann dauern!)
Remove-VMCheckpoint -VMName "SRV-WEB01" -Name "Vor Update KB5034441"

# Alle Checkpoints einer VM löschen
Get-VMCheckpoint -VMName "SRV-WEB01" | Remove-VMCheckpoint

# Standard-Checkpoint-Typ für eine VM festlegen
Set-VM -Name "SRV-WEB01" -CheckpointType Production
Set-VM -Name "SRV-WEB01" -CheckpointType Standard
Set-VM -Name "SRV-WEB01" -CheckpointType Disabled   # Checkpoints komplett sperren
Set-VM -Name "SRV-WEB01" -CheckpointType ProductionOnly  # Kein Fallback auf Standard

# Checkpoint-Typ aller VMs anzeigen
Get-VM | Select-Object Name, CheckpointType
```

> [!DANGER] Checkpoints sind kein Backup-Ersatz
> Ein Checkpoint ist an die originale VHDX-Datei gebunden. Geht die Basis-Disk verloren (Festplattenausfall, versehentliches Löschen), sind **alle Checkpoints wertlos**. Checkpoints schützen nicht vor Hardwareausfall, Dateikorruption oder Ransomware. Für echte Datensicherheit ist ein vollständiges Backup (z. B. Windows Server Backup, Veeam, Azure Backup) unerlässlich.

---

## Disk erweitern

Das Erweitern einer VHDX-Disk ist ein **Zwei-Schritt-Prozess**: Zuerst wird die virtuelle Festplatte auf Hypervisor-Ebene vergrößert, dann muss das Gast-Betriebssystem den neuen Speicherplatz nutzen.

> [!WARNING] Checkpoints vor der Erweiterung entfernen
> Eine VHDX mit aktiven Checkpoints (AVHDX-Kette) kann nicht direkt erweitert werden — `Resize-VHD` schlägt mit einem Fehler fehl. Alle Checkpoints müssen zuerst gelöscht und der Merge-Vorgang abgeschlossen sein, bevor die Disk erweitert werden kann.

### Schritt 1 — VHDX-Datei vergrößern (Hypervisor)

```powershell
# Aktuellen Pfad und Größe der Disk ermitteln
Get-VMHardDiskDrive -VMName "SRV-WEB01" | Select-Object Path
Get-VHD -Path "D:\VMs\SRV-WEB01\SRV-WEB01-OS.vhdx" | Select-Object Path, Size, FileSize

# VHDX auf 120 GB vergrößern (VM kann laufen — Online-Resize)
Resize-VHD -Path "D:\VMs\SRV-WEB01\SRV-WEB01-OS.vhdx" -SizeBytes 120GB

# Ergebnis prüfen
Get-VHD -Path "D:\VMs\SRV-WEB01\SRV-WEB01-OS.vhdx" |
    Select-Object Path,
        @{N="Größe (GB)"; E={[math]::Round($_.Size/1GB,0)}},
        @{N="Genutzt (GB)"; E={[math]::Round($_.FileSize/1GB,1)}}
```

### Schritt 2a — Windows: Partition erweitern (im Gast)

```powershell
# Option A: PowerShell im Gast (empfohlen)
# Partition und maximale Größe ermitteln
$disk = Get-Disk | Where-Object {$_.OperationalStatus -eq "Online"} | Select-Object -First 1
$part = Get-Partition -DiskNumber $disk.Number | Where-Object {$_.Type -eq "Basic"}

# Maximale Größe abfragen
$maxSize = ($part | Get-PartitionSupportedSize).SizeMax

# Partition auf Maximum erweitern
Resize-Partition -DiskNumber $disk.Number -PartitionNumber $part.PartitionNumber -Size $maxSize

# Option B: diskpart (klassisch)
# diskpart
# list disk
# select disk 0
# list partition
# select partition 3
# extend
```

### Schritt 2b — Linux: Partition und Dateisystem erweitern (im Gast)

```bash
# Schritt 1: Neue Disk-Größe dem Kernel mitteilen (ohne Neustart)
echo 1 > /sys/class/block/sda/device/rescan

# Schritt 2: Partition erweitern (growpart aus cloud-utils-growpart)
# Syntax: growpart <Gerät> <Partitionsnummer>
growpart /dev/sda 2

# Schritt 3a: ext4-Dateisystem erweitern (online möglich)
resize2fs /dev/sda2

# Schritt 3b: XFS-Dateisystem erweitern (nur online, Mountpoint angeben)
xfs_growfs /

# Schritt 3c: LVM — Physical Volume und Logical Volume erweitern
pvresize /dev/sda2
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv   # oder xfs_growfs für XFS auf LVM

# Ergebnis prüfen
df -hT
lsblk
```

> [!TIP] Online-Resize unter Linux
> Bei modernen Linux-Distributionen (Kernel 4.x+) mit Hyper-V Integration Services ist ein Neustart für die Disk-Erkennung in der Regel nicht nötig. Der `rescan`-Befehl reicht aus. Bei älteren Systemen ggf. `partprobe /dev/sda` ausführen.

---

## VM exportieren und importieren

Export und Import ermöglichen das Verschieben, Klonen und Archivieren von VMs zwischen Hyper-V-Hosts oder als Offline-Backup.

### VM exportieren

```powershell
# VM in Zielordner exportieren (VM kann laufen — Online-Export)
Export-VM -Name "SRV-WEB01" -Path "E:\Exports"

# Ergebnis: E:\Exports\SRV-WEB01\
#   ├── Virtual Hard Disks\  (VHDX-Dateien)
#   ├── Snapshots\           (Checkpoint-AVHDXs, falls vorhanden)
#   └── SRV-WEB01.vmcx       (Konfigurationsdatei)

# Mehrere VMs gleichzeitig exportieren
Get-VM -Name "SRV-*" | Export-VM -Path "E:\Exports"

# Exportfortschritt anzeigen
Export-VM -Name "SRV-WEB01" -Path "E:\Exports" -AsJob | Wait-Job | Receive-Job
```

> [!NOTE] Export bei laufender VM
> Hyper-V nutzt VSS für den Export laufender VMs. Die exportierte VM ist **anwendungskonsistent**. Checkpoints werden mit exportiert — das kann den Export deutlich verlangsamen und vergrößern.

### Import-Typen

| Import-Typ | PowerShell-Parameter | Verhalten | Neue VM-ID | Geeignet für |
|------------|---------------------|-----------|:----------:|--------------|
| **Register in place** | _(Standard ohne -Copy)_ | VM direkt am Export-Pfad registrieren, keine Dateikopie | ❌ Gleiche ID | Schnelles Reaktivieren am selben Host |
| **Restore** | `-Copy` (ohne `-GenerateNewId`) | Dateien in Standard-VM-Pfade kopieren, gleiche ID behalten | ❌ Gleiche ID | Wiederherstellung auf gleichem Host nach Datenverlust |
| **Copy** | `-Copy -GenerateNewId` | Dateien kopieren + neue eindeutige VM-ID generieren | ✅ Neue ID | Klonen auf beliebigen Host, paralleler Betrieb |

```powershell
# Import-Konfigurationsdatei finden
$vmcx = Get-ChildItem "E:\Exports\SRV-WEB01" -Recurse -Filter "*.vmcx" | Select-Object -First 1

# Typ 1: Register in place (keine Dateikopie)
Import-VM -Path $vmcx.FullName

# Typ 2: Restore (Dateien kopieren, gleiche ID)
Import-VM -Path $vmcx.FullName `
    -Copy `
    -VhdDestinationPath "D:\VMs\SRV-WEB01\Virtual Hard Disks" `
    -VirtualMachinePath "D:\VMs\SRV-WEB01"

# Typ 3: Copy mit neuer ID (Klon-Betrieb auf gleichem Host)
Import-VM -Path $vmcx.FullName `
    -Copy `
    -GenerateNewId `
    -VhdDestinationPath "D:\VMs\SRV-WEB01-KLON\Virtual Hard Disks" `
    -VirtualMachinePath "D:\VMs\SRV-WEB01-KLON"

# Nach dem Import: VM umbenennen und Netzwerk anpassen
Rename-VM -Name "SRV-WEB01" -NewName "SRV-WEB01-KLON"
Get-VMNetworkAdapter -VMName "SRV-WEB01-KLON" | Connect-VMNetworkAdapter -SwitchName "vSwitch-External"
```

---

## VM-Templates und Differencing Disks

Templates ermöglichen das schnelle Klonen vorkonfigurierter VMs. **Differencing Disks** (untergeordnete VHDX-Dateien) teilen sich eine gemeinsame Parent-Disk und speichern nur die individuellen Unterschiede.

### Parent → Differencing Diagramm

```
Template-Konzept mit Differencing Disks:

  ┌──────────────────────────────────────┐
  │  Windows-Server-2022-Template.vhdx   │  ← Parent-Disk
  │  [Sysprepped, Read-Only, ~15 GB]     │    NIEMALS modifizieren!
  └───────┬─────────────┬────────────────┘
          │             │
          │ referenziert│ referenziert
  ┌───────▼──────┐  ┌───▼──────────┐
  │ SRV-WEB01    │  │ SRV-APP01    │   ← Differencing Disks
  │ .avhdx ~2 GB │  │ .avhdx ~3 GB │     (nur Delta zum Parent)
  └──────────────┘  └──────────────┘

Vorteil: 10 VMs teilen eine Parent-Disk → massiv weniger Speicherverbrauch
Nachteil: Parent-Ausfall legt alle abhängigen VMs lahm
```

### Sysprep-Workflow und Template-Erstellung

```powershell
# ── Schritt 1: Basis-VM vorbereiten ──────────────────────────────────────────
# Windows im Gast vollständig installieren und konfigurieren
# Dann Sysprep im Gast ausführen:
# C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown

# ── Schritt 2: Parent-Disk schreibschützen ────────────────────────────────────
# VM ist nach Sysprep ausgeschaltet
# VHDX aus VM entfernen (nicht löschen!) und als schreibgeschützt markieren
$parentPath = "D:\Templates\WS2022-Template.vhdx"
Set-ItemProperty -Path $parentPath -Name IsReadOnly -Value $true

# ── Schritt 3: Differencing Disk für neue VM erstellen ────────────────────────
New-VHD `
    -Path "D:\VMs\SRV-WEB02\SRV-WEB02-OS.avhdx" `
    -ParentPath $parentPath `
    -Differencing

# ── Schritt 4: Neue VM anlegen und Differencing Disk anhängen ─────────────────
New-VM `
    -Name "SRV-WEB02" `
    -Generation 2 `
    -MemoryStartupBytes 4GB `
    -SwitchName "vSwitch-External"

Add-VMHardDiskDrive `
    -VMName "SRV-WEB02" `
    -Path "D:\VMs\SRV-WEB02\SRV-WEB02-OS.avhdx"

# Boot-Reihenfolge anpassen (SCSI-Disk zuerst)
$hd = Get-VMHardDiskDrive -VMName "SRV-WEB02"
Set-VMFirmware "SRV-WEB02" -FirstBootDevice $hd

# ── Schritt 5: VM starten — Windows Out-of-Box-Experience läuft durch ─────────
Start-VM -Name "SRV-WEB02"
```

> [!WARNING] Parent-Disk niemals verändern
> Wird die Parent-Disk nach dem Anlegen von Differencing Disks modifiziert, werden alle davon abhängigen VMs **unwiederbringlich beschädigt**. Die Checksummen der Blöcke stimmen nicht mehr überein. Deshalb: Parent-Disk immer als schreibgeschützt markieren und nur als Basis für neue Differencing Disks nutzen. Für Updates ein neues Template erstellen.

```powershell
# Differencing Disk in eigenständige VHDX konvertieren (Merge)
# Nützlich, wenn eine VM unabhängig vom Parent werden soll
Convert-VHD `
    -Path "D:\VMs\SRV-WEB02\SRV-WEB02-OS.avhdx" `
    -DestinationPath "D:\VMs\SRV-WEB02\SRV-WEB02-OS-merged.vhdx" `
    -VHDType Dynamic

# Anschließend VM auf die neue, unabhängige VHDX umstellen
Remove-VMHardDiskDrive -VMName "SRV-WEB02" -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 0
Add-VMHardDiskDrive -VMName "SRV-WEB02" -Path "D:\VMs\SRV-WEB02\SRV-WEB02-OS-merged.vhdx"
```

---

## Integration Services

Integration Services (auch: Enlightenments) sind Treiber und Dienste, die zwischen Hyper-V-Host und Gast kommunizieren. Sie verbessern Performance, Stabilität und Verwaltbarkeit erheblich.

### Alle 7 Integration Services im Überblick

| Dienst | Name (englisch) | Funktion | Auswirkung bei Deaktivierung |
|--------|----------------|----------|------------------------------|
| **Taktgebersynchronisierung** | Time Synchronization | Hält Gastzeit mit Hostzeit synchron | Zeitdrift im Gast — problematisch für Kerberos, Zertifikate, Logs |
| **Taktgeber (Heartbeat)** | Heartbeat | Host überwacht ob Gast reagiert | Host kann VM-Status nicht feststellen |
| **Schlüsselwertepaare** | Key-Value Pair Exchange (KVP) | Datenaustausch Host ↔ Gast (IP, OS-Info) | Keine Metadaten im Hyper-V Manager sichtbar |
| **Herunterfahren** | Shutdown | Host kann Gast per ACPI sauber herunterfahren | „Shut Down" im Manager funktioniert nicht |
| **VSS** | Volume Shadow Copy | Ermöglicht anwendungskonsistente Sicherungen | Nur crash-konsistente Backups möglich |
| **Gastdienste** | Guest Services | Dateiübertragung Host → Gast ohne Netzwerk | `Copy-VMFile` nicht nutzbar |
| **Dynamischer Arbeitsspeicher** | Dynamic Memory | RAM-Ballooning durch den Host | Dynamic Memory nicht nutzbar |

> [!INFO] Linux und Integration Services
> Ab **Linux-Kernel 4.x** sind die Hyper-V Integration Services direkt im Kernel integriert (`hv_vmbus`, `hv_storvsc`, `hv_netvsc` u. a.). Eine manuelle Installation ist bei modernen Distributionen (Ubuntu 18.04+, RHEL/CentOS 7+, Debian 9+) nicht mehr nötig. Der Status der einzelnen Dienste kann dennoch über PowerShell abgefragt werden.

```powershell
# Alle Integration Services einer VM anzeigen
Get-VMIntegrationService -VMName "SRV-WEB01" |
    Select-Object Name, Enabled, PrimaryOperationalStatus, SecondaryOperationalStatus

# Einzelnen Dienst aktivieren
Enable-VMIntegrationService -VMName "SRV-WEB01" -Name "Guest Service Interface"

# Einzelnen Dienst deaktivieren (z. B. Time Sync, wenn NTP-Server genutzt wird)
Disable-VMIntegrationService -VMName "SRV-WEB01" -Name "Time Synchronization"

# Alle Integration Services für eine VM aktivieren
Get-VMIntegrationService -VMName "SRV-WEB01" | Enable-VMIntegrationService

# Integration Services aller VMs prüfen (Heartbeat-Fokus)
Get-VM | ForEach-Object {
    $hb = Get-VMIntegrationService -VMName $_.Name -Name "Heartbeat"
    [PSCustomObject]@{
        VM       = $_.Name
        Status   = $_.State
        Heartbeat = $hb.PrimaryOperationalStatus
    }
} | Format-Table -AutoSize

# Datei per Guest Services übertragen (ohne Netzwerk)
Copy-VMFile "SRV-WEB01" `
    -SourcePath "C:\Temp\agent-setup.exe" `
    -DestinationPath "C:\Temp\agent-setup.exe" `
    -CreateFullPath `
    -FileSource Host
```

> [!TIP] Time Synchronization und NTP
> Wenn im Gast ein eigener NTP-Client (Windows Time Service mit externem Server, chrony, ntpd) konfiguriert ist, sollte der Hyper-V **Time Synchronization**-Dienst deaktiviert werden. Andernfalls kämpfen beide Dienste gegeneinander und die Uhrzeit schwankt.

---

## VM-Lifecycle-Operationen

Hyper-V bietet mehrere Möglichkeiten, eine VM zu starten, zu stoppen und ihren Zustand zu ändern. Die Unterschiede sind besonders beim Stoppen relevant.

### Zustandsübersicht

| Operation | Hyper-V Manager | PowerShell-Befehl | VM-Status danach | Datenverlust? | Beschreibung |
|-----------|----------------|-------------------|-----------------|:------------:|--------------|
| **Start** | Starten | `Start-VM` | Running | ❌ | VM hochfahren |
| **Shut Down** | Herunterfahren | `Stop-VM` | Off | ❌ | Sauberes OS-Shutdown über ACPI (Integration Services nötig) |
| **Turn Off** | Ausschalten | `Stop-VM -Force` | Off | ⚠️ **Ja** | Sofortiger Stromabbruch — entspricht Stecker ziehen |
| **Pause** | Anhalten | `Suspend-VM` | Paused | ❌ | VM eingefroren, RAM bleibt belegt |
| **Save** | Speichern | `Save-VM` | Saved | ❌ | Hibernate: RAM auf Disk gesichert, dann VM gestoppt |
| **Resume** | Fortsetzen | `Resume-VM` | Running | ❌ | Aus Paused oder Saved-Zustand fortsetzen |
| **Reset** | Zurücksetzen | `Restart-VM -Force` | Running | ⚠️ **Ja** | Hard Reset — wie Stecker ziehen + sofort starten |
| **Checkpoint** | Prüfpunkt | `Checkpoint-VM` | Unverändert | ❌ | Momentaufnahme des aktuellen Zustands |

> [!WARNING] Turn Off vs. Shut Down — kritischer Unterschied
> **Shut Down** (`Stop-VM`) sendet ein ACPI-Signal an den Gast, der dann regulär herunterfährt — Dateisysteme werden ordnungsgemäß ausgehängt, laufende Transaktionen abgeschlossen. **Turn Off** (`Stop-VM -Force`) entspricht dem abrupten Trennen der Stromversorgung. Offene Dateien können beschädigt werden, Datenbanken müssen beim nächsten Start ein Recovery durchführen. Turn Off nur im Notfall verwenden (z. B. wenn der Gast nicht mehr reagiert).

### PowerShell — Alle Lifecycle-Befehle

```powershell
# ── Starten ───────────────────────────────────────────────────────────────────
Start-VM -Name "SRV-WEB01"

# Mehrere VMs starten
Start-VM -Name "SRV-WEB01", "SRV-WEB02", "SRV-DB01"

# Alle gestoppten VMs starten
Get-VM | Where-Object {$_.State -eq "Off"} | Start-VM

# ── Sauber herunterfahren (Shut Down) ────────────────────────────────────────
Stop-VM -Name "SRV-WEB01"

# Mit Timeout: Wenn Gast nicht antwortet, nach 60s abbrechen (kein Force)
Stop-VM -Name "SRV-WEB01" -Force:$false

# ── Sofort ausschalten (Turn Off) ─────────────────────────────────────────────
Stop-VM -Name "SRV-WEB01" -Force   # Datenverlust möglich!

# ── Anhalten und Fortsetzen (Pause/Resume) ────────────────────────────────────
Suspend-VM -Name "SRV-WEB01"
Resume-VM  -Name "SRV-WEB01"

# ── Speichern (Hibernate / Save State) ───────────────────────────────────────
Save-VM -Name "SRV-WEB01"
# Zum Fortsetzen:
Start-VM -Name "SRV-WEB01"

# ── Neustart ──────────────────────────────────────────────────────────────────
# Sauber (über OS-Shutdown-Mechanismus)
Restart-VM -Name "SRV-WEB01"

# Hard Reset (sofort, Datenverlust möglich)
Restart-VM -Name "SRV-WEB01" -Force

# ── Automatisches Startverhalten konfigurieren ────────────────────────────────
# VM beim Hoststart automatisch starten
Set-VM -Name "SRV-WEB01" `
    -AutomaticStartAction Start `
    -AutomaticStartDelay 30        # Sekunden Verzögerung nach Host-Boot

# VM im letzten Zustand wiederherstellen (Running → Start, Paused → Paused)
Set-VM -Name "SRV-WEB01" -AutomaticStartAction StartIfRunning

# VM nie automatisch starten
Set-VM -Name "SRV-WEB01" -AutomaticStartAction Nothing

# Automatisches Stoppverhalten konfigurieren (bei Host-Shutdown)
Set-VM -Name "SRV-WEB01" -AutomaticStopAction ShutDown   # Sauber herunterfahren
Set-VM -Name "SRV-WEB01" -AutomaticStopAction Save       # Zustand speichern
Set-VM -Name "SRV-WEB01" -AutomaticStopAction TurnOff    # Sofort ausschalten

# ── Aktuellen Status aller VMs anzeigen ──────────────────────────────────────
Get-VM | Select-Object Name, State, CPUUsage,
    @{N="RAM (GB)"; E={[math]::Round($_.MemoryAssigned/1GB,1)}},
    Uptime, Status |
    Format-Table -AutoSize
```

### AutomaticStartAction — Optionen im Vergleich

| Option | Verhalten beim Host-Start | Typischer Einsatz |
|--------|--------------------------|-------------------|
| `Nothing` | VM bleibt ausgeschaltet | Test-VMs, manuell verwaltete VMs |
| `Start` | VM startet immer | Produktionsserver, Dienste |
| `StartIfRunning` | Startet nur, wenn VM beim Host-Shutdown lief | Flexible Produktionsumgebungen |

---

## Verwandte Notizen

- [[Was ist Hyper-V]]
- [[VM Generationen und Konfigurationsversionen]]
- [[Hyper-V Storage]]
- [[Hyper-V Netzwerk]]
- [[Hyper-V Best Practices]]
- [[Hyper-V Troubleshooting]]