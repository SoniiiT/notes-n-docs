---
tags: [windows-server, print-server, drucker, druckerwarteschlange, treiber, gpo, druckerfreigabe]
aliases: ["Print Server", "Druckserver", "Druckerfreigabe", "Windows Druck"]
---

# Print Server

> [!ABSTRACT]
> Windows Print Server zentralisiert die Druckerverwaltung: Treiber werden einmalig auf dem Server installiert und automatisch an Clients verteilt (Point and Print). GPO ermöglicht die automatische Druckerbereitstellung für Benutzer und Standorte.

---

## Was ist ein Print Server?

Ein Windows Print Server ist eine **zentrale Verwaltungsinstanz für Drucker und Druckwarteschlangen** im Netzwerk. Statt Druckertreiber auf jedem Client einzeln zu installieren und zu pflegen, werden Treiber einmalig auf dem Print Server hinterlegt und bei Bedarf automatisch an Clients verteilt — dieses Modell heißt **Point and Print**.

Der **Windows-Druckdienst** (Print Spooler, `spoolsv.exe`) ist seit Windows NT 3.1 (1993) Bestandteil des Betriebssystems. Die **Print Management Console** (`printmanagement.msc`) für zentrale Verwaltung mehrerer Server wurde mit **Windows Server 2003 R2** eingeführt.

### Komponenten des Drucksystems

| Komponente | Funktion |
|-----------|---------|
| **Print Spooler** (`spoolsv.exe`) | Windows-Dienst — nimmt Druckaufträge entgegen, stellt sie in die Warteschlange, leitet sie an den Drucker weiter |
| **Druckerwarteschlange** | Listet alle ausstehenden Druckaufträge; ermöglicht Priorisierung und Abbruch |
| **Druckertreiber** | Konvertiert Druckaufträge in druckerverständliches Format (PCL, PostScript, XPS) |
| **Druckeranschluss** | Verbindung zum physischen Drucker (TCP/IP-Port, USB-Port, LPR-Port) |
| **Druckprozessor** | Verarbeitet den Datentyp des Druckauftrags (RAW, EMF, TEXT) |

### Druckprotokolle

| Protokoll | Standard | Port | Beschreibung |
|-----------|---------|------|-------------|
| **SMB/Windows Printing** | Microsoft | TCP 445 | Standard für Windows-Netzwerke — Freigabe über `\\Server\Drucker` |
| **IPP** (Internet Printing Protocol) | RFC 2911 | TCP 631 | Offener Standard, plattformübergreifend |
| **LPR/LPD** | RFC 1179 | TCP 515 | Älteres Unix-Protokoll — für Legacy-Drucker und Linux-Integration |
| **WSD** (Web Services for Devices) | Microsoft/OASIS | HTTP | Automatische Druckererkennung (Plug-and-Play übers Netzwerk) |
| **RAW** | Microsoft | TCP 9100 | Direktes Senden von Druckdaten an Drucker-IP |

### Point and Print

Point and Print ist das Windows-Modell zur **automatischen Treiber-Verteilung**:

1. Admin installiert Treiber einmalig auf dem Print Server
2. Client verbindet sich erstmals mit `\\SRV-PRINT01\HP-Marketing`
3. Windows erkennt fehlenden Treiber und lädt ihn automatisch vom Server herunter
4. Bei Treiber-Updates auf dem Server: Clients erhalten die neue Version automatisch beim nächsten Verbindungsaufbau

> [!IMPORTANT] Point and Print Sicherheit
> Ab Windows 10/11 (2021-Patches, CVE-2021-34527 "PrintNightmare") verlangt Windows standardmäßig **Admin-Rechte** für Point-and-Print-Installationen von Nicht-Domain-Druckservern. In AD-Umgebungen mit korrekt konfigurierten GPOs ist das kein Problem.

### Druckerbereitstellung per GPO

Für automatische Druckerbereitstellung ohne Benutzerinteraktion:

| GPO-Methode | Zugewiesene Drucker | Geeignet für |
|-------------|---------------------|-------------|
| **Computer-Konfiguration** | Fest einem Computer zugewiesen | Öffentliche Terminals, Druckstationen |
| **Benutzer-Konfiguration** | Benutzer erhält Drucker unabhängig vom Gerät | Mobile Worker, standardisierte Abteilungsdrucker |
| **Standortbasiert** (Location-Aware Printing) | Automatisch je nach AD-Subnetz | Benutzer in verschiedenen Gebäuden/Standorten |

---

## Print Server Rolle installieren

```powershell
# Druckdienste-Rolle installieren
Install-WindowsFeature Print-Server -IncludeManagementTools

# Print Management Console
printmanagement.msc

# Druckwarteschlangen-Verwaltung (klassisch)
printui.msc
```

---

## Drucker hinzufügen

### GUI (Print Management Console)

```
Druckverwaltung (printmanagement.msc):
→ Druckserver → SRV-PRINT01 → Drucker → Rechtsklick → Drucker hinzufügen

Assistent "Netzwerkdrucker hinzufügen":
1. TCP/IP-Drucker:
   IP-Adresse oder Hostname:  192.168.1.150
   Portname:                  IP_192.168.1.150
   Druckertyp automatisch erkennen: ✓
   → Weiter

2. Treiber:
   Aus Liste wählen: HP Universal Printing PCL 6
   (oder: Datenträger → INF-Datei des Herstellers)
   → Weiter

3. Druckereinstellungen:
   Druckername:    HP-Marketing-3Etage
   Freigabename:   HP-MKT-3F
   ✓ Diesen Drucker freigeben
   Standort:       3. Etage, Raum 301
   ✓ Im Verzeichnis auflisten (AD-Veröffentlichung)
   → Weiter → Fertigstellen

Drucker entfernen:
→ Drucker → Rechtsklick → Löschen
```

### PowerShell

```powershell
# --- TCP/IP-Drucker (Netzwerkdrucker mit IP) ---

# Druckeranschluss erstellen
Add-PrinterPort `
    -Name "IP_192.168.1.150" `
    -PrinterHostAddress "192.168.1.150"

# Treiber installieren (muss in Driver Store vorhanden sein)
Add-PrinterDriver -Name "HP Universal Printing PCL 6"

# Drucker hinzufügen
Add-Printer `
    -Name "HP-Marketing-3Etage" `
    -DriverName "HP Universal Printing PCL 6" `
    -PortName "IP_192.168.1.150" `
    -ShareName "HP-MKT-3F" `
    -Shared $true `
    -Published $true `
    -Location "3. Etage, Raum 301" `
    -Comment "Marketing-Abteilung Drucker"

# Alle Drucker anzeigen
Get-Printer | Format-Table Name, ShareName, PortName, Shared, Published

# Drucker entfernen
Remove-Printer -Name "HP-Marketing-3Etage"
Remove-PrinterPort -Name "IP_192.168.1.150"
```

---

## Drucker in AD veröffentlichen

```powershell
# Drucker in AD-Verzeichnisdienst veröffentlichen (für Benutzer-Suche)
Set-Printer -Name "HP-Marketing-3Etage" -Published $true

# Drucker suchen (aus Benutzerperspektive)
# Systemsteuerung → Drucker → Drucker hinzufügen → Drucker im Verzeichnis suchen
# Filterbar nach: Name, Standort (Location), Farbe, Doppelseitig

# AD-Standort-/Location-Richtlinie:
# Computer-Standort in AD Sites → automatisches Location-Filtering
```

---

## Drucker per GPO bereitstellen

```
GPO → Benutzerkonfiguration → Einstellungen → Systemsteuerung-Einstellungen → Drucker

Methode 1: Windows-Drucker bereitstellen (einfach)
- Drucker: \\SRV-PRINT01\HP-MKT-3F
- Standarddrucker: Ja/Nein
- Element-Zielgruppenadressierung: Sicherheitsgruppe "Marketing-Mitarbeiter"

Methode 2: TCP/IP-Drucker (mit lokalem Anschluss auf Client)
- Für direkte IP-Verbindungen ohne Printserver

Methode 3: Freigegebener Drucker (veraltet, aber kompatibel)
- Verbindung zu UNC-Pfad des Druckers
```

```powershell
# GPO-Variante per PowerShell (Printer Connections GPO)
# GPO: Benutzerkonfiguration → Administrative Vorlagen →
#      Drucker → Drucker zu Verzeichnis hinzufügen oder löschen

# Oder: Anmeldeskript
Add-Printer -ConnectionName "\\SRV-PRINT01\HP-MKT-3F"
```

---

## Treiberverwaltung (Point and Print)

```powershell
# Installierte Druckertreiber anzeigen
Get-PrinterDriver | Format-Table Name, MfgName, PrinterEnvironment

# Treiber aus Treiberkatalog hinzufügen
Add-PrinterDriver -Name "HP Universal Printing PCL 6"

# Treiber vom Hersteller (INF-Datei) hinzufügen
pnputil.exe /add-driver "C:\Drivers\HP\hpupd.inf"
Add-PrinterDriver -Name "HP Universal Printing PCL 6" -InfPath "C:\Drivers\HP\hpupd.inf"

# Point and Print (Clients laden Treiber automatisch vom Server)
# GPO: Computer Konfiguration → Administrative Vorlagen → Drucker
# → "Sicherheitsaufforderungen für Point and Print deaktivieren" → Aktiviert
# → "Point and Print-Einschränkungen" → Vertrauenswürdige Server angeben

# Sicherheitshinweis: PrintNightmare (CVE-2021-1675/34527)
# Patch installieren + Gruppenrichtlinien für Druckertreiber-Installation einschränken
Set-ItemProperty `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Printers\PointAndPrint" `
    -Name "RestrictDriverInstallationToAdministrators" `
    -Value 1
```

---

## Druckerwarteschlange verwalten

```powershell
# Druckerjobs anzeigen
Get-PrintJob -PrinterName "HP-Marketing-3Etage" | Format-Table Id, UserName, Document, Size, Submitted

# Druckerjob löschen
Remove-PrintJob -PrinterName "HP-Marketing-3Etage" -ID 5

# Alle Jobs löschen
Get-PrintJob -PrinterName "HP-Marketing-3Etage" | Remove-PrintJob

# Drucker anhalten/fortsetzen
Set-Printer -Name "HP-Marketing-3Etage" -WorkOffline $true   # Anhalten
Set-Printer -Name "HP-Marketing-3Etage" -WorkOffline $false  # Fortsetzen

# Druckspooler neu starten (bei hängenden Jobs)
Restart-Service Spooler
```

---

## Print Spooler Troubleshooting

```powershell
# Spooler-Status
Get-Service Spooler

# Spooler-Warteschlange leeren (hängende Jobs)
Stop-Service Spooler
Get-ChildItem "C:\Windows\System32\spool\PRINTERS\" | Remove-Item -Force
Start-Service Spooler

# Druckerverbindung testen
Test-NetConnection -ComputerName SRV-PRINT01 -Port 9100   # RAW/Port 9100
Test-NetConnection -ComputerName SRV-PRINT01 -Port 445    # SMB/Windows Freigabe

# Druckerevents
Get-WinEvent -LogName "Microsoft-Windows-PrintService/Operational" -MaxEvents 30
```

---

## Verwandte Notizen

- [[Group Policy]] — Drucker per GPO bereitstellen
- [[AD Grundlagen]] — AD-Veröffentlichung für Drucker-Suche
- [[SMB Freigaben]] — Druckerfreigaben via SMB
