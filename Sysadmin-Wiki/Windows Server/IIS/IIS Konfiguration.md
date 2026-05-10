---
tags: [windows-server, iis, url-rewrite, reverse-proxy, arr, authentication, konfiguration]
aliases: ["IIS Konfiguration", "IIS URL Rewrite", "IIS Reverse Proxy", "IIS ARR"]
---

# IIS Konfiguration

> [!ABSTRACT]
> Erweiterte IIS-Konfiguration: URL Rewrite für saubere URLs und HTTP→HTTPS-Umleitungen, ARR (Application Request Routing) als Reverse Proxy, Windows-Authentifizierung, MIME-Types, Komprimierung und Sicherheits-Hardening.

---

## URL Rewrite

URL Rewrite ermöglicht Weiterleitungen und URL-Transformationen ohne Code-Änderungen:

### GUI

```
Voraussetzung: URL Rewrite Modul installiert (Web Platform Installer oder manuell)

IIS Manager (inetmgr) → Website auswählen → URL Rewrite

Neue Regel erstellen:
→ "Regel hinzufügen..." → Leer
  Name:             HTTP zu HTTPS
  Muster:           .*
  Bedingungen:      {HTTPS}  Übereinstimmt   ^OFF$
  Aktion:           Umleitung
  Umleitungs-URL:   https://{HTTP_HOST}/{R:0}
  Umleitungstyp:    Permanent (301)
→ Übernehmen
```

### PowerShell

```powershell
# URL Rewrite Modul installieren (nicht standardmäßig dabei)
# Muss von https://www.iis.net/downloads/microsoft/url-rewrite heruntergeladen werden
# Oder per Web Platform Installer

# URL Rewrite per web.config (in Dateisystem der Website):
# D:\Websites\firma-web\web.config

# HTTP → HTTPS Weiterleitung
@"
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="HTTP to HTTPS" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="^OFF$" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{R:1}"
                  redirectType="Permanent" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
"@ | Set-Content "D:\Websites\firma-web\web.config"
```

---

## Reverse Proxy mit ARR

ARR (Application Request Routing) macht IIS zum Reverse Proxy:

### GUI

```
Voraussetzung: ARR + URL Rewrite Module installiert

IIS Manager → Server-Ebene (nicht Website!) → Application Request Routing Cache
→ "Proxyeinstellungen..." → ☑ Proxy aktivieren → Übernehmen

Website anlegen (z.B. "ReverseProxy"):
→ Website → URL Rewrite → Regel hinzufügen → Leer
  Name:    ReverseProxy zu Backend
  Muster:  (.*)
  Aktion:  Umschreiben
  URL:     http://192.168.1.100/{R:1}
→ Übernehmen
```

### PowerShell

```powershell
# ARR und URL Rewrite müssen installiert sein
# ARR: https://www.iis.net/downloads/microsoft/application-request-routing

# Reverse Proxy per web.config konfigurieren:
@"
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="ReverseProxyRule" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="http://192.168.1.100/{R:1}" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
"@ | Set-Content "D:\Websites\reverseproxy\web.config"

# ARR Server Farm (für Load Balancing auf mehrere Backend-Server)
Add-WebConfigurationProperty `
    -PSPath "MACHINE/WEBROOT/APPHOST" `
    -Filter "webFarms" `
    -Name "." `
    -Value @{name="BackendFarm"}

Add-WebConfigurationProperty `
    -PSPath "MACHINE/WEBROOT/APPHOST" `
    -Filter "webFarms/webFarm[@name='BackendFarm']/server" `
    -Name "." `
    -Value @{address="192.168.1.100"}
```

---

## Authentifizierung

```powershell
# Authentifizierungsmodi
# Anonyme Authentifizierung deaktivieren (für interne Apps)
Set-WebConfigurationProperty `
    -Filter /system.webServer/security/authentication/anonymousAuthentication `
    -Name enabled -Value False `
    -PSPath "IIS:\Sites\firma-web"

# Windows-Authentifizierung aktivieren (AD-Kerberos/NTLM)
Set-WebConfigurationProperty `
    -Filter /system.webServer/security/authentication/windowsAuthentication `
    -Name enabled -Value True `
    -PSPath "IIS:\Sites\firma-web"

# Basis-Authentifizierung (nur über HTTPS!)
Set-WebConfigurationProperty `
    -Filter /system.webServer/security/authentication/basicAuthentication `
    -Name enabled -Value True `
    -PSPath "IIS:\Sites\firma-web"

# Authentifizierung prüfen
Get-WebConfigurationProperty `
    -Filter /system.webServer/security/authentication/windowsAuthentication `
    -Name enabled `
    -PSPath "IIS:\Sites\firma-web"
```

---

## IP-Beschränkungen

```powershell
# Nur bestimmte IPs erlauben (Allowlist-Modus)
Add-WebConfigurationProperty `
    -Filter /system.webServer/security/ipSecurity `
    -PSPath "IIS:\Sites\firma-web" `
    -Name "." `
    -Value @{ipAddress="192.168.1.0"; subnetMask="255.255.255.0"; allowed="true"}

# Standardzugriff verweigern (Allowlist)
Set-WebConfigurationProperty `
    -Filter /system.webServer/security/ipSecurity `
    -PSPath "IIS:\Sites\firma-web" `
    -Name allowUnlisted -Value $false

# Einzelne IP sperren (Blocklist)
Add-WebConfigurationProperty `
    -Filter /system.webServer/security/ipSecurity `
    -PSPath "IIS:\Sites\firma-web" `
    -Name "." `
    -Value @{ipAddress="203.0.113.0"; subnetMask="255.255.255.255"; allowed="false"}
```

---

## Komprimierung (Performance)

```powershell
# Statische und dynamische Komprimierung aktivieren
Set-WebConfigurationProperty `
    -Filter /system.webServer/urlCompression `
    -PSPath "MACHINE/WEBROOT/APPHOST" `
    -Name doStaticCompression -Value True

Set-WebConfigurationProperty `
    -Filter /system.webServer/urlCompression `
    -PSPath "MACHINE/WEBROOT/APPHOST" `
    -Name doDynamicCompression -Value True

# Komprimierungslevel (1-9)
Set-WebConfigurationProperty `
    -Filter /system.webServer/httpCompression `
    -PSPath "MACHINE/WEBROOT/APPHOST" `
    -Name staticCompressionLevel -Value 9
```

---

## MIME-Types hinzufügen

```powershell
# Neuen MIME-Type hinzufügen (z.B. .woff2 für Web-Fonts)
Add-WebConfigurationProperty `
    -Filter /system.webServer/staticContent `
    -PSPath "MACHINE/WEBROOT/APPHOST" `
    -Name "." `
    -Value @{fileExtension=".woff2"; mimeType="font/woff2"}

# Alle MIME-Types anzeigen
Get-WebConfiguration "/system.webServer/staticContent/mimeMap"
```

---

## Security Headers (Hardening)

```powershell
# Wichtige Security-Header per web.config:
@"
<configuration>
  <system.webServer>
    <httpProtocol>
      <customHeaders>
        <add name="X-Content-Type-Options" value="nosniff" />
        <add name="X-Frame-Options" value="SAMEORIGIN" />
        <add name="X-XSS-Protection" value="1; mode=block" />
        <add name="Strict-Transport-Security" value="max-age=31536000; includeSubDomains" />
        <add name="Referrer-Policy" value="strict-origin-when-cross-origin" />
        <remove name="X-Powered-By" />
        <remove name="Server" />
      </customHeaders>
    </httpProtocol>
  </system.webServer>
</configuration>
"@

# Server-Header entfernen (IIS-Version nicht preisgeben)
Set-WebConfigurationProperty `
    -Filter /system.webServer/security/requestFiltering `
    -PSPath "MACHINE/WEBROOT/APPHOST" `
    -Name removeServerHeader -Value True
```

---

## IIS-Diagnose

```powershell
# IIS-Dienststatus
Get-Service W3SVC, WAS | Format-Table Name, Status

# Worker Processes anzeigen
Get-Process w3wp | Format-Table Id, CPU, WorkingSet, ProcessName

# Anwendungspool und Worker Process zuordnen
Get-WmiObject -Namespace "root\WebAdministration" -Class WorkerProcess |
    Select-Object AppPoolName, ProcessId

# HTTP-Fehler-Log
Get-Content "$env:SystemRoot\System32\LogFiles\HTTPERR\httperr*.log" -Tail 20

# Fehlgeschlagene Anfragen (FREB) aktivieren
# IIS Manager → Site → Failed Request Tracing Rules
```

---

## Verwandte Notizen

- [[IIS Grundlagen]] — Installation, Sites, AppPools, SSL-Grundlagen
- [[PKI - AD CS]] — Interne CA für IIS-Zertifikate
- [[Windows DNS]] — DNS für IIS-Websites
- [[Windows Server Installation]] — IIS-Feature bei Erstinstallation
