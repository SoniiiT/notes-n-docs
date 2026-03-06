---
tags: [powershell, remoting, winrm, pssession, invoke-command, remote-administration, jea]
aliases: ["PowerShell Remoting", "WinRM", "PSSession", "Invoke-Command"]
---

# PowerShell Remoting

> [!ABSTRACT]
> PowerShell Remoting ermöglicht die Ausführung von Befehlen auf Remote-Servern — via **WinRM** (Windows Remote Management, HTTP/HTTPS). Grundlage für moderne Windows-Administration: `Invoke-Command` für einzelne Befehle, `Enter-PSSession` für interaktive Sessions, `New-PSSession` für persistente Verbindungen.

---

## WinRM aktivieren

```powershell
# Auf Zielserver (als Admin)
Enable-PSRemoting -Force

# Was Enable-PSRemoting macht:
# 1. WinRM-Dienst starten und auf Autostart setzen
# 2. WinRM-Listener auf HTTP (Port 5985) erstellen
# 3. Firewall-Ausnahme für WinRM hinzufügen
# 4. LocalAccountTokenFilterPolicy aktivieren

# WinRM-Status prüfen
Get-Service WinRM
Test-WSMan -ComputerName SRV-DC01
```

---

## Invoke-Command — Remote-Befehl ausführen

```powershell
# Einzelner Befehl auf Remote-Server
Invoke-Command -ComputerName SRV-DC01 -ScriptBlock {
    Get-Service AD* | Select-Object Name, Status
}

# Mit Credentials (workgroup / andere Domain)
$cred = Get-Credential
Invoke-Command -ComputerName SRV-DC01 -Credential $cred -ScriptBlock {
    hostname
}

# Variablen übergeben ($using:)
$serverName = "WebApp01"
Invoke-Command -ComputerName SRV-DC01 -ScriptBlock {
    Get-Service -Name $using:serverName
}

# Auf mehreren Servern gleichzeitig
$server = @("SRV-DC01", "SRV-DC02", "SRV-FS01")
Invoke-Command -ComputerName $server -ScriptBlock {
    [PSCustomObject]@{
        Server  = $env:COMPUTERNAME
        Uptime  = (Get-Date) - (gcim Win32_OperatingSystem).LastBootUpTime
        MemFree = [math]::Round((gcim Win32_OperatingSystem).FreePhysicalMemory / 1MB, 2)
    }
} | Format-Table Server, Uptime, MemFree

# Skript-Datei remote ausführen
Invoke-Command -ComputerName SRV-DC01 -FilePath "C:\Scripts\Deploy.ps1"
```

---

## Enter-PSSession — Interaktive Remote-Session

```powershell
# Interaktive Session (wie SSH)
Enter-PSSession -ComputerName SRV-DC01

# Mit anderem User
Enter-PSSession -ComputerName SRV-DC01 -Credential (Get-Credential)

# Prompt ändert sich:
# [SRV-DC01]: PS C:\Users\Administrator\Documents>

# Session beenden
Exit-PSSession
```

---

## New-PSSession — Persistente Sessions

```powershell
# Session erstellen (bleibt offen)
$session = New-PSSession -ComputerName SRV-DC01

# Befehle in Session ausführen (Zustand bleibt erhalten!)
Invoke-Command -Session $session -ScriptBlock { $x = 42 }
Invoke-Command -Session $session -ScriptBlock { Write-Host $x }
# Ausgabe: 42 (Variable bleibt in Session!)

# Mehrere Sessions
$sessions = New-PSSession -ComputerName SRV-DC01, SRV-DC02, SRV-FS01

# Auf allen Sessions ausführen
Invoke-Command -Session $sessions -ScriptBlock {
    Get-EventLog -LogName System -EntryType Error -Newest 5
}

# Offene Sessions anzeigen
Get-PSSession

# Sessions schließen
Remove-PSSession -Session $session
Get-PSSession | Remove-PSSession  # Alle schließen
```

---

## HTTPS-Remoting (Production)

```powershell
# Zertifikat erstellen (auf Zielserver)
$cert = New-SelfSignedCertificate `
    -DnsName SRV-DC01.firma.local `
    -CertStoreLocation Cert:\LocalMachine\My

# HTTPS-Listener konfigurieren
New-Item WSMan:\localhost\Listener `
    -Transport HTTPS `
    -Address * `
    -CertificateThumbprint $cert.Thumbprint

# Firewall für HTTPS-WinRM (Port 5986)
New-NetFirewallRule `
    -DisplayName "WinRM HTTPS" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 5986 `
    -Action Allow

# Verbindung via HTTPS
Enter-PSSession `
    -ComputerName SRV-DC01.firma.local `
    -UseSSL `
    -Credential (Get-Credential)
```

---

## Trusted Hosts (Workgroup-Umgebungen)

Ohne Domain-Mitgliedschaft muss der Client den Server als vertrauenswürdig einstufen:

```powershell
# Auf CLIENT-Seite:
# Einen Server hinzufügen
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "SRV-DC01" -Force

# Mehrere Server
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "SRV-DC01,SRV-DC02,192.168.1.*" -Force

# Alle erlauben (nicht empfohlen in Produktion)
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force

# Aktuelle Trusted Hosts anzeigen
Get-Item WSMan:\localhost\Client\TrustedHosts
```

---

## Remote-Module importieren (Implicit Remoting)

```powershell
# Module von Remote-Server importieren und lokal nutzen
$session = New-PSSession -ComputerName SRV-DC01

# AD-Modul importieren (auch wenn RSAT lokal nicht installiert ist!)
Import-PSSession -Session $session -Module ActiveDirectory -Prefix Remote

# Jetzt können AD-Cmdlets lokal ausgeführt werden (laufen aber remote)
Get-RemoteADUser -Filter {SamAccountName -eq "jdoe"}

# Session beenden
Remove-PSSession $session
```

---

## WinRM-Konfiguration

```powershell
# WinRM-Konfiguration anzeigen
winrm get winrm/config
Get-WSManInstance winrm/config

# Maximale Verbindungen erhöhen
Set-Item WSMan:\localhost\Shell\MaxConcurrentUsers -Value 25
Set-Item WSMan:\localhost\Shell\MaxShellsPerUser -Value 25

# Timeout erhöhen (Standard: 60 Minuten)
Set-Item WSMan:\localhost\Shell\MaxIdleTimeoutms -Value 7200000  # 2 Stunden

# Listener anzeigen
Get-WSManInstance winrm/config/listener -Enumerate
```

---

## Verbindungsprobleme debuggen

```powershell
# Verbindungstest
Test-WSMan -ComputerName SRV-DC01
Test-NetConnection -ComputerName SRV-DC01 -Port 5985

# Häufige Fehler:
# "Access is denied"      → Benutzer hat keine Rechte, oder Credential-Problem
# "Connection refused"    → WinRM nicht aktiv, Firewall blockiert
# "Host not trusted"      → TrustedHosts nicht konfiguriert
# "Kerberos failure"      → DNS-Problem (Hostname muss auflösbar sein!)

# WinRM-Diagnose
winrm id -remote:SRV-DC01

# Firewall auf Zielserver prüfen
Get-NetFirewallRule -DisplayName "*WinRM*" | Select-Object DisplayName, Enabled
```

---

## Verwandte Notizen

- [[PowerShell für Admins]] — PowerShell-Grundlagen
- [[PowerShell Cheatsheet]] — Befehlsreferenz
- [[Windows Server Installation]] — WinRM bei der Grundinstallation aktivieren
- [[AD Troubleshooting]] — Kerberos-Probleme mit Remoting
