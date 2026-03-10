---
tags:
  - azure
  - arc
  - edge-computing
  - iot
  - appliance
aliases:
  - Stack Edge
  - Azure Stack Edge Pro
  - ASE
---

# Azure Stack Edge

> [!ABSTRACT] TL;DR
> Azure Stack Edge ist Microsofts Hardware-as-a-Service-Familie für Edge-Computing. Die Appliances bringen Azure-Computing (VMs, Container, Kubernetes, KI/ML mit GPU) direkt an Standorte mit eingeschränkter oder keiner Azure-Verbindung — für IoT-Datenverarbeitung, Machine Learning am Edge und hybride Szenarien.

---

## Was ist Azure Stack Edge?

**Azure Stack Edge** ist eine Linie von Cloud-managed Hardware-Appliances, die Azure-Compute, -Storage und -KI-Dienste am Edge bereitstellen. Im Gegensatz zu Azure Stack HCI (Standard-Server-Hardware) sind Azure Stack Edge-Geräte **von Microsoft designte und gelieferte Appliances**.

Das Grundprinzip: Daten lokal verarbeiten, bevor sie — gefiltert und reduziert — in die Azure Cloud hochgeladen werden.

```mermaid
graph LR
    IOT_DEVICES[IoT Geräte Sensoren Kameras PLCs]
    ASE[Azure Stack Edge Appliance]
    PROCESS[Lokale Verarbeitung KI Filterung Aggregation]
    UPLOAD[Selektiver Upload nur relevante Daten]
    AZURE[Azure Cloud Speicher Analytics]

    IOT_DEVICES --> ASE
    ASE --> PROCESS
    PROCESS --> UPLOAD
    UPLOAD --> AZURE
```

---

## Edge-Computing-Szenarien

**Warum Edge-Computing?**

| Szenario | Problem | Azure Stack Edge Lösung |
|---|---|---|
| **Niedrige Latenz** | Cloud-Roundtrip zu langsam für Echtzeit | Lokale Inferenz in <10ms |
| **Hohe Datenmenge** | Alle Rohdaten in Azure zu teuer | Lokale Vorverarbeitung, nur Ergebnisse in Cloud |
| **Schlechte Konnektivität** | Offshore, Fabrik ohne stabiles Internet | Offline-fähiger Betrieb |
| **Compliance** | Daten dürfen Standort nicht verlassen | Lokale Verarbeitung, nur Aggregate in Cloud |
| **Echtzeit-KI** | Bildverarbeitung, Anomalieerkennung | NVIDIA GPU für lokale Inferenz |

---

## Produktfamilie

### Überblick aller SKUs

| Modell | Formfaktor | GPU | Use Case |
|---|---|---|---|
| **Pro 2** | 1U Rack | 1–2x NVIDIA A2 (optional) | Rechenzentrum, Edge-Standorte |
| **Pro R** | Robuste 1U Box | 1x NVIDIA T4 | Raue Umgebungen, Outdoor |
| **Mini R** | Ultra-Portabel | Intel VPU | Taktische Edge, Feldarbeit |

### Azure Stack Edge Pro 2

Der **Pro 2** ist die aktuelle Hauptversion für Unternehmenseinsatz:

```mermaid
graph TD
    PRO2[Azure Stack Edge Pro 2]
    COMPUTE[Compute Intel Xeon Gold bis 32 vCPUs nutzbar]
    MEMORY[Memory bis 204 GB RAM]
    STORAGE[Storage bis 2.5 TB lokales SSD]
    GPU_OPT[GPU Option 1-2x NVIDIA A2 für KI-Inferenz]
    NETWORK[Netzwerk bis 100GbE Ports]
    MGMT[Management Azure Portal lokal]

    PRO2 --> COMPUTE
    PRO2 --> MEMORY
    PRO2 --> STORAGE
    PRO2 --> GPU_OPT
    PRO2 --> NETWORK
    PRO2 --> MGMT
```

**Technische Spezifikationen Pro 2:**

| Komponente | Spezifikation |
|---|---|
| CPU | Intel Xeon Gold, bis 32 nutzbare vCPUs |
| RAM | Bis 204 GB |
| Storage | Bis 2,5 TB SSD lokal |
| GPU | Optional: 1–2x NVIDIA A2 (16 GB VRAM je) |
| Netzwerk | 2x 1GbE (Management), 4x 25GbE oder 2x 100GbE (Data) |
| Formfaktor | 1U Rack |

### Azure Stack Edge Pro R

Der **Pro R** ist für **raue Umgebungen** ausgelegt:

| Merkmal | Beschreibung |
|---|---|
| **Zertifizierungen** | MIL-STD-810H, IP-54-Schutz |
| **Temperaturen** | -20°C bis +65°C Betrieb |
| **Erschütterungen** | Stoß- und vibrationsfest |
| **Stromversorgung** | Redundante USV integriert |
| **GPU** | NVIDIA T4 (16 GB VRAM) |
| **Use Cases** | Militär, Offshore, Fahrzeugmontage, Baustellen |

### Azure Stack Edge Mini R

Der **Mini R** ist das ultra-portable Modell für Feldeinsätze:

| Merkmal | Beschreibung |
|---|---|
| **Gewicht** | ca. 3 kg |
| **Formfaktor** | Tragbares Gerät mit Handgriff |
| **Compute** | Intel VPU für KI-Inferenz |
| **Konnektivität** | WiFi + LTE (SIM-Karte) |
| **Akku** | Integrierter Akku |
| **Use Cases** | Militär, Katastrophenschutz, Feldforschung |

> [!NOTE] Subscription-Modell
> Azure Stack Edge wird als **Hardware-as-a-Service** angeboten: Monatliche Azure-Subscription inkl. Hardware-Support und Ersatz. Keine einmaligen Kaufkosten, stattdessen laufende opex-basierte Kosten.

---

## Azure Stack Edge als Arc-fähiges Gerät

Azure Stack Edge-Geräte sind **Arc-fähige Ressourcen** und werden über Azure Arc verwaltet:

```mermaid
graph TD
    ASE_DEVICE[Azure Stack Edge Gerät vor Ort]
    ARC[Azure Arc Registrierung]
    ARM[Azure Resource Manager]
    PORTAL[Azure Portal]
    EXTENSIONS[Arc Extensions Monitor Defender]

    ASE_DEVICE --> ARC
    ARC --> ARM
    ARM --> PORTAL
    ARC --> EXTENSIONS
```

**Was Arc auf ASE ermöglicht:**
- Zentrale Inventarisierung aller Edge-Geräte
- Azure Policy für Compliance-Überprüfung
- Remote-Diagnose und Troubleshooting
- Extension-Deployment (Azure Monitor Agent, Defender)
- Softwareupdate-Management via Azure Update Manager

---

## IoT Edge auf Azure Stack Edge

**Azure IoT Edge** kann auf Azure Stack Edge als Workload deployt werden:

```mermaid
graph LR
    IOT_HUB[Azure IoT Hub in Azure]
    ASE[Azure Stack Edge]
    IOTEDGE_RT[IoT Edge Runtime]
    MODULES[IoT Edge Module Container]
    DEVICES[Downstream IoT Geräte]

    IOT_HUB -->|Konfiguration Manifeste| IOTEDGE_RT
    IOTEDGE_RT --> MODULES
    MODULES --> DEVICES
    MODULES -->|Gefilterte Daten| IOT_HUB
```

**Typische IoT Edge-Module auf ASE:**
- Stream Analytics Edge (lokale Datenverarbeitung)
- Custom Vision (Bildanalyse am Edge)
- Anomalie-Detection-Module
- OPC UA Bridge (Industrie-4.0-Protokoll)
- Zeit-Reihen-Datenbank (InfluxDB, TimescaleDB)

---

## Azure Kubernetes Service auf Edge

Azure Stack Edge kann einen **Kubernetes-Cluster** hosten, der über Azure Arc verwaltet wird:

```bash
# Kubernetes auf ASE ist automatisch aktiviert
# Zugriff via kubectl über lokales Profil
kubectl get nodes

# Arc-Verbindung für ASE-Kubernetes
az connectedk8s connect \
  --name "myASE-K8s" \
  --resource-group "myRG" \
  --location "westeurope" \
  --kube-config "/path/to/ase-kubeconfig"
```

**Kubernetes-Workloads auf ASE:**
- Container-Applikationen (Microservices)
- Machine Learning Inferenz-Services (TensorFlow Serving, ONNX)
- IoT Edge-Module als Kubernetes-Pods
- Lokale Datenbank-Container

---

## Compute: VMs, Container, Kubernetes

Azure Stack Edge unterstützt drei Compute-Paradigmen:

```mermaid
graph TD
    ASE[Azure Stack Edge]
    VMS[Virtual Machines Hyper-V basiert]
    CONTAINERS[Container Docker kompatibel]
    K8S[Kubernetes Arc-verbunden]

    ASE --> VMS
    ASE --> CONTAINERS
    ASE --> K8S
```

### Virtual Machines auf ASE

VMs auf Azure Stack Edge werden über den **Lokalen Azure Resource Manager** (lokaler ARM-Endpunkt) oder das Azure Portal verwaltet:

```powershell
# VM über lokalen ARM-Endpunkt erstellen (PowerShell)
$password = ConvertTo-SecureString "<password>" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("admin", $password)

New-AzVM `
  -ResourceGroupName "localRG" `
  -Name "myEdgeVM" `
  -Location "dbelocal" `
  -Image "Win2019Datacenter" `
  -Credential $cred `
  -Size "Standard_D2s_v3"
```

---

## Lokaler Azure Resource Manager

Ein einzigartiges Feature von Azure Stack Edge ist der **lokale ARM-Endpunkt** — ein vollständiger Azure Resource Manager, der lokal auf der Appliance läuft:

```mermaid
graph TD
    USER[Administrator]
    LOCAL_ARM[Lokaler ARM auf ASE-Gerät]
    LOCAL_PORTAL[Lokales Azure Portal https://local-mgmt-ip]
    CLOUD_ARM[Azure ARM in der Cloud]
    SYNC[Synchronisation Status und Metadaten]

    USER --> LOCAL_ARM
    USER --> LOCAL_PORTAL
    LOCAL_ARM --> LOCAL_PORTAL
    LOCAL_ARM -.->|Online: Sync| CLOUD_ARM
    LOCAL_ARM -->|Offline: Lokal funktionsfähig| LOCAL_ARM
    CLOUD_ARM --> SYNC
```

**Lokaler ARM ermöglicht:**
- VM- und Container-Deployment **offline** (ohne Azure-Verbindung)
- Lokales Portal für Geräteverwaltung
- ARM-Templates für konsistentes lokales Deployment
- Identische Erfahrung wie in der Azure Cloud

---

## Netzwerk-Konfiguration

```mermaid
graph LR
    subgraph ASE_Network
        PORT1[Port 1 1 GbE Management]
        PORT2[Port 2 1 GbE Management Redundant]
        PORT3[Port 3 10-25 GbE Data]
        PORT4[Port 4 10-25 GbE Data]
        PORT5[Port 5 10-100 GbE Data High Speed]
        PORT6[Port 6 10-100 GbE Data High Speed]
    end

    MGMT_NET[Management-Netzwerk]
    DATA_NET[Datennetzwerk]

    PORT1 --> MGMT_NET
    PORT2 --> MGMT_NET
    PORT3 --> DATA_NET
    PORT4 --> DATA_NET
    PORT5 --> DATA_NET
    PORT6 --> DATA_NET
```

### VPN-Konfiguration

```powershell
# VPN auf Azure Stack Edge konfigurieren (PowerShell via lokaler ARM)
New-AzVirtualNetworkGateway `
  -Name "myASE-VPN" `
  -ResourceGroupName "localRG" `
  -Location "dbelocal" `
  -GatewayType "Vpn" `
  -VpnType "RouteBased"
```

---

## Storage: Lokale Freigaben und Blob-Endpunkt

Azure Stack Edge bietet lokalen Speicher, der mit Azure Blob Storage synchronisiert werden kann:

```mermaid
graph LR
    APPS[Lokale Applikationen VMs Container K8s]
    LOCAL_SHARE[Lokale SMB/NFS-Freigabe auf ASE]
    CACHE[Edge Cache automatische Tiering]
    BLOB[Azure Blob Storage in Azure]

    APPS --> LOCAL_SHARE
    LOCAL_SHARE --> CACHE
    CACHE -->|Automatischer Upload| BLOB
    BLOB -->|Download on demand| CACHE
```

**Storage-Typen:**
- **Edge Shares**: SMB/NFS-Freigaben, die mit Azure Blob synchronisieren
- **Local Shares**: Rein lokale Freigaben (kein Cloud-Sync)
- **Edge Local Storage**: Direkte Kubernetes Persistent Volumes
- **Blob-Endpunkt**: Lokaler S3-kompatibler Blob-Endpunkt

```bash
# Blob-Endpunkt auf ASE verwenden (Azure CLI)
az storage blob upload \
  --account-name "myASEAccount" \
  --account-key "<key>" \
  --container-name "myContainer" \
  --file "localdata.csv" \
  --name "data/localdata.csv" \
  --connection-string "DefaultEndpointsProtocol=https;AccountName=myASE;AccountKey=<key>;BlobEndpoint=https://<ASE-IP>:11002/myASE;"
```

---

## Deployment-Szenarien

### Szenario 1: Vollständig verbunden (Connected)

```mermaid
graph LR
    ASE[ASE Gerät]
    AZURE[Azure Cloud]
    REAL_TIME[Echtzeit-Management Monitoring Updates]

    ASE <-->|Permanente Verbindung| AZURE
    AZURE --> REAL_TIME
```

### Szenario 2: Zeitweise verbunden (Semi-connected)

```mermaid
graph LR
    ASE[ASE Gerät Offshore Schiff]
    SATLINK[Satellitenverbindung periodisch]
    AZURE[Azure Cloud]

    ASE -.->|1-2x täglich| SATLINK
    SATLINK --> AZURE
```

### Szenario 3: Vollständig getrennt (Disconnected)

```mermaid
graph LR
    ASE[ASE Gerät Klassifiziertes Netzwerk]
    AIRGAP[Air-Gap keine Verbindung]
    MANUAL[Manueller Export USB Datenträger]
    AZURE[Azure Portal nur Konfiguration vorab]

    ASE --> AIRGAP
    ASE -.->|Periodisch physisch| MANUAL
    AZURE -.->|Konfiguration einmalig| ASE
```

---

## Machine Learning am Edge

Azure Stack Edge mit NVIDIA GPU ist optimiert für **KI-Inferenz am Edge**:

```mermaid
graph TD
    MODEL[ML-Modell trainiert in Azure ML]
    EXPORT[Export ONNX TensorFlow Lite]
    DEPLOY[Deployment auf ASE als Container]
    INFERENCE[Lokale Inferenz ohne Cloud-Latenz]
    RESULTS[Ergebnisse lokal verarbeitet]
    UPLOAD[Upload nur Anomalien in Azure]

    MODEL --> EXPORT
    EXPORT --> DEPLOY
    DEPLOY --> INFERENCE
    INFERENCE --> RESULTS
    RESULTS --> UPLOAD
```

**Unterstützte ML-Frameworks auf ASE:**
- **TensorFlow / TensorFlow Serving**
- **PyTorch** (über ONNX Runtime)
- **ONNX Runtime** (CPU und GPU)
- **NVIDIA Triton Inference Server** (für Pro 2 mit A2-GPU)
- **Azure Machine Learning Inferencing** (als Container)

---

## Verwaltung via Azure Portal und lokal

| Aufgabe | Azure Portal | Lokales Web-Interface | PowerShell |
|---|---|---|---|
| Geräte-Onboarding | ✅ | ❌ | ✅ |
| Netzwerk-Konfiguration | Eingeschränkt | ✅ | ✅ |
| Share-Verwaltung | ✅ | ✅ | ✅ |
| VM-Deployment | ✅ (lokal ARM) | ✅ | ✅ |
| Kubernetes | ✅ (via Arc) | ❌ | ✅ |
| Logs und Diagnose | ✅ (Azure Monitor) | ✅ | ✅ |
| Updates | ✅ (Update Manager) | ✅ | ✅ |
| Offline-Verwaltung | ❌ | ✅ | ✅ |

---

## Verwandte Notizen
- [[Azure Arc Grundlagen]]
- [[Azure Stack HCI]]
- [[Azure IoT Edge]]
- [[Azure IoT Hub]]
