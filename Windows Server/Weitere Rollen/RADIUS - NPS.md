---
tags: [windows-server, radius, nps, 802-1x, wlan, vpn, authentifizierung, netzwerkzugriff]
aliases: ["RADIUS", "NPS", "Network Policy Server", "802.1X", "WLAN Authentifizierung"]
---

# RADIUS — Network Policy Server (NPS)

> [!ABSTRACT]
> Der Network Policy Server (NPS) ist der Microsoft RADIUS-Server. Er authentifiziert und autorisiert Netzwerkzugriffe: WLAN (802.1X/WPA2-Enterprise), VPN-Einwahl, und kabelgebundene 802.1X-Portauthentifizierung. NPS integriert sich direkt mit Active Directory für Benutzer- und Gruppenvalidierung.

---

## Was ist RADIUS?

RADIUS (Remote Authentication Dial-In User Service) ist ein **Netzwerkprotokoll für zentralisierte Authentifizierung, Autorisierung und Abrechnung** (AAA) von Netzwerkzugriffen. Es wurde 1991 von Livingston Enterprises entwickelt und ist in **RFC 2865** (Authentication/Authorization) und **RFC 2866** (Accounting) standardisiert.

RADIUS ermöglicht es, die Zugangskontrolle für heterogene Netzwerkgeräte (WLAN Access Points, VPN-Gateways, Switches) zentral über einen einzigen Server zu verwalten, der gegen Active Directory authentifiziert.

### Das AAA-Modell

| Komponente | Funktion | Beispiel |
|------------|---------|---------|
| **Authentication** (Authentifizierung) | Identität prüfen — Wer bist du? | Benutzername + Passwort oder Zertifikat gegen AD prüfen |
| **Authorization** (Autorisierung) | Berechtigungen prüfen — Was darfst du? | Welches VLAN, welche Bandbreite, welche Zugriffszeit |
| **Accounting** (Abrechnung/Protokollierung) | Verwendung aufzeichnen — Was hast du gemacht? | Session-Start/-Ende, übertragene Datenmenge (UDP Port 1813) |

### RADIUS-Kommunikationsmodell

```
Endgerät (PC/Smartphone)
    ↕ EAP (802.1X)
Authenticator (NAS: Access Point / Switch / VPN-Gateway)
    ↕ RADIUS (UDP Port 1812/1813) + Shared Secret
NPS/RADIUS-Server
    ↕ LDAP/Kerberos
Active Directory
```

**Rollentrennung:** Der Access Point (NAS — Network Access Server) übernimmt nur die Weiterleitung. Die eigentliche Entscheidung trifft der RADIUS-Server anhand von Netzwerkrichtlinien.

### EAP-Methoden

EAP (Extensible Authentication Protocol, RFC 3748) ist das Protokoll für den Authentifizierungsdialog innerhalb von RADIUS:

| EAP-Methode | Standard | Authentifizierung | Zertifikat | Sicherheit |
|-------------|---------|------------------|------------|------------|
| **PEAP-MSCHAPv2** | Microsoft | Benutzer/Passwort in TLS-Tunnel | Nur Server-Zertifikat | Hoch, Standard für WLAN |
| **EAP-TLS** | RFC 5216 | Gegenseitige Zertifikats-Authentifizierung | Client + Server | Sehr hoch, PKI erforderlich |
| **EAP-TTLS** | RFC 5281 | Beliebige Methode in TLS-Tunnel | Nur Server-Zertifikat | Hoch, plattformübergreifend |
| **LEAP** | Cisco (proprietär) | Challenge-Response | Nein | **Unsicher, nicht verwenden** |

### RADIUS-Pakettypen

| Paket | Von → An | Bedeutung |
|-------|---------|-----------|
| **Access-Request** | NAS → RADIUS | Zugriffsanfrage mit Benutzerinformationen |
| **Access-Accept** | RADIUS → NAS | Zugriff gewährt (+ optionale Attribute wie VLAN-ID) |
| **Access-Reject** | RADIUS → NAS | Zugriff verweigert |
| **Access-Challenge** | RADIUS → NAS | Weitere Informationen anfordern (EAP-Verhandlung) |
| **Accounting-Request** | NAS → RADIUS | Session-Start, -Ende oder -Update |
| **Accounting-Response** | RADIUS → NAS | Accounting bestätigt |

### NPS (Network Policy Server)

NPS ist Microsofts RADIUS-Server-Implementierung. Er verwendet zwei Richtlinientypen:

| Richtlinientyp | Funktion |
|----------------|---------|
| **Verbindungsanforderungsrichtlinie** | Entscheidet: lokal authentifizieren oder an anderen RADIUS-Server weiterleiten |
| **Netzwerkrichtlinie** | Entscheidet: Zugriff gewähren oder verweigern, Bedingungen (Gruppe, Zeit, Gerättyp), VLAN-Zuweisung |

---

---

## NPS installieren

### GUI

```
Server Manager → Verwalten → Rollen und Features hinzufügen
→ Serverrollen → Netzwerkrichtlinien- und Zugriffsdienste
  → Netzwerkrichtlinienserver (NPS)
→ Installieren

NPS in AD registrieren:
nps.msc → NPS (Lokal) → Rechtsklick →
"Server in Active Directory registrieren"
→ OK
```

### PowerShell

```powershell
# NPS-Rolle installieren
Install-WindowsFeature NPAS -IncludeManagementTools

# NPS Manager öffnen
nps.msc

# NPS in AD registrieren (für Konto-Leseberechtigung)
netsh nps add registeredserver
# Oder:
Register-NpsServer -Domain "firma.local"
```

---

## RADIUS-Client konfigurieren (Access Point / Switch / VPN)

### GUI

```
nps.msc → RADIUS-Clients und -Server → RADIUS-Clients → Rechtsklick → "Neu"

Anzeigename:    AP-Empfang
Adresse (IP):   192.168.1.30
Hersteller:     RADIUS Standard (oder Cisco, Juniper usw.)
Gemeinsamer geheimer Schlüssel:
  ⦿ Manuell
  Schlüssel: SuperGeheim123!    ← Derselbe wie im Access Point!
  Bestätigen: SuperGeheim123!
→ OK
```

### PowerShell

```powershell
# RADIUS-Client im NPS registrieren
New-NpsRadiusClient `
    -Name "AP-Empfang" `
    -Address "192.168.1.30" `           # IP des Access Points
    -SharedSecret "SuperGeheim123!" `   # Gleiches Secret wie im AP eingetragen
    -ManufacturerName "Cisco"

# Alle RADIUS-Clients anzeigen
Get-NpsRadiusClient

# RADIUS-Client entfernen
Remove-NpsRadiusClient -Name "AP-Empfang"
```

---

## Verbindungsanforderungsrichtlinie

```
NPS Manager → Richtlinien → Verbindungsanforderungsrichtlinien

Standard: "Use Windows authentication for all users"
→ NPS authentifiziert gegen AD

Für NPS-Proxy (an anderen RADIUS weiterleiten):
→ Neue Richtlinie → Weiterleiten an Remote-RADIUS-Servergruppe
```

---

## Netzwerkrichtlinie (Zugangsregeln)

```
NPS Manager → Richtlinien → Netzwerkrichtlinien

Neue Richtlinie erstellen:
1. Name: "WLAN VPN Mitarbeiter"
2. Bedingungen:
   - Windows-Gruppe: FIRMA\VPN-Benutzer
   - NAS-Port-Typ: Virtual (VPN) oder Wireless-IEEE 802.11 (WLAN)
3. Berechtigungen: Zugriff gewähren
4. Einschränkungen:
   - Authentifizierungsmethode: EAP (PEAP-MSCHAPv2 für WLAN)
   - Tageszeit, Verbindungsdauer
5. Einstellungen:
   - VLAN-Zuweisung (Tunnel-Private-Group-ID = VLAN-ID)
```

---

## WPA2-Enterprise WLAN (802.1X)

```powershell
# Ablauf:
# 1. Client verbindet mit SSID "Firma-WLAN-Sicher"
# 2. AP fragt NPS (RADIUS): Darf der User?
# 3. NPS prüft AD-Credentials via PEAP-MSCHAPv2 oder EAP-TLS (Zertifikat)
# 4. NPS antwortet: Accept + optional VLAN-Zuweisung

# EAP-Typen für 802.1X:
# PEAP-MSCHAPv2    → Benutzername + Passwort (einfach, Zertifikat nur auf Server)
# EAP-TLS          → Zertifikat auf Client (sicherer, benötigt PKI/Auto-Enrollment)

# NPS-Zertifikat (für PEAP muss NPS ein Server-Zertifikat haben)
# Über PKI/AD CS ausstellen oder kommerzielles Zertifikat importieren

# VLAN-Zuweisung per NPS (in Netzwerkrichtlinie):
# Tunnel-Type:              13 (Virtual LANs)
# Tunnel-Medium-Type:       6 (802)
# Tunnel-Pvt-Group-Id:      100 (VLAN-Nummer)
```

---

## VPN-Authentifizierung

```powershell
# NPS als RADIUS-Server für RRAS (Routing and Remote Access)
# RRAS installieren + konfigurieren
Install-WindowsFeature RemoteAccess, RRAS -IncludeManagementTools

# RRAS VPN konfigurieren
netsh ras add authtype MS-CHAP-V2

# RRAS auf NPS-Authentifizierung umstellen
# RRAS → Eigenschaften → Sicherheit → Authentifizierungsanbieter: RADIUS
# RADIUS-Server: localhost (wenn NPS auf gleichem Server) oder NPS-IP

# NPS-Netzwerkrichtlinie für VPN:
# Bedingung: NAS-Port-Typ = Virtual (VPN)
# Gruppe: FIRMA\VPN-Benutzer
# Auth: PEAP oder MS-CHAPv2
```

---

## NPS-Protokollierung

```powershell
# SQL-Protokollierung aktivieren (für Berichte/Auditing)
# NPS Manager → Buchhaltung → Einstellungen konfigurieren

# Textdatei-Protokollierung
Set-NpsAccounting `
    -EnableAccountingInterimUpdates $true `
    -AccountingOnOffRequests $true

# NPS Event Log
Get-WinEvent -LogName "Security" |
    Where-Object {$_.Id -in @(6272, 6273)} |    # 6272=Accept, 6273=Deny
    Select-Object -First 20

# NPS-Log-Dateien
Get-Content "C:\Windows\System32\LogFiles\IN*.log" -Tail 20
```

---

## Troubleshooting

```powershell
# NPS-Authentifizierungstest
# Event 6272: Netzwerkzugriffsserver hat Anforderung gewährt
# Event 6273: Netzwerkzugriffsserver hat Anforderung verweigert

# Häufige Fehler:
# "The user attempted to use an authentication method that is not enabled"
#   → Authentifizierungsmethode in Netzwerkrichtlinie falsch
#   
# "An NPS extension dynamic link library that is installed on the NPS server"
#   → NPS-DLL Fehler, Dienst neu starten

# NPS-Dienst neu starten
Restart-Service IAS    # NPS läuft als "IAS"-Dienst

# Verbindung zu NPS von RADIUS-Client testen
Test-NetConnection -ComputerName SRV-NPS01 -Port 1812
```

---

## Verwandte Notizen

- [[AD Grundlagen]] — AD-Gruppen für RADIUS-Autorisierung
- [[PKI - AD CS]] — Zertifikate für EAP-TLS
- [[Group Policy]] — 802.1X Client-Konfiguration per GPO
