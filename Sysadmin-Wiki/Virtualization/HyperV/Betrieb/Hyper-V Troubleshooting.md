---
tags:
  - hyper-v
  - troubleshooting
  - fehler
  - diagnose
  - logs
aliases:
  - Hyper-V Fehleranalyse
  - Hyper-V Diagnose
---

# Hyper-V Troubleshooting

> [!ABSTRACT] TL;DR
> Strukturierte Fehlerdiagnose für Hyper-V — Event-Viewer-Pfade, kritische Event-IDs, häufige Fehlerbilder mit Ursachen und Lösungen, PowerShell-Diagnosebefehle sowie Analyse von Performance-Problemen (CPU, Memory Pressure, Disk-Latenz).

---

## Log-Dateien & Event Viewer

### Hyper-V Event-Log-Pfade (Event Viewer)

```
Ereignisanzeige → Anwendungs- und Dienstprotokolle
  → Microsoft
    → Windows
      → Hyper-V-Compute          (Laufzeit-Ereignisse, VM-Prozesse)
      → Hyper-V-Config           (Konfigurationsänderungen)
      → Hyper-V-High-Availability (Cluster / Failover-Ereignisse)
      → Hyper-V-Hypervisor       (Hypervisor-Kernel-Ereignisse)
      → Hyper-V-Integration      (Integration Services)
      → Hyper-V-Network-VSP      (Virtueller Switch / Netzwerk)
      → Hyper-V-StorageVSP       (Virtueller Storage)
      → Hyper-V-VMMS             (Virtual Machine Management Service)
      → Hyper-V-Worker           (VM-Worker-Prozesse, pro VM)
```

### Wichtige Event-IDs (Schnellreferenz)

| Event-ID | Quelle | Bedeutung | Priorität |
|:---------|:-------|:----------|:----------|
| **18590** | VMMS | VM erfolgreich gestartet | Info |
| **18592** | VMMS | VM erfolgreich gestoppt | Info |
| **12010** | VMMS | VM konnte nicht gestartet werden | 🔴 Kritisch |
| **13003** | VMMS | Konfigurationsdatei beschädigt | 🔴 Kritisch |
| **3040** | VMMS | Checkpoint-Merge fehlgeschlagen | 🟠 Hoch |
| **3080** | VMMS | Live Migration gestartet | Info |
| **21024** | VMMS | Live Migration fehlgeschlagen | 🔴 Kritisch |
| **32022** | VMMS | Replica: Replikation fehlgeschlagen | 🟠 Hoch |
| **32086** | VMMS | Replica: RPO-Verletzung | 🟠 Hoch |
| **12100** | Worker | VM-Prozess abgestürzt | 🔴 Kritisch |
| **4096** | Integration | Integration Services nicht erreichbar | 🟡 Mittel |
| **14050** | Hypervisor | NUMA-Spanning aktiv (Perf-Warnung) | 🟡 Mittel |

### Event-Logs per PowerShell filtern

```powershell
# VMMS-Log: letzte 50 Fehler und Warnungen
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" |
    Where-Object { $_.LevelDisplayName -in 'Error', 'Warning' } |
    Select-Object TimeCreated, Id, LevelDisplayName, Message -First 50 |
    Format-List

# Alle Hyper-V-Logs auf einmal durchsuchen (nach Event-ID filtern)
$hvLogs = Get-WinEvent -ListLog "Microsoft-Windows-Hyper-V-*" |
    Select-Object -ExpandProperty LogName
Get-WinEvent -LogName $hvLogs -ErrorAction SilentlyContinue |
    Where-Object { $_.Id -eq 12010 } |
    Select-Object TimeCreated, LogName, Message

# Ereignisse einer bestimmten VM finden
$vmName = "MeineVM"
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" |
    Where-Object { $_.Message -like "*$vmName*" } |
    Select-Object TimeCreated, Id, Message -First 20
```

---

## Häufige Fehler und Lösungen

### VM startet nicht

**Symptom:** `Start-VM` gibt Fehler zurück oder VM bleibt im Zustand "Starting" hängen.

#### Ursache A: VHDX-Datei gesperrt

```
Fehlertext: "The process cannot access the file because it is being used by another process."

Diagnose:
```

```powershell
# Prüfen ob eine andere VM oder ein Prozess die VHDX hält
Get-VM | Get-VMHardDiskDrive | Where-Object { $_.VMName -ne "MeineVM" } |
    Where-Object { $_.Path -eq "E:\Hyper-V\VHDs\MeineVM.vhdx" }

# Alternativ: Handle-Tool (Sysinternals) auf dem Host ausführen
handle.exe "MeineVM.vhdx"
```

```
Lösung:
1. Alle VMs, die dieselbe VHDX referenzieren, stoppen
2. VMMS-Dienst neu starten: Restart-Service -Name vmms
3. VM erneut starten
```

#### Ursache B: Speicherplatz voll

```powershell
# Volume-Auslastung prüfen
Get-PSDrive -PSProvider FileSystem | Select-Object Name,
    @{N="Frei (GB)"; E={[math]::Round($_.Free/1GB, 1)}}

# Checkpoints löschen (häufigste Ursache für volles Volume)
Get-VM | Get-VMCheckpoint | Remove-VMCheckpoint -Confirm:$false
```

#### Ursache C: Konfigurationsdatei beschädigt

```powershell
# VM de-registrieren und neu registrieren
$vmcxPath = "D:\Hyper-V\VMs\MeineVM\MeineVM.vmcx"
$vmid = (Get-VM -Name "MeineVM" -ErrorAction SilentlyContinue).Id

# VM aus Inventar entfernen
Remove-VM -Name "MeineVM" -Force -ErrorAction SilentlyContinue

# VM aus VMCX-Datei neu importieren
Import-VM -Path $vmcxPath
```

---

### VM bootet nicht (Betriebssystem startet nicht)

#### Ursache A: Falsches Secure Boot Template (Linux)

```
Symptom: Generation-2-VM mit Linux bleibt bei "No bootable device" stehen.
```

```powershell
# Secure Boot Template auf Microsoft UEFI CA umstellen (für Linux)
Set-VMFirmware -VMName "Ubuntu-VM" `
    -SecureBootTemplate "MicrosoftUEFICertificateAuthority"

# Oder Secure Boot temporär deaktivieren (nur zum Test)
Set-VMFirmware -VMName "Ubuntu-VM" -EnableSecureBoot Off
```

#### Ursache B: Falsche Boot-Reihenfolge

```powershell
# Aktuelle Boot-Reihenfolge anzeigen (nur Generation 2)
Get-VMFirmware -VMName "MeineVM" | Select-Object -ExpandProperty BootOrder

# Boot-Reihenfolge korrigieren: VHDX an erste Stelle
$bootDisk = Get-VMHardDiskDrive -VMName "MeineVM" | Select-Object -First 1
Set-VMFirmware -VMName "MeineVM" -FirstBootDevice $bootDisk
```

---

### Live Migration schlägt fehl

**Symptom:** `Move-VM` gibt Fehler zurück, Event-ID 21024 im VMMS-Log.

#### Ursache A: CPU-Inkompatibilität

```powershell
# CPU-Kompatibilitätsmodus aktivieren (vor der Migration)
Set-VMProcessor -VMName "MeineVM" -CompatibilityForMigrationEnabled $true

# CPU-Features der Hosts vergleichen
Get-VMHost -ComputerName hv01 | Select-Object -ExpandProperty ProcessorCapabilities
Get-VMHost -ComputerName hv02 | Select-Object -ExpandProperty ProcessorCapabilities
```

#### Ursache B: Kerberos-Probleme

```
Diagnose-Checkliste:
1. Beide Hosts im gleichen AD? → Get-ADComputer hv01, hv02
2. SPN korrekt registriert?
   setspn -L hv01
   setspn -L hv02
3. Zeitdifferenz < 5 Minuten?
   w32tm /query /status        (auf beiden Hosts ausführen)
4. Constrained Delegation konfiguriert? (bei Workgroup-Hosts nötig)
```

```powershell
# Live Migration mit Kerberos testen (explizit angeben)
Move-VM -Name "MeineVM" -DestinationHost "hv02" `
    -AuthenticationType Kerberos
```

#### Ursache C: Live Migration-Netzwerk nicht erreichbar

```powershell
# Netzwerkkonnektivität zwischen den Hosts prüfen
Test-NetConnection -ComputerName hv02 -Port 6600   # Live Migration Port
Test-NetConnection -ComputerName hv02 -Port 443    # HTTPS

# Live Migration Netzwerk auf dem Host prüfen
Get-VMHostSupportedVersion | Select-Object Name, Version
netsh advfirewall firewall show rule name="Hyper-V"
```

---

### Replica-Fehler

#### Ursache A: Zertifikat abgelaufen (HTTPS-Replica)

```powershell
# Zertifikat-Ablaufdatum auf beiden Hosts prüfen
Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.Subject -like "*hv*" } |
    Select-Object Subject, NotAfter, Thumbprint

# Replikation auf neues Zertifikat umstellen
Set-VMReplication -VMName "MeineVM" `
    -CertificateThumbprint "NEUER_THUMBPRINT_HIER"
```

#### Ursache B: RPO-Verletzung (Event-ID 32086)

```
Ursachen für RPO-Verletzung:
- Netzwerkbandbreite zwischen Primary und Replica zu gering
- Zu viele Write-I/Os auf dem Primary (Replikation kommt nicht nach)
- Replica-Server überlastet

Diagnose:
```

```powershell
# Replikationsstatistiken anzeigen
Get-VMReplication -VMName "MeineVM" |
    Select-Object VMName, State, Health,
                  LastReplicationTime, AverageReplicationSize,
                  ReplicationFrequency, LastReplicationError

# Netzwerkauslastung während Replikation messen
Get-NetAdapterStatistics -Name "NIC-Replica" |
    Select-Object Name, ReceivedBytes, SentBytes
```

---

### Checkpoint-Probleme

#### Merge hängt nach Checkpoint-Löschen

```
Symptom: VHDX-Volume füllt sich trotz "Checkpoint gelöscht", VM läuft langsam.
         Differenz-Disk (.avhdx) wird nicht zusammengeführt.
```

```powershell
# Merge-Status prüfen
Get-VMHardDiskDrive -VMName "MeineVM" | ForEach-Object {
    Get-VHD -Path $_.Path | Select-Object Path, VhdType, ParentPath, FileSize
}

# Manuellen Merge erzwingen (VM muss ausgeschaltet sein)
Stop-VM -Name "MeineVM" -Force
Merge-VHD -Path "E:\Hyper-V\VHDs\MeineVM_1234.avhdx" `
          -DestinationPath "E:\Hyper-V\VHDs\MeineVM.vhdx"
```

> [!DANGER] Merge nie bei laufender VM erzwingen
> Ein manueller `Merge-VHD` darf nur bei gestoppter VM durchgeführt werden. Bei laufender VM führt ein Merge zur sofortigen VHDX-Korruption.

#### Datastore voll — Checkpoint kann nicht erstellt werden

```powershell
# Freien Speicher prüfen und größte Dateien finden
Get-ChildItem -Path "E:\Hyper-V" -Recurse -File |
    Sort-Object Length -Descending |
    Select-Object FullName, @{N="GB"; E={[math]::Round($_.Length/1GB, 2)}} -First 20
```

---

### Integration Services veraltet

```powershell
# IS-Status aller laufenden VMs
Get-VM | Where-Object { $_.State -eq 'Running' } |
    Get-VMIntegrationService |
    Select-Object VMName, Name, Enabled, PrimaryStatusDescription |
    Where-Object { $_.PrimaryStatusDescription -ne 'OK' }

# IS für eine VM neu starten (ohne VM-Neustart)
Restart-VMIntegrationService -VMName "MeineVM" -Name "Heartbeat"
Restart-VMIntegrationService -VMName "MeineVM" -Name "Guest Service Interface"
```

> [!TIP] Linux IS aktualisieren
> Unter Ubuntu/Debian: `sudo apt update && sudo apt install linux-tools-$(uname -r) linux-cloud-tools-$(uname -r)`
> Unter RHEL/CentOS: `sudo dnf install hyperv-daemons`

---

## Diagnose-Befehle (PowerShell)

### VM-Gesundheit umfassend prüfen

```powershell
function Test-VMHealth {
    param([string]$VMName)

    $vm = Get-VM -Name $VMName
    Write-Host "`n=== VM: $VMName ===" -ForegroundColor Cyan

    # State
    Write-Host "State:      $($vm.State)"
    Write-Host "CPU-Nutzung: $($vm.CPUUsage) %"
    Write-Host "RAM:        $([math]::Round($vm.MemoryAssigned/1GB,1)) GB"
    Write-Host "Uptime:     $($vm.Uptime)"

    # Integration Services
    $is = Get-VMIntegrationService -VMName $VMName
    $isProblems = $is | Where-Object { -not $_.Enabled -or $_.PrimaryStatusDescription -ne 'OK' }
    if ($isProblems) {
        Write-Host "`nIS-Probleme:" -ForegroundColor Yellow
        $isProblems | Select-Object Name, Enabled, PrimaryStatusDescription | Format-Table
    } else {
        Write-Host "Integration Services: OK" -ForegroundColor Green
    }

    # Checkpoints
    $cp = Get-VMCheckpoint -VMName $VMName
    if ($cp) {
        Write-Host "`nCheckpoints ($($cp.Count) gefunden):" -ForegroundColor Yellow
        $cp | Select-Object Name, CreationTime | Format-Table
    }

    # Replikation
    $repl = Get-VMReplication -VMName $VMName -ErrorAction SilentlyContinue
    if ($repl) {
        Write-Host "Replica: $($repl.State) / $($repl.Health)" -ForegroundColor $(
            if ($repl.Health -eq 'Normal') { 'Green' } else { 'Red' }
        )
    }
}

# Aufruf
Test-VMHealth -VMName "MeineVM"
```

### Replica-Status aller VMs

```powershell
# Vollständige Replica-Übersicht
Get-VMReplication | Select-Object `
    VMName,
    @{N="Zustand";       E={$_.State}},
    @{N="Gesundheit";    E={$_.Health}},
    @{N="Letzte Replik"; E={$_.LastReplicationTime}},
    @{N="Letzter Fehler";E={$_.LastReplicationError}},
    FrequencySec |
    Format-Table -AutoSize

# Nur fehlerhafte Replikationen anzeigen
Get-VMReplication |
    Where-Object { $_.Health -ne 'Normal' } |
    Select-Object VMName, State, Health, LastReplicationError | Format-Table -AutoSize
```

---

## Hyper-V Manager verbindet nicht zum Remote-Host

```
Symptom: "Fehler beim Verbinden mit dem Server" in Hyper-V Manager

Checkliste:
1. WinRM aktiv?
   Test-WSMan -ComputerName hv01

2. Firewall-Regeln vorhanden?
   Get-NetFirewallRule -DisplayGroup "Hyper-V" | Select-Object DisplayName, Enabled

3. Kerberos / DNS funktioniert?
   Resolve-DnsName hv01.example.local
   Test-NetConnection hv01 -Port 5985   # WinRM HTTP
   Test-NetConnection hv01 -Port 5986   # WinRM HTTPS

4. Remote-Management im lokalen Netz erlaubt?
   winrm quickconfig
   Set-Item WSMan:\localhost\Client\TrustedHosts -Value "hv01"

5. DCOM-Konfiguration prüfen (für ältere Hyper-V Manager Versionen)
   dcomcnfg → Component Services → Computers → My Computer → Properties
   → COM Security → Edit Limits → Remote Access erlauben
```

```powershell
# Verbindungstest vollständig
$host = "hv01.example.local"
Test-NetConnection $host -Port 135   # RPC Endpoint Mapper
Test-NetConnection $host -Port 445   # SMB
Test-NetConnection $host -Port 5985  # WinRM HTTP
Invoke-Command -ComputerName $host -ScriptBlock { Get-VM } -ErrorAction Stop
```

---

## Performance-Probleme analysieren

### CPU-Engpass analysieren

```powershell
# Aktuellen CPU-Verbrauch aller VMs anzeigen
Get-VM | Where-Object { $_.State -eq 'Running' } |
    Select-Object Name,
        @{N="CPU %"; E={$_.CPUUsage}} |
    Sort-Object "CPU %" -Descending | Format-Table -AutoSize

# Hyper-V Hypervisor Logical Processor Auslastung
Get-Counter "\Hyper-V Hypervisor Logical Processor(_Total)\% Total Run Time" `
    -SampleInterval 2 -MaxSamples 5 |
    Select-Object -ExpandProperty CounterSamples |
    Select-Object Timestamp, CookedValue
```

> [!WARNING] CPU-Überbuchung erkennen
> Wenn `% Total Run Time` dauerhaft über 80 % liegt und VMs träge reagieren, ist der Host CPU-überlastet. Lösung: VMs auf anderen Host migrieren oder vCPU-Anzahl der VMs reduzieren (NUMA-Grenzen beachten).

### Memory Pressure analysieren

```powershell
# Memory Pressure des Hosts (Dynamic Memory Balancer)
Get-Counter "\Hyper-V Dynamic Memory Balancer(*)\Average Pressure" `
    -SampleInterval 5 -MaxSamples 6 |
    Select-Object -ExpandProperty CounterSamples |
    Select-Object InstanceName, CookedValue

# RAM-Zuweisung vs. Bedarf aller VMs
Get-VM | Where-Object { $_.State -eq 'Running' } |
    Select-Object Name,
        @{N="Zugewiesen (MB)"; E={[math]::Round($_.MemoryAssigned/1MB)}},
        @{N="Bedarf (MB)";     E={[math]::Round($_.MemoryDemand/1MB)}},
        @{N="Druck (%)";       E={
            if ($_.MemoryAssigned -gt 0) {
                [math]::Round($_.MemoryDemand / $_.MemoryAssigned * 100, 1)
            }
        }} |
    Sort-Object "Druck (%)" -Descending | Format-Table -AutoSize
```

> [!DANGER] Memory Pressure > 100 %
> Ein Memory Pressure-Wert über 100 % bedeutet, dass der Host aktiv Ballooning betreibt — Gast-OS gibt RAM zurück, obwohl er ihn braucht. Performance-Einbrüche sind die Folge. Sofortmaßnahme: RAM einbauen oder VMs migrieren.

### Disk-Latenz analysieren

```powershell
# VHDX Read/Write-Latenz überwachen
Get-Counter `
    "\Hyper-V Virtual Storage Device(*)\Read Bytes/sec",
    "\Hyper-V Virtual Storage Device(*)\Write Bytes/sec",
    "\Hyper-V Virtual Storage Device(*)\Read Operations/sec",
    "\Hyper-V Virtual Storage Device(*)\Write Operations/sec" `
    -SampleInterval 2 -MaxSamples 10 |
    Select-Object -ExpandProperty CounterSamples |
    Select-Object InstanceName, Path, CookedValue |
    Format-Table -AutoSize

# Physische Disk-Latenz des Hosts
Get-Counter "\PhysicalDisk(*)\Avg. Disk sec/Read",
            "\PhysicalDisk(*)\Avg. Disk sec/Write" `
    -SampleInterval 2 -MaxSamples 5 |
    Select-Object -ExpandProperty CounterSamples |
    Select-Object InstanceName, CookedValue |
    Format-Table -AutoSize
```

| Latenz-Richtwert | Bewertung |
|:-----------------|:----------|
| < 5 ms | ✅ Gut |
| 5–20 ms | 🟡 Akzeptabel |
| 20–50 ms | 🟠 Kritisch — Ursache suchen |
| > 50 ms | 🔴 Inakzeptabel — sofort handeln |

---

## Verwandte Notizen

- [[Was ist Hyper-V]]
- [[Hyper-V VM Verwaltung]]
- [[Hyper-V Best Practices]]
- [[Hyper-V PowerShell Cheatsheet]]
- [[Installation Hyper-V (Windows Server 2022)]]