---
tags:
  - esxi
  - vsphere
  - vmware
  - cli
  - cheatsheet
  - shell
  - esxcli
  - referenz
aliases:
  - ESXCLI Cheatsheet
  - ESXi Shell Befehle
---

# ESXi CLI Cheatsheet

> [!ABSTRACT] TL;DR
> Schnellreferenz für `esxcli`, `vim-cmd`, `vmkfstools` und weitere Shell-Befehle auf ESXi-Hosts. Alle wichtigen Namespaces, VM-Operationen, Netzwerk-, Storage- und Log-Kommandos auf einen Blick.

---

## Zugang zur ESXi Shell

### DCUI (Direkte Konsole)

Am Host-Bildschirm oder über iDRAC/iLO:

1. `F2` drücken → **Troubleshooting Options**
2. **Enable ESXi Shell** → `Enter`
3. **Enable SSH** → `Enter`
4. Mit `Alt+F1` zur Shell wechseln (lokal am Server)

### SSH aktivieren (vSphere Client)

**Host → Actions → Services → Enable Secure Shell (SSH)**

Oder direkt in der DCUI unter **Troubleshooting Options → Enable SSH**.

### PowerCLI (Remote)

```powershell
# Verbindung zu einem ESXi-Host herstellen
Connect-VIServer -Server esxi01.example.com -User root -Password 'Passwort'

# EsxCli-Objekt für Remote-Ausführung von esxcli-Befehlen
$esxcli = Get-EsxCli -VMHost esxi01.example.com -V2
```

> [!TIP] SSH-Timeout
> Standardmäßig trennt ESXi SSH-Sessions nach kurzer Inaktivität. `screen` oder `tmux` nutzen um Sessions bei langen Operationen zu halten.

---

## ESXCLI — Grundstruktur

```bash
esxcli <namespace> <object> <command> [options]
```

### Wichtige Namespaces

| Namespace | Beschreibung |
|:----------|:-------------|
| `system` | Host-Konfiguration, Zeit, Wartungsmodus |
| `network` | NICs, vSwitches, Firewall, IP-Konfiguration |
| `storage` | Datastores, Geräte, Pfade, VMFS |
| `vm` | Laufende VM-Prozesse |
| `software` | VIBs, Patches, Profile |
| `hardware` | CPU, RAM, PCI, BIOS |
| `iscsi` | iSCSI-Initiator und -Targets |

### Hilfe & Discovery

```bash
esxcli --help                        # Alle Namespaces
esxcli system --help                 # Objekte eines Namespace
esxcli network nic list --help       # Optionen eines Befehls
```

---

## System-Befehle

```bash
# Host-Informationen
esxcli system hostname get
esxcli system hostname set --host=esxi01 --fqdn=esxi01.example.com
esxcli system version get
vmware -v
esxcli system time get
esxcli system stats uptime get

# Wartungsmodus
esxcli system maintenanceMode set --enable true
esxcli system maintenanceMode set --enable false
esxcli system maintenanceMode get
```

> [!WARNING] Wartungsmodus im Cluster
> Bei vSphere-Cluster-Hosts immer DRS und vMotion prüfen — ohne DRS müssen VMs manuell migriert werden.

```bash
# Neustart & Herunterfahren
esxcli system shutdown reboot --reason="Patch-Wartung"
esxcli system shutdown poweroff --reason="Hardware-Austausch"
```

> [!DANGER] Sofortige Wirkung
> `shutdown poweroff` und `reboot` werden ohne weitere Bestätigung ausgeführt — laufende VMs werden hart unterbrochen.

---

## VM-Befehle (vim-cmd)

### VMs auflisten

```bash
vim-cmd vmsvc/getallvms          # Alle registrierten VMs mit VMID
esxcli vm process list           # Laufende Prozesse mit World-ID
```

### Power-Management

```bash
vim-cmd vmsvc/power.on <vmid>
vim-cmd vmsvc/power.off <vmid>
vim-cmd vmsvc/power.reset <vmid>
vim-cmd vmsvc/power.suspend <vmid>
vim-cmd vmsvc/power.getstate <vmid>
```

### VM-Details

```bash
vim-cmd vmsvc/get.summary <vmid>   # Status, Ressourcen, IP
vim-cmd vmsvc/get.networks <vmid>  # Netzwerk-Info
vim-cmd vmsvc/get.config <vmid>    # Hardware-Konfiguration, VMX-Pfad
```

### Snapshots

```bash
vim-cmd vmsvc/snapshot.create <vmid> "Vor-Patch" "Beschreibung" 0 0
vim-cmd vmsvc/snapshot.get <vmid>
vim-cmd vmsvc/snapshot.removeall <vmid>
```

> [!WARNING] Snapshots im Produktivbetrieb
> Snapshots sind kein Backup-Ersatz und belasten den Datastore-I/O erheblich. Nie länger als 72 Stunden bestehen lassen.

### VM zwangsbeenden (Notfall)

```bash
# Schritt 1: World-ID der hängenden VM ermitteln
esxcli vm process list

# Schritt 2: Eskalierend beenden
esxcli vm process kill --type=soft  --world-id=<id>   # Graceful
esxcli vm process kill --type=hard  --world-id=<id>   # Erzwungen
esxcli vm process kill --type=force --world-id=<id>   # Letzter Ausweg
```

> [!DANGER] force-Kill
> Entspricht einem Stromausfall für die VM — Datenverlust und Filesystem-Korruption im Guest-OS möglich. Nur wenn `soft` und `hard` versagen.

---

## Netzwerk-Befehle

### Physische NICs

```bash
esxcli network nic list                    # Alle NICs
esxcli network nic get -n vmnic0           # Details (Speed, Duplex, Driver)
esxcli network nic stats get -n vmnic0     # Statistiken (Fehler, Throughput)
esxcli network nic down -n vmnic0          # NIC deaktivieren
esxcli network nic up   -n vmnic0          # NIC aktivieren
```

### vSwitches & VMkernel-Ports

```bash
esxcli network vswitch standard list       # Standard-vSwitches
esxcli network ip interface list           # VMkernel-Interfaces
esxcli network ip interface ipv4 get       # IP-Adressen
esxcfg-vmknic -l                           # VMkernel-NICs (klassisch)
```

### Routing & Firewall

```bash
esxcli network ip route ipv4 list                                    # Routing-Tabelle
esxcfg-route                                                         # Standard-Gateway
esxcli network firewall ruleset list                                 # Firewall-Regelsets
esxcli network firewall ruleset set --ruleset-id=sshServer --enabled=true
```

### Verbindungstest

```bash
ping -c 4 8.8.8.8                        # Standard-Ping (Management-Stack)
vmkping 192.168.10.1                     # VMkernel-Ping
vmkping -I vmk1 -s 8972 192.168.10.1    # Jumbo Frames testen (MTU 9000)
```

> [!TIP] vmkping vs. ping
> `vmkping` sendet über den VMkernel-Stack — ideal für Storage- und vMotion-Netzwerke, die vom Management-Interface aus nicht erreichbar sind.

---

## Storage-Befehle

### Übersicht

```bash
esxcli storage core device list     # Blockgeräte (lokale Disks, SAN-LUNs, NVMe)
esxcli storage filesystem list      # Datastores / VMFS-Volumes
esxcli storage nmp device list      # NMP-Pfade (Multipathing)
ls -lh /vmfs/volumes/               # Datastores im Filesystem
df -h                               # Disk-Auslastung (human-readable)
```

### VMDK-Operationen mit vmkfstools

```bash
# VMDK klonen
vmkfstools -i source.vmdk dest.vmdk

# Klonen als Thin Provision
vmkfstools -i source.vmdk dest.vmdk -d thin

# VMDK erweitern (VM muss ausgeschaltet sein!)
vmkfstools -X 100G /vmfs/volumes/datastore1/VM/vm.vmdk

# Neue leere VMDK erstellen (50 GB, Thin)
vmkfstools -c 50G -d thin /vmfs/volumes/datastore1/VM/neue-disk.vmdk

# VMDK-Metadaten anzeigen
vmkfstools -D /vmfs/volumes/datastore1/VM/vm.vmdk
```

> [!DANGER] Nur bei ausgeschalteter VM
> `vmkfstools -X` (Resize) und `-i` (Klonen) dürfen nur bei heruntergefahrener VM ausgeführt werden — bei laufender VM drohen Datenverlust und VMFS-Korruption.

---

## Log-Befehle

### Wichtige Log-Dateien

| Log-Datei | Inhalt |
|:----------|:-------|
| `/var/log/vmkernel.log` | VMkernel-Ereignisse, Storage, Netzwerk, Hardware |
| `/var/log/hostd.log` | Host-Daemon: VM-Operationen, Client-Zugriffe |
| `/var/log/vpxa.log` | vCenter-Agent-Kommunikation |
| `/var/log/esxi.log` | ESXi-Systemereignisse |
| `/var/log/syslog.log` | Allgemeiner Syslog |
| `/var/log/auth.log` | Authentifizierung, SSH-Zugriffe |
| `/var/log/shell.log` | Alle Shell-Befehle (Audit-Trail) |
| `/var/log/vobd.log` | VMkernel-Observation-Daemon (Hardware-Events) |

### Log-Analyse

```bash
tail -f /var/log/vmkernel.log                          # Live verfolgen
tail -n 100 /var/log/hostd.log                         # Letzte 100 Zeilen
grep -i error /var/log/hostd.log                       # Fehler filtern
grep -i "warn\|error\|fail" /var/log/vmkernel.log
grep -i "MeineVM" /var/log/hostd.log | tail -50        # Nach VM suchen

# Remote-Syslog konfigurieren
esxcli system syslog config set --loghost=udp://syslog.example.com:514
esxcli system syslog reload
esxcli system syslog config get
```

> [!TIP] Support-Bundle erstellen
> `vm-support` in der Shell erzeugt ein vollständiges Diagnose-Archiv unter `/var/tmp/` — per SCP für VMware-Support herunterladen.

---

## Software & Patches

```bash
# VIBs verwalten
esxcli software vib list                                # Installierte VIBs
esxcli software vib list | grep -i tools               # Filtern
esxcli software vib install -v /vmfs/volumes/.../patch.vib
esxcli software vib remove -n <vib-name>

# Profile & Offline-Updates
esxcli software profile get                            # Aktuelles Profil
esxcli software profile update \
  -d /vmfs/volumes/datastore1/VMware-ESXi-8.x-depot.zip \
  -p ESXi-8.0U3-standard
```

> [!IMPORTANT] Wartungsmodus vor Updates
> VIB-Installationen erfordern meist einen Neustart — immer zuerst Wartungsmodus aktivieren und VMs migrieren.

---

## Hardware-Informationen

```bash
esxcli hardware cpu list          # CPU-Details (Sockel, Kerne, Threads)
esxcli hardware cpu global get    # CPU-Gesamtübersicht
esxcli hardware memory get        # RAM (Gesamtgröße, nutzbar)
esxcli hardware pci list          # PCI-Geräte (NICs, HBAs, GPUs, NVMe)
esxcli hardware platform get      # Hersteller, Modell, Seriennummer
smbiosDump                        # BIOS/DMI-Rohdaten
```

---

## Nützliche Einzeiler

```bash
# Alle VMs mit World-ID und Status
esxcli vm process list

# VM-Prozess eskalierend beenden
esxcli vm process list | grep -A6 "MeineVM"
esxcli vm process kill --type=soft --world-id=<id>

# Netzwerktest — Management, vMotion, Storage
ping -c 3 192.168.1.1
vmkping -I vmk1 192.168.20.1
vmkping -I vmk2 -s 8972 192.168.30.10

# Disk-Latenz und I/O prüfen
esxcli storage core device stats get -d naa.<device-id>

# Echtzeit-Performance-Monitoring
esxtop
# Tastenkürzel: c=CPU  m=Memory  d=Disk  n=Network  v=VMs  q=Beenden

# esxtop im Batch-Modus für Skriptauswertung
esxtop -b -d 2 -n 5 > /tmp/esxtop.csv

# Datastore-Belegung kompakt
df -h | grep -v tmpfs | grep -v ramfs
```

---

## Verwandte Notizen

- [[ESXi Troubleshooting]]
- [[PowerCLI]]
- [[ESXi Best Practices]]
- [[VM Verwaltung]]
- [[ESXi Grundlagen]]
