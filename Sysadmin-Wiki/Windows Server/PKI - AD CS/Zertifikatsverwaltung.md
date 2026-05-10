---
tags: [windows-server, pki, zertifikate, erneuerung, revocation, subordinate-ca, ocsp, troubleshooting]
aliases: ["Zertifikatsverwaltung", "PKI Betrieb", "CA Verwaltung", "Zertifikate erneuern"]
---

# Zertifikatsverwaltung

> [!ABSTRACT]
> Laufende Verwaltung einer Windows PKI: Zertifikate erneuern, sperren, CRL-Verteilung überwachen, CA-Zertifikat erneuern, Datenbank bereinigen und PKI-Fehler diagnostizieren.

---

## CA-Überwachung

```powershell
# CA-Dienststatus
Get-Service CertSvc

# Alle ausgestellten Zertifikate
certutil -view -out "Issued" | Select-String "Certificate Template"

# CA-Datenbankgröße und ausstehende Anforderungen
certutil -getconfig
certutil -cainfo

# Ausstehende Anforderungen (warten auf manuelles Approve)
certutil -pend
```

---

## Zertifikate ausstellen, genehmigen, ablehnen

### GUI

```
certsrv.msc → [CA-Name] → Ausstehende Anforderungen

Anforderung genehmigen:
→ Ausstehende Anforderungen → Anforderung auswählen
→ Rechtsklick → Alle Tasks → Ausstellen
→ Zertifikat erscheint nun unter "Ausgestellte Zertifikate"

Anforderung ablehnen:
→ Ausstehende Anforderungen → Anforderung auswählen
→ Rechtsklick → Alle Tasks → Verweigern
```

### PowerShell

```powershell
# Ausstehende Anforderungen anzeigen
Get-CACertificate -Status Pending

# Anforderung genehmigen (ID aus der Liste)
Approve-CACertificateRequest -RequestID 42

# Anforderung ablehnen
Deny-CACertificateRequest -RequestID 42

# Zertifikat direkt ausstellen (ohne manuelle Genehmigung — für automatische Templates)
# In Template: "Automatically issue certificate" aktivieren
```

---

## Zertifikate sperren (Revocation)

### GUI

```
certsrv.msc → [CA-Name] → Ausgestellte Zertifikate

Zertifikat suchen:
→ Ansicht → Spalten hinzufügen/entfernen → Seriennummer, Antragsteller
→ Rechtsklick auf Zertifikat → Alle Tasks → Zertifikat sperren

Sperrgrund auswählen:
  Schlüsselkompromittierung   ← Bei Schlüsseldiebstahl (Datum: jetzt)
  Nicht mehr in Betrieb       ← Dienst eingestellt
  Ersetzt                     ← Durch neues Zertifikat erneuert
  Zertifikat angehalten       ← Temporäre Sperrung (kann rückgängig gemacht werden)
→ Datum und Uhrzeit → OK

CRL veröffentlichen:
→ certsrv.msc → [CA-Name] → Gesperrte Zertifikate →
  Rechtsklick → Alle Tasks → Veröffentlichen → OK
```

### PowerShell

```powershell
# Zertifikat sperren
$cert = Get-CACertificate -RequestID 42
Revoke-CACertificate `
    -SerialNumber $cert.SerialNumber `
    -Reason CeaseOfOperation    # Grund: Unspecified, KeyCompromise, CACompromise, 
                                 #        AffiliationChanged, Superseded, CeaseOfOperation

# Sperrgründe:
# KeyCompromise       → Privater Schlüssel kompromittiert (sofort wirksam)
# Superseded          → Durch neues Zertifikat ersetzt
# CeaseOfOperation    → Dienst eingestellt

# Nach Sperrung: CRL veröffentlichen
certutil -crl

# CRL prüfen
certutil -verify -urlfetch <Zertifikat.cer>
```

---

## Zertifikat erneuern

```powershell
# Zertifikat manuell erneuern (auf Client)
# Methode 1: Auto-Renewal per GPO (wenn in Template konfiguriert)
gpupdate /force    # Löst Auto-Enrollment aus

# Methode 2: certlm.msc → Persönlich → Zertifikat → Alle Tasks → Erneuern

# Methode 3: PowerShell
Get-ChildItem "Cert:\LocalMachine\My" |
    Where-Object {$_.NotAfter -lt (Get-Date).AddDays(30)} |
    ForEach-Object {
        $templateName = ($_.Extensions | Where-Object {$_.Oid.FriendlyName -eq "Certificate Template Information"}).Format($true)
        Write-Host "Erneuert: $($_.Subject) — Ablauf: $($_.NotAfter)"
        # certreq -enroll -machine -q "$templateName"
    }
```

---

## CA-Zertifikat erneuern

```powershell
# Issuing CA-Zertifikat erneuern (läuft ab)
# Muss von Root CA neu signiert werden

# Auf Issuing CA:
certutil -renew ReuseKeys    # Vorhandene Schlüssel wiederverwenden
# oder
certutil -renew              # Neue Schlüssel generieren (sicherer)

# CSR an Root CA senden
# Root CA: CSR einreichen, signieren
# Issuing CA: Zertifikat installieren
certutil -installcert "issuing-ca-new.cer"

# Dienst neu starten
Restart-Service CertSvc
```

---

## CRL Verteilung und Monitoring

```powershell
# CRL Distribution Points anzeigen
Get-CACRLDistributionPoint

# CRL manuell veröffentlichen
certutil -crl

# CRL auf Webserver (CDP URL) überprüfen
$crlUrl = "http://pki.firma.local/CertEnroll/Firma-Issuing-CA.crl"
Invoke-WebRequest -Uri $crlUrl -UseBasicParsing | Select-Object StatusCode

# CRL-Gültigkeitsdauer konfigurieren
certutil -setreg ca\CRLPeriodUnits 1
certutil -setreg ca\CRLPeriod "Weeks"
certutil -setreg ca\CRLDeltaPeriodUnits 1
certutil -setreg ca\CRLDeltaPeriod "Days"

Restart-Service CertSvc
```

---

## PKI-Datenbank bereinigen

```powershell
# Abgelaufene Zertifikate aus CA-Datenbank löschen
certutil -deleterow 01/01/2023 Expired     # Alle vor diesem Datum abgelaufenen
certutil -deleterow 01/01/2023 Request     # Ausstehende Anfragen vor Datum

# Defragmentierung der CA-Datenbank
# Auf CA-Server (Dienst muss laufen):
certutil -dbinfo    # Datenbankgröße anzeigen

# Offline-Komprimierung (Dienst anhalten):
Stop-Service CertSvc
esentutl /d "C:\Windows\system32\CertLog\<CA-Name>.edb"
Start-Service CertSvc
```

---

## PKI-Troubleshooting

```powershell
# Zertifikatspfad prüfen (vertraute Kette)
certutil -verify -urlfetch <Zertifikat.cer>

# CA-Konnektivität testen
certutil -ping <CA-Server>

# PKI-Zustand im AD anzeigen
certutil -viewstore "ldap:///CN=Certification Authorities,CN=Public Key Services,CN=Services,CN=Configuration,DC=firma,DC=local"

# Root-CA-Zertifikat im AD prüfen
certutil -store -enterprise Root

# Fehlersuche bei Auto-Enrollment
# Event Log: Application → Source: AutoEnrollment
Get-WinEvent -LogName Application |
    Where-Object {$_.ProviderName -eq "AutoEnrollment"} |
    Select-Object -First 10

# Zertifikat-Anforderungsdetails
certutil -req <request.csr>

# OCSP-Antwort prüfen
certutil -ocsp <Zertifikat.cer>
```

---

## PKI Health Check (Enterprise PKI Snap-In)

```powershell
# PKI View öffnen (in CA-Tools enthalten)
pkiview.msc

# Zeigt:
# ├── CA-Status (Online/Offline)
# ├── CRL-Status (aktuell, Ablauf)
# ├── AIA-URLs (erreichbar?)
# └── CDP-URLs (erreichbar?)

# Häufige Fehler in pkiview.msc:
# "CRL is not yet valid"  → Zeitzone/Uhr falsch
# "AIA not accessible"    → Webserver für AIA-Verteilung nicht erreichbar
# "CA certificate expired" → CA-Zertifikat erneuern!
```

---

## Verwandte Notizen

- [[PKI - AD CS]] — Installation und Grundkonfiguration
- [[AD Grundlagen]] — AD-Veröffentlichung von PKI-Objekten
- [[IIS Grundlagen]] — CDP/AIA-Webserver für CRL-Distribution
- [[Group Policy]] — Auto-Enrollment GPO-Konfiguration
