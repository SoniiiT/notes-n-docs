---
tags: [windows-server, rds, remoteapp, remotedesktop, web-access, anwendungsveröffentlichung]
aliases: ["RemoteApp", "RDS RemoteApp", "Published Applications"]
---

# RemoteApp

> [!ABSTRACT]
> RemoteApp ermöglicht das Veröffentlichen einzelner Windows-Anwendungen über RDS — Benutzer öffnen die Anwendung als wäre sie lokal installiert, sie läuft aber auf dem Server. Zugriff erfolgt über RD Web Access (Browser) oder verteilte `.rdp`-/`.msi`-Dateien.

---

## Was ist RemoteApp?

RemoteApp ist eine **RDS-Funktion zur Veröffentlichung einzelner Windows-Anwendungen** über Remote Desktop Services. Statt einen vollständigen Remote-Desktop zu öffnen, erscheint die Anwendung auf dem lokalen Desktop des Benutzers — sie sieht aus wie eine lokal installierte Anwendung, wird aber auf dem RDS Session Host-Server ausgeführt.

RemoteApp wurde mit **Windows Server 2008** eingeführt und ist seitdem Bestandteil der RDS-Rolle. Technisch ist RemoteApp eine reguläre RDP-Sitzung, bei der das Remote-Desktop-Fenster ausgeblendet ist und nur das Anwendungsfenster angezeigt wird.

### Vergleich: Volldesktop vs. RemoteApp

| Merkmal | Volldesktop (RDS) | RemoteApp |
|---------|------------------|-----------|
| **Benutzererwartung** | Ganzer Windows-Desktop remote | Einzelne Anwendung auf lokalem Desktop |
| **Integration** | Klarer Rahmen "das ist remote" | Nahtlos im lokalen Desktop integriert |
| **Taskleiste** | Remote-Desktop in einem Fenster | Anwendungs-Icons direkt in lokaler Taskleiste |
| **Zwischenablage** | Muss explizit geteilt werden | Automatisch verfügbar |
| **Typischer Anwender** | Thin-Client-Nutzer, vollständig remote | Hybridarbeiter mit lokaler + remote Software |

### Verteilungsmethoden

| Methode | Beschreibung |
|---------|-------------|
| **RD Web Access** | Webportal — Benutzer klicken Anwendung im Browser an (`https://server/rdweb`) |
| **.rdp-Datei** | Direktstarter — per E-Mail, Netzlaufwerk oder GPO verteilt |
| **.msi-Paket** | Installiert Desktop-Icon und Start-Menü-Eintrag — verhält sich wie lokale Installation |
| **GPO (RemoteApp and Desktop Connections)** | Automatische Bereitstellung für Domänen-Benutzer über Gruppenrichtlinie |

> [!NOTE] Voraussetzungen
> RemoteApp benötigt mindestens einen **RD Session Host**. Für Hochverfügbarkeit und Lastverteilung wird zusätzlich ein **RD Connection Broker** empfohlen. RD Web Access ist für die Browser-Bereitstellung erforderlich.

---

## RemoteApp veröffentlichen

### GUI

```
Server Manager → Remote Desktop Services → Collections → Desktop Collection
→ Aufgaben → RemoteApp-Programme veröffentlichen

1. Anwendung auswählen (z.B. Calculator, Notepad, eigene App)
2. RD Web Access: automatisch sichtbar unter /rdweb
3. .rdp und .msi können exportiert werden
```

### PowerShell

```powershell
# RemoteApp-Programm veröffentlichen
New-RDRemoteApp `
    -CollectionName "Desktop Collection" `
    -DisplayName "Taschenrechner" `
    -FilePath "C:\Windows\System32\calc.exe" `
    -Alias "calc" `
    -ConnectionBroker "SRV-RDCB01.firma.local"

# Mit Icon
New-RDRemoteApp `
    -CollectionName "Desktop Collection" `
    -DisplayName "Faktura Pro" `
    -FilePath "C:\Program Files\FakturaPro\faktura.exe" `
    -Alias "faktura" `
    -IconPath "C:\Program Files\FakturaPro\faktura.exe" `
    -IconIndex 0 `
    -ConnectionBroker "SRV-RDCB01.firma.local"

# Alle veröffentlichten RemoteApps anzeigen
Get-RDRemoteApp -CollectionName "Desktop Collection" -ConnectionBroker "SRV-RDCB01.firma.local"

# RemoteApp entfernen
Remove-RDRemoteApp `
    -CollectionName "Desktop Collection" `
    -Alias "calc" `
    -ConnectionBroker "SRV-RDCB01.firma.local" `
    -Force
```

---

## RD Web Access

```powershell
# RD Web Access Zertifikat konfigurieren (HTTPS, kein selbstsigniertes Zertifikat!)
Set-RDCertificate `
    -Role RDWebAccess `
    -Thumbprint "ABCDEF1234567890..." `
    -ConnectionBroker "SRV-RDCB01.firma.local" `
    -Force

# URL für Benutzer:
# https://rds.firma.local/rdweb
# Benutzer meldet sich an → sieht veröffentlichte RemoteApps

# IIS für RD Web Access konfigurieren
# Standardmäßig wird IIS auf dem RDWA-Server automatisch konfiguriert
```

---

## .rdp-Datei erstellen und verteilen

```powershell
# .rdp-Datei für RemoteApp exportieren
Export-RDPersonalSessionDesktop    # Für Full Desktop
# RemoteApp .rdp direkt aus Web Access herunterladen

# Manuelle .rdp-Datei (für Skripte/Test)
@"
full address:s:SRV-RDCB01.firma.local
remoteapplicationmode:i:1
remoteapplicationname:s:Faktura Pro
remoteapplicationprogram:s:||faktura
remoteapplicationcmdline:s:
workspace id:s:SRV-RDCB01.firma.local
gatewayhostname:s:rdgw.firma.local
gatewayusagemethod:i:1
"@ | Set-Content "C:\Temp\faktura.rdp"

# .rdp per GPO verteilen
# Benutzerkonfiguration → Einstellungen → Windows-Einstellungen → Dateien
```

---

## Sicherheit: Benutzergruppen für RemoteApp

```powershell
# Zugriff auf bestimmte Collection per Gruppe einschränken
Set-RDSessionCollectionConfiguration `
    -CollectionName "Desktop Collection" `
    -ConnectionBroker "SRV-RDCB01.firma.local" `
    -UserGroup "FIRMA\RDS-Benutzer"    # Nur diese Gruppe darf zugreifen

# Für bestimmtes RemoteApp: per RD Gateway CAP/RAP weiter einschränken
# (Wer darf, über Gateway, auf welche Ressource zugreifen)
```

---

## RemoteApp-MSI-Paket erstellen

```
In der RD Web Access GUI:
→ RemoteApp-Details anzeigen → "Create .rdp file" oder "Create .msi package"

Das MSI-Paket:
├── Installiert .rdp-Datei unter Dokumente
├── Erstellt Verknüpfung auf Desktop und im Startmenü
└── Kann per WSUS, GPO oder SCCM verteilt werden
```

---

## Troubleshooting RemoteApp

```powershell
# RemoteApp startet nicht?
# 1. Pfad auf Session Host prüfen
Test-Path "C:\Program Files\FakturaPro\faktura.exe" -PathType Leaf

# 2. Benutzer hat Zugriff auf Collection?
Get-RDSessionCollectionConfiguration -CollectionName "Desktop Collection" |
    Select-Object UserGroup

# 3. Firewall auf Session Host
Get-NetFirewallRule -DisplayGroup "Remote Desktop"

# 4. Event Log auf RDSH
Get-WinEvent -LogName "Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational" -MaxEvents 20

# 5. Test-Verbindung direkt auf Session Host (ohne Broker)
mstsc /v:SRV-RDSH01 /admin
```

---

## Verwandte Notizen

- [[RDS Grundlagen]] — Voraussetzungen und Architektur
- [[RDS Konfiguration]] — Farm-Setup, HA, Profile
- [[Group Policy]] — RemoteApp-Einstellungen per GPO
