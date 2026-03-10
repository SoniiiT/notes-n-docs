---
tags:
  - azure
  - compute
  - virtual-machines
  - iaas
  - vm
  - bicep
aliases:
  - Azure VMs
  - Virtual Machines Azure
  - Azure IaaS
---

# Azure Virtual Machines

> [!ABSTRACT] TL;DR
> Azure Virtual Machines (VMs) sind der grundlegende IaaS-Baustein in Azure. Sie bieten vollständige Kontrolle über das Betriebssystem, erlauben eigene Images und unterstützen alle gängigen Workloads. Die Wahl der richtigen VM-Serie, Disk-Typen und Verfügbarkeits-Optionen ist entscheidend für Performance, SLA und Kosten.

---

## VM-Typen und Serien

Azure VMs sind in Serien organisiert, die für spezifische Workloads optimiert sind:

| VM-Serie | Beispiel-SKU | Anwendungsfall |
|----------|-------------|---------------|
| **General Purpose (D/Dsv5)** | Standard_D4s_v5 | Web-Server, Dev/Test, kleine Datenbanken |
| **Compute Optimized (F)** | Standard_F8s_v2 | CI/CD, Gaming, Web-Frontiends mit hohem CPU-Bedarf |
| **Memory Optimized (E/M)** | Standard_E16s_v5 | Datenbanken, In-Memory Analytics, SAP |
| **Storage Optimized (L)** | Standard_L8s_v3 | NoSQL-Datenbanken (Cassandra, MongoDB), Data Warehousing |
| **GPU (N)** | Standard_NC6s_v3 | ML-Training, 3D-Rendering, Grafikverarbeitung |
| **HPC (H/HB/HC)** | Standard_HB120rs_v3 | Finite-Elemente, Fluidsimulation, Genomik |
| **Confidential (DC)** | Standard_DC4s_v3 | Verarbeitung sensibler Daten mit TEE |

---

## VM-Größen-Notation

Die SKU-Notation folgt einem einheitlichen Schema:

```
Standard_D4s_v5
│       │││  │
│       │││  └── Version (v5 = neueste Generation)
│       ││└───── Feature-Flag (s = Premium Storage, d = lokale NVMe)
│       │└────── vCPU-Anzahl (4 vCPUs)
│       └─────── VM-Serie (D = General Purpose)
└─────────────── Tier (Standard, Basic veraltet)
```

**Weitere Feature-Flags:**
- `a` = AMD-Prozessor
- `b` = Burst-fähig (Burstable)
- `d` = lokaler NVMe-Temp-Disk
- `i` = Isolated (dedizierter Host)
- `l` = Low Memory (weniger RAM)
- `m` = Memory-optimiert (mehr RAM)
- `n` = Netzwerk-optimiert (hoher Durchsatz)
- `s` = Premium SSD unterstützt

---

## Images

### Marketplace-Images
- Fertige OS-Images von Microsoft und Drittanbietern
- Windows Server (2019, 2022), Ubuntu, Red Hat, SUSE, etc.
- Anwendungs-Images: SQL Server, SAP, Cisco, Palo Alto

### Custom Images
- Eigene Images aus vorbereiteten VMs erstellen (Sysprep + Generalize)
- Ideal für standardisierte Unternehmens-Deployments mit vorinstallierten Tools

### Azure Compute Gallery (früher: Shared Image Gallery)
- Zentrales Repository für eigene VM-Images
- Image Definitions + Image Versions
- **Geo-Replikation** in beliebig viele Regionen
- RBAC-basierter Zugriff (auch Cross-Tenant möglich)
- Unterstützt VM-Images, Disk-Images und Application Packages

---

## OS Disk Typen

| Disk-Typ | IOPS (max) | Throughput | Latenz | Kosten | Anwendungsfall |
|----------|-----------|-----------|--------|--------|---------------|
| **Standard HDD** | 500 | 60 MiB/s | ~10ms | Niedrig | Backup, seltene Zugriffe |
| **Standard SSD** | 6.000 | 750 MiB/s | ~1-2ms | Mittel | Leichte Web-Workloads, Dev/Test |
| **Premium SSD v1** | 20.000 | 900 MiB/s | <1ms | Hoch | Produktion, Datenbanken |
| **Premium SSD v2** | 80.000 | 1.200 MiB/s | <1ms | Hoch (flexibel) | Hochperformante Datenbanken |
| **Ultra Disk** | 160.000 | 4.000 MiB/s | <0,5ms | Sehr hoch | SAP HANA, Top-Tier-DBs |

> [!NOTE] Managed Disks immer bevorzugen
> Unmanaged Disks (in Storage Accounts) sind veraltet. **Managed Disks** sind einfacher, sicherer (kein eigener Storage Account) und bieten SLA für einzelne VMs.

---

## Availability Sets vs. Availability Zones

| Aspekt | Availability Set | Availability Zone |
|--------|-----------------|------------------|
| **Schutz vor** | Hardware-Ausfall im Datacenter | Datacenter-Ausfall (ganze Zone) |
| **Fehlerdomänen** | 2-3 (physische Racks) | 1 Zone = 1 Datacenter |
| **Update-Domänen** | 5-20 (gleichzeitige Wartung) | Regionen-weit koordiniert |
| **SLA** | 99,95% (2+ VMs) | 99,99% (2+ VMs in 2+ Zonen) |
| **Kosten** | Kein Aufpreis | Kein Aufpreis (aber Egress-Kosten) |
| **Wann nutzen** | Regions ohne AZs, Legacy | Moderne Deployments, HA-Anforderungen |

> [!TIP] Zone Redundant bevorzugen
> In Regionen mit Availability Zones (z.B. West Europe, East US) sind **Availability Zones** dem Availability Set immer vorzuziehen — höhere SLA, bessere Isolierung.

---

## VM Scale Sets (VMSS)

VMSS verwaltet eine Gruppe identischer VMs mit automatischer Skalierung.

### Flexible vs. Uniform Orchestration

| Aspekt | Flexible Orchestration | Uniform Orchestration |
|--------|----------------------|----------------------|
| **VM-Typen** | Individuelle VMs (eigene IDs) | Identische Instanzen |
| **Mischung** | Verschiedene VM-Größen möglich | Nur eine VM-Größe |
| **SLA** | 99,99% mit 2+ Zonen | 99,95% Availability Set-Level |
| **Empfehlung** | Neue Deployments | Legacy-Workloads |
| **Portal-Verwaltung** | Jede VM einzeln sichtbar | Als Gruppe verwaltet |

**Skalierungs-Trigger:**
- CPU-Auslastung (z.B. bei >70% Scale-Out, bei <30% Scale-In)
- Memory, Netzwerk, benutzerdefinierte Metriken
- Schedule-basiert (vorhersehbare Last)

---

## Weitere Compute-Optionen

### Azure Spot VMs
Preemptible VMs mit bis zu 90% Rabatt — geeignet für unterbrechbare Workloads. Siehe [[Azure Spot VMs]].

### Proximity Placement Groups
- Stellt sicher, dass VMs physisch nah beieinander sind
- Maximiert Netzwerk-Performance, minimiert Latenz
- Ideal für eng gekoppelte Anwendungen (z.B. HANA + App-Server)

### Dedicated Hosts
- Physischer Azure-Server exklusiv für einen Kunden
- Compliance-Anforderungen (kein Shared Hardware mit anderen Tenants)
- Kontrolle über Wartungsfenster
- Azure Hybrid Benefit kompatibel

### Hibernation (Preview)
- VM pausieren und Zustand im OS Disk speichern
- Neustart in Sekunden (Zustand wiederhergestellt)
- Spart Compute-Kosten bei langen Pausen (nur Storage-Kosten)
- Unterstützt: Windows 10/11, bestimmte Linux-Distros

---

## VM-Erweiterungen

Extensions ermöglichen Post-Deployment-Konfiguration ohne Image-Rebuild. Übersicht der wichtigsten Extensions:

- **Custom Script Extension**: Skript nach Deployment ausführen
- **Azure Monitor Agent (AMA)**: Metriken und Logs sammeln
- **Disk Encryption Extension**: BitLocker/dm-crypt aktivieren
- **Domain Join Extension**: Windows VMs in AD einbinden

Vollständige Übersicht: [[Azure VM Erweiterungen]]

---

## Auto-Shutdown

Automatisches Herunterfahren für Dev/Test-VMs (Kosten sparen):

```powershell
# Auto-Shutdown per PowerShell konfigurieren
$subscriptionId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$resourceGroup = "rg-dev"
$vmName = "vm-dev-01"
$shutdownTime = "1900"  # 19:00 Uhr
$timeZone = "W. Europe Standard Time"

$properties = @{
    status = "Enabled"
    taskType = "ComputeVmShutdownTask"
    dailyRecurrence = @{ time = $shutdownTime }
    timeZoneId = $timeZone
    notificationSettings = @{ status = "Disabled" }
    targetResourceId = "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Compute/virtualMachines/$vmName"
}

$uri = "https://management.azure.com/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/microsoft.devtestlab/schedules/shutdown-computevm-$vmName"
Invoke-AzRestMethod -Method PUT -Path "$uri?api-version=2018-09-15" -Payload ($properties | ConvertTo-Json -Depth 5)
```

---

## Serial Console

Notfall-Konsole für den Fall, dass SSH/RDP nicht verfügbar ist:

- Zugriff über Azure Portal → VM → Hilfe → Serial Console
- Erfordert Boot-Diagnose (Storage Account)
- Funktioniert bei Netzwerkproblemen, falsch konfigurierten Firewall-Regeln
- Unterstützt SAC (Special Administration Console) für Windows

> [!IMPORTANT] Boot-Diagnostik aktivieren
> Serial Console erfordert aktivierte **Boot-Diagnostik** (Managed Storage Account reicht). Ohne Boot-Diagnostik ist kein Serial Console-Zugriff möglich.

---

## VM Limits und Quotas

| Limit | Standard-Wert | Erhöhbar |
|-------|--------------|---------|
| **vCPUs pro Region** | 10–350 (je nach Subscription) | Ja (Support-Ticket) |
| **vCPUs pro VM-Familie** | 10–100 (variiert) | Ja |
| **VMs pro Availability Set** | 200 | Nein |
| **VMs pro Scale Set** | 1.000 (Flexible), 1.000 (Uniform) | Nein |
| **Managed Disks pro Subscription** | 50.000 | Ja |

```bash
# Aktuelle vCPU-Quotas anzeigen
az vm list-usage --location "westeurope" --output table

# Quota-Anfrage per CLI (oder Support-Ticket)
az quota create --resource-name "cores" \
  --scope "/subscriptions/{subId}/providers/Microsoft.Compute/locations/westeurope" \
  --limit-object value=100 \
  --resource-type dedicated
```

---

## Via Azure Portal (UI)

> [!TIP] Schnelleinstieg
> Das Azure Portal führt durch alle VM-Konfigurationsschritte mit validierter Eingabe – ideal für einmalige Deployments.

### Virtuelle Maschine erstellen

1. **portal.azure.com** → **Virtuelle Computer** → **+ Erstellen** → **Virtuelle Azure-Maschine**
2. **Grundlagen**-Tab:
   - Abonnement & Ressourcengruppe
   - **VM-Name**, **Region**, **Verfügbarkeitsoptionen** (Zone / Verfügbarkeitsgruppe / Keine)
   - **Image**: z. B. `Ubuntu Server 24.04 LTS`, `Windows Server 2025 Datacenter`
   - **Größe**: → **Alle Größen anzeigen** → Filtere nach Familie (D, E, F usw.)
   - **Administratorkonto**: SSH-Public-Key (Linux) oder Benutzername/Passwort (Windows)
   - **Eingehende Ports**: RDP (3389) oder SSH (22) erlauben
3. **Datenträger**-Tab:
   - OS-Datenträger-Typ: Premium SSD, Standard SSD, Standard HDD
   - **+ Neuen Datenträger erstellen und anfügen** für Datendatenträger
4. **Netzwerk**-Tab:
   - Virtuelles Netzwerk, Subnetz, öffentliche IP auswählen
   - **NIC-Netzwerksicherheitsgruppe**: Keine, Einfach oder Erweitert
5. **Verwaltung**-Tab:
   - **System-seitig zugewiesene Identität** aktivieren (Managed Identity)
   - **Automatisches Herunterfahren** konfigurieren
6. **Überwachung**-Tab: Startdiagnose aktivieren
7. **Überprüfen + erstellen** → **Erstellen**

### VM-Größe nachträglich ändern

1. VM → **Größe** im linken Menü → Neue Größe auswählen → **Größe ändern**
2. ⚠️ VM wird neu gestartet

### VMSS erstellen

1. portal.azure.com → **VM-Skalierungsgruppen** → **+ Erstellen**
2. Orchestrierungsmodus, Image, Größe, Instanzanzahl konfigurieren
3. **Skalierung** → automatische Skalierungsregeln festlegen

---

## Bicep-Beispiel: Linux VM

```bicep
param location string = resourceGroup().location
param vmName string = 'vm-app-prod-01'
param adminUsername string
@secure()
param adminPasswordOrKey string

resource publicIP 'Microsoft.Network/publicIPAddresses@2023-09-01' = {
  name: '${vmName}-pip'
  location: location
  sku: { name: 'Standard' }
  properties: { publicIPAllocationMethod: 'Static' }
}

resource nic 'Microsoft.Network/networkInterfaces@2023-09-01' = {
  name: '${vmName}-nic'
  location: location
  properties: {
    ipConfigurations: [{
      name: 'ipconfig1'
      properties: {
        privateIPAllocationMethod: 'Dynamic'
        publicIPAddress: { id: publicIP.id }
        subnet: { id: resourceId('Microsoft.Network/virtualNetworks/subnets', 'vnet-prod', 'snet-app') }
      }
    }]
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2023-09-01' = {
  name: vmName
  location: location
  properties: {
    hardwareProfile: { vmSize: 'Standard_D4s_v5' }
    storageProfile: {
      imageReference: {
        publisher: 'Canonical'
        offer: '0001-com-ubuntu-server-jammy'
        sku: '22_04-lts-gen2'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
        managedDisk: { storageAccountType: 'Premium_LRS' }
        diskSizeGB: 64
      }
    }
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      linuxConfiguration: {
        disablePasswordAuthentication: true
        ssh: {
          publicKeys: [{
            path: '/home/${adminUsername}/.ssh/authorized_keys'
            keyData: adminPasswordOrKey
          }]
        }
      }
    }
    networkProfile: {
      networkInterfaces: [{ id: nic.id }]
    }
  }
}
```

---

## Kosten-Optimierung

| Strategie | Einsparung | Anwendungsfall |
|-----------|-----------|---------------|
| **Azure Spot VMs** | Bis 90% | Unterbrechbare Workloads |
| **Reserved Instances (1 Jahr)** | ~40% | Bekannte, kontinuierliche Workloads |
| **Reserved Instances (3 Jahre)** | ~60% | Stabile Langzeit-Workloads |
| **Azure Hybrid Benefit** | Bis 40% | Windows Server / SQL Server BYOL |
| **Richtige VM-Größe** | Variabel | Regelmäßiges Right-Sizing (Advisor) |
| **Auto-Shutdown** | Bis 75% | Dev/Test-Umgebungen |
| **Deallocate statt Running** | ~100% Compute | VMs pausieren (zahle nur Storage) |

> [!TIP] Azure Advisor nutzen
> Der **Azure Advisor** analysiert VM-Metriken und empfiehlt konkrete Right-Sizing-Maßnahmen (z.B. "VM ist zu 15% ausgelastet — wechsle zu D2s_v5").

---

## Verwandte Notizen
- [[Azure VM Erweiterungen]]
- [[Azure Spot VMs]]
- [[Azure RBAC]]
