---
tags:
  - virtualization
  - hyper-v
  - netzwerk
  - set
  - nic-teaming
  - lbfo
  - high-availability
---

# SET und NIC Teaming

> [!ABSTRACT] TL;DR
> Windows Server unterstützt zwei Teaming-Technologien: das veraltete **LBFO (NIC Teaming)** und das moderne **SET (Switch Embedded Teaming)**. LBFO ist seit Windows Server 2022 deprecated und in Windows Server 2025 nicht mehr verfügbar. SET ist der empfohlene Nachfolger für alle Hyper-V-Umgebungen und bietet bessere Performance, RDMA-Unterstützung sowie native vSwitch-Integration.

---

## Was ist NIC Teaming (LBFO)?

**LBFO** (Load Balancing and Failover) ist das klassische Windows-Feature zum Bündeln mehrerer physischer Netzwerkkarten zu einem logischen Team. Es ist unabhängig von Hyper-V einsetzbar und funktioniert sowohl auf physischen als auch auf virtuellen Maschinen.

### Betriebsmodi

| Modus | Beschreibung | Switch-Anforderung |
|---|---|---|
| **Switch Independent** | Jede NIC kann an einem anderen Switch hängen | Kein spezieller Switch nötig |
| **Switch Dependent – Static** | Switch und Host sind manuell konfiguriert (IEEE 802.3ad Static) | Switch-Unterstützung erforderlich |
| **Switch Dependent – LACP** | Dynamische Aushandlung per Link Aggregation Control Protocol (802.1ax) | LACP-fähiger Switch erforderlich |

### Load-Balancing-Modi

| Modus | Beschreibung |
|---|---|
| **Address Hash** | Traffic wird anhand von IP/MAC/Port-Hash auf NICs verteilt |
| **Hyper-V Port** | Jeder VM-Port wird fest einer physischen NIC zugeordnet |
| **Dynamic** | Kombiniert Address Hash und Hyper-V Port; passt Verteilung automatisch an |

### Einsatzzweck

LBFO bietet zwei Hauptvorteile:
- **Redundanz** — Fällt eine NIC aus, übernehmen die verbleibenden NICs automatisch
- **Bandbreitenbündelung** — Mehrere NICs erscheinen nach außen als eine logische Verbindung mit höherer Gesamtkapazität

### Verfügbarkeit nach Windows Server Version

| Windows Server Version | LBFO-Status |
|---|---|
| 2016 | Vollständig unterstützt |
| 2019 | Vollständig unterstützt |
| 2022 | **Deprecated** (noch funktionsfähig, keine Weiterentwicklung) |
| 2025 | **Entfernt** — nicht mehr verfügbar |

> [!DANGER] LBFO ist abgekündigt
> Microsoft hat LBFO offiziell als deprecated markiert. In Windows Server 2025 ist das Feature vollständig entfernt. Neue Deployments sollten ausschließlich SET verwenden. Bestehende LBFO-Konfigurationen müssen vor einem Upgrade auf WS 2025 migriert werden.

---

## Was ist SET (Switch Embedded Teaming)?

**SET** (Switch Embedded Teaming) ist die moderne Teaming-Technologie von Microsoft, die direkt in den **Hyper-V Virtual Switch** integriert ist. Anstatt ein separates Teaming-Objekt (wie bei LBFO) zu erstellen, werden mehrere physische NICs beim Erstellen des vSwitch direkt gebündelt.

### Architektur

```
  Physische Netzwerkkarten
  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
  │  pNIC1  │   │  pNIC2  │   │  pNIC3  │   │  pNIC4  │
  └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘
       │              │              │              │
       └──────────────┴──────────────┴──────────────┘
                              │
                ┌─────────────▼─────────────┐
                │  SET-fähiger Hyper-V vSwitch │
                │   (EmbeddedTeaming = $true)  │
                └──────┬─────────────┬────────┘
                       │             │
          ┌────────────┘             └──────────────┐
          │                                         │
  ┌───────▼────────┐                   ┌────────────▼──────────┐
  │  VM-Adapter    │                   │  Host-vNICs (ManagOS) │
  │  (VM Traffic)  │                   │  Management / Storage  │
  └────────────────┘                   └───────────────────────┘
```

### Voraussetzungen

> [!IMPORTANT] Hyper-V ist Pflicht
> SET ist **ausschließlich in Hyper-V-Umgebungen** verfügbar. Ohne installierte Hyper-V-Rolle kann SET nicht genutzt werden. Für Bare-Metal-Server ohne Hyper-V muss (solange noch verfügbar) LBFO verwendet werden.

- Hyper-V-Rolle muss installiert sein
- Alle NICs im Team müssen vom **gleichen physischen Typ** sein (gleiche Geschwindigkeit und Hersteller empfohlen)
- Ausschließlich **Switch Independent** als Teaming-Modus (kein LACP auf SET-Ebene)
- Maximal **8 physische NICs** pro SET-Team

### Load-Balancing-Modi

| Modus | Beschreibung | Standard? |
|---|---|---|
| **Dynamic** | Ausgehender Traffic wird per Flowlet-Analyse dynamisch balanciert | Ja |
| **Hyper-V Port** | Jeder vSwitch-Port wird fest einer physischen NIC zugeordnet | Nein |

### Vorteile gegenüber LBFO

| Merkmal | SET | LBFO |
|---|---|---|
| vSwitch-Integration | Nativ, kein Extra-Layer | Separates Objekt über vSwitch |
| Overhead | Gering (direkte Integration) | Höher (zusätzliche Abstraktionsschicht) |
| SR-IOV | Kompatibel | Nicht kompatibel |
| RDMA / SMB Direct | Vollständig unterstützt | Eingeschränkt |
| VMQ (Virtual Machine Queue) | Unterstützt | Eingeschränkt |
| Zukunftssicherheit | Aktiv entwickelt | Deprecated / entfernt |

---

## Vergleichstabelle SET vs. LBFO

| Kriterium | SET | LBFO |
|---|---|---|
| **Verfügbarkeit** | Windows Server 2016+ mit Hyper-V | Windows Server 2012 R2 – 2022 |
| **Hyper-V erforderlich** | Ja (zwingend) | Nein |
| **Betriebsmodi** | Nur Switch Independent | Switch Independent, Static, LACP |
| **Max. NICs im Team** | 8 | 32 |
| **SR-IOV** | Unterstützt | Nicht unterstützt |
| **RDMA / SMB Direct** | Vollständig | Eingeschränkt |
| **VMQ** | Unterstützt | Eingeschränkt |
| **Live Migration** | Unterstützt | Unterstützt |
| **Verwaltung** | PowerShell, Hyper-V Manager | PowerShell, Server Manager |
| **Zukunft** | Aktiv gepflegt | In WS 2025 entfernt |

---

## SET konfigurieren (PowerShell)

### Neuen vSwitch mit SET erstellen

```powershell
# Externes SET-Team mit zwei NICs erstellen
New-VMSwitch `
    -Name "vSwitch-SET" `
    -NetAdapterName @("NIC1", "NIC2") `
    -EnableEmbeddedTeaming $true `
    -AllowManagementOS $false
```

> [!NOTE] AllowManagementOS
> Mit `-AllowManagementOS $false` wird keine automatische Management-vNIC erstellt. In Produktionsumgebungen empfiehlt sich die manuelle Erstellung dedizierter Host-vNICs, um volle Kontrolle über VLANs und Traffic-Klassen zu behalten.

### Vorhandenen vSwitch auf SET prüfen

```powershell
# Alle vSwitches und SET-Status anzeigen
Get-VMSwitch | Select-Object Name, EmbeddedTeamingEnabled, NetAdapterInterfaceDescriptions

# NIC-Mitglieder im SET-Team prüfen
Get-VMSwitchTeam -Name "vSwitch-SET"
```

### Host-vNIC für Management erstellen

```powershell
# Management-vNIC am vSwitch anlegen
Add-VMNetworkAdapter `
    -ManagementOS `
    -Name "vNIC-Management" `
    -SwitchName "vSwitch-SET"

# IP-Adresse zuweisen
New-NetIPAddress `
    -InterfaceAlias "vEthernet (vNIC-Management)" `
    -IPAddress "192.168.1.10" `
    -PrefixLength 24 `
    -DefaultGateway "192.168.1.1"
```

### VLAN auf Host-vNIC setzen

```powershell
# Management-vNIC auf VLAN 10
Set-VMNetworkAdapterVlan `
    -ManagementOS `
    -VMNetworkAdapterName "vNIC-Management" `
    -Access `
    -VlanId 10

# Storage-vNIC auf VLAN 20
Set-VMNetworkAdapterVlan `
    -ManagementOS `
    -VMNetworkAdapterName "vNIC-Storage" `
    -Access `
    -VlanId 20
```

### Load-Balancing-Algorithmus ändern

```powershell
# Auf Hyper-V Port wechseln
Set-VMSwitchTeam -Name "vSwitch-SET" -LoadBalancingAlgorithm HyperVPort

# Zurück auf Dynamic (Standard)
Set-VMSwitchTeam -Name "vSwitch-SET" -LoadBalancingAlgorithm Dynamic
```

---

## RDMA und SMB Direct mit SET

### Wozu RDMA mit SET?

RDMA (Remote Direct Memory Access) ermöglicht Netzwerkkommunikation mit minimaler CPU-Last und sehr hohem Durchsatz. In Kombination mit SMB Direct (SMB über RDMA) wird dies vor allem für **Storage-Traffic** genutzt — etwa beim Zugriff auf Scale-Out File Server (SOFS) oder bei Hyper-V Cluster Shared Volumes.

SET ist die einzige Teaming-Technologie, die RDMA auf Host-vNICs vollständig unterstützt.

### Unterstützte RDMA-Technologien

| Technologie | Beschreibung | Typische Hardware |
|---|---|---|
| **RoCE v1/v2** | RDMA over Converged Ethernet | Mellanox ConnectX-4/5/6, Broadcom |
| **iWARP** | Internet Wide Area RDMA Protocol | Intel X550, Chelsio |
| **InfiniBand** | Dediziertes Hochgeschwindigkeitsnetz | Mellanox InfiniBand HCAs |

### RDMA-Konfiguration mit SET

```powershell
# 1. RDMA auf physischen NICs aktivieren
Enable-NetAdapterRDMA -Name "NIC1", "NIC2"

# 2. SET-vSwitch erstellen
New-VMSwitch `
    -Name "vSwitch-Converged" `
    -NetAdapterName @("NIC1", "NIC2") `
    -EnableEmbeddedTeaming $true `
    -AllowManagementOS $false

# 3. Storage-vNICs erstellen
Add-VMNetworkAdapter -ManagementOS -Name "vNIC-Storage-A" -SwitchName "vSwitch-Converged"
Add-VMNetworkAdapter -ManagementOS -Name "vNIC-Storage-B" -SwitchName "vSwitch-Converged"

# 4. RDMA auf Host-vNICs aktivieren
Enable-NetAdapterRDMA -Name "vEthernet (vNIC-Storage-A)"
Enable-NetAdapterRDMA -Name "vEthernet (vNIC-Storage-B)"

# 5. Status prüfen
Get-NetAdapterRDMA | Select-Object Name, Enabled
Get-SmbClientNetworkInterface | Select-Object IpAddress, RdmaCapable
```

> [!TIP] Dedizierte Storage-vNICs
> Verwende **separate Host-vNICs** für Storage- und Management-Traffic. Weise jeder Storage-vNIC ein eigenes VLAN zu und konfiguriere QoS/DCB für priorisierten RDMA-Traffic.

---

## Von LBFO auf SET migrieren

> [!WARNING] Downtime erforderlich
> Eine Online-Migration von LBFO auf SET ist **nicht möglich**. Der Umbau erfordert kurze Netzwerk-Unterbrechungen. In Clustern kann dies durch rollendes Vorgehen (Node für Node) minimiert werden.

### Migration (Einzelner Host)

```powershell
# 1. Bestehende Konfiguration anzeigen
Get-NetLbfoTeam
Get-VMSwitch

# 2. VMs ggf. auf anderen Host migrieren (Cluster)
# Move-VM -Name "MeineVM" -DestinationHost "AndererHost"

# 3. Alten vSwitch entfernen
Remove-VMSwitch -Name "AlterVSwitch" -Force

# 4. LBFO-Team auflösen
Remove-NetLbfoTeam -Name "AltesTEAM" -Confirm:$false

# 5. Neuen SET-vSwitch erstellen
New-VMSwitch `
    -Name "vSwitch-SET" `
    -NetAdapterName @("NIC1", "NIC2") `
    -EnableEmbeddedTeaming $true `
    -AllowManagementOS $false

# 6. Host-vNICs neu erstellen und konfigurieren
Add-VMNetworkAdapter -ManagementOS -Name "vNIC-Management" -SwitchName "vSwitch-SET"
Add-VMNetworkAdapter -ManagementOS -Name "vNIC-Storage"    -SwitchName "vSwitch-SET"
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "vNIC-Management" -Access -VlanId 10
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "vNIC-Storage"    -Access -VlanId 20

# 7. VMs mit neuem vSwitch verbinden
Get-VM | Get-VMNetworkAdapter | Connect-VMNetworkAdapter -SwitchName "vSwitch-SET"
```

### Rolling Migration im Failover-Cluster

> [!TIP] Knotenweise Migration
> 1. Alle VMs von **Node A** per Live Migration auf andere Knoten verschieben
> 2. Node A pausieren und aus dem Cluster drainieren
> 3. LBFO auflösen, SET-vSwitch erstellen, vNICs konfigurieren
> 4. Node A wieder in den Cluster aufnehmen und VMs zurückmigrieren
> 5. Vorgang für jeden weiteren Knoten wiederholen

---

## Typische Einsatzszenarien

| Szenario | Empfehlung | Begründung |
|---|---|---|
| **Kleiner Hyper-V-Host (2 NICs)** | SET mit 2 NICs, Dynamic LB | Redundanz ohne zusätzlichen Switch-Aufwand |
| **Mittlerer Host (4 NICs)** | SET mit 4 NICs + RDMA | Storage und VM-Traffic konvergiert |
| **Enterprise-Cluster (4–8 NICs)** | SET + RDMA + DCB/QoS | Vollständiges Converged Fabric |
| **Standalone-Server ohne Hyper-V** | LBFO (nur bis WS 2022) | SET erfordert Hyper-V |
| **Windows Server 2025** | SET (LBFO nicht mehr verfügbar) | Keine Wahlmöglichkeit |

### Konvergentes Netzwerk (Converged Fabric)

```
  2x oder 4x 25 GbE RDMA-NICs
              │
  SET-vSwitch (EmbeddedTeaming)
              │
  ┌───────────┼───────────┐
  │           │           │
vNIC-Mgmt  vNIC-Storage  VM-Adapter
VLAN 10    VLAN 20       VLAN 100-199
RDMA: nein RDMA: ja      QoS: VM-Traffic
```

---

## Troubleshooting

### SET-Status prüfen

```powershell
# Vollständiger Status aller SET-Teams
Get-VMSwitchTeam | Format-List *

# NIC-Status im Team
Get-VMSwitch -Name "vSwitch-SET" |
    Get-VMSwitchTeam |
    ForEach-Object {
        $_.NetAdapterInterfaceDescription | ForEach-Object {
            Get-NetAdapter -InterfaceDescription $_ |
                Select-Object Name, Status, LinkSpeed
        }
    }
```

### Failover testen

```powershell
# NIC deaktivieren (simulierter Ausfall)
Disable-NetAdapter -Name "NIC1" -Confirm:$false

# Konnektivität prüfen, dann NIC wieder aktivieren
Enable-NetAdapter -Name "NIC1"
```

### Häufige Fehler

> [!WARNING] Bekannte Probleme
>
> **NICs unterschiedlicher Geschwindigkeit:** Microsoft empfiehlt **identische NICs** im Team. Mischbetrieb kann zu unvorhersehbarem Load-Balancing führen.
>
> **SR-IOV Failover:** SR-IOV Virtual Functions sind nur auf einer physischen NIC aktiv. Bei NIC-Failover fällt die VM kurzzeitig auf Software-Switching zurück.
>
> **RDMA auf vNICs nicht aktiv:** Prüfe (1) RDMA auf physischer NIC aktiv, (2) vSwitch mit `-EnableEmbeddedTeaming $true` erstellt, (3) NIC-Treiber unterstützt RDMA über SET.
>
> **SET-vSwitch lässt sich nicht erstellen:** Stelle sicher, dass die NICs nicht bereits in einem LBFO-Team, einem anderen vSwitch oder einer Bridge sind.

```powershell
# Ereignisprotokoll auf vSwitch-Fehler prüfen
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VmSwitch/Operational" `
    -MaxEvents 50 |
    Where-Object LevelDisplayName -ne "Information" |
    Select-Object TimeCreated, Message
```

---

## Verwandte Notizen

- [[Hyper-V Netzwerk]]
- [[Hyper-V Best Practices]]
- [[Hyper-V Storage]]
