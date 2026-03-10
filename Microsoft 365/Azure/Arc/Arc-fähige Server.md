---
tags:
  - azure
  - arc
  - server
  - hybrid-cloud
aliases:
  - Arc-enabled Servers
  - Arc Server
  - Connected Machine
---

# Arc-fähige Server

> [!ABSTRACT] TL;DR
> Arc-fähige Server (Arc-enabled Servers) ermöglichen die Verwaltung von Windows- und Linux-Servern außerhalb von Azure über den Azure Resource Manager. Der **Connected Machine Agent** stellt die Verbindung her und ermöglicht die Nutzung von Extensions, Azure Monitor, Defender for Cloud, Update Manager, ESU und SSH-Zugang direkt aus dem Azure Portal.

---

## Was sind Arc-enabled Servers?

**Arc-enabled Servers** sind physische oder virtuelle Windows/Linux-Maschinen, die per **Connected Machine Agent** mit Azure verbunden wurden. Nach der Registrierung erscheinen sie als Ressourcen vom Typ `microsoft.hybridcompute/machines` im Azure Resource Manager.

Dies ermöglicht:
- Zentrales Inventory aller Server unabhängig vom Standort
- Azure Policy und Compliance-Prüfung
- Extensions wie Azure Monitor Agent, Defender, Custom Script
- SSH-Zugang ohne VPN oder öffentliche IP
- Extended Security Updates (ESU) für End-of-Support-Systeme

---

## Unterstützte Betriebssysteme

### Windows
| Betriebssystem | Architektur |
|---|---|
| Windows Server 2012 / 2012 R2 | x64 |
| Windows Server 2016 | x64 |
| Windows Server 2019 | x64 |
| Windows Server 2022 | x64 |
| Windows 10 (IoT/Enterprise) | x64 |
| Windows 11 | x64 |

### Linux
| Distribution | Versionen | Architektur |
|---|---|---|
| Ubuntu | 18.04, 20.04, 22.04, 24.04 | x64, ARM64 |
| RHEL / CentOS | 7.x, 8.x, 9.x | x64 |
| AlmaLinux | 8.x, 9.x | x64 |
| Rocky Linux | 8.x, 9.x | x64 |
| SUSE Linux Enterprise | 12 SP3+, 15 | x64 |
| openSUSE Leap | 15.x | x64 |
| Debian | 10, 11, 12 | x64, ARM64 |
| Oracle Linux | 7.x, 8.x, 9.x | x64 |
| Amazon Linux | 2, 2023 | x64 |
| Azure Linux (CBL-Mariner) | 1.0, 2.0 | x64, ARM64 |

> [!WARNING] 32-Bit nicht unterstützt
> Azure Arc unterstützt keine 32-Bit-Architekturen. ARM64-Unterstützung ist auf bestimmte Linux-Distributionen beschränkt; für Windows ist aktuell nur x64 unterstützt.

---

## Connected Machine Agent

### Architektur und Komponenten

Der Agent besteht aus vier Hauptkomponenten, die als separate Dienste laufen:

```mermaid
graph TD
    subgraph ConnectedMachineAgent
        HIMDS[HIMDS\nHybrid Instance Metadata Service]
        GC[GuestConfigAgent\nPolicy und Compliance]
        EXT[ExtensionManagerAgent\nExtensions verwalten]
        PROXY[ArcProxyService\nTraffic-Aggregation]
    end

    subgraph Azure
        ARM[Azure Resource Manager]
        POLICY[Azure Policy]
        EXTREPO[Extension Repository]
    end

    HIMDS --> PROXY
    GC --> PROXY
    EXT --> PROXY
    PROXY -->|HTTPS 443| ARM
    ARM --> POLICY
    ARM --> EXTREPO
```

| Dienst | Windows-Name | Linux-Name | Aufgabe |
|---|---|---|---|
| **HIMDS** | himds | himds | Identität, Managed Identity Token, Heartbeat |
| **GuestConfigAgent** | GCService | gcad | Policy-Compliance (DSC/Chef InSpec) |
| **ExtensionManager** | ExtensionService | extd | Extensions installieren/aktualisieren/entfernen |
| **ArcProxyService** | ArcProxyService | arcproxyd | Netzwerk-Proxy (ab v1.51, standardmäßig aktiv) |

### Agent-Version prüfen

```powershell
# Windows
azcmagent version

# Status aller Dienste
azcmagent show
```

```bash
# Linux
azcmagent version

# Verbindungsstatus
azcmagent show
```

---

## Installation des Connected Machine Agent

### Methode 1: Interaktiv (einzelner Server)

```powershell
# Windows — PowerShell als Administrator
$env:SUBSCRIPTION_ID = "<subscriptionId>"
$env:RESOURCE_GROUP = "<resourceGroup>"
$env:TENANT_ID = "<tenantId>"
$env:LOCATION = "westeurope"
$env:AUTH_TYPE = "principal"
$env:CORRELATION_ID = "<correlationId>"
$env:CLOUD = "AzureCloud"

# Agent herunterladen und installieren
Invoke-WebRequest -Uri "https://aka.ms/azcmagent-windows" -OutFile "$env:TEMP\install_windows_azcmagent.ps1"
& "$env:TEMP\install_windows_azcmagent.ps1"

# Mit Azure verbinden
azcmagent connect `
  --subscription-id $env:SUBSCRIPTION_ID `
  --resource-group $env:RESOURCE_GROUP `
  --tenant-id $env:TENANT_ID `
  --location $env:LOCATION `
  --cloud $env:CLOUD
```

```bash
# Linux — als root
export subscriptionId="<subscriptionId>"
export resourceGroup="<resourceGroup>"
export tenantId="<tenantId>"
export location="westeurope"

# Agent installieren
wget https://aka.ms/azcmagent-linux -O /tmp/install_linux_azcmagent.sh
bash /tmp/install_linux_azcmagent.sh

# Mit Azure verbinden (Device Code Flow)
azcmagent connect \
  --subscription-id "$subscriptionId" \
  --resource-group "$resourceGroup" \
  --tenant-id "$tenantId" \
  --location "$location" \
  --cloud "AzureCloud"
```

### Methode 2: Skript-basiert (Service Principal)

```powershell
# Service Principal für Onboarding verwenden
azcmagent connect `
  --subscription-id "<subscriptionId>" `
  --resource-group "<resourceGroup>" `
  --tenant-id "<tenantId>" `
  --location "westeurope" `
  --service-principal-id "<spClientId>" `
  --service-principal-secret "<spSecret>" `
  --cloud "AzureCloud" `
  --tags "Environment=Production,Owner=IT"
```

### Methode 3: Skaliert via Azure Policy

Azure Policy mit **DeployIfNotExists**-Effekt kann den Arc-Agent automatisch auf Azure-VMs deployen — für Hybrid-Szenarien empfiehlt sich die Kombination mit einem Konfigurations-Management-Tool (SCCM, Ansible, Chef).

```json
{
  "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/bfea026e-043f-4ff4-9d1b-bf301ca7ff46",
  "displayName": "Configure Windows machines to run Azure Monitor Agent",
  "effect": "DeployIfNotExists"
}
```

### Methode 4: Skript-generiert im Azure Portal

Im Azure Portal unter **Azure Arc > Server > +Hinzufügen** kann ein maßgeschneidertes Onboarding-Skript für die jeweilige Umgebung generiert werden (inklusive Proxy, Tags, Resource Group).

---

## Azure Extensions auf Arc-Servern

Extensions erweitern Arc-Server um Azure-Funktionalität, ähnlich wie VM Extensions auf Azure-VMs:

| Extension | Zweck |
|---|---|
| **AzureMonitorWindowsAgent / AzureMonitorLinuxAgent** | Azure Monitor Agent (AMA) für Logs und Metriken |
| **MicrosoftDefenderForEndpoint** | Microsoft Defender for Endpoint Onboarding |
| **WindowsOSConfig / LinuxOSConfig** | Guest Configuration Agent |
| **CustomScriptExtension** | Skript-Ausführung auf Arc-Servern |
| **AzureKeyVaultSecretsProvider** | Secrets aus Azure Key Vault auf dem Server bereitstellen |
| **DependencyAgent** | Service Map / VM Insights Dependency Tracking |
| **WindowsPatchExtension / LinuxPatchExtension** | Azure Update Manager Patch-Verwaltung |
| **AzureSecurityWindowsAgent** | Defender for Cloud Sicherheits-Scanning |

```bash
# Extension manuell installieren (AMA für Linux)
az connectedmachine extension create \
  --resource-group "myRG" \
  --machine-name "myServer" \
  --name "AzureMonitorLinuxAgent" \
  --type "AzureMonitorLinuxAgent" \
  --publisher "Microsoft.Azure.Monitor" \
  --location "westeurope"
```

---

## Hybrid Benefits für Arc-Server

Der **Azure Hybrid Benefit** für Arc-Server ermöglicht die Nutzung vorhandener Windows Server-Lizenzen mit Software Assurance auf Arc-Servern — ohne zusätzliche Lizenzgebühren für bestimmte Azure-Dienste.

```powershell
# Hybrid Benefit aktivieren
az connectedmachine update \
  --name "myServer" \
  --resource-group "myRG" \
  --license-profile "{softwareAssurance:{softwareAssuranceCustomer:true}}"
```

> [!IMPORTANT] Software Assurance erforderlich
> Hybrid Benefit für Arc-Server setzt aktive **Windows Server Software Assurance** oder Windows Server-Abonnements voraus. Dies muss durch den Kunden nachgewiesen werden.

---

## Extended Security Updates (ESU) via Arc

Für **Windows Server 2012/2012 R2** (End of Support: Oktober 2023) können über Azure Arc kostenlos Extended Security Updates bezogen werden — bis Oktober 2026 (3 Jahre).

```mermaid
graph LR
    WS2012[Windows Server 2012 EOS Okt 2023]
    ARC[Arc-Registrierung]
    ESU[ESU-Lizenz zuweisen]
    UPDATES[Sicherheits-Updates via WU]
    PLAN1[Jahr 1: 75% des Listenpreises]
    PLAN2[Jahr 2: 100%]
    PLAN3[Jahr 3: 125%]

    WS2012 --> ARC
    ARC --> ESU
    ESU --> UPDATES
    ESU --> PLAN1
    ESU --> PLAN2
    ESU --> PLAN3
```

**Vorteile von ESU via Arc:**
- **Kostenlos** für Arc-Server die direkt in Azure laufen würden (aber eben on-premises sind)
- **Kein Produktschlüssel** erforderlich — Aktivierung erfolgt automatisch über Arc
- **Monatliche Abrechnung** über Azure-Subscription
- **Centralisiertes Management** via Azure Update Manager

```powershell
# ESU-Lizenz erstellen und Arc-Server zuweisen
az arcappliance create-esu-license \
  --resource-group "myRG" \
  --name "WS2012-ESU-License" \
  --location "westeurope" \
  --state "Activated" \
  --edition "Standard" \
  --type "vCore" \
  --processors 16
```

---

## Azure Monitor für Arc-Server

```mermaid
graph TD
    ARCSERVER[Arc-fähiger Server]
    AMA[Azure Monitor Agent]
    DCR[Data Collection Rule]
    LAW[Log Analytics Workspace]
    AZURE_MON[Azure Monitor]
    INSIGHTS[VM Insights]
    ALERTS[Alerts und Dashboards]

    ARCSERVER --> AMA
    AMA --> DCR
    DCR --> LAW
    LAW --> AZURE_MON
    AZURE_MON --> INSIGHTS
    AZURE_MON --> ALERTS
```

**Data Collection Rules (DCR)** definieren, welche Daten gesammelt werden:
- Windows Event Logs (Security, System, Application)
- Syslog (Linux)
- Performance Counters (CPU, Memory, Disk, Network)
- IIS Logs
- Custom Text Logs

---

## Defender for Cloud für Arc-Server

**Microsoft Defender for Servers** (Plan 1 oder Plan 2) kann auf Arc-Servern aktiviert werden:

| Feature | Plan 1 | Plan 2 |
|---|---|---|
| Microsoft Defender for Endpoint | ✅ | ✅ |
| Sicherheitsempfehlungen | ✅ | ✅ |
| Sicherheitsbewertung (Secure Score) | ✅ | ✅ |
| Just-in-Time VM Access | ❌ | ✅ |
| Adaptive Application Controls | ❌ | ✅ |
| File Integrity Monitoring | ❌ | ✅ |
| Threat Intelligence | ❌ | ✅ |
| Preis | ~7,20 $/Server/Monat | ~15 $/Server/Monat |

---

## Update Manager für Arc-Server

**Azure Update Manager** bietet zentrales Patch-Management für Arc-Server:

```powershell
# Update-Bewertung manuell starten
az connectedmachine assess-patches \
  --resource-group "myRG" \
  --name "myServer"

# Updates installieren
az connectedmachine install-patches \
  --resource-group "myRG" \
  --name "myServer" \
  --maximum-duration "PT2H" \
  --reboot-setting "IfRequired" \
  --windows-parameters "{classificationsToInclude:['Security','Critical']}"
```

---

## SSH-Zugang via Arc

Arc ermöglicht SSH-Verbindungen zu Servern **ohne öffentliche IP und ohne VPN** — direkt über den Arc-Tunnel:

```bash
# SSH via Arc (Azure CLI)
az ssh arc \
  --resource-group "myRG" \
  --name "myLinuxServer" \
  --local-user "azureuser"

# RDP-Tunnel für Windows (Port-Forwarding)
az network bastion tunnel \
  --name "myServer" \
  --resource-group "myRG" \
  --target-resource-id "<arcResourceId>" \
  --resource-port 3389 \
  --port 3389
```

> [!NOTE] Run Commands
> Über **Run Commands** können Skripte direkt aus dem Azure Portal oder per CLI auf Arc-Servern ausgeführt werden — ohne SSH-Session:
> ```bash
> az connectedmachine run-command create \
>   --resource-group "myRG" \
>   --machine-name "myServer" \
>   --run-command-name "GetOSInfo" \
>   --script "Get-ComputerInfo"
> ```

---

## Netzwerkanforderungen

Der Arc-Agent benötigt ausgehende HTTPS-Verbindungen (Port 443) zu folgenden Endpunkten:

| Endpunkt | Zweck |
|---|---|
| `management.azure.com` | ARM-Endpunkt |
| `login.microsoftonline.com` | Authentifizierung |
| `pas.windows.net` | Authentifizierung |
| `*.his.arc.azure.com` | Arc-Metadaten-Service |
| `*.guestconfiguration.azure.com` | Guest Configuration |
| `guestnotificationservice.azure.com` | Notifications |
| `*.blob.core.windows.net` | Extension Downloads |

### Proxy-Konfiguration

```bash
# Proxy für Arc-Agent konfigurieren (Linux)
azcmagent config set proxy.url "http://proxy:8080"

# Proxy mit Authentifizierung
azcmagent config set proxy.url "http://user:password@proxy:8080"

# Proxy deaktivieren
azcmagent config clear proxy.url
```

### Arc Gateway (Preview)

Das **Arc Gateway** reduziert die erforderlichen Firewall-Ausnahmen auf einen einzigen Endpunkt:

```mermaid
graph LR
    SERVERS[Arc Server 1..N]
    GW[Arc Gateway]
    AZURE[Azure Endpoints]

    SERVERS -->|Einzelner Endpunkt| GW
    GW -->|Intern geroutet| AZURE
```

---

## Troubleshooting Agent-Konnektivität

```powershell
# Konnektivität zu Azure-Endpunkten prüfen
azcmagent check

# Diagnose-Log erstellen
azcmagent logs

# Verbindungsstatus und Details anzeigen
azcmagent show

# Agent-Konfiguration anzeigen
azcmagent config list
```

**Häufige Fehler:**

| Fehler | Ursache | Lösung |
|---|---|---|
| `AZCM0050` | Netzwerk-Verbindungsfehler | Firewall-Regeln prüfen, `azcmagent check` ausführen |
| `AZCM0016` | Service Principal abgelaufen | SP-Credentials erneuern |
| `AZCM0041` | Resource Provider nicht registriert | `az provider register --namespace Microsoft.HybridCompute` |
| `AZCM0056` | TLS 1.2 nicht aktiviert | TLS 1.2 auf dem Server aktivieren |

```bash
# Logs einsehen (Linux)
journalctl -u himdsd -f
journalectl -u gcad -f

# Windows Event Log
Get-WinEvent -LogName "Microsoft-AzureArcAgent/Operational" | Select-Object -First 50
```

---

## Verwandte Notizen
- [[Azure Arc Grundlagen]]
- [[Arc-fähige Kubernetes]]
- [[Azure Monitor]]
- [[Azure Update Manager]]
- [[Azure Policy]]
