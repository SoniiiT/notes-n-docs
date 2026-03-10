---
tags:
  - azure
  - compute
  - vm-extensions
  - custom-script
  - azure-monitor-agent
aliases:
  - VM Extensions
  - Azure VM Erweiterungen
  - VM Extensions Azure
---

# Azure VM Erweiterungen

> [!ABSTRACT] TL;DR
> VM-Erweiterungen (Extensions) ermöglichen Post-Deployment-Konfiguration und Verwaltung von Azure VMs ohne Image-Rebuild. Sie werden als Agent-Prozesse innerhalb der VM ausgeführt und decken Szenarien wie Skript-Ausführung, Monitoring, Verschlüsselung, Domain-Join und Konfigurationsmanagement ab.

---

## Was sind VM-Erweiterungen?

VM-Erweiterungen sind kleine Anwendungen, die nach dem Deployment einer VM ausgeführt werden. Sie lösen die häufigsten Konfigurationsaufgaben, ohne ein neues VM-Image erstellen zu müssen:

- **Automatisierung**: Skripte und Konfigurationen nach Deployment anwenden
- **Monitoring**: Agenten für Metriken, Logs und Diagnose installieren
- **Sicherheit**: Verschlüsselung, Antimalware und Compliance-Checks aktivieren
- **Integration**: VMs in Active Directory einbinden, Zertifikate bereitstellen

Extensions werden über den **Azure VM Agent** verwaltet, der standardmäßig in allen Azure Marketplace-Images vorinstalliert ist. Jede Extension hat einen Provisioning-State (Succeeded, Failed, Updating).

---

## Übersicht aller wichtigen Extensions

| Extension Name | Publisher | Platform | Zweck |
|---------------|-----------|----------|-------|
| **CustomScriptExtension** | Microsoft.Compute | Windows | Skripte nach Deployment ausführen |
| **CustomScript** | Microsoft.Azure.Extensions | Linux | Skripte nach Deployment ausführen |
| **AzureMonitorWindowsAgent** | Microsoft.Azure.Monitor | Windows | Metriken und Logs (AMA) |
| **AzureMonitorLinuxAgent** | Microsoft.Azure.Monitor | Linux | Metriken und Logs (AMA) |
| **IaaSAntimalware** | Microsoft.Azure.Security | Windows | Microsoft Antimalware (Defender) |
| **AzureDiskEncryption** | Microsoft.Azure.Security | Windows | BitLocker-Verschlüsselung |
| **AzureDiskEncryptionForLinux** | Microsoft.Azure.Security | Linux | dm-crypt-Verschlüsselung |
| **JsonADDomainExtension** | Microsoft.Compute | Windows | Active Directory Domain Join |
| **DSC** | Microsoft.Powershell | Windows | Desired State Configuration |
| **DependencyAgentWindows** | Microsoft.Azure.Monitoring.DependencyAgent | Windows | Application Map (Service Map) |
| **DependencyAgentLinux** | Microsoft.Azure.Monitoring.DependencyAgent | Linux | Application Map (Service Map) |
| **NetworkWatcherAgentWindows** | Microsoft.Azure.NetworkWatcher | Windows | Paket-Captures, Verbindungs-Diagnose |
| **NetworkWatcherAgentLinux** | Microsoft.Azure.NetworkWatcher | Linux | Paket-Captures, Verbindungs-Diagnose |
| **KeyVaultForWindows** | Microsoft.Azure.KeyVault | Windows | Zertifikate automatisch bereitstellen |
| **KeyVaultForLinux** | Microsoft.Azure.KeyVault | Linux | Zertifikate automatisch bereitstellen |
| **LinuxDiagnostic** | Microsoft.Azure.Diagnostics | Linux | Metriken und Logs zu Storage |
| **GuestConfiguration** | Microsoft.GuestConfiguration | Windows/Linux | Policy-Compliance-Checks |
| **BGInfo** | Microsoft.Compute | Windows | Desktop-Hintergrundinformationen |

---

## Custom Script Extension

Die Custom Script Extension führt Skripte direkt nach dem VM-Deployment aus — ideal für Bootstrapping, Software-Installation und Konfiguration.

### Windows: CustomScriptExtension

```powershell
# Custom Script Extension für Windows per PowerShell
Set-AzVMExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-web-01" `
  -Name "CustomScriptExtension" `
  -Publisher "Microsoft.Compute" `
  -ExtensionType "CustomScriptExtension" `
  -TypeHandlerVersion "1.10" `
  -Settings @{
    "fileUris" = @("https://staccount.blob.core.windows.net/scripts/setup.ps1")
    "commandToExecute" = "powershell -ExecutionPolicy Unrestricted -File setup.ps1"
  }
```

### Linux: CustomScript

```bash
# Custom Script Extension für Linux per Azure CLI
az vm extension set \
  --resource-group rg-prod \
  --vm-name vm-app-01 \
  --name customScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{
    "fileUris": ["https://staccount.blob.core.windows.net/scripts/install.sh"],
    "commandToExecute": "bash install.sh"
  }'
```

### Bicep-Beispiel: Custom Script Extension

```bicep
resource customScriptExtension 'Microsoft.Compute/virtualMachines/extensions@2023-09-01' = {
  name: '${vmName}/CustomScriptExtension'
  location: location
  properties: {
    publisher: 'Microsoft.Compute'
    type: 'CustomScriptExtension'
    typeHandlerVersion: '1.10'
    autoUpgradeMinorVersion: true
    settings: {
      fileUris: [
        'https://${storageAccountName}.blob.core.windows.net/scripts/setup.ps1'
      ]
    }
    protectedSettings: {
      commandToExecute: 'powershell -ExecutionPolicy Unrestricted -File setup.ps1 -AdminPassword ${adminPassword}'
      storageAccountName: storageAccountName
      storageAccountKey: storageAccountKey
    }
  }
}
```

> [!IMPORTANT] Protected Settings für Secrets
> Zugangsdaten, Passwörter und Storage-Keys immer in `protectedSettings` (nicht in `settings`) übergeben. Protected Settings werden verschlüsselt übertragen und nicht in Azure-Logs gespeichert.

> [!WARNING] Nur einmal ausführbar
> Die Custom Script Extension kann pro VM nur einmal deployed werden. Für erneute Ausführung muss die Extension entfernt und neu hinzugefügt werden.

---

## Azure Monitor Agent (AMA)

Der Azure Monitor Agent (AMA) ersetzt den veralteten Log Analytics Agent (MMA/OMS) und den Diagnostics Agent. Er sammelt Metriken und Logs basierend auf **Data Collection Rules (DCR)**.

### Vorteile gegenüber dem alten MMA

| Aspekt | AMA (neu) | MMA/OMS (alt) |
|--------|-----------|--------------|
| **Konfiguration** | Data Collection Rules (DCR) | Workspace-Einstellungen |
| **Multi-Workspace** | Ja, pro DCR | Nur ein Workspace |
| **Granularität** | Pro VM, Pro Resource Group | Workspace-weit |
| **Sicherheit** | Managed Identity | Workspace Key |
| **End-of-Support** | Aktiv entwickelt | August 2024 (EOL) |

### AMA per PowerShell installieren

```powershell
# AMA auf Windows VM installieren
Set-AzVMExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-app-01" `
  -Name "AzureMonitorWindowsAgent" `
  -Publisher "Microsoft.Azure.Monitor" `
  -ExtensionType "AzureMonitorWindowsAgent" `
  -TypeHandlerVersion "1.0" `
  -Location "westeurope"

# AMA auf Linux VM installieren
Set-AzVMExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-linux-01" `
  -Name "AzureMonitorLinuxAgent" `
  -Publisher "Microsoft.Azure.Monitor" `
  -ExtensionType "AzureMonitorLinuxAgent" `
  -TypeHandlerVersion "1.0" `
  -Location "westeurope"
```

### Data Collection Rule (DCR) erstellen

```json
{
  "location": "westeurope",
  "properties": {
    "dataSources": {
      "performanceCounters": [
        {
          "streams": ["Microsoft-Perf"],
          "samplingFrequencyInSeconds": 60,
          "counterSpecifiers": [
            "\\Processor Information(_Total)\\% Processor Time",
            "\\Memory\\Available Bytes",
            "\\LogicalDisk(_Total)\\Disk Read Bytes/sec"
          ],
          "name": "perfCounters"
        }
      ],
      "windowsEventLogs": [
        {
          "streams": ["Microsoft-Event"],
          "xPathQueries": [
            "Application!*[System[(Level=1 or Level=2 or Level=3)]]",
            "System!*[System[(Level=1 or Level=2)]]"
          ],
          "name": "eventLogs"
        }
      ]
    },
    "destinations": {
      "logAnalytics": [
        {
          "workspaceResourceId": "/subscriptions/{subId}/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-prod",
          "name": "lawProd"
        }
      ]
    },
    "dataFlows": [
      {
        "streams": ["Microsoft-Perf", "Microsoft-Event"],
        "destinations": ["lawProd"]
      }
    ]
  }
}
```

---

## Microsoft Antimalware Extension

Aktiviert Microsoft Antimalware (Windows Defender) auf Windows VMs mit konfigurierbaren Scans und Ausnahmen:

```powershell
# Antimalware Extension konfigurieren
$Settings = @{
    "AntimalwareEnabled" = $true
    "RealtimeProtectionEnabled" = $true
    "ScheduledScanSettings" = @{
        "isEnabled" = $true
        "scanType" = "Quick"
        "day" = "7"       # Sonntag
        "time" = "120"    # 02:00 Uhr
    }
    "Exclusions" = @{
        "Extensions" = ".log;.tmp"
        "Paths" = "D:\Data\Uploads"
        "Processes" = "CustomApp.exe"
    }
}

Set-AzVMExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-app-01" `
  -Name "IaaSAntimalware" `
  -Publisher "Microsoft.Azure.Security" `
  -ExtensionType "IaaSAntimalware" `
  -TypeHandlerVersion "1.5" `
  -Settings $Settings
```

> [!NOTE] Defender for Cloud Alternative
> In Umgebungen mit **Microsoft Defender for Cloud** wird Antimalware automatisch über Policies bereitgestellt. Die manuelle Extension-Konfiguration ist für Standalone-Szenarien gedacht.

---

## Disk Encryption Extension

Verschlüsselt VM-Disks mit BitLocker (Windows) oder dm-crypt (Linux) mithilfe von Schlüsseln aus Azure Key Vault.

### Azure Disk Encryption aktivieren

```powershell
# Key Vault für Disk Encryption vorbereiten
$keyVaultName = "kv-diskenc-prod"
$keyVaultResourceGroup = "rg-security"

# Key Vault für Disk Encryption aktivieren
Set-AzKeyVaultAccessPolicy `
  -VaultName $keyVaultName `
  -ResourceGroupName $keyVaultResourceGroup `
  -EnabledForDiskEncryption

# Encryption Key (KEK) erstellen
$kek = Add-AzKeyVaultKey `
  -VaultName $keyVaultName `
  -Name "DiskEncryptionKey" `
  -Destination "Software"

# Windows VM verschlüsseln (OS + Data Disks)
$keyVault = Get-AzKeyVault -VaultName $keyVaultName -ResourceGroupName $keyVaultResourceGroup

Set-AzVMDiskEncryptionExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-app-01" `
  -DiskEncryptionKeyVaultUrl $keyVault.VaultUri `
  -DiskEncryptionKeyVaultId $keyVault.ResourceId `
  -KeyEncryptionKeyUrl $kek.Key.Kid `
  -KeyEncryptionKeyVaultId $keyVault.ResourceId `
  -VolumeType "All" `
  -Force
```

> [!WARNING] VM-Neustart erforderlich
> Die Disk-Encryption erfordert einen VM-Neustart. Plant entsprechende Wartungsfenster ein.

---

## Domain Join Extension

Bindet Windows VMs automatisch in eine Active Directory-Domäne ein — ideal für goldene Images ohne Domänen-Präkonfiguration.

```json
{
  "type": "Microsoft.Compute/virtualMachines/extensions",
  "name": "[concat(parameters('vmName'),'/JsonADDomainExtension')]",
  "apiVersion": "2023-09-01",
  "location": "[parameters('location')]",
  "dependsOn": [
    "[resourceId('Microsoft.Compute/virtualMachines', parameters('vmName'))]"
  ],
  "properties": {
    "publisher": "Microsoft.Compute",
    "type": "JsonADDomainExtension",
    "typeHandlerVersion": "1.3",
    "autoUpgradeMinorVersion": true,
    "settings": {
      "Name": "corp.internal",
      "OUPath": "OU=AzureVMs,DC=corp,DC=internal",
      "User": "corp.internal\\svc-domainjoin",
      "Restart": "true",
      "Options": "3"
    },
    "protectedSettings": {
      "Password": "[parameters('domainJoinPassword')]"
    }
  }
}
```

**Options-Werte:**
- `1` = NetSetupJoinDomain
- `3` = NetSetupJoinDomain + NetSetupCreateComputerAccount (Konto anlegen falls nicht vorhanden)

---

## DSC Extension (Desired State Configuration)

Wendet PowerShell DSC-Konfigurationen auf Windows VMs an — geeignet für deklaratives Konfigurationsmanagement.

```powershell
# DSC-Konfiguration kompilieren und hochladen
Publish-AzVMDscConfiguration `
  -ConfigurationPath ".\WebServerConfig.ps1" `
  -ResourceGroupName "rg-prod" `
  -StorageAccountName "stprod001" `
  -Force

# DSC Extension auf VM anwenden
Set-AzVMDscExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-web-01" `
  -ArchiveBlobName "WebServerConfig.ps1.zip" `
  -ArchiveStorageAccountName "stprod001" `
  -ConfigurationName "WebServerConfig" `
  -Version "2.83"
```

### Beispiel DSC-Konfiguration

```powershell
Configuration WebServerConfig {
    Import-DscResource -ModuleName PSDesiredStateConfiguration

    Node "localhost" {
        WindowsFeature IIS {
            Ensure = "Present"
            Name   = "Web-Server"
        }

        WindowsFeature IISManagement {
            Ensure = "Present"
            Name   = "Web-Mgmt-Console"
            DependsOn = "[WindowsFeature]IIS"
        }

        File WebContent {
            Ensure          = "Present"
            Type            = "Directory"
            DestinationPath = "C:\inetpub\wwwroot\app"
        }
    }
}
```

---

## Azure Policy Guest Configuration

Prüft OS-Compliance direkt ohne zusätzlichen Agent — nutzt den Azure VM Agent als Grundlage.

- Eingebaut in Azure Policy (kein separater Agent)
- Überprüft OS-Einstellungen: Passwort-Komplexität, Audit-Logs, Software-Installation
- **Audit**: Nur melden (kein Eingriff)
- **DeployIfNotExists**: Automatisch korrigieren
- Unterstützt Windows (DSC-basiert) und Linux (Chef InSpec-basiert)

> [!TIP] In Defender for Cloud integriert
> Guest Configuration ist in **Microsoft Defender for Cloud** als "Guest OS Security Recommendations" integriert. Dort werden nicht-konforme Settings direkt gemeldet.

---

## Dependency Agent

Ermöglicht **Application Map** in Azure Monitor — visualisiert Abhängigkeiten zwischen Services und VMs.

```bash
# Dependency Agent per CLI installieren (Linux)
az vm extension set \
  --resource-group rg-prod \
  --vm-name vm-app-01 \
  --name DependencyAgentLinux \
  --publisher Microsoft.Azure.Monitoring.DependencyAgent \
  --version 9.10

# Windows
az vm extension set \
  --resource-group rg-prod \
  --vm-name vm-win-01 \
  --name DependencyAgentWindows \
  --publisher Microsoft.Azure.Monitoring.DependencyAgent \
  --version 9.10
```

> [!NOTE] AMA als Voraussetzung
> Der Dependency Agent benötigt den **Azure Monitor Agent (AMA)** als Voraussetzung. Beide müssen auf der VM installiert sein.

---

## Network Watcher Agent

Ermöglicht Paket-Captures, Verbindungs-Diagnose und IP-Flow-Tests aus dem Azure Network Watcher.

```bash
# Network Watcher Agent (Linux) installieren
az vm extension set \
  --resource-group rg-prod \
  --vm-name vm-app-01 \
  --name NetworkWatcherAgentLinux \
  --publisher Microsoft.Azure.NetworkWatcher \
  --version 1.4

# Paket-Capture starten
az network watcher packet-capture create \
  --resource-group rg-prod \
  --vm vm-app-01 \
  --name capture-session-01 \
  --storage-account stnetworkwatcher \
  --time-limit 300 \
  --filters '[{"protocol":"TCP","remoteIPAddress":"10.0.0.5","localIPAddress":"","localPort":"","remotePort":"443"}]'
```

---

## Key Vault VM Extension

Stellt Zertifikate aus Azure Key Vault automatisch auf VMs bereit und erneuert sie bei Ablauf.

```json
{
  "type": "Microsoft.Compute/virtualMachines/extensions",
  "name": "[concat(parameters('vmName'), '/KeyVaultForWindows')]",
  "apiVersion": "2023-09-01",
  "properties": {
    "publisher": "Microsoft.Azure.KeyVault",
    "type": "KeyVaultForWindows",
    "typeHandlerVersion": "3.0",
    "autoUpgradeMinorVersion": true,
    "settings": {
      "secretsManagementSettings": {
        "pollingIntervalInS": "3600",
        "certificateStoreName": "MY",
        "certificateStoreLocation": "LocalMachine",
        "observedCertificates": [
          "https://kv-prod.vault.azure.net/secrets/ssl-cert"
        ]
      },
      "authenticationSettings": {
        "msiEndpoint": "http://169.254.169.254/metadata/identity/oauth2/token",
        "msiClientId": "[parameters('userAssignedIdentityClientId')]"
      }
    }
  }
}
```

---

## Linux Diagnostics Extension

Sendet Metriken und Logs an einen Azure Storage Account — nützlich für ältere Monitoring-Setups.

```bash
# Linux Diagnostics Extension konfigurieren
az vm diagnostics set \
  --resource-group rg-prod \
  --vm-name vm-linux-01 \
  --settings '{"storageAccount": "stdiagprod"}' \
  --protected-settings '{"storageAccountName": "stdiagprod", "storageAccountKey": "KEY=="}'
```

---

## Erweiterungen verwalten

### Portal
Azure Portal → VM → Einstellungen → Erweiterungen + Anwendungen

### PowerShell

```powershell
# Alle Extensions einer VM anzeigen
Get-AzVMExtension -ResourceGroupName "rg-prod" -VMName "vm-app-01"

# Extension hinzufügen (generisch)
Set-AzVMExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-app-01" `
  -Name "ExtensionName" `
  -Publisher "Publisher.Name" `
  -ExtensionType "TypeName" `
  -TypeHandlerVersion "1.0" `
  -Location "westeurope" `
  -Settings @{ "key" = "value" }

# Extension entfernen
Remove-AzVMExtension `
  -ResourceGroupName "rg-prod" `
  -VMName "vm-app-01" `
  -Name "ExtensionName" `
  -Force
```

### Azure CLI

```bash
# Alle Extensions anzeigen
az vm extension list \
  --resource-group rg-prod \
  --vm-name vm-app-01 \
  --output table

# Extension installieren
az vm extension set \
  --resource-group rg-prod \
  --vm-name vm-app-01 \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{"commandToExecute": "apt-get install -y nginx"}'

# Extension löschen
az vm extension delete \
  --resource-group rg-prod \
  --vm-name vm-app-01 \
  --name CustomScript
```

---

## Fehlerbehebung

### Extension-Logs auf Linux

```bash
# Allgemeines Extension-Log-Verzeichnis
ls /var/log/azure/

# Custom Script Extension Logs
cat /var/log/azure/Microsoft.Azure.Extensions.CustomScript/extension.log

# Azure Monitor Agent Logs
cat /var/log/azure/Microsoft.Azure.Monitor.AzureMonitorLinuxAgent/azure-mdsd.log
journalctl -u mdsd -n 100
```

### Extension-Logs auf Windows

```
C:\WindowsAzure\Logs\Plugins\<PublisherName>.<TypeName>\<Version>\
C:\WindowsAzure\Logs\WaAppAgent.log  (VM Agent selbst)
```

### Provision-Status prüfen

```bash
# Provision-Status per CLI
az vm extension list \
  --resource-group rg-prod \
  --vm-name vm-app-01 \
  --query "[].{Name:name, State:provisioningState, Status:instanceView.statuses[0].message}" \
  --output table

# Detaillierte Status-Informationen
az vm get-instance-view \
  --resource-group rg-prod \
  --name vm-app-01 \
  --query "extensions[?name=='CustomScript'].statuses"
```

### Häufige Fehler und Lösungen

| Fehler | Ursache | Lösung |
|--------|---------|--------|
| `ProvisioningFailed` | Skript-Fehler, falscher Pfad | Extension-Log prüfen, Skript testen |
| `Handler not ready` | VM Agent nicht gestartet | VM neu starten, Agent-Status prüfen |
| `Timeout` | Skript dauert zu lang (Standard: 90 Min) | Skript optimieren oder async gestalten |
| `Storage access denied` | SAS-Token abgelaufen | Neues SAS-Token generieren |
| `Extension already installed` | Doppeltes Deployment | Extension zuerst entfernen |

> [!TIP] Azure VM Agent Diagnose
> Bei Problemen mit dem VM Agent selbst: `waagent -version` (Linux) oder Dienst `WindowsAzureGuestAgent` prüfen (Windows). Der Agent muss laufen, damit Extensions funktionieren.

---

## Verwandte Notizen
- [[Azure Virtual Machines]]
- [[Azure Spot VMs]]
- [[Azure Policy]]
- [[Azure RBAC]]
