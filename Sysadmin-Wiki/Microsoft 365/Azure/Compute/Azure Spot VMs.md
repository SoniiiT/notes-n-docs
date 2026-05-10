---
tags:
  - azure
  - compute
  - spot-vms
  - kostenoptimierung
  - eviction
aliases:
  - Azure Spot
  - Spot VMs Azure
  - Preemptible VMs
---

# Azure Spot VMs

> [!ABSTRACT] TL;DR
> Azure Spot VMs nutzen ungenutzte Azure-Kapazität zu massiv reduzierten Preisen — bis zu 90% günstiger als On-Demand. Im Gegenzug können Spot VMs jederzeit unterbrochen (evicted) werden, wenn Azure die Kapazität zurückbenötigt. Geeignet für unterbrechbare Workloads wie Batch-Verarbeitung, CI/CD und KI-Training.

---

## Was sind Azure Spot VMs?

Spot VMs ermöglichen den Zugriff auf überschüssige Azure-Compute-Kapazität zu stark reduzierten Preisen. Azure stellt diese freie Kapazität zur Verfügung, solange sie nicht von regulären On-Demand-Kunden benötigt wird. Sobald die Kapazität gebraucht wird oder der Spot-Preis den gesetzten Maximalpreis übersteigt, wird die VM unterbrochen.

**Kernprinzipien:**
- Preis variiert basierend auf Angebot und Nachfrage nach VM-Kapazität in einer Region
- Unterbrechung mit 2-Minuten-Vorwarnung
- Kein SLA — keine Verfügbarkeitsgarantie
- Verfügbar in allen Azure-Regionen und für die meisten VM-Serien

---

## Preismodell

### Max-Preis festlegen

Bei der VM-Erstellung wird ein **Max-Preis** (maximaler Stundensatz) definiert:

| Max-Preis-Option | Verhalten |
|-----------------|-----------|
| `-1` (Standard) | VM läuft bis zur Kapazitäts-Eviction; Preis nie > On-Demand-Preis |
| `> 0` (spezifischer Betrag) | VM wird evictet wenn Spot-Preis > Max-Preis ODER Kapazität fehlt |
| `0` | VM sofort evictet wenn Spot-Preis > 0 (selten sinnvoll) |

> [!TIP] Max-Preis auf -1 setzen
> In den meisten Fällen empfiehlt sich `-1` als Max-Preis. Damit läuft die VM zum aktuellen Spot-Preis (immer ≤ On-Demand-Preis) und wird nur bei Kapazitätsengpässen unterbrochen — nicht bei Preisänderungen.

### Aktuelle Spot-Preise abfragen

```bash
# Spot-Preise für eine VM-Größe in einer Region anzeigen
az vm list-skus \
  --location westeurope \
  --size Standard_D4s_v5 \
  --output table

# Spot-Preishistorie (letzten 30 Tage)
az rest \
  --method GET \
  --uri "https://management.azure.com/subscriptions/{subId}/providers/Microsoft.Compute/locations/westeurope/spotPriceHistory?api-version=2024-03-01&vmSizes=Standard_D4s_v5"
```

---

## Eviction Policy

Die Eviction Policy bestimmt, was mit der VM passiert, wenn sie unterbrochen wird:

| Eviction Policy | Verhalten | Kosten nach Eviction | Wiederherstellung |
|----------------|-----------|---------------------|------------------|
| **Deallocate** | VM wird angehalten, Disk bleibt | Kein Compute, Storage weiter | Manueller Neustart möglich |
| **Delete** | VM + alle Disks werden gelöscht | Keine | Keine Wiederherstellung möglich |

**Wann welche Policy:**
- **Deallocate**: Stateful Workloads, Dev/Test, wenn Daten auf lokalem Disk wichtig sind
- **Delete**: Zustandslose Workloads, wenn Daten extern gespeichert werden (Blob, Datenbank)

---

## Eviction-Typen

| Typ | Auslöser | Vorwarnung |
|-----|---------|-----------|
| **Price-based** | Spot-Preis überschreitet gesetzten Max-Preis | 2 Minuten via Scheduled Events |
| **Capacity-based** | Azure benötigt Kapazität zurück | 2 Minuten via Scheduled Events |

> [!WARNING] Keine Garantie für Vorwarnung
> Die 2-Minuten-Vorwarnung wird nach bestem Bemühen bereitgestellt — sie ist nicht garantiert. Workloads müssen mit sofortiger Unterbrechung umgehen können.

---

## Geeignete vs. Nicht-geeignete Workloads

| Workload | Spot-geeignet | Begründung |
|----------|--------------|-----------|
| Batch-Verarbeitung (Azure Batch) | Ja | Checkpointing möglich, kein SLA nötig |
| CI/CD Build-Agents | Ja | Build-Abbruch tolerierbar, einfacher Neustart |
| ML-Training mit Checkpointing | Ja | Modell-Zustand regelmäßig speichern |
| Daten-ETL-Pipelines | Ja | Idempotent ausführbar |
| Rendering-Farmen | Ja | Frame-basiert, einzelne Frames wiederholbar |
| Genomik-Analysen | Ja | Parallelisierbar, Teilergebnisse speicherbar |
| Produktions-Web-Server | Nein | SLA-Anforderungen, kein Ausfall tolerierbar |
| Datenbanken (primär) | Nein | Datenverlust-Risiko, SLA-Anforderungen |
| SAP-Produktionssysteme | Nein | Business-critical, SLA erforderlich |
| Active Directory Controller | Nein | Verfügbarkeit kritisch für gesamte Infrastruktur |
| Stateful Legacy-Anwendungen | Nein | Kein Checkpoint-Mechanismus |

---

## Spot VMs in VM Scale Sets

VM Scale Sets unterstützen eine Mixed-Instance-Policy, die On-Demand und Spot VMs kombiniert.

### Mixed Instance Policy

| Konfiguration | Beschreibung | Anwendungsfall |
|--------------|-------------|----------------|
| **100% Spot** | Alle Instanzen als Spot | Maximale Kostenersparnis, hohe Unterbrechungstoleranz |
| **Spot mit Fallback** | Spot-First, On-Demand als Fallback | Balance zwischen Kosten und Verfügbarkeit |
| **Spot-Anteil (%)** | X% Spot, Rest On-Demand | Definierte Kosten-SLA-Balance |

```bash
# VMSS mit Spot-Instanzen erstellen (100% Spot, Deallocate)
az vmss create \
  --resource-group rg-batch \
  --name vmss-spot-workers \
  --image Ubuntu2204 \
  --vm-sku Standard_D4s_v5 \
  --instance-count 5 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1 \
  --admin-username azureuser \
  --generate-ssh-keys

# VMSS mit Spot-Fallback auf On-Demand
az vmss create \
  --resource-group rg-batch \
  --name vmss-mixed-workers \
  --image Ubuntu2204 \
  --vm-sku Standard_D4s_v5 \
  --instance-count 10 \
  --priority Spot \
  --eviction-policy Delete \
  --max-price -1 \
  --single-placement-group false
```

### Autoscaling mit Spot VMSS

```json
{
  "profiles": [{
    "name": "SpotScaleProfile",
    "capacity": {
      "minimum": "2",
      "maximum": "50",
      "default": "5"
    },
    "rules": [{
      "metricTrigger": {
        "metricName": "Percentage CPU",
        "operator": "GreaterThan",
        "threshold": 70,
        "timeAggregation": "Average",
        "statistic": "Average",
        "timeWindow": "PT5M",
        "timeGrain": "PT1M"
      },
      "scaleAction": {
        "direction": "Increase",
        "type": "ChangeCount",
        "value": "3",
        "cooldown": "PT3M"
      }
    }]
  }]
}
```

---

## Azure Batch mit Low Priority Nodes

In Azure Batch werden Spot-ähnliche VMs als **Low Priority Nodes** bezeichnet. Sie bieten bis zu 80% Rabatt.

```python
# Azure Batch Pool mit Low Priority Nodes
import azure.batch as batch
from azure.batch import models

pool_config = models.PoolAddParameter(
    id="low-priority-pool",
    vm_size="Standard_D4s_v5",
    virtual_machine_configuration=models.VirtualMachineConfiguration(
        image_reference=models.ImageReference(
            publisher="Canonical",
            offer="0001-com-ubuntu-server-jammy",
            sku="22_04-lts-gen2",
            version="latest"
        ),
        node_agent_sku_id="batch.node.ubuntu 22.04"
    ),
    target_dedicated_nodes=2,       # On-Demand Nodes
    target_low_priority_nodes=8,    # Low Priority (Spot) Nodes
    resize_timeout=timedelta(minutes=15)
)
```

| Aspekt | Batch Low Priority | Spot VMs (direkt) |
|--------|-------------------|------------------|
| **Rabatt** | Bis 80% | Bis 90% |
| **Eviction Policy** | Requeue (Task neu starten) | Deallocate oder Delete |
| **Management** | Azure Batch verwaltet | Selbst verwaltet |
| **Checkpointing** | Automatisch via Batch | Manuell implementieren |

---

## Eviction-Behandlung: Scheduled Events

Azure gibt 2 Minuten vor der Eviction eine Vorwarnung über den **Instance Metadata Service (IMDS)**.

### Scheduled Events abfragen

```python
import requests
import json
import time
import signal
import sys

METADATA_URL = "http://169.254.169.254/metadata/scheduledevents"
HEADERS = {"Metadata": "true"}
PARAMS = {"api-version": "2020-07-01"}

def check_eviction():
    """Prüft ob eine Eviction bevorsteht und gibt True zurück wenn ja."""
    try:
        response = requests.get(
            METADATA_URL,
            headers=HEADERS,
            params=PARAMS,
            timeout=5
        )
        events = response.json().get("Events", [])
        for event in events:
            if event.get("EventType") == "Preempt":
                event_id = event.get("EventId")
                not_before = event.get("NotBefore")
                print(f"Eviction angekündigt: EventId={event_id}, NotBefore={not_before}")
                return True, event_id
        return False, None
    except Exception as e:
        print(f"Fehler beim Abfragen der Scheduled Events: {e}")
        return False, None

def acknowledge_event(event_id):
    """Bestätigt das Ereignis (beschleunigt die Eviction nach Checkpoint)."""
    body = {"StartRequests": [{"EventId": event_id}]}
    requests.post(
        METADATA_URL,
        headers=HEADERS,
        params=PARAMS,
        json=body
    )

def graceful_shutdown():
    """Checkpoint speichern, Verbindungen schließen, State sichern."""
    print("Starte Graceful Shutdown...")
    # Hier: Verarbeitungsstand speichern, Queues drainieren, etc.
    time.sleep(5)
    print("Graceful Shutdown abgeschlossen.")

# Polling-Loop
while True:
    eviction_pending, event_id = check_eviction()
    if eviction_pending:
        graceful_shutdown()
        if event_id:
            acknowledge_event(event_id)
        sys.exit(0)
    time.sleep(10)  # Alle 10 Sekunden prüfen
```

---

## Via Azure Portal (UI)

> [!TIP] Schnelleinstieg
> Spot VMs werden beim Erstellen einer regulären VM über die **Erweitert**-Einstellungen aktiviert.

### Spot VM erstellen

1. **portal.azure.com** → **Virtuelle Computer** → **+ Erstellen** → **Virtuelle Azure-Maschine**
2. **Grundlagen**-Tab: Abonnement, Ressourcengruppe, VM-Name, Region, Image wählen
3. **Größe**: VM-Größe auswählen (Spot-fähige Größen: D-, E-, F-Serie usw.)
4. Im **Grundlagen**-Tab → **Azure Spot-Rabatt**: **Ja** aktivieren
   - **Entfernungstyp**: Kapazitätsbasiert oder Preisbasiert
   - **Entfernungsrichtlinie**: Freigeben (Deallocate) oder Löschen (Delete)
   - **Max. Preis (USD/Stunde)**: Eigenes Limit festlegen oder „Aktueller Spot-Preis" wählen
5. Netzwerk, Datenträger, Verwaltung wie bei regulärer VM konfigurieren
6. **Überprüfen + erstellen** → **Erstellen**

### Spot-Preise prüfen

1. portal.azure.com → **Virtuelle Computer** → **+ Erstellen**
2. Bei der Größenauswahl → **Preisdetails anzeigen** → Vergleich Spot vs. On-Demand

### Spot VMs in VM Scale Sets (VMSS)

1. portal.azure.com → **VM-Skalierungsgruppen** → **+ Erstellen**
2. **Orchestrierungsmodus**: Einheitlich oder Flexibel
3. **Instanzdetails** → **Azure Spot-Instanz**: Ja
4. **Entfernungsrichtlinie** & Spot-Prioritätsmix (% Spot vs. Regular) konfigurieren

---

## Spot VM erstellen

### Azure CLI

```bash
# Spot VM erstellen
az vm create \
  --resource-group rg-batch \
  --name vm-spot-worker-01 \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --priority Spot \
  --max-price -1 \
  --eviction-policy Deallocate \
  --admin-username azureuser \
  --generate-ssh-keys \
  --output json

# Spot VM Eviction-Status prüfen
az vm get-instance-view \
  --resource-group rg-batch \
  --name vm-spot-worker-01 \
  --query "instanceView.statuses[?starts_with(code,'PowerState')]"
```

### PowerShell

```powershell
# Spot VM per PowerShell erstellen
$vmConfig = New-AzVMConfig `
  -VMName "vm-spot-01" `
  -VMSize "Standard_D4s_v5" `
  -Priority "Spot" `
  -MaxPrice -1 `
  -EvictionPolicy "Deallocate"

New-AzVM `
  -ResourceGroupName "rg-batch" `
  -Location "westeurope" `
  -VM $vmConfig
```

---

## Bicep-Beispiel: Spot VM

```bicep
param location string = resourceGroup().location
param vmName string = 'vm-spot-worker-01'
param adminUsername string
@secure()
param adminPublicKey string

resource spotNic 'Microsoft.Network/networkInterfaces@2023-09-01' = {
  name: '${vmName}-nic'
  location: location
  properties: {
    ipConfigurations: [{
      name: 'ipconfig1'
      properties: {
        privateIPAllocationMethod: 'Dynamic'
        subnet: {
          id: resourceId('Microsoft.Network/virtualNetworks/subnets', 'vnet-batch', 'snet-workers')
        }
      }
    }]
  }
}

resource spotVM 'Microsoft.Compute/virtualMachines@2023-09-01' = {
  name: vmName
  location: location
  properties: {
    hardwareProfile: {
      vmSize: 'Standard_D4s_v5'
    }
    priority: 'Spot'
    evictionPolicy: 'Deallocate'
    billingProfile: {
      maxPrice: -1   // Nie mehr als On-Demand-Preis
    }
    storageProfile: {
      imageReference: {
        publisher: 'Canonical'
        offer: '0001-com-ubuntu-server-jammy'
        sku: '22_04-lts-gen2'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
        managedDisk: { storageAccountType: 'Standard_LRS' }
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
            keyData: adminPublicKey
          }]
        }
      }
    }
    networkProfile: {
      networkInterfaces: [{ id: spotNic.id }]
    }
  }
}
```

---

## Vergleich: Spot vs. Reserved Instances vs. Savings Plans

| Aspekt | Spot VMs | Reserved Instances | Savings Plans |
|--------|---------|-------------------|--------------|
| **Rabatt** | Bis 90% | 40–60% | 15–35% |
| **Verpflichtung** | Keine | 1 oder 3 Jahre | 1 oder 3 Jahre |
| **Unterbrechung** | Ja (jederzeit) | Nein | Nein |
| **Workload** | Unterbrechbar | Kontinuierlich bekannt | Flexibel, kontinuierlich |
| **VM-Bindung** | Keine (Spot-Preis) | Spezifische VM-Serie/Region | Compute-Familie |
| **Kapazitätsreservierung** | Nein | Optional (Capacity Reservation) | Nein |
| **Ideal für** | Batch, CI/CD, Training | Datenbanken, Web-Server | Gemischte, wechselnde Workloads |

> [!TIP] Kombination empfohlen
> Produktions-Workloads mit **Reserved Instances** absichern und flexible/batch Workloads mit **Spot VMs** optimieren. Damit werden beide Kostenoptimierungen gleichzeitig genutzt.

---

## Verfügbarkeit und Kapazität

Spot-Verfügbarkeit variiert je nach Region, VM-Serie und Tageszeit:

```bash
# Spot-Eviction-Rate für VM-Größen anzeigen (Schätzung)
az rest \
  --method GET \
  --uri "https://management.azure.com/subscriptions/{subId}/providers/Microsoft.Compute/locations/westeurope/spotEvictionRates?api-version=2024-03-01"

# Verfügbarkeit prüfen (versuche VM zu erstellen, prüfe auf Fehler)
az vm create \
  --resource-group rg-test \
  --name test-spot-avail \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --priority Spot \
  --max-price -1 \
  --eviction-policy Delete \
  --no-wait
```

**Strategie für höhere Verfügbarkeit:**
- Mehrere VM-Serien mit ähnlicher Leistung als Alternativen konfigurieren (z.B. D4s_v5 und E4s_v5)
- Über mehrere Regionen verteilen
- Eviction-tolerante Architektur mit Queue-basierter Arbeitsteilung

---

## Verwandte Notizen
- [[Azure Virtual Machines]]
- [[Azure Batch]]
- [[Azure Kubernetes Service]]
