---
tags:
  - azure
  - netzwerk
  - expressroute
  - hybrid-connectivity
  - private-peering
  - bgp
  - dedizierte-verbindung
aliases:
  - ER
  - ExpressRoute Azure
  - Dedicated Connection Azure
  - Azure Private Connectivity
  - ExpressRoute Circuit
---

# Azure ExpressRoute

> [!ABSTRACT] TL;DR
> Azure ExpressRoute ist eine dedizierte private Netzwerkverbindung zwischen On-Premises-Netzwerken und Azure — ohne das öffentliche Internet. Bietet konsistente Latenz, höheren Durchsatz und SLA-garantierte Verbindung. Drei Circuit-SKUs (Local, Standard, Premium) und vier Verbindungsmodelle je nach Provider und Infrastruktur.

---

## Was ist Azure ExpressRoute?

ExpressRoute erstellt eine **private, dedizierte Netzwerkverbindung** zu Azure:

```mermaid
graph LR
    ONPREM[On-Premises Rechenzentrum] -->|Dedizierte Leitung Layer 2 oder 3| PROVIDER[Connectivity Provider oder colocation]
    PROVIDER -->|Microsoft Enterprise Edge Router MSEE| AZURE[Microsoft Azure Global Backbone]
    AZURE --> VNET[Azure VNets]
    AZURE --> M365[Microsoft 365 Dynamics 365]
```

**Vorteile gegenüber VPN:**
- Kein öffentliches Internet im Datenpfad
- Konsistente Latenz (keine Internet-Jitter)
- Höherer Durchsatz (50 Mbps bis 100 Gbps)
- SLA-garantierte Verbindung
- Direkte Anbindung an Microsoft 365 (über Microsoft Peering)

---

## Verbindungsmodelle

### CloudExchange Colocation

Eigenes Equipment in einem Colocation-Rechenzentrum das auch Microsofts Equipment hostet:

```mermaid
graph LR
    CORP_EQUIP[Eigene Router im Co-lo RZ] -->|Cross-Connect Layer 2| MSEE[Microsoft Enterprise Edge]
    MSEE --> AZURE[Azure]
```

**Anbieter:** Equinix, Digital Realty, CyrusOne, Interxion

### Point-to-Point Ethernet

Direkte Ethernet-Verbindung vom eigenen RZ zum Microsoft-Netzwerk über einen Provider:

```mermaid
graph LR
    CORP[Eigenes RZ] -->|Dedicated Ethernet Link| PROVIDER[Connectivity Provider]
    PROVIDER -->|MSEE| AZURE[Azure]
```

**Anbieter:** Telekom, Vodafone, BT, Orange, Telia

### Any-to-Any (IPVPN / MPLS)

Integration in ein bestehendes MPLS/IPVPN-WAN:

```mermaid
graph LR
    HQ[Hauptsitz] -->|MPLS| WAN[Provider WAN]
    BRANCH[Filiale] -->|MPLS| WAN
    WAN -->|MSEE| AZURE[Azure]
```

Azure erscheint als ein weiterer Standort im bestehenden WAN.

### ExpressRoute Direct

Direkte physische Verbindung zu Microsoft-Routern ohne zwischengeschalteten Provider:

```mermaid
graph LR
    CORP[Eigene Router bei Microsoft RZ] -->|Direktverbindung 10G oder 100G Ports| MSEE[Microsoft Enterprise Edge MSEE]
    MSEE --> AZURE[Azure Global Backbone]
```

- **10 Gbps** oder **100 Gbps** dedizierte Ports
- Eigene Ports direkt beim Microsoft-Backbone
- Für höchste Durchsatzanforderungen

---

## Circuit SKUs

| SKU | Abrechnung | Regionen | M365 | Anwendungsfall |
|-----|-----------|---------|------|---------------|
| **Local** | Inkludierte Datenübertragung | Nur eine Geo-Region | ❌ | Lokale Anbindung, günstig |
| **Standard** | Metered oder Unlimited | Alle Regionen einer Geo (z.B. Europa) | ❌ | Standard-Hybridverbindung |
| **Premium** | Metered oder Unlimited | Global (alle Regionen) | ✅ | Globale Firmen, M365 |

**Geo-Regionen für Standard:** Europa, Nordamerika, Asien-Pazifik, Japan, Australien, etc.

---

## Peering-Typen

### Private Peering

Verbindet On-Premises direkt mit Azure VNets:

```mermaid
graph LR
    ONPREM[On-Prem 192 168 0 0/16] -->|BGP Session Private Peering| ER[ExpressRoute Circuit]
    ER -->|ER Gateway| VNET[Azure VNet 10 0 0 0/16]
    ER -->|Gateway Transit| VNET2[Peered VNet 10 1 0 0/16]
```

- BGP-Session zwischen On-Prem-Router und Microsoft (primärer + sekundärer Pfad)
- Routet zu Azure VNet-Adressräumen

### Microsoft Peering

Direkter Zugriff auf Microsoft-Public-Dienste (Microsoft 365, Azure PaaS) ohne Internet:

```mermaid
graph LR
    ONPREM[On-Prem] -->|BGP Session Microsoft Peering| ER[ExpressRoute]
    ER -->|Public Microsoft IPs| M365[Microsoft 365 Exchange, SharePoint]
    ER -->|Public Azure IPs| AZUREPUBLIC[Azure PaaS Storage, SQL etc]
```

- Erfordert eigene öffentliche IP-Adressen (ASN + öffentliche IPs)
- Route-Filter konfigurieren (welche Microsoft-Services über ER)

---

## BGP-Konfiguration

ExpressRoute nutzt **BGP** für dynamisches Routing:

```mermaid
graph LR
    CE[Customer Edge Router CE] -->|BGP Session ASN: 65001| MSEE1[MSEE Primary ASN: 12076]
    CE -->|BGP Session ASN: 65001| MSEE2[MSEE Secondary ASN: 12076]
```

| Parameter | Wert |
|-----------|------|
| Microsoft BGP ASN | 12076 |
| Primärer BGP-Link | /30-Subnet (Peering-IP für Primary) |
| Sekundärer BGP-Link | /30-Subnet (Peering-IP für Secondary) |
| VLAN-ID | Vom Provider zugewiesen |
| MD5 Auth | Optional aber empfohlen |

```powershell
# ExpressRoute Circuit erstellen
$circuit = New-AzExpressRouteCircuit `
  -Name "er-circuit-prod" `
  -ResourceGroupName "rg-connectivity" `
  -Location "westeurope" `
  -SkuTier Standard `
  -SkuFamily MeteredData `
  -ServiceProviderName "Telekom" `
  -PeeringLocation "Frankfurt" `
  -BandwidthInMbps 500

# Private Peering konfigurieren
Add-AzExpressRouteCircuitPeeringConfig `
  -Name PrivatePeering `
  -ExpressRouteCircuit $circuit `
  -PeeringType AzurePrivatePeering `
  -PeerASN 65001 `
  -PrimaryPeerAddressPrefix "192.168.100.0/30" ` # Primary BGP Link
  -SecondaryPeerAddressPrefix "192.168.100.4/30" ` # Secondary BGP Link
  -VlanId 100
```

---

## Bandbreiten

| Bandbreite | Anwendungsfall |
|-----------|---------------|
| 50 Mbps | Kleine Unternehmen, Pilot |
| 100 Mbps | Kleines Unternehmen |
| 200 Mbps | Mittelständler |
| 500 Mbps | Mittelständler |
| 1 Gbps | Enterprise |
| 2 Gbps | Große Enterprise |
| 5 Gbps | Sehr große Enterprise |
| 10 Gbps | Große Datacenter-Integration |
| 40 Gbps | ExpressRoute Direct |
| 100 Gbps | ExpressRoute Direct |

> [!NOTE] Bandwidth-Upgrade
> Bandwidth kann jederzeit **erhöht** (nicht verringert) werden. Bei erhöhen kein Outage. Downgrade erfordert neuen Circuit.

---

## ExpressRoute Direct

```mermaid
graph LR
    CORP[Eigene Fiber bei MSEE] -->|100G Port| MSEE[Microsoft MSEE Router]
    MSEE -->|Multiple Circuits auf einem Port| C1[Circuit 1 10G] & C2[Circuit 2 10G] & C3[Circuit 3 5G]
```

```powershell
# ExpressRoute Direct Port Pair
$directPort = New-AzExpressRoutePort `
  -Name "er-direct-100g" `
  -ResourceGroupName "rg-connectivity" `
  -Location "westeurope" `
  -PeeringLocation "Amsterdam" `
  -BandwidthInGbps 100 `
  -Encapsulation Dot1Q # oder QinQ
```

---

## ExpressRoute Global Reach

On-Premises-Standorte über Microsofts Backbone verbinden (On-Prem zu On-Prem):

```mermaid
graph LR
    CORP_DE[On-Prem Deutschland] -->|ER Circuit 1| MSBB[Microsoft Global Backbone]
    CORP_US[On-Prem USA] -->|ER Circuit 2| MSBB
    MSBB -.->|Global Reach Direkte Verbindung| CORP_DE & CORP_US
```

```powershell
# Global Reach zwischen zwei ER Circuits konfigurieren
Add-AzExpressRouteCircuitConnectionConfig `
  -Name "global-reach-de-us" `
  -ExpressRouteCircuit $circuitDE `
  -PeerExpressRouteCircuitPeering $circuitUS.Peerings[0].Id `
  -AddressPrefix "192.168.200.0/29" # IP-Raum für Global Reach BGP
```

---

## FastPath (Gateway Bypass)

Normaler Traffic-Pfad: On-Prem → ER Circuit → **ER Gateway** → VNet VMs

Mit FastPath: On-Prem → ER Circuit → **direkt zu VM NIC** (Gateway wird umgangen)

```mermaid
graph LR
    ONPREM[On-Prem] -->|Daten| ERGW[ER Gateway Nur noch Control Plane]
    ONPREM -->|FastPath Data Plane bypass| VM[Azure VM Direkt]
```

```powershell
# ExpressRoute Gateway mit FastPath
New-AzVirtualNetworkGateway `
  -GatewayType ExpressRoute `
  -GatewaySku ErGw3AZ ` # ErGw1AZ, ErGw2AZ, ErGw3AZ, ErGwScale unterstützen FastPath
  -EnableFastPathActivation $true
```

**FastPath-Anforderungen:** ErGw3AZ SKU oder höher, Ultra Performance, ErGwScale.

---

## Hochverfügbarkeit

### Zwei Circuits (empfohlen für Produktion)

```mermaid
graph LR
    ONPREM[On-Prem Dual-Router] -->|ER Circuit 1 Provider A| MSEE_A[MSEE A]
    ONPREM -->|ER Circuit 2 Provider B| MSEE_B[MSEE B]
    MSEE_A & MSEE_B --> VNET[Azure VNet]
```

- **Geo-Redundanz:** Circuits in verschiedenen Peering-Locations (z.B. Frankfurt + Amsterdam)
- **Provider-Redundanz:** Verschiedene Provider für physische Diversität
- Beide Circuits aktiv (BGP läuft auf beiden, Load Balancing über AS Path Prepending)

---

## ExpressRoute + VPN Gateway (Failover)

```mermaid
graph LR
    ONPREM[On-Prem] -->|Primär ExpressRoute| ERGW[ER Gateway]
    ONPREM -.->|Failover VPN| VPNGW[VPN Gateway]
    ERGW & VPNGW --> VNET[Azure VNet]
```

BGP-Konfiguration: ER-Routes haben kürzeren AS Path → bevorzugt. Bei ER-Ausfall übernehmen VPN-BGP-Routes automatisch.

---

## Monitoring

```powershell
# ExpressRoute Circuit Health prüfen
Get-AzExpressRouteCircuit -Name "er-circuit-prod" -ResourceGroupName "rg-connectivity" | 
  Select-Object Name, ServiceProviderProvisioningState, CircuitProvisioningState

# BGP-Routing-Informationen
Get-AzExpressRouteCircuitRouteTable `
  -ResourceGroupName "rg-connectivity" `
  -ExpressRouteCircuitName "er-circuit-prod" `
  -PeeringType AzurePrivatePeering `
  -DevicePath Primary
```

### KQL — ExpressRoute Metriken

```kusto
// BitsInPerSecond und BitsOutPerSecond
AzureMetrics
| where ResourceId contains "er-circuit-prod"
| where MetricName in ("BitsInPerSecond", "BitsOutPerSecond")
| summarize avg(Average) by MetricName, bin(TimeGenerated, 5m)
| render timechart
```

---

## Kosten und Provider-Auswahl

**Kostenkomponenten:**
| Komponente | Details |
|-----------|---------|
| **Circuit-Gebühr** | Provider-Preis (Leitung selbst) — variiert stark |
| **Azure Port Fee** | Basierend auf Bandbreite (z.B. 1 Gbps: ~$2.000/Monat) |
| **Metered Data** | Ausgehende Daten in Rechnung (Local/Standard/Premium) |
| **Unlimited Data** | Pauschal, höhere Grundgebühr aber kein Datenlimit |
| **ER Gateway** | ErGw1AZ: ~$0,37/h (~$270/Monat) |

**Provider-Auswahl Kriterien:**
- Latenz zur nächsten Peering-Location (Frankfurt, Amsterdam, Wien, etc.)
- SLA des Providers (99,9% vs. 99,99%)
- Bandbreiten-Flexibilität (Upgrade ohne Provider-Wechsel?)
- Kosten (Port Fee + Mileage)

---

## Verwandte Notizen

- [[Azure Virtual Network]]
- [[Azure VPN Gateway]]
- [[Azure Virtual WAN]]
- [[Azure Network Watcher]]
