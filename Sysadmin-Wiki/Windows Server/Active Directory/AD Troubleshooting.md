---
tags: [active-directory, troubleshooting, replikation, kerberos, dcdiag, repadmin, fehler]
aliases: ["AD Troubleshooting", "Active Directory Fehler", "DC Diagnose"]
---

# AD Troubleshooting

> [!ABSTRACT]
> Systematische Fehlerdiagnose für häufige Active Directory-Probleme: Replikationsfehler, Kerberos-Authentifizierungsprobleme, gesperrte Konten, DNS-Ausfälle, GPO wird nicht angewendet und Domain-Controller-Probleme.

---

## Diagnosewerkzeuge im Überblick

| Tool                       | Funktion                          |
| -------------------------- | --------------------------------- |
| `dcdiag`                   | DC-Gesundheitscheck (alle Tests)  |
| `repadmin`                 | Replikationsstatus und -diagnose  |
| `nltest`                   | Netlogon-Tests, DC-Suche, Trust   |
| `klist`                    | Kerberos-Tickets anzeigen/löschen |
| `nslookup`                 | DNS-Auflösung testen              |
| `netdom`                   | Domain-Diagnose, FSMO-Rollen      |
| `Get-WinEvent`             | Windows Event Log durchsuchen     |
| `Get-ADReplicationFailure` | PowerShell: Replikationsfehler    |

---

## DC-Gesundheitscheck

```powershell
# Vollständigen DC-Test ausführen
dcdiag /v

# Nur Replikations-Tests
dcdiag /test:replications

# Nur DNS-Tests
dcdiag /test:dns /v

# Tests auf Remote-DC
dcdiag /s:SRV-DC02

# Wichtige Tests:
# Connectivity      → DC erreichbar?
# Replications      → Replikation funktioniert?
# SysVolCheck       → SYSVOL korrekt?
# KccEvent          → KCC-Fehler?
# NetLogons         → Netlogon-Dienst?
# Services          → Alle AD-Dienste laufen?
# Advertising       → DC registriert sich korrekt?
```

---

## Replikationsfehler

```powershell
# Replikationsstatus aller DCs
repadmin /showrepl

# Replikationsfehler-Zusammenfassung
repadmin /replsummary

# Replizierung erzwingen
repadmin /syncall /AdP          # Alle DCs, alle Partitionen
repadmin /sync dc=firma,dc=local SRV-DC01 SRV-DC02  # Gezielt

# Replikationspartner anzeigen
repadmin /showrepl SRV-DC01

# PowerShell-Variante
Get-ADReplicationFailure -Target SRV-DC01
Get-ADReplicationPartnerMetadata -Target SRV-DC01

# Häufige Replikationsfehler:
# 8606: "Insufficient attributes were given to create an object" → Tombstone zu alt
# 8453: "Replication access was denied" → Berechtigungsproblem
# 1722: "RPC server unavailable" → Netzwerk/Firewall-Problem
# 8614: "The directory service cannot replicate" → USN-Rollback
```

### SYSVOL-Replikation prüfen

```powershell
# SYSVOL-Status (sollte "SYSVOL ready" sein)
netlogon  # Dienststatus
net share | findstr SYSVOL

# DFS-Replikation Status (moderne Methode, seit 2008 R2)
dfsrdiag pollad
Get-DfsrState

# Replikation erzwingen
dfsrdiag syncnow /partner:SRV-DC02 /rgname:"Domain System Volume" /time:1
```

---

## Kerberos-Probleme

```powershell
# Aktuelle Tickets anzeigen
klist tickets

# Tickets für Service
klist get cifs/SRV-FS01.firma.local

# Alle Tickets löschen (erzwingt neue Authentifizierung)
klist purge

# Häufige Kerberos-Fehler:
# "The time at the Primary Domain Controller is different than..." 
#   → Zeitabweichung > 5 Minuten — Zeit synchronisieren!

# Zeit synchronisieren
w32tm /resync /force
w32tm /query /status

# Kerberos-Events im Security Log
Get-WinEvent -LogName Security |
    Where-Object {$_.Id -in @(4768, 4769, 4771)} |
    Select-Object -First 20 |
    Format-Table TimeCreated, Message -AutoSize

# Event 4768: TGT Request
# Event 4769: TGS Request
# Event 4771: Pre-Authentication failed (falsches Passwort)
```

---

## DNS-Probleme (AD benötigt DNS!)

```powershell
# AD-DNS-Einträge prüfen
nslookup -type=SRV _ldap._tcp.firma.local
nslookup -type=SRV _kerberos._tcp.firma.local
nslookup SRV-DC01.firma.local

# DC registriert sich in DNS
nltest /dsregdns

# Alle DC-DNS-Einträge neu registrieren
ipconfig /registerdns
net stop netlogon && net start netlogon

# DNS-Server testen
Resolve-DnsName "_ldap._tcp.firma.local" -Type SRV
Resolve-DnsName "SRV-DC01" -Server 192.168.1.10

# DNS-Diagnose
dcdiag /test:dns /v /c
```

---

## Anmeldeprobleme

```powershell
# Gesperrte Konten
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LockedOut, BadLogonCount

# Konto entsperren
Unlock-ADAccount -Identity "jdoe"

# Ursprung von Kontosperrungen finden
# Security Event 4740 = Account locked out
Get-WinEvent -LogName Security |
    Where-Object {$_.Id -eq 4740} |
    Select-Object -First 10 |
    ForEach-Object {
        [xml]$xml = $_.ToXml()
        [PSCustomObject]@{
            Time        = $_.TimeCreated
            LockedUser  = $xml.Event.EventData.Data | Where-Object Name -eq "TargetUserName" | Select-Object -ExpandProperty "#text"
            CallerHost  = $xml.Event.EventData.Data | Where-Object Name -eq "CallerComputerName" | Select-Object -ExpandProperty "#text"
        }
    }

# Netlogon-Log (enthält Ursache der Sperrung)
Get-Content "C:\Windows\debug\netlogon.log" | Select-String -Pattern "jdoe" | Select-Object -Last 20
```

---

## Netlogon und DC-Suche

```powershell
# DC für Domain finden
nltest /dsgetdc:firma.local

# Netlogon-Dienst testen
nltest /sc_verify:firma.local

# Secure Channel zurücksetzen (bei Anmeldeproblemen mit Computerkonto)
Test-ComputerSecureChannel -Verbose
Reset-ComputerMachinePassword -Server SRV-DC01 -Credential (Get-Credential)

# Netlogon-Service neu starten
Restart-Service Netlogon

# DC-Liste für Site abfragen
nltest /dnsgetdc:firma.local
```

---

## GPO wird nicht angewendet

```powershell
# GPO-Verarbeitung auf einem Client prüfen
gpresult /r
gpresult /h C:\Temp\report.html /f

# GPO sofort anwenden
gpupdate /force

# GPO-Anwendungsereignisse im Event Log
Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" |
    Where-Object {$_.Level -le 3} |    # Warnings und Errors
    Select-Object -First 20 |
    Format-Table TimeCreated, Message -AutoSize

# Häufige GPO-Probleme:
# GPO nicht verknüpft? → GPMC prüfen
# WMI-Filter passt nicht? → Filter debuggen
# Sicherheitsfilterung → Authenticated Users vorhanden?
# Loopback Processing? → Benutzer-GPO auf Server-OU?
```

---

## FSMO-Rollen neu zuweisen

```powershell
# Aktuelle FSMO-Rolleninhaber
netdom query fsmo

# Rollen übertragen (graceful — alter DC läuft noch)
Move-ADDirectoryServerOperationMasterRole `
    -Identity SRV-DC02 `
    -OperationMasterRole PDCEmulator, RIDMaster, InfrastructureMaster

# Rollen übernehmen (seize — alter DC nicht mehr verfügbar!)
# NTDSUTIL:
ntdsutil
> roles
> connections
> connect to server SRV-DC02
> quit
> seize PDC
> seize RID master
> seize infrastructure master
> quit
> quit
```

---

## AD-Datenbank-Reparatur (Offline)

```powershell
# DSRM-Modus starten (für Offline-Reparatur)
bcdedit /set safeboot dsrepair
Restart-Computer

# In DSRM: NTDS-Datenbank prüfen
ntdsutil "activate instance ntds" "files" "integrity" quit quit

# Datenbank defragmentieren (NTDS.DIT)
ntdsutil "activate instance ntds" "files" "compact to C:\Temp\NTDS" quit quit
# Dann: alte ntds.dit durch komprimierte ersetzen

# Normalen Boot wiederherstellen
bcdedit /deletevalue safeboot
Restart-Computer
```

---

## Verwandte Notizen

- [[AD Grundlagen]] — FSMO, Replikation, DNS-Abhängigkeit
- [[AD Installation]] — Korrekte Installation vermeidet viele Probleme
- [[Windows DNS]] — DNS ist Voraussetzung für AD
- [[PowerShell Remoting]] — Remote-Diagnose via PS
