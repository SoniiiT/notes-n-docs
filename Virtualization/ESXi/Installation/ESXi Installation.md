---
tags: [virtualization, esxi, vsphere, vmware, installation, konfiguration]
aliases: ["ESXi Installation", "ESXi einrichten", "vSphere Installation"]
---

# ESXi Installation

> [!ABSTRACT]
> Diese Anleitung führt durch die Installation von ESXi 8.x auf einem physischen Server — von den Hardwarevoraussetzungen über die Grundkonfiguration im DCUI bis zur ersten VM. Die Verwaltung über den Browser-basierten vSphere Client (Host Client) ist der Standard ohne vCenter.

---

## Was ist ESXi?

Siehe [[ESXi Grundlagen]] für Theorie, Architektur, Lizenzierung und Produktfamilie.

---

## Voraussetzungen

### Hardware-Anforderungen

| Komponente | Minimum | Empfohlen |
|-----------|---------|-----------|
| **CPU** | 64-bit mit Intel VT-x oder AMD-V | Intel VT-x + VT-d / AMD-V + IOMMU |
| **RAM** | 8 GB | 64 GB+ (je nach VM-Last) |
| **Systemspeicher** | 8 GB (Boot-USB/SD) | 32 GB SSD für ESXi |
| **VM-Storage** | 1 SATA/SAS HDD/SSD | NVMe SSDs oder SAN-Anbindung |
| **Netzwerk** | 1x 1 GbE NIC | 2+ NICs (Management + VM-Traffic getrennt) |

> [!NOTE] Hardware Compatibility List (HCL)
> VMware pflegt eine offizielle HCL unter `compatibilityguide.broadcom.com`. Für Produktionsumgebungen nur HCL-zertifizierte Hardware verwenden. Consumer-Hardware (Intel NUC, Consumer-Mainboards) läuft meist, wird aber nicht offiziell unterstützt.

### BIOS/UEFI Einstellungen

Vor der Installation sicherstellen, dass folgende Optionen im BIOS/UEFI aktiviert sind:

- **Intel:** `Intel Virtualization Technology (VT-x)` und `VT-d` (für Passthrough)
- **AMD:** `AMD-V (SVM Mode)` und `AMD-Vi / IOMMU`
- **Secure Boot:** Deaktivieren (ESXi 7+ unterstützt Secure Boot, aber einfacher ohne)
- **Boot-Reihenfolge:** USB/DVD zuerst für Installation

### ESXi-Image herunterladen

```
Broadcom Customer Connect Portal (registrierter Account erforderlich):
https://support.broadcom.com → Software → VMware vSphere

Für kostenloses ESXi 8 (Non-Production):
https://support.broadcom.com/group/ecx/free-downloads
→ VMware vSphere Hypervisor (ESXi) 8.x

ISO auf USB-Stick schreiben:
Linux/macOS:  dd if=VMware-VMvisor-Installer-8.x-XXXXX.iso of=/dev/sdX bs=1M
Windows:      Rufus → ISO-Modus → GPT → Starten
```

---

## Installation

### GUI (Installer)

```
1. Server von USB/DVD booten
   → ESXi-Boot-Menü erscheint → "ESXi Installer" auswählen

2. Willkommensbildschirm → Enter

3. EULA → F11 (Accept and Continue)

4. Installationsziel wählen:
   → Verfügbare Datenträger werden angezeigt
   → Ziel-Disk auswählen (für ESXi selbst — min. 8 GB)
   → Enter → Warnung "Disk wird überschrieben" → Enter

5. Tastaturlayout: Deutsch → d → Enter

6. Root-Passwort setzen (min. 7 Zeichen, Komplexitätsanforderung):
   → Passwort eingeben → Bestätigen → Enter

7. Installations-Bestätigung:
   → F11 (Install) → Installation läuft (~5 Minuten)

8. Installation abgeschlossen:
   → USB/DVD entfernen
   → Enter (Neustart)
```

---

## Erstkonfiguration im DCUI

Nach dem Neustart erscheint der **DCUI** (Direct Console User Interface) — die lokale Textkonsole von ESXi.

### Management-Netzwerk konfigurieren

```
DCUI → F2 (Customize System)
→ Root-Passwort eingeben

→ Configure Management Network

   Network Adapters:
   → Verfügbare NICs auswählen (primäre Management-NIC)
   → Empfehlung: dedizierte NIC für Management

   VLAN (optional):
   → VLAN ID eingeben wenn Management-Traffic in eigenem VLAN

   IPv4 Configuration:
   ⦿ Set static IPv4 address and network configuration
      IP-Adresse:     192.168.1.10
      Subnetzmaske:   255.255.255.0
      Gateway:        192.168.1.1
   oder:
   ⦿ Use dynamic IPv4 address and network configuration (DHCP)

   DNS Configuration:
   → Primary DNS Server:    192.168.1.1
   → Secondary DNS Server:  8.8.8.8
   → Hostname:              esxi01.firma.local

→ Enter (Einstellungen übernehmen)
→ Escape → Y (Netzwerk neu starten)
```

### SSH aktivieren (optional, für Remote-Zugriff)

```
DCUI → F2 → Troubleshooting Options
→ Enable SSH → Enter
→ Enable ESXi Shell → Enter (optional)
→ Escape
```

> [!WARNING] SSH im Produktionsbetrieb
> SSH und ESXi Shell sollten im Produktionsbetrieb **nur bei Bedarf** aktiviert werden. Im Normalbetrieb reicht der vSphere Host Client (Web-UI). Dauerhaft aktives SSH ist ein Sicherheitsrisiko und generiert Health-Warnungen in vCenter.

---

## Verwaltung über vSphere Host Client

### Browser-Zugriff

```
Browser → https://192.168.1.10
→ Sicherheitswarnung ignorieren (self-signed cert)
→ Username: root
→ Passwort: (beim Installieren gesetzt)

Oder direkt: https://192.168.1.10/ui
```

### Erste Aufgaben im Host Client

```
Navigator (links):

Host
├── Manage → Networking → Virtual Switches → Standardkonfiguration prüfen
├── Manage → Storage → Datastores → lokale Datenspeicher anzeigen
├── Manage → Security & Users → Users → weitere Admin-Konten anlegen
└── Manage → System → Time & Date → NTP konfigurieren!
```

### NTP konfigurieren

```
Host → Manage → System → Time & Date
→ "Use Network Time Protocol (Enable NTP client)"
→ NTP Service Status: Running
→ NTP Servers: 192.168.1.1, pool.ntp.org
→ NTP Startup Policy: Start and stop with host

→ Speichern
```

---

## Virtuelle Maschine erstellen

### GUI (vSphere Host Client)

```
Host Client → Virtuelle Maschinen → VM erstellen/registrieren

Schritt 1: Erstellungstyp
→ ⦿ Neue virtuelle Maschine erstellen

Schritt 2: Name und Gastbetriebssystem
→ VM-Name:        WS-APP01
→ Kompatibilität:  ESXi 8.0 und höher
→ Gastbetriebssystemfamilie: Windows
→ Gastbetriebssystemversion: Microsoft Windows Server 2022 (64-Bit)

Schritt 3: Storage
→ Datastore auswählen (z.B. "datastore1")

Schritt 4: Einstellungen anpassen
→ CPU:    4 vCPU
→ RAM:    8 GB (Memory Hot Add aktivieren empfohlen)
→ HDD:    80 GB (Thin Provision)
→ CD/DVD: Datenspeicher-ISO-Datei → ISO auswählen
→ Netzwerk: VM Network

Schritt 5: Abschließen → Fertig stellen

VM starten → Console öffnen → Betriebssystem installieren
```

---

## VMware Tools installieren

VMware Tools verbessern Performance und Integration deutlich:

### GUI

```
VM auswählen → Aktionen → Gastbetriebssystem → VMware Tools installieren
→ In der VM: Setup starten → Vollständige Installation → Neustart
```

### PowerShell (innerhalb der Windows-VM)

```powershell
# VMware Tools aus Windows heraus aktualisieren (wenn bereits installiert)
& "C:\Program Files\VMware\VMware Tools\VMwareToolsd.exe" --cmd "update"

# Status prüfen
Get-Service -Name VMTools
```

---

## Verwandte Notizen

- [[ESXi Grundlagen]]
- [[vCenter]]
- [[ESXi Netzwerk und Storage]]
- [[ESXi Troubleshooting]]
