---
tags:
  - virtualization
  - esxi
  - vsphere
  - vmware
  - best-practices
  - betrieb
  - sicherheit
---

# ESXi Best Practices

> [!ABSTRACT] TL;DR
> Bewährte Konfigurationsempfehlungen für professionell betriebene ESXi/vSphere-Umgebungen — von Host-Härtung und VM-Konfiguration über Storage, Netzwerk und Sicherheit bis zu Updates, Backup und Performance-Monitoring. Gültig für ESXi 7.x und 8.x, skaliert von KMU bis Enterprise.

---

## 1. Host-Konfiguration

### NTP — Zeitsynchronisation

Eine saubere Zeitsynchronisation ist Grundvoraussetzung für Logging, Kerberos-Auth und vMotion.

```bash
# NTP-Status auf einem Host prüfen (ESXi Shell)
esxcli system time get
esxcli network ntp get
```

| Einstellung | Empfehlung |
|---|---|
| NTP-Quelle | vCenter als primäre Quelle, externe NTP-Server als Fallback |
| NTP-Dienst | Startrichtlinie auf **Start and stop with host** setzen |
| Maximale Abweichung | < 1 Sekunde zwischen allen Hosts im Cluster |
| Konfigurationsweg | Host-Profil oder PowerCLI für Konsistenz über alle Hosts |

```powershell
# NTP-Konfiguration per PowerCLI auf alle Hosts ausrollen
Get-VMHost | Add-VMHostNtpServer -NtpServer "ntp.example.local"
Get-VMHost | Get-VMHostService | Where-Object {$_.Key -eq "ntpd"} |
    Set-VMHostService -Policy On | Start-VMHostService
```

### Syslog — Zentrales Logging

> [!IMPORTANT] Syslog immer extern weiterleiten
> Ohne zentralen Syslog-Server gehen Host-Logs bei einem Ausfall unwiederbringlich verloren. Logs auf dem Host selbst sind zu klein und werden überschrieben.

```bash
# Syslog-Ziel setzen (ESXi Shell)
esxcli system syslog config set --loghost="udp://syslog.example.local:514"
esxcli system syslog reload
```

### SSH und ESXi Shell

> [!WARNING] SSH und Shell niemals dauerhaft aktiviert lassen
> Beide Dienste stellen eine potenzielle Angriffsfläche dar. Sie sollten nur für aktive Wartungsarbeiten aktiviert und danach sofort wieder deaktiviert werden.

| Service | Empfehlung | Timeout |
|---|---|---|
| SSH | Nur bei Bedarf aktivieren | 15 Minuten |
| ESXi Shell | Nur bei Bedarf aktivieren | 15 Minuten |
| Startrichtlinie | `Start and stop manually` | — |

```powershell
# SSH- und Shell-Timeout per PowerCLI auf 15 Minuten setzen
Get-VMHost | Get-AdvancedSetting -Name "UserVars.ESXiShellInteractiveTimeOut" |
    Set-AdvancedSetting -Value 900
Get-VMHost | Get-AdvancedSetting -Name "UserVars.ESXiShellTimeOut" |
    Set-AdvancedSetting -Value 900
```

### Lockdown Mode

| Modus | Zugriff erlaubt | Empfehlung |
|---|---|---|
| **Disabled** | Direkte Host-Anmeldung möglich | Nur während Ersteinrichtung |
| **Normal** | Nur über vCenter; Exception Users können per DCUI zugreifen | Standard für Produktion |
| **Strict** | Kein lokaler Zugriff, DCUI deaktiviert | High-Security / PCI-DSS |

> [!TIP] Normal Lockdown als Standard
> Strict Lockdown sollte nur eingesetzt werden, wenn eine Out-of-Band-Konsole (iDRAC, iLO, IPMI) zuverlässig zur Verfügung steht. Andernfalls droht ein kompletter Zugangsverlust bei vCenter-Ausfall.

### Host-Profile

Host-Profile (vCenter Enterprise Plus) sichern Konfigurationskonsistenz über alle Cluster-Hosts:

- Referenz-Host konfigurieren → Profil extrahieren → auf alle anderen Hosts anwenden
- Nach jedem Patch-Zyklus Compliance-Check durchführen
- Host-Profile dokumentieren und als `.vpf` exportieren

### Management-Netzwerk und NIC Teaming

```
Empfohlene Netzwerktrennung:
┌─────────────────────────────────────────────────┐
│  VLAN 10 — Management (ESXi DCUI, Host Client)  │
│  VLAN 20 — vMotion                              │
│  VLAN 30 — VM-Traffic                           │
│  VLAN 40 — Storage (iSCSI / NFS / vSAN)         │
└─────────────────────────────────────────────────┘
```

- Management-VMkernel immer auf dediziertem VLAN mit eingeschränktem Firewall-Zugriff
- NIC Teaming mit mindestens **2 physischen Uplinks** pro vSwitch (Active/Active oder Active/Standby)
- Für Storage-Netzwerke: Route Based on IP Hash für MPIO-Balance

---

## 2. VM-Konfiguration

### VMware Tools und Hardware-Version

> [!IMPORTANT] VMware Tools sind keine Option
> Ohne aktuelle VMware Tools fehlen Quiesce-Snapshots, korrekte Ressourcenstatistiken und saubere Shutdown-Signale. VMs ohne Tools gelten als nicht produktionstauglich.

| Komponente | Empfehlung |
|---|---|
| VMware Tools | Immer aktuell halten, Auto-Update aktivieren |
| VM Hardware (HWv) | Regelmäßig upgraden — im Wartungsfenster |
| Netzwerkadapter | **VMXNET3** für alle neuen VMs — E1000/E1000e nur für Legacy |
| SCSI-Controller | Paravirtual SCSI (PVSCSI) für alle neuen Windows/Linux-VMs |

### Thin Provisioning vs. Eager Zeroed Thick

| Provisioning-Typ | Vorteil | Empfehlung |
|---|---|---|
| **Thin** | Geringster Speicherverbrauch, schnelle Bereitstellung | Standard für alle VMs |
| **Lazy Zeroed Thick** | Volle Kapazität reserviert | Wenige Sonderfälle |
| **Eager Zeroed Thick** | Maximale I/O-Performance, für Cluster-FS geeignet | Datenbanken, Exchange, FCI |

### Snapshots

> [!DANGER] Snapshots sind temporär — kein Backup-Ersatz
> Snapshots wachsen mit jeder Schreiboperation. Ein Snapshot, der wochenlang läuft, kann den gesamten Datastore füllen und die VM zum Stillstand bringen.

| Regel | Wert |
|---|---|
| Maximale Snapshot-Laufzeit | **72 Stunden** |
| Maximale Snapshot-Tiefe | **3 Ebenen** |
| Produktive Snapshots | **Niemals** dauerhaft |
| ISO-Images gemountet lassen | Nach Installation immer trennen |

```powershell
# Alle VMs mit Snapshots älter als 3 Tage finden
Get-VM | Get-Snapshot | Where-Object {$_.Created -lt (Get-Date).AddDays(-3)} |
    Select-Object VM, Name, Created, SizeGB | Format-Table -AutoSize
```

### CPU und RAM — Reservierungen & NUMA

- **Reservierungen** nur für wirklich kritische Workloads (DBs, Echtzeitsysteme) setzen — sie erhöhen Fragmentierung und erschweren vMotion
- **CPU Ready** ist der wichtigste VM-Performance-Indikator (Ziel: < 5 %)
- **NUMA-Awareness:** VMs, die mehr vCPUs/RAM haben als ein NUMA-Node physisch bietet, erzeugen Remote-Memory-Zugriffe → deutlicher Performance-Einbruch

> [!TIP] NUMA-Sizing-Regel
> vCPU-Anzahl und RAM einer VM sollten die Kapazität eines einzelnen NUMA-Nodes nicht überschreiten. Für große Workloads lieber zwei kleinere VMs als eine Gigant-VM.

---

## 3. Storage

### Datastore-Management

| Metrik | Warnschwelle | Kritisch | Aktion |
|---|---|---|---|
| Datastore-Auslastung | > 80 % | > 90 % | Kapazität erweitern / VMs migrieren |
| Snapshot-Delta-Größe | > 30 % VM-Größe | > 50 % VM-Größe | Snapshot konsolidieren |
| Disk Read Latency | > 20 ms | > 50 ms | Storage-Analyse |
| Disk Write Latency | > 20 ms | > 50 ms | Storage-Analyse |

```powershell
# Datastore-Auslastung aller Datastores anzeigen
Get-Datastore | Select-Object Name,
    @{N="Gesamt GB";E={[math]::Round($_.CapacityGB,1)}},
    @{N="Frei GB";E={[math]::Round($_.FreeSpaceGB,1)}},
    @{N="Belegt %";E={[math]::Round((1 - $_.FreeSpaceGB/$_.CapacityGB)*100,1)}} |
    Sort-Object "Belegt %" -Descending | Format-Table -AutoSize
```

### Weitere Storage-Empfehlungen

- **Getrennte Datastores** für produktive VMs, Templates und ISO-Bibliothek
- **vSAN Deduplication & Compression** nur auf All-Flash aktivieren — auf Hybrid-vSAN ist der Performance-Overhead zu groß
- **Snapshot-Konsolidierung:** Wöchentlich per PowerCLI prüfen — nicht konsolidierte Snapshots verursachen stille Performance-Probleme
- **Storage I/O Control (SIOC)** aktivieren: automatische I/O-Priorisierung verhindert, dass einzelne VMs den Datastore monopolisieren

```powershell
# VMs mit nicht konsolidierten Snapshots finden
Get-VM | Where-Object {$_.Extensiondata.Runtime.ConsolidationNeeded} |
    Select-Object Name, PowerState | Format-Table -AutoSize
```

---

## 4. Netzwerk

### Netzwerksegmentierung

> [!IMPORTANT] Netzwerktypen immer strikt trennen
> Management- und VM-Traffic dürfen niemals auf demselben VMkernel oder VLAN laufen. Ein kompromittierter VM-Gast darf keinen Zugriff auf die Management-Plane haben.

| Netzwerktyp | Eigenes VMkernel | Eigenes VLAN | MTU |
|---|---|---|---|
| Management | ✅ | ✅ | 1500 |
| vMotion | ✅ | ✅ | **9000** |
| VM-Traffic | ❌ (Port Groups) | ✅ | 1500 |
| iSCSI / NFS / vSAN | ✅ | ✅ | **9000** |

### Jumbo Frames

> [!TIP] Jumbo Frames müssen end-to-end stimmen
> MTU 9000 muss auf jedem Glied der Kette konfiguriert sein: physischer Switch, VMkernel-Port, vSwitch-Uplink und Storage-System. Ein einziges Gerät mit MTU 1500 macht die gesamte Konfiguration wirkungslos.

```bash
# MTU-Test auf ESXi Shell (VMkernel vmk1 → Storage-IP)
vmkping -I vmk1 -d -s 8972 <storage-ip>
# -d = Don't Fragment | -s 8972 = 8972 Byte Payload + 28 Byte Header = 9000 Byte gesamt
```

### vSwitch-Typen

| Feature | Standard vSwitch (VSS) | Distributed vSwitch (VDS) |
|---|---|---|
| Zentrales Management | ❌ | ✅ (über vCenter) |
| LACP / Link Aggregation | ❌ | ✅ |
| NetFlow / Port Mirroring | ❌ | ✅ |
| Empfehlung | Test/Lab | **Produktion** |

### Port Group Security

| Security-Policy | Standard | Ausnahme |
|---|---|---|
| Promiscuous Mode | **Reject** | Network Monitoring VMs, IDS/IPS |
| MAC Address Changes | **Reject** | Legacy-Applikationen mit MAC-Binding |
| Forged Transmits | **Reject** | Nested Virtualization, NSX Edge |

---

## 5. Sicherheit

### Zugriffskontrolle und Authentifizierung

- **Least Privilege:** Vordefinierte vCenter-Rollen nutzen; eigene Rollen nur anlegen, wenn Standard-Rollen nicht ausreichen
- **AD-Integration:** Lokale ESXi-Accounts auf ein Minimum reduzieren — alle Admins authentifizieren sich über vCenter mit AD/LDAP
- **root-Account:** Regelmäßige Passwortrotation (alle 90 Tage), komplexe Passphrase (min. 20 Zeichen)
- **Lockdown Mode:** Nach der Ersteinrichtung aktivieren — mindestens Normal, Strict in hochsicheren Umgebungen

```powershell
# root-Passwort per PowerCLI auf allen Hosts rotieren
$cred = Get-Credential -UserName root -Message "Neues root-Passwort eingeben"
Get-VMHost | ForEach-Object {
    $hostView = $_.ExtensionData
    $hostView.UpdateLocalSwpUser($cred.GetNetworkCredential().Password)
}
```

### Audit-Logging und TLS

| Maßnahme | Umsetzung |
|---|---|
| Audit-Logging | vCenter Audit-Log aktivieren, extern an SIEM weiterleiten |
| TLS-Verschlüsselung | Nur TLS 1.2+ erlauben — SSLv3, TLS 1.0 und 1.1 deaktivieren |
| vSphere API | Ausschließlich über HTTPS — HTTP-Zugriff auf Port 80 blockieren |
| Zertifikate | VMware CA oder eigene PKI — selbstsignierte Zertifikate nur im Lab |

> [!WARNING] TLS-Downgrade vor Härtung testen
> ESXi akzeptiert standardmäßig ältere TLS-Versionen. In gehärteten Umgebungen (PCI-DSS, ISO 27001) TLS 1.0/1.1 explizit deaktivieren — Kompatibilität aller angebundenen Systeme vorher prüfen.

---

## 6. Wartung und Updates

### Update-Reihenfolge

> [!IMPORTANT] vCenter immer zuerst aktualisieren
> Ein neuerer ESXi-Host an einem älteren vCenter kann zu Inkompatibilitäten und Funktionsausfällen führen. Die Reihenfolge ist verbindlich.

```
Empfohlene Update-Reihenfolge:
1. vCenter Server (VCSA)
2. PSC / External Identity Sources (falls vorhanden)
3. NSX Manager (falls vorhanden)
4. ESXi-Hosts — Rolling: einen nach dem anderen
5. VMware Tools in den VMs
6. VM Hardware-Version (optional, separates Wartungsfenster)
```

### Rolling Host-Updates

```powershell
# Host in Maintenance Mode versetzen (VMs automatisch per vMotion migrieren)
Get-VMHost "esxi01.example.local" | Set-VMHost -State Maintenance

# Nach abgeschlossenem Update: Host wieder in Betrieb nehmen
Get-VMHost "esxi01.example.local" | Set-VMHost -State Connected
```

- **vSphere Lifecycle Manager (vLCM)** für alle Hosts nutzen — Image-basiertes Management eliminiert Konfigurationsdrift zuverlässiger als Baselines
- **Vor Updates:** Snapshot aller Management-VMs erstellen (vCenter, NSX-Manager, vROps)
- **Change Freeze:** In definierten Produktions-Freeze-Perioden keine Updates — Ausnahme: kritische Sicherheits-Patches (CVSS ≥ 9.0)

---

## 7. Backup und Disaster Recovery

> [!DANGER] Snapshots sind KEIN Backup — niemals
> Fällt der Datastore aus, sind Snapshot-Daten und VM-Daten gemeinsam verloren. Snapshots sind ein kurzfristiges Rollback-Werkzeug für Change-Windows — nicht für Disaster Recovery.

### Backup-Tools im Überblick

| Tool | Stärken | Lizenzmodell |
|---|---|---|
| **Veeam Backup & Replication** | Marktführer, CBT, Instant Recovery, SOBR, CDP | Pro Socket / per VM |
| **NAKIVO Backup & Replication** | Kosteneffizient, NAS-Appliance-tauglich | Pro Socket / per VM |
| **Commvault** | Enterprise, umfangreiche Compliance-Features | Enterprise |
| **vSphere Replication** | VMware-nativ, RPO ab 5 Minuten, kein Drittanbieter nötig | Im vSphere-Lizenzumfang |

### DR-Mindeststandards

| Maßnahme | Mindeststandard |
|---|---|
| Backup-Häufigkeit | Täglich — kritische VMs mehrmals täglich |
| Aufbewahrung | 30 Tage On-Site, 90 Tage Off-Site |
| Recovery-Test | **Mindestens 1x pro Jahr** — ungetestete Backups sind wertlos |
| RPO < 24 h | vSphere Replication oder Veeam CDP einsetzen |
| vCenter-Backup | VCSA File-Based Backup täglich auf externen NFS/CIFS-Share |
| Ransomware-Schutz | S3 Object Lock (Immutable) oder WORM-Tape als letzte Verteidigungslinie |

---

## 8. Performance-Monitoring

### Schwellwerte und Alarmkonfiguration

> [!TIP] Alarme proaktiv konfigurieren — nicht reaktiv
> Der Standard-Alarm-Satz von VMware ist zu konservativ. Eigene Alarme für alle kritischen Metriken in vCenter anlegen und an das Monitoring-System weiterleiten.

| Metrik | Warnschwelle | Kritisch | Maßnahme |
|---|---|---|---|
| VM CPU Ready | > 5 % | > 10 % | vCPU-Anzahl reduzieren oder Host entlasten |
| Host CPU-Auslastung | > 70 % | > 85 % | Workload migrieren / Kapazität erweitern |
| Memory Balloon | > 0 MB | > 1 % RAM | RAM-Überbuchung reduzieren |
| Memory Swap | > 0 MB | > 100 MB | **Kritisch** — sofortige Eskalation |
| Disk Read Latency | > 20 ms | > 50 ms | Storage-Performance-Analyse |
| Disk Write Latency | > 20 ms | > 50 ms | Storage-Performance-Analyse |
| Cluster CPU-Reserve | < 30 % frei | < 15 % frei | Kapazitätsplanung einleiten |
| Cluster RAM-Reserve | < 30 % frei | < 15 % frei | Kapazitätsplanung einleiten |

```powershell
# CPU-Ready-Durchschnitt aller VMs der letzten 24 Stunden abfragen
Get-VM | ForEach-Object {
    $stat = Get-Stat -Entity $_ -Stat "cpu.ready.summation" `
        -Start (Get-Date).AddDays(-1) -IntervalMins 30 -ErrorAction SilentlyContinue
    if ($stat) {
        [PSCustomObject]@{
            VM                = $_.Name
            "CPU Ready Avg %" = [math]::Round(($stat.Value |
                Measure-Object -Average).Average / 200, 2)
        }
    }
} | Sort-Object "CPU Ready Avg %" -Descending | Format-Table -AutoSize
```

### Kapazitätsplanung

- **Cluster-Faustregel:** CPU/RAM-Auslastung unter **70 %** halten — Reserve für HA-Failover des größten Hosts (N+1-Prinzip)
- **vROps / Aria Operations** (Enterprise): Predictive Analytics, Right-Sizing-Empfehlungen, automatisierte Kapazitätsberichte
- **Minimalstandard:** Wöchentliche Auswertung der vCenter Performance Charts für alle Cluster

> [!NOTE] HA-Kapazitätsreserve berechnen
> Beispiel: 4-Host-Cluster, je 25 % Auslastung. Fällt ein Host aus, tragen 3 Hosts 33 % mehr Last — Gesamtauslastung steigt auf ~33 %. Bei 75 % Ausgangslast wären die 3 verbliebenen Hosts sofort überlastet. Daher: nie über ~75 % auslasten.

---

## Verwandte Notizen

- [[ESXi Grundlagen]]
- [[VM Verwaltung]]
- [[ESXi Troubleshooting]]
- [[PowerCLI]]
- [[vCenter]]
- [[ESXi Netzwerk und Storage]]
- [[vSphere Lifecycle Manager]]
- [[vSphere Replication]]
