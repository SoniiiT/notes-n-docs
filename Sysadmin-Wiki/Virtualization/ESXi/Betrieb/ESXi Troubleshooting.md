---
tags: [virtualization, esxi, vsphere, vmware, troubleshooting, logs, diagnose, fehler]
aliases: ["ESXi Troubleshooting", "vSphere Troubleshooting", "ESXi Fehleranalyse"]
---

# ESXi Troubleshooting

> [!ABSTRACT]
> Troubleshooting in ESXi-Umgebungen erfolgt über Log-Dateien, ESXi Shell-Befehle, den DCUI für lokalen Notfallzugriff und den vSphere Client für Remote-Diagnose. Diese Dokumentation deckt häufige Fehlerbilder und deren Lösung ab.

---

## Log-Dateien

### Wichtige Log-Pfade auf ESXi

| Pfad | Inhalt |
|------|--------|
| `/var/log/vmkernel.log` | VMkernel-Kernprotokoll — Storage, Netzwerk, Treiber |
| `/var/log/hostd.log` | Host Daemon — vSphere Client Verbindungen, Host-Management-Operationen |
| `/var/log/vpxa.log` | vCenter Agent — Kommunikation zwischen Host und vCenter |
| `/var/log/vobd.log` | VM Observer Daemon — Ereignisse und Alarme |
| `/var/log/syslog.log` | Allgemeines System-Log |
| `/var/log/shell.log` | ESXi Shell Befehle (Audit) |
| `/var/log/auth.log` | Authentifizierungsereignisse (SSH, Host Client) |
| `/var/log/fdm.log` | vSphere HA — Fault Domain Manager |
| VM-Verzeichnis `vmware.log` | VM-spezifisches Log (auf dem Datastore) |

### Logs im vSphere Client einsehen

```
Host Client → Host → Überwachen → Protokolle
→ Log auswählen → Aktuelle Einträge werden angezeigt

Oder:
vSphere Client (vCenter) → Host → Konfigurieren → System → Erweiterte Einstellungen
→ Protokollebene festlegen
```

### Support Bundle erstellen

```
Host Client → Host → Aktionen → Support → Support-Bundle generieren
→ ZIP wird erstellt → Herunterladen

Oder via ESXi Shell:
vm-support -s -d /tmp/support
```

---

## ESXi Shell — Häufige Befehle

SSH oder DCUI-Shell auf dem ESXi-Host:

### Host-Status

```bash
# ESXi-Version und Build anzeigen
vmware -v

# Systeminfo
esxcli system version get
esxcli hardware platform get

# Aktive Prozesse
esxcli vm process list

# NIC-Informationen
esxcli network nic list
esxcli network nic stats get --nic-name vmnic0

# vSwitch-Konfiguration anzeigen
esxcli network vswitch standard list
esxcli network vswitch dvs vmware list

# IP-Konfiguration der VMkernel-Interfaces
esxcli network ip interface ipv4 get

# Routing-Tabelle
esxcli network ip route ipv4 list

# DNS-Konfiguration
esxcli network ip dns server list
esxcli network ip dns search list
```

### Storage

```bash
# Datastore-Übersicht
esxcli storage filesystem list

# Verbundene Storage-Geräte
esxcli storage core device list

# iSCSI-Verbindungen
esxcli iscsi adapter list
esxcli iscsi session list

# Storage-Pfade (Multipath)
esxcli storage core path list
```

### Dienste

```bash
# Alle laufenden Dienste
/etc/init.d/ (Dienste auflisten)
esxcli system process list

# Hostd neu starten (wenn Host Client nicht reagiert)
/etc/init.d/hostd restart

# Management Network neu starten
/etc/init.d/management-network restart

# SSH starten/stoppen
/etc/init.d/SSH start
/etc/init.d/SSH stop
```

---

## Häufige Fehler und Lösungen

### Host nicht in vCenter erreichbar

**Symptom:** Host erscheint als "Nicht reagierend" in vCenter.

```
Diagnose:
1. Ping von vCenter zum ESXi-Host testen
2. SSH auf ESXi-Host prüfen
3. vpxa-Log prüfen: tail -f /var/log/vpxa.log
4. Hostd-Log prüfen: tail -f /var/log/hostd.log

Lösung A — vpxa neu starten:
/etc/init.d/vpxa restart

Lösung B — hostd neu starten:
/etc/init.d/hostd restart

Lösung C — Host in vCenter re-konfigurieren:
vSphere Client → Host → Rechtsklick
→ vCenter-Verbindung → Host erneut verbinden
```

### VM lässt sich nicht starten (locked)

**Symptom:** "Unable to access a file since it is locked."

```
Ursache: VM-Dateien (.vmdk, .vmx) sind von einem anderen Host gesperrt
         (passiert nach ungeplanten Ausfällen oder fehlgeschlagenen Migrationen)

Diagnose:
esxcli vm process list
vmkfstools -D /vmfs/volumes/datastore/VM/VM.vmdk

Lösung:
1. Alle Snapshots prüfen und konsolidieren
2. VM ausschalten (nicht nur stoppen)
3. vmx-Datei aus dem Datastore re-registrieren:
   Datastores → VM-Ordner → vmx-Datei → Rechtsklick → "VM registrieren"

Falls das fehlschlägt (SSH):
vim-cmd vmsvc/getallvms             # VM-IDs auflisten
vim-cmd vmsvc/power.off <ID>       # VM stoppen
vim-cmd vmsvc/unregister <ID>      # VM deregistrieren
vim-cmd solo/registervm /vmfs/volumes/datastore/VM/VM.vmx  # Re-registrieren
```

### vMotion schlägt fehl

**Symptom:** "The virtual machine is configured to use a device that prevents the migration."

```
Häufige Ursachen:
1. CD/DVD-Laufwerk ist mit ISO verbunden (Datastore-ISO auf Quell-Host)
   → VM → Hardware → CD/DVD → Keine verbinden / Client-Gerät

2. CPUs der Hosts inkompatibel (ohne EVC)
   → Cluster → EVC aktivieren (passender CPU-Modus)

3. Kein gemeinsamer Datastore (bei Standard vMotion)
   → Shared Nothing vMotion nutzen oder gemeinsamen Datastore bereitstellen

4. vMotion-Netzwerk nicht konfiguriert
   → VMkernel-Port mit aktiviertem vMotion-Dienst prüfen
```

### vSphere HA startet VMs nicht nach Ausfall

**Symptom:** VMs wurden nach Host-Ausfall nicht neu gestartet.

```
Ursachen:
1. Admission Control blockiert Neustart (zu wenig freie Ressourcen)
   → Cluster → Konfigurieren → vSphere-Verfügbarkeit → Zugangskontrolle
   → Reservierten Prozentsatz reduzieren oder Hosts hinzufügen

2. HA nicht konfiguriert / Cluster-Dienst inaktiv
   → fdm.log prüfen: tail -f /var/log/fdm.log

3. Alle Hosts gleichzeitig ausgefallen (Stromausfall o.ä.)
   → HA kann VMs nur auf laufenden Hosts neu starten
```

### Datastore voll

**Symptom:** "No space left on device" oder VMs pausieren unerwartet.

```
Sofortmaßnahme:
1. Alte Snapshots löschen (häufigste Ursache!)
   → VM → Aktionen → Snapshots → Alle löschen

2. Nicht genutzte ISOs entfernen
   → Datastore Browser → ISO-Ordner durchsuchen

3. Verwaiste VM-Dateien suchen
   esxcli storage filesystem list
   ls -lah /vmfs/volumes/datastore-name/

Langfristig:
→ Storage vMotion: VMs auf größeren Datastore migrieren
→ VMFS-Datastore erweitern (LUN-Erweiterung auf Storage-Seite, dann VMFS rescan)
```

### ESXi Host reagiert nicht — PSOD (Purple Screen of Death)

**Symptom:** Lila Bildschirm mit Kernel-Dump-Information (ähnlich Windows BSoD).

```
PSOD dokumentieren:
→ Foto/Screenshot des PSOD machen (Fehlercode, Stack Trace)
→ Core Dump wird nach Neustart unter /var/core/ gespeichert

Häufige Ursachen:
- Fehlerhafte Hardware (RAM, NIC, HBA)
- Treiberproblem (Drittanbieter-VIB)
- ESXi-Bug

Diagnose nach Neustart:
cat /var/core/vmkernel-zdump* | less
esxcli system coredump file list

Support-Bundle für VMware einreichen:
vm-support -s
```

---

## Netzwerk-Diagnose

```bash
# Ping von ESXi
vmkping 8.8.8.8
vmkping -I vmk1 192.168.30.1    # Ping über spezifisches VMkernel-Interface

# DNS auflösen
nslookup vcenter.firma.local

# Netzwerkverbindungen anzeigen
esxcli network connection list

# Portgruppen-VLANs prüfen
esxcli network vswitch standard portgroup list
```

---

## Wartungsmodus

Vor Arbeiten an einem Host immer in den Wartungsmodus wechseln — VMs werden dann automatisch per vMotion migriert (bei konfiguriertem DRS).

```
vSphere Client → Host → Rechtsklick
→ Wartungsmodus → Wartungsmodus aufrufen
→ ✓ Virtuelle Maschinen migrieren (vMotion, benötigt DRS)

Wartungsmodus beenden:
→ Host → Rechtsklick → Wartungsmodus → Wartungsmodus beenden
```

```powershell
# PowerCLI
Set-VMHost -VMHost esxi01.firma.local -State Maintenance
# Arbeiten durchführen...
Set-VMHost -VMHost esxi01.firma.local -State Connected
```

---

## Verwandte Notizen

- [[ESXi Grundlagen]]
- [[ESXi Installation]]
- [[vCenter]]
- [[ESXi Netzwerk und Storage]]
