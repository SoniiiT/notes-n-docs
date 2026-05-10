---
tags:
  - virtualization
  - hyper-v
  - vm-generation
  - konfigurationsversion
---

# VM Generationen und Konfigurationsversionen

> [!ABSTRACT] TL;DR
> Hyper-V kennt zwei VM-Generationen (Gen1 = Legacy-BIOS, Gen2 = UEFI) und separate Konfigurationsversionen (5.0–12.0), die das Speicherformat der VM-Metadaten bestimmen. Die Generation ist nach der VM-Erstellung **permanent unveränderlich**. Die Konfigurationsversion kann upgradet, aber **nicht downgegradet** werden.

---

## VM-Generationen — Überblick

Jede Hyper-V-VM wird bei ihrer Erstellung einer Generation zugewiesen. Diese Wahl legt den emulierte Hardware-Typ fest und ist danach nicht mehr änderbar.

| Merkmal | Generation 1 | Generation 2 |
|---|---|---|
| **Firmware** | BIOS (Legacy) | UEFI |
| **Secure Boot** | ✗ Nicht verfügbar | ✓ Unterstützt |
| **Boot-Geräte** | IDE-Disk, Legacy-NIC, DVD | SCSI-Disk, Synthetic NIC, PXE über Synthetic NIC |
| **Netzwerkkarte** | Legacy NIC (emuliert) + Synthetic NIC | Nur Synthetic NIC (VMBus) |
| **Festplatten-Controller** | IDE (Boot) + SCSI (Daten) | SCSI (Boot + Daten) |
| **32-bit-Gastbetriebssysteme** | ✓ Unterstützt | ✗ Nicht unterstützt |
| **Linux ohne UEFI-Support** | ✓ Funktioniert | ✗ Nicht kompatibel |
| **PXE-Boot via Legacy NIC** | ✓ Möglich | ✗ Kein Legacy NIC |
| **PXE-Boot via Synthetic NIC** | ✗ Nicht möglich | ✓ Möglich |
| **Hot-Add SCSI-Disk** | ✗ Nur SCSI-Daten-Disks im Betrieb | ✓ Auch SCSI-Boot-Disk (mit Einschränkungen) |
| **vTPM** | ✗ Nicht verfügbar | ✓ Unterstützt |
| **Shielded VM** | ✗ Nicht möglich | ✓ Möglich |
| **Boot-Geschwindigkeit** | Langsamer (BIOS-POST) | Schneller (UEFI-Direktstart) |
| **RAM-Maximum** | 1 TB | 12 TB |
| **vCPU-Maximum** | 64 | 240 |

> [!DANGER] Generation ist permanent — kein Wechsel möglich!
> Die VM-Generation kann nach der Erstellung **unter keinen Umständen geändert werden**. Es gibt keinen PowerShell-Befehl, kein Registry-Hack und keine GUI-Option, um nachträglich von Gen1 auf Gen2 (oder umgekehrt) zu wechseln.
>
> **Einzige Möglichkeit:** VM neu erstellen und Daten-VHDs anhängen. Das Betriebssystem muss neu installiert oder über einen P2V/V2V-Prozess migriert werden.
>
> → Generationswahl **vor der Erstellung** sorgfältig treffen!

---

## Generation 1 — Details

### Wann ist Gen1 noch die richtige Wahl?

Generation 1 sollte nur noch dann eingesetzt werden, wenn einer der folgenden Punkte zutrifft:

- **32-bit-Gastbetriebssystem** — Gen2 unterstützt ausschließlich 64-bit-Gäste
- **Ältere Linux-Distributionen ohne UEFI-Support** — z. B. CentOS 6, RHEL 6, ältere Debian-Versionen
- **PXE-Boot mit Legacy-NIC-Abhängigkeit** — Deployment-Umgebungen, die zwingend eine emulierte Netzwerkkarte erwarten
- **Sehr alte Windows-Versionen** — Windows XP, Windows Server 2003 (nicht mehr supportet, aber noch in Legacy-Umgebungen)
- **Appliance-VMs mit fester BIOS-Abhängigkeit** — Importierte VMs aus anderen Hypervisoren, die auf BIOS ausgelegt sind

> [!TIP] Im Zweifelsfall Gen2 verwenden
> Alle modernen Betriebssysteme (Windows Server 2012 R2+, Windows 8.1+, Ubuntu 14.04+, RHEL/CentOS 7+, Debian 8+) unterstützen UEFI und laufen stabil auf Gen2. Für neue VMs immer Gen2 bevorzugen.

### Einschränkungen von Gen1

- Boot-Disk muss zwingend an einem **IDE-Controller** hängen (IDE 0, Channel 0 oder 1)
- Maximale IDE-Disk-Größe: 2.040 GB (MBR-Partitionstabellen-Limit)
- Kein vTPM, kein Shielded VM, kein Secure Boot
- Legacy NIC unterstützt kein SR-IOV
- Geringere maximale RAM- und vCPU-Werte gegenüber Gen2

### Typische Gen1-VM erstellen (PowerShell)

```powershell
# Generation 1 VM erstellen
New-VM `
    -Name "Legacy-Server01" `
    -Generation 1 `
    -MemoryStartupBytes 2GB `
    -Path "D:\VMs" `
    -NewVHDPath "D:\VMs\Legacy-Server01\OS.vhdx" `
    -NewVHDSizeBytes 80GB `
    -SwitchName "vSwitch-Production"

# SCSI-Controller für Daten-Disks hinzufügen
Add-VMScsiController -VMName "Legacy-Server01"
Add-VMHardDiskDrive -VMName "Legacy-Server01" -ControllerType SCSI -ControllerNumber 0 `
    -Path "D:\VMs\Legacy-Server01\Data.vhdx"
```

---

## Generation 2 — Details

### Vorteile von Gen2

Generation 2 ist der empfohlene Standard für alle modernen Gäste:

- **Schnelleres Booten** — UEFI überspringt den langsamen BIOS-POST
- **Secure Boot** — Schutz vor Bootkit-Malware, konfigurierbar per Template
- **SCSI als primärer Controller** — Boot und Daten auf demselben Controller-Typ
- **vTPM** — Ermöglicht BitLocker ohne physisches TPM, Voraussetzung für Shielded VMs
- **Hot-Add von SCSI-Disks** — Daten-Disks können im laufenden Betrieb hinzugefügt werden
- **Größere RAM- und vCPU-Limits** — Bis zu 12 TB RAM und 240 vCPUs
- **PXE-Boot über Synthetic NIC** — Schneller und kompatibler als Legacy-NIC-PXE

### Secure Boot Templates

Gen2-VMs haben Secure Boot standardmäßig **aktiviert**. Das Template bestimmt, welche Signaturen als vertrauenswürdig gelten:

| Template | Verwendung | Enthaltene Zertifikate |
|---|---|---|
| `MicrosoftWindows` | Windows-Gäste | Microsoft Windows Production CA |
| `MicrosoftUEFICertificateAuthority` | Linux-Gäste, Shim-basierte Distros | Microsoft UEFI CA (inkl. Linux-Shim-Signaturen) |
| `OpenSourceShieldedVM` | Shielded Linux | Erweiterter Linux-Support |

> [!WARNING] Secure Boot und Linux
> Für Linux-Gäste (Ubuntu, Debian, RHEL, SUSE etc.) **muss** das Template auf `MicrosoftUEFICertificateAuthority` gesetzt werden — sonst startet die VM nicht. Das Standardtemplate `MicrosoftWindows` lässt Linux-Bootloader nicht durch.

### Linux-Kompatibilität (Gen2)

| Distribution | Gen2-fähig | Secure-Boot-Template | Anmerkung |
|---|---|---|---|
| Ubuntu 14.04+ | ✓ | MicrosoftUEFICertificateAuthority | Ab 14.04 mit UEFI-Shim |
| Debian 8+ | ✓ | MicrosoftUEFICertificateAuthority | Stretch und neuer |
| RHEL / CentOS 7+ | ✓ | MicrosoftUEFICertificateAuthority | CentOS 6 nur Gen1 |
| SUSE SLES 12+ | ✓ | MicrosoftUEFICertificateAuthority | |
| Fedora 18+ | ✓ | MicrosoftUEFICertificateAuthority | |
| CentOS 6 / RHEL 6 | ✗ | — | Kein UEFI-Support → Gen1 |
| Ältere Linux (< 2012) | ✗ | — | Gen1 verwenden |

### Typische Gen2-VM erstellen (PowerShell)

```powershell
# Generation 2 VM erstellen (Windows-Gast)
New-VM `
    -Name "WinServer2022-01" `
    -Generation 2 `
    -MemoryStartupBytes 4GB `
    -Path "D:\VMs" `
    -NewVHDPath "D:\VMs\WinServer2022-01\OS.vhdx" `
    -NewVHDSizeBytes 127GB `
    -SwitchName "vSwitch-Production"

Set-VMFirmware -VMName "WinServer2022-01" `
    -SecureBootTemplate "MicrosoftWindows"

# Generation 2 VM für Linux-Gast (z. B. Ubuntu)
New-VM `
    -Name "Ubuntu-24-01" `
    -Generation 2 `
    -MemoryStartupBytes 2GB `
    -Path "D:\VMs" `
    -NewVHDPath "D:\VMs\Ubuntu-24-01\OS.vhdx" `
    -NewVHDSizeBytes 40GB `
    -SwitchName "vSwitch-Production"

# Secure Boot auf Linux-Template umstellen
Set-VMFirmware -VMName "Ubuntu-24-01" `
    -SecureBootTemplate "MicrosoftUEFICertificateAuthority"

# Aktuelle Firmware-Einstellungen prüfen
Get-VMFirmware -VMName "Ubuntu-24-01" | Select-Object SecureBoot, SecureBootTemplate
```

---

## Konfigurationsversionen

### Was sind Konfigurationsversionen?

Die **Konfigurationsversion** beschreibt das Format, in dem Hyper-V die VM-Metadaten speichert (`*.vmcx`, `*.vmrs`). Die Versionsnummer bestimmt, welche Features im Management-Format verfügbar sind.

> [!NOTE] Konfigurationsversion ≠ VM-Generation
> Diese beiden Konzepte sind **vollständig unabhängig** voneinander. Eine Gen1-VM kann Konfigurationsversion 10.0 haben. Eine Gen2-VM kann noch auf Version 8.0 sein. Beide Achsen werden getrennt verwaltet.

### Versionsübersicht

| Version | Eingeführt mit | Features / Änderungen |
|---|---|---|
| **5.0** | Windows Server 2012 R2 | Baseline für das VMCX-Format |
| **6.2** | Windows Server 2016 (frühe Builds) | Übergangsversion, selten in Produktion |
| **8.0** | Windows Server 2016 (RTM) | Production Checkpoints, Hot-Add Memory (Gen2) |
| **9.0** | Windows Server 2019 | Verbesserungen bei Checkpoints und vTPM |
| **10.0** | Windows Server 2022 | Nested Virtualization, Hibernate-Support |
| **11.0** | Windows Server 2025 (frühe Builds) | Erweiterte Sicherheitsfeatures |
| **12.0** | Windows Server 2025 (RTM) | Aktuelles Format, neue VM-Konfigurationsoptionen |

```powershell
# Alle VMs mit ihrer aktuellen Konfigurationsversion anzeigen
Get-VM | Select-Object Name, Version, Generation, State | Sort-Object Name
```

---

## Upgrade der Konfigurationsversion

> [!DANGER] Upgrade ist irreversibel — kein Downgrade möglich!
> Nach dem Upgrade kann die VM nicht mehr auf einem älteren Hyper-V-Host gestartet werden.
>
> Beispiel: Von 8.0 auf 10.0 upgraden → VM ist danach **nicht mehr auf Windows Server 2016-Hosts lauffähig**.

> [!WARNING] Cluster: Erst upgraden wenn ALLE Knoten aktualisiert sind!
> In einem Hyper-V-Failovercluster darf die VM-Konfigurationsversion erst upgradet werden, wenn **alle Clusterknoten** auf der neuen Windows-Version laufen.

### Einzelne VM upgraden

```powershell
# VM muss ausgeschaltet sein!
Update-VMVersion -VMName "FileServer-01" -Force

# Version nach dem Upgrade prüfen
Get-VM -VMName "FileServer-01" | Select-Object Name, Version
```

### Mehrere VMs upgraden

```powershell
# Alle ausgeschalteten VMs upgraden
Get-VM | Where-Object { $_.State -eq 'Off' } | Update-VMVersion -Force

# Nur VMs unterhalb einer bestimmten Version upgraden
Get-VM | Where-Object { $_.Version -lt '10.0' -and $_.State -eq 'Off' } |
    Update-VMVersion -Force

# Übersicht: Welche VMs sind noch auf alter Version?
Get-VM | Where-Object { $_.Version -lt '10.0' } |
    Select-Object Name, Version, State | Sort-Object Version
```

### Cluster-Upgrade — Empfohlene Vorgehensweise

```powershell
# 1. Sicherstellen, dass alle Knoten auf neuer OS-Version laufen
Get-ClusterNode | Select-Object Name, State

# 2. Cluster Functional Level upgraden
Get-Cluster | Select-Object Name, ClusterFunctionalLevel
Update-ClusterFunctionalLevel

# 3. VMs nacheinander upgraden
$vmsToUpgrade = Get-VM | Where-Object { $_.Version -lt '10.0' }
foreach ($vm in $vmsToUpgrade) {
    Write-Host "Upgrading $($vm.Name) von Version $($vm.Version)..."
    if ($vm.State -ne 'Off') {
        Stop-VM -VMName $vm.Name -Force
        do { Start-Sleep -Seconds 2 } until ((Get-VM -VMName $vm.Name).State -eq 'Off')
    }
    Update-VMVersion -VMName $vm.Name -Force
    Write-Host "  -> Neue Version: $((Get-VM -VMName $vm.Name).Version)"
}
```

> [!TIP] Nach dem Upgrade sofort testen
> VM starten und grundlegende Funktionstests durchführen, bevor weitere VMs upgradet werden.

---

## Zusammenspiel Generation und Konfigurationsversion

```
VM-Generation (Hardware-Typ)          Konfigurationsversion (Speicher-Format)
─────────────────────────────         ─────────────────────────────────────────
Gen1 = BIOS, IDE-Boot, Legacy NIC     5.0  = WS 2012 R2-Format
Gen2 = UEFI, SCSI-Boot, Secure Boot  10.0 = WS 2022-Format
         │                                        │
         │ permanent unveränderlich               │ upgradebar (einmalig)
         └────────────── unabhängig ──────────────┘
```

| Kombination | Typischer Einsatz |
|---|---|
| Gen1 + Version 5.0 | Sehr alte WS 2012 R2-Umgebungen |
| Gen1 + Version 10.0 | Legacy-App auf aktuellem WS 2022-Host |
| Gen2 + Version 8.0 | WS 2016-VM noch nicht upgradet |
| Gen2 + Version 9.0 | Typischer WS 2019-Produktionsgast |
| Gen2 + Version 10.0 | Empfehlung für neue WS 2022-Deployments |
| Gen2 + Version 12.0 | WS 2025-Umgebungen |

---

## Best Practices

> [!TIP] Entscheidungsbaum — Generation
> ```
> Modernes OS (Win 10+, WS 2016+, Ubuntu 14.04+)? → Generation 2
> 32-bit-OS benötigt?                              → Generation 1 (zwingend)
> Älteres Linux ohne UEFI (CentOS 6, RHEL 6)?      → Generation 1
> Alles andere?                                    → Generation 2
> ```

- Neue VMs **immer mit Gen2** erstellen, sofern das Gastbetriebssystem es unterstützt
- Bei Linux Gen2: Secure Boot auf `MicrosoftUEFICertificateAuthority` setzen
- Konfigurationsversion **erst nach vollständigem Cluster-Upgrade** erhöhen
- Vor jedem Upgrade **Backup oder Checkpoint** erstellen
- `Get-VM | Select Name, Version` ins Monitoring aufnehmen

```powershell
# Monitoring-Report: Generationen und Versionen aller VMs
Get-VM | Select-Object `
    @{N='VM-Name';    E={$_.Name}},
    @{N='Generation'; E={$_.Generation}},
    @{N='Version';    E={$_.Version}},
    @{N='Status';     E={$_.State}} |
    Sort-Object Generation, Version |
    Format-Table -AutoSize
```

---

## Verwandte Notizen

- [[Was ist Hyper-V]]
- [[Hyper-V VM Verwaltung]]
- [[Hyper-V Best Practices]]
