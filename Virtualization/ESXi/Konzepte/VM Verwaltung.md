---
tags:
  - virtualization
  - esxi
  - vsphere
  - vmware
  - vm
  - administration
  - grundlagen
aliases:
  - VM erstellen
  - Virtuelle Maschine verwalten
  - vSphere VM Management
---

# VM Verwaltung

> [!ABSTRACT] TL;DR
> Alles rund um den Lebenszyklus virtueller Maschinen in ESXi/vSphere — von der Erstellung über Hardware-Konfiguration, Disk-Erweiterung und Snapshots bis hin zu Templates, Klonen und VMware Tools. Mit Schritt-für-Schritt-Anleitungen für Anfänger und tiefergehenden Infos für Fortgeschrittene.

---

## VM erstellen (New Virtual Machine Wizard)

### Voraussetzungen

Vor dem Erstellen einer VM werden benötigt:
- Ein vorhandener **Datastore** (Speicherort für VM-Dateien)
- Ein konfiguriertes **Netzwerk** (Port Group)
- Eine **ISO-Datei** des Gastbetriebssystems im Datastore oder einer Content Library

### Schritt-für-Schritt (vSphere Host Client / vCenter Client)

1. **Rechtsklick** auf den Host oder Cluster → **New Virtual Machine**
2. **Creation type** wählen:
   - `Create a new virtual machine` — leere VM von Grund auf
   - `Deploy from template` — aus Template deployen
   - `Clone` — bestehende VM klonen
3. **Name** vergeben und **Speicherort** (Datacenter/Ordner) wählen
4. **Compute Resource** wählen (Host oder Cluster)
5. **Datastore** wählen — Speicherort für alle VM-Dateien
6. **Compatibility** — VM Hardware Version:
   - Höhere Version = mehr Features, aber ältere ESXi-Hosts können nicht mehr booten
   - In gemischten Cluster-Umgebungen die niedrigste gemeinsame Version wählen
7. **Guest OS Family & Version** korrekt auswählen (wichtig für VMware Tools & Optimierungen):
   - Windows Server 2022 → `Windows`, `Microsoft Windows Server 2022`
   - Ubuntu 22.04 → `Linux`, `Ubuntu Linux (64-Bit)`
8. **Hardware anpassen:**
   - CPU, RAM, Festplatte, Netzwerkadapter (siehe Abschnitt Hardware)
   - CD/DVD: ISO aus Datastore oder Content Library einlegen
9. **Zusammenfassung prüfen** → **Finish**

> [!TIP] Typical vs. Custom
> Der Wizard bietet je nach Client-Version eine vereinfachte oder detaillierte Ansicht. Für Produktions-VMs immer die erweiterten Optionen nutzen, um Adapter-Typ (VMXNET3, PVSCSI) und Provisioning-Typ (Thin) korrekt zu setzen.

---

## VM Hardware konfigurieren

Hardware-Einstellungen können bei ausgeschalteter VM jederzeit geändert werden. Bei laufender VM sind Hot-Add-Optionen möglich, wenn vorher aktiviert.

### CPU

| Einstellung | Beschreibung | Empfehlung |
|---|---|---|
| **vCPUs** | Anzahl virtueller Prozessoren | Nur so viele wie nötig — zu viele erhöhen CPU-Scheduling-Overhead |
| **Cores per Socket** | CPU-Topologie (beeinflusst Lizenzierung des Gastbetriebssystems) | Windows Server: 1 Core per Socket vermeiden, sinnvoll konfigurieren |
| **CPU Hot Add** | vCPUs im laufenden Betrieb hinzufügen | Aktivieren wenn nötig — erfordert Neustart zur Erstaktivierung |
| **Reservierung** | Garantierte CPU-MHz | Nur für kritische Workloads, erhöht Fragmentierung |
| **Limit** | Maximale CPU-MHz | Vermeiden — kann unbemerkt Performance drosseln |
| **Shares** | Relative Priorität bei Ressourcenknappheit | Standard: Normal |

> [!IMPORTANT] CPU Ready als Qualitätsmaß
> CPU Ready (Wartezeit der VM auf eine physische CPU) sollte unter 5 % liegen. Zu viele vCPUs auf einem VM erhöhen diesen Wert paradoxerweise — weniger vCPUs sind oft schneller.

### RAM

| Einstellung | Beschreibung | Empfehlung |
|---|---|---|
| **Memory** | RAM-Zuteilung | Realistisch dimensionieren — Überdimensionierung belastet den Host |
| **Memory Hot Add** | RAM im laufenden Betrieb hinzufügen | Nur bei Gastbetriebssystemen die Hot-Add unterstützen |
| **Reservierung** | Garantierter physischer RAM | Nur für Echtzeit-Workloads oder wenn Ballooning/Swap verhindert werden soll |

> [!WARNING] Memory Balloon und Swap
> Wenn der Host zu wenig RAM hat, entzieht VMware VMs aktiv RAM (Ballooning) oder lagert aus (Swap) — das führt zu drastischen Performance-Einbrüchen. Symptom: VM reagiert extrem langsam, `memory.balloon` > 0 in vCenter-Charts.

---

## Hot-Add und Hot-Plug

VMware unterscheidet zwischen **Hot-Add** (CPU/RAM im laufenden Betrieb hinzufügen) und **Hot-Plug** (Geräte wie Disks oder NICs ein-/ausstecken). Beide Features müssen vorab aktiviert werden und setzen Unterstützung im Gastbetriebssystem voraus.

### Hot-Add aktivieren (CPU & RAM)

> [!IMPORTANT] Einmalig Neustart erforderlich
> Hot-Add muss bei **ausgeschalteter VM** aktiviert werden. Danach steht es im laufenden Betrieb zur Verfügung — bis die VM erneut ausgeschaltet wird (beim nächsten Neustart bleibt die Einstellung erhalten).

**Via vSphere Client:**
1. VM ausschalten
2. VM → **Actions → Edit Settings → VM Options → Advanced**
3. **Memory Hot Add** und/oder **CPU Hot Add** auf `Enable` setzen
4. **OK** → VM starten

**Via PowerCLI:**
```powershell
# CPU und Memory Hot-Add auf einer VM aktivieren
$vm = Get-VM "MeineVM"
$spec = New-Object VMware.Vim.VirtualMachineConfigSpec
$spec.memoryHotAddEnabled = $true
$spec.cpuHotAddEnabled    = $true
$vm.ExtensionData.ReconfigVM($spec)
```

### CPU Hot-Add — vCPUs im laufenden Betrieb erhöhen

Wenn CPU Hot-Add aktiviert ist, können vCPUs **ohne VM-Neustart** hinzugefügt werden — aber **nicht reduziert**.

**Via vSphere Client (laufende VM):**
1. VM → **Actions → Edit Settings**
2. **CPU** → Anzahl der vCPUs erhöhen
3. **OK** — wirkt sofort

**Via PowerCLI:**
```powershell
# vCPU-Anzahl einer laufenden VM erhöhen (von 2 auf 4)
Set-VM -VM "MeineVM" -NumCpu 4 -Confirm:$false
```

> [!WARNING] CPU-Reduzierung erfordert Neustart
> Hot-Add gilt nur in eine Richtung — vCPUs können im laufenden Betrieb **nur erhöht**, nicht reduziert werden. Für eine Reduzierung muss die VM neu gestartet werden.

### Memory Hot-Add — RAM im laufenden Betrieb erhöhen

RAM kann ebenfalls ohne Neustart erhöht werden — sofern das Gastbetriebssystem Hot-Add unterstützt.

**Via vSphere Client (laufende VM):**
1. VM → **Actions → Edit Settings**
2. **Memory** → Wert erhöhen (z.B. von 8 GB auf 16 GB)
3. **OK** — wirkt sofort (wenn vom Gastbetriebssystem akzeptiert)

**Via PowerCLI:**
```powershell
# RAM einer laufenden VM erhöhen (auf 16 GB)
Set-VM -VM "MeineVM" -MemoryGB 16 -Confirm:$false
```

**RAM im Gastbetriebssystem aktivieren (Windows):**
```powershell
# Prüfen ob neuer RAM vom Windows-Kernel erkannt wurde
Get-WmiObject -Class Win32_PhysicalMemory | Measure-Object -Property Capacity -Sum |
    Select-Object @{N="RAM GB";E={$_.Sum / 1GB}}
```

Unter Linux ist Hot-Add-RAM meist sofort verfügbar — im Zweifel prüfen mit:
```bash
free -h
cat /proc/meminfo | grep MemTotal
```

### Hot-Plug — Geräte im laufenden Betrieb hinzufügen

Neben CPU und RAM können auch einzelne **virtuelle Geräte** im laufenden Betrieb hinzugefügt werden:

| Gerät | Hot-Plug möglich | Einschränkung |
|---|---|---|
| **Virtuelle Disk (VMDK)** | ✅ Ja | Nur hinzufügen — nicht entfernen bei laufender VM |
| **Netzwerkadapter (NIC)** | ✅ Ja | Nur hinzufügen — Entfernen erfordert Neustart |
| **USB-Gerät** | ✅ Ja (USB 3.x Controller) | Gastbetriebssystem muss USB Hot-Plug unterstützen |
| **CD/DVD-Laufwerk** | ✅ Ja | ISO jederzeit einlegen/wechseln |
| **SCSI-Controller** | ❌ Nein | Neustart erforderlich |
| **PCI-Gerät (Passthrough)** | ❌ Nein | Neustart erforderlich |

**Disk Hot-Plug (neue Disk hinzufügen ohne Neustart):**
1. VM → **Actions → Edit Settings → Add New Device → Hard Disk**
2. Größe und Provisioning-Typ wählen → **OK**
3. Im Gastbetriebssystem die neue Disk initialisieren:

```powershell
# Windows — neue Disk online bringen und formatieren
Get-Disk | Where-Object PartitionStyle -eq 'RAW' | Initialize-Disk -PartitionStyle GPT
New-Partition -DiskNumber <n> -UseMaximumSize -AssignDriveLetter |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "Daten" -Confirm:$false
```

```bash
# Linux — neue Disk erkennen und einrichten
echo "- - -" > /sys/class/scsi_host/host0/scan   # SCSI-Rescan
lsblk                                              # Neue Disk sehen (z.B. /dev/sdb)
fdisk /dev/sdb                                     # Partitionieren
mkfs.ext4 /dev/sdb1                               # Formatieren
mount /dev/sdb1 /mnt/daten                        # Einhängen
```

### OS-Kompatibilität für Hot-Add

Nicht alle Gastbetriebssysteme unterstützen CPU- und RAM-Hot-Add gleich gut:

| Betriebssystem | CPU Hot-Add | RAM Hot-Add | Hinweis |
|---|---|---|---|
| Windows Server 2016+ | ✅ | ✅ | Sofort verfügbar |
| Windows Server 2012 R2 | ✅ | ✅ | Nur Datacenter/Standard Edition |
| Windows 10/11 | ❌ | ✅ | CPU Hot-Add nicht unterstützt |
| RHEL / CentOS 7+ | ✅ | ✅ | Kernel-Unterstützung vorhanden |
| Ubuntu 18.04+ | ✅ | ✅ | Standard-Kernel unterstützt beides |
| Debian 10+ | ✅ | ✅ | Standard-Kernel unterstützt beides |
| ältere Linux (Kernel < 3.x) | ❌ | ❌ | Neustart erforderlich |

> [!TIP] Hot-Add und NUMA
> Wenn Hot-Add-CPU dazu führt, dass die VM mehr vCPUs hat als ein NUMA-Node physisch bietet, aktiviert VMware automatisch vNUMA — das kann die Performance beeinflussen. Bei sehr großen VMs vorher die NUMA-Topologie des Hosts prüfen (`esxcli hardware cpu list`).

> [!NOTE] Hot-Add deaktiviert bestimmte CPU-Features
> Wenn CPU Hot-Add aktiviert ist, deaktiviert VMware intern **vNUMA** für die VM (bis die Anzahl der vCPUs einen NUMA-Node übersteigt). In leistungskritischen Umgebungen Hot-Add daher nur aktivieren wenn wirklich benötigt.

### Netzwerkadapter

| Adapter-Typ | Beschreibung | Empfehlung |
|---|---|---|
| **VMXNET3** | Paravirtualisiert, maximale Performance, bis 10 GbE | **Standard für alle neuen VMs** |
| **E1000e** | Emulierter Intel-Adapter, breite OS-Kompatibilität | Ältere OS ohne VMXNET3-Treiber |
| **E1000** | Älterer emulierter Adapter | Nur Legacy |
| **SR-IOV** | Direktdurchleitung physischer NIC-Funktion | High-Performance / NFV |

### Empfohlene Hardware nach OS-Typ

| Betriebssystem | Netzwerk | SCSI-Controller | Sonstiges |
|---|---|---|---|
| Windows Server 2016+ | VMXNET3 | PVSCSI | VMware Tools installieren |
| Windows 10/11 | VMXNET3 | PVSCSI | — |
| Ubuntu / Debian | VMXNET3 | PVSCSI | open-vm-tools aus Repo |
| RHEL / CentOS 8+ | VMXNET3 | PVSCSI | open-vm-tools aus Repo |
| ältere Linux-Distros | VMXNET3 (falls Treiber vorhanden) | LSI Logic | VMware Tools ISO |
| Windows Server 2008 R2 | VMXNET3 | PVSCSI | VMware Tools ISO |

---

## Disk Provisioning — Thin vs. Thick

### Die drei Provisioning-Typen

| Typ | Speicherverbrauch | Performance | Wann einsetzen |
|---|---|---|---|
| **Thin Provision** | Wächst on-demand ab 0 | Leicht geringer I/O beim Allokieren | **Standard** — alle normalen Workloads |
| **Thick Lazy Zeroed** | Vollständig reserviert, Nullen beim Schreiben | Ähnlich Thin | Selten sinnvoll — kaum Vorteile |
| **Thick Eager Zeroed** | Vollständig reserviert UND vorgenullt | **Beste Performance** | Datenbanken, Exchange, WSFC, vSAN |

> [!TIP] Thin ist fast immer richtig
> In den meisten Umgebungen ist Thin Provisioning optimal — solange die Datastore-Auslastung überwacht wird (Warnung ab 80 %). Thin-Disks können nachträglich nicht in Eager Thick konvertiert werden ohne Storage vMotion oder vmkfstools.

### Konvertierung zwischen Typen

```bash
# Thin → Eager Zeroed Thick (ESXi Shell — VM muss ausgeschaltet sein)
vmkfstools -i /vmfs/volumes/datastore1/VM/vm.vmdk \
           /vmfs/volumes/datastore1/VM/vm-thick.vmdk \
           -d eagerzeroedthick
```

Oder per vCenter: **Storage vMotion** mit Änderung des Provisioning-Typs im Wizard.

---

## VM-Festplatte erweitern

> [!IMPORTANT] Zwei separate Schritte
> Eine Disk-Erweiterung besteht immer aus zwei Teilen: **(1)** die virtuelle Disk in ESXi/vCenter vergrößern und **(2)** die Partition im Gastbetriebssystem erweitern. ESXi vergrößert nur den Container — das Gastbetriebssystem muss den neuen Platz selbst beanspruchen.

### Schritt 1: Disk in vSphere vergrößern

**Voraussetzungen:**
- VM hat **keine aktiven Snapshots** (Snapshots sperren Disk-Änderungen — vorher löschen/konsolidieren!)
- VM kann laufen (Hot-Extend möglich) oder ausgeschaltet sein

**Via vSphere Client:**
1. VM auswählen → **Actions → Edit Settings**
2. Auf die gewünschte **Hard Disk** klicken
3. **Disk Size** auf den neuen Wert erhöhen (z.B. von 50 GB auf 100 GB)
4. **OK** klicken

**Via ESXi Shell (vmkfstools):**
```bash
# VM muss ausgeschaltet sein!
vmkfstools -X 100G /vmfs/volumes/datastore1/MeineVM/MeineVM.vmdk
```

> [!WARNING] Disk grayed-out / nicht veränderbar?
> Ursachen: aktive Snapshots, VM läuft im Independent-Disk-Modus, oder Disk ist Teil eines Multi-Writer-Setups. Alle Snapshots löschen und konsolidieren, dann erneut versuchen.

### Schritt 2a: Partition im Gastbetriebssystem erweitern — Windows

**Methode 1 — Datenträgerverwaltung (GUI):**
1. `diskmgmt.msc` öffnen
2. Aktion → **Datenträger aktualisieren** (damit Windows die neue Größe erkennt)
3. Rechtsklick auf die Partition → **Volume erweitern**
4. Assistent durchlaufen → Fertig stellen

**Methode 2 — PowerShell:**
```powershell
# Verfügbaren Speicherplatz auf Laufwerk C: anzeigen
Get-PartitionSupportedSize -DriveLetter C

# Partition auf maximale Größe erweitern
$size = (Get-PartitionSupportedSize -DriveLetter C).SizeMax
Resize-Partition -DriveLetter C -Size $size
```

### Schritt 2b: Partition im Gastbetriebssystem erweitern — Linux

```bash
# Schritt 1: Neue Disk-Größe dem Kernel bekanntmachen (ohne Neustart)
echo 1 > /sys/class/block/sda/device/rescan
# oder bei LVM-PV auf /dev/sda:
pvresize /dev/sda

# Schritt 2: Partition erweitern (growpart — cloud-utils-growpart installieren)
growpart /dev/sda 2          # Partition 2 auf /dev/sda erweitern

# Schritt 3: Dateisystem erweitern
# ext4:
resize2fs /dev/sda2

# xfs (z.B. RHEL/CentOS Standard):
xfs_growfs /

# LVM-Workflow (häufig bei Enterprise-Linux):
pvresize /dev/sda3            # Physical Volume erweitern
lvextend -l +100%FREE /dev/vg0/lv_root   # Logical Volume erweitern
resize2fs /dev/vg0/lv_root    # Filesystem erweitern (ext4)
# oder:
xfs_growfs /dev/vg0/lv_root   # Filesystem erweitern (xfs)
```

> [!TIP] Neustart oft nicht nötig
> Bei modernen Linux-Systemen (Kernel 3.x+) und VMware Tools ist ein Neustart für die Disk-Erkennung meist nicht erforderlich. `lsblk` oder `fdisk -l` zeigen die neue Größe nach dem Rescan.

---

## Snapshots

### Was ist ein Snapshot?

Ein Snapshot friert den Zustand einer VM zu einem bestimmten Zeitpunkt ein. Alle Schreiboperationen gehen ab diesem Punkt in eine **Delta-VMDK-Datei** — die originale VMDK bleibt unverändert. Beim Zurücksetzen wird die Delta-Datei verworfen.

```
VM-Disk vor Snapshot:      vm.vmdk (z.B. 40 GB belegt)
Nach Snapshot-Erstellung:  vm.vmdk (eingefroren)
                           vm-000001-delta.vmdk (wächst mit jeder Schreiboperation)
```

### Snapshot erstellen

**Via vSphere Client:**
1. VM → **Actions → Snapshots → Take Snapshot**
2. **Name** und optionale Beschreibung vergeben
3. Optionen:
   - `Snapshot the virtual machine's memory` — RAM-Zustand einfrieren (länger, ermöglicht vollständige Wiederherstellung inkl. RAM)
   - `Quiesce guest file system` — VMware Tools sendet Flush-Signal ans OS für konsistenten Zustand (empfohlen für Backups)

**Via ESXi Shell:**
```bash
vim-cmd vmsvc/snapshot.create <vmid> "Vor-Update" "Beschreibung" 0 1
# Parameter: vmid, name, beschreibung, memory(0/1), quiesce(0/1)
```

### Snapshot wiederherstellen / löschen

```bash
# Snapshot-Liste anzeigen
vim-cmd vmsvc/snapshot.get <vmid>

# Alle Snapshots löschen (konsolidieren)
vim-cmd vmsvc/snapshot.removeall <vmid>
```

**Via vSphere Client:** VM → **Actions → Snapshots → Manage Snapshots**

Im **Snapshot-Manager** sind alle Snapshots als Baum dargestellt. Einzelne Snapshots können wiederhergestellt (`Revert to`) oder gelöscht werden.

> [!DANGER] Snapshot-Wachstum ist gefährlich
> Eine Delta-VMDK kann theoretisch so groß werden wie die originale Disk. Füllt die Delta-Disk den Datastore, friert die VM ein und kann nicht mehr schreiben → Produktionsausfall. Immer Datastore-Freiraum überwachen.

### Best Practices für Snapshots

| Regel | Empfehlung |
|---|---|
| Maximale Laufzeit | **72 Stunden** — danach löschen |
| Maximale Tiefe | **3 Ebenen** im Snapshot-Baum |
| Produktive Umgebungen | Snapshots **niemals** dauerhaft lassen |
| Nach dem Löschen | Konsolidierung prüfen (vCenter warnt bei Bedarf) |
| Snapshots = Backup? | **NEIN** — bei Datastore-Ausfall verloren |

---

## VM Templates und Klonen

### Template erstellen

Templates sind VMs im „eingefrorenen" Zustand — sie können nicht gestartet, nur deployed werden.

**VM → Rechtsklick → Clone → Clone to Template:**
1. Template-Name vergeben
2. Speicherort (Ordner) und Datastore wählen
3. **Finish** — die VM wird in ein Template konvertiert (Original verschwindet als VM)

**Oder: Template aus laufender VM erstellen (ohne VM zu löschen):**
VM → Rechtsklick → **Clone → Clone to Template in Library** (Content Library)

> [!TIP] VM vor Template-Erstellung vorbereiten
> - VMware Tools aktuell halten
> - Alle temporären Dateien löschen
> - Logs leeren
> - Windows: `sysprep` ausführen (generalisiert das System)
> - Linux: `cloud-init clean` oder SSH-Host-Keys löschen

### VM aus Template deployen

Template → Rechtsklick → **New VM from This Template:**
1. VM-Name vergeben
2. Compute Resource und Datastore wählen
3. **Customization Specification** auswählen (optional, aber empfohlen):
   - Setzt automatisch Hostname, IP-Adresse, Zeitzone
   - Für Windows: Sysprep-basiert
   - Für Linux: Hostname und Netzwerk-Konfiguration

### Linked Clone vs. Full Clone

| Eigenschaft | Full Clone | Linked Clone |
|---|---|---|
| Speicherplatz | Vollständige Kopie | Nur Delta zur Quelldisk |
| Abhängigkeit | Unabhängig | Abhängig von Quelldisk (Template) |
| Performance | Normal | Leicht schlechter (Reads gehen zur Quelldisk) |
| Einsatz | Produktion | Dev/Test, VDI, schnelles Deployen |
| Snapshot nötig? | Nein | Ja (Linked Clones basieren auf einem Snapshot) |

### Customization Specification

Customization Specifications (früher: "Sysprep-Konfiguration") automatisieren die Personalisierung nach dem Klonen:

**vCenter → Policies and Profiles → VM Customization Specifications → New:**

| Einstellung | Windows | Linux |
|---|---|---|
| Hostname | Automatisch oder aus VM-Name | Automatisch oder manuell |
| Netzwerk | DHCP oder statisch | DHCP oder statisch |
| Domain Join | Ja (inkl. AD-Credentials) | Nein (manuell per Skript) |
| Lizenznummer | Optional eintragbar | Nicht relevant |
| Zeitzone | Auswählbar | Auswählbar |

---

## VM-Lebenszyklus-Operationen

### Power-Operationen im Vergleich

| Operation | Beschreibung | Wann nutzen |
|---|---|---|
| **Power On** | VM starten | Normal |
| **Shut Down Guest** | Sauberes Herunterfahren via VMware Tools | **Standard** — immer bevorzugen |
| **Power Off** | Sofortiger Stromverlust | Nur wenn Guest nicht reagiert |
| **Restart Guest** | Sauberer Neustart via VMware Tools | **Standard** |
| **Reset** | Harter Reset (wie Stromkabel ziehen/einstecken) | Nur Notfall |
| **Suspend** | RAM-Zustand auf Disk sichern, Host-Ressourcen freigeben | Dev/Test |

> [!WARNING] Power Off vs. Shut Down Guest
> `Power Off` entspricht dem Ziehen des Netzsteckers — kein sauberes Herunterfahren, kein Filesystem-Flush. Immer `Shut Down Guest` bevorzugen. `Power Off` nur wenn das Guest-OS nicht mehr reagiert.

### VM migrieren

| Migrationsart | Voraussetzung | Downtime |
|---|---|---|
| **vMotion** | Shared Storage, vCenter, kompatible CPUs | ~0 s |
| **Storage vMotion** | Kein Shared Storage nötig | ~0 s |
| **Cold Migration** | VM ausgeschaltet | Ja |

Für Details → [[vCenter]]

### VM entfernen vs. löschen

| Option | Was passiert | Wiederherstellbar? |
|---|---|---|
| **Remove from Inventory** | VM verschwindet aus vCenter/Host Client, Dateien bleiben auf dem Datastore | Ja — VMX-Datei erneut registrieren |
| **Delete from Disk** | VM und alle Dateien (VMDK, VMX, Logs) werden **unwiderruflich gelöscht** | **Nein** |

> [!DANGER] Delete from Disk ist endgültig
> Es gibt keine Papierkorb-Funktion. Vor dem Löschen sicherstellen, dass die VM tatsächlich nicht mehr benötigt wird und kein Backup-Job noch auf sie zeigt.

---

## VMware Tools

### Was sind VMware Tools?

VMware Tools ist ein Paket aus Treibern und Diensten, das im Gastbetriebssystem läuft und die Integration zwischen VM und Hypervisor drastisch verbessert.

| Feature | Ohne VMware Tools | Mit VMware Tools |
|---|---|---|
| Maussteuerung | Springt beim Verlassen des VM-Fensters | Nahtloser Übergang |
| Shutdown/Reboot | Nur Power Off/Reset möglich | Sauberer Shut Down / Restart |
| IP-Adresse in vCenter | Nicht sichtbar | Sichtbar in VM-Übersicht |
| Quiesce Snapshots | Nicht möglich | Dateisystem-konsistente Snapshots |
| Performance-Metriken | Eingeschränkt | Vollständige CPU/RAM/Disk/Net-Metriken |
| Zeitsynchonisation | Abweichungen möglich | Automatische NTP-Synchronisation |
| Drag & Drop / Copy-Paste | Nicht verfügbar | Verfügbar (wenn aktiviert) |

### Installation auf Windows

1. VM → **Actions → Guest OS → Install VMware Tools**
2. Im Gastbetriebssystem: DVD-Laufwerk öffnen → `setup64.exe` ausführen
3. Installationstyp: **Typical** (Standard) oder **Complete** (für Troubleshooting)
4. Neustart der VM

**Oder per PowerShell (wenn VMware Tools bereits veraltet installiert):**
```powershell
# VMware Tools über Windows Update aktualisieren (neuere Versionen)
# Alternativ: MSI-Paket aus Content Library deployen
```

### Installation auf Linux

```bash
# Empfohlen: Open VM Tools aus dem Paket-Repository (automatische Updates)

# Ubuntu / Debian:
sudo apt update && sudo apt install -y open-vm-tools

# Für Desktop-VMs (mit GUI):
sudo apt install -y open-vm-tools-desktop

# RHEL / CentOS / AlmaLinux:
sudo dnf install -y open-vm-tools

# Dienst starten und aktivieren:
sudo systemctl enable --now vmtoolsd
```

> [!TIP] Open VM Tools vs. VMware Tools ISO
> `open-vm-tools` aus dem OS-Repository ist der empfohlene Weg für alle modernen Linux-Distributionen — keine manuelle Installation, automatische Updates mit OS-Updates. Die VMware-Tools-ISO ist nur für ältere oder nicht unterstützte Distributionen nötig.

### VMware Tools aktuell halten

```powershell
# Alle VMs mit veralteten VMware Tools finden (PowerCLI)
Get-VM | Get-VMGuest |
    Where-Object {$_.ToolsVersionStatus -ne "guestToolsCurrent"} |
    Select-Object VmName, ToolsVersion, ToolsVersionStatus |
    Format-Table -AutoSize
```

---

## Verwandte Notizen

- [[ESXi Grundlagen]]
- [[ESXi Installation]]
- [[ESXi Netzwerk und Storage]]
- [[vCenter]]
- [[ESXi Troubleshooting]]
- [[ESXi Best Practices]]
- [[ESXi CLI Cheatsheet]]
