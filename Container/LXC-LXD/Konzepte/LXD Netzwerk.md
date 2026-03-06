---
tags: [lxd, netzwerk, bridge, macvlan, ovn, routing, nat]
aliases: ["LXD Netzwerk", "LXD Networking"]
---

# LXD Netzwerk

> [!ABSTRACT]
> LXD bietet flexible Netzwerkkonfiguration: Managed Bridges mit NAT, macvlan für direkten Zugriff im physischen Netz, OVN für Software-Defined Networking im Cluster und Proxy-Devices für Port-Forwarding. Dieser Artikel erklärt alle Netzwerktypen und typische Setups.

---

## Netzwerk-Typen im Überblick

| Typ | Beschreibung | Anwendungsfall |
|-----|-------------|---------------|
| `bridge` | Managed Bridge mit NAT | Standard-Setup, internes Netz |
| `macvlan` | Direkter physischer Netz-Zugang | Container als LAN-Gerät |
| `ipvlan` | Wie macvlan, IP-basiert | Viele IPs auf einem Interface |
| `sriov` | SR-IOV Hardware-Netz | High-Performance |
| `ovn` | Open Virtual Network | Cluster, SDN |
| `physical` | Physisches Interface übergeben | Direkte HW-Übergabe |

---

## Managed Bridge (Standard)

Die einfachste und häufigste Netzwerkkonfiguration:

```
Host
├── eth0 (172.16.0.5 - physisches Interface)
└── lxdbr0 (10.10.10.1/24 - Bridge)
    ├── Container A: 10.10.10.2
    ├── Container B: 10.10.10.3
    └── Container C: 10.10.10.4
```

Container kommunizieren untereinander über die Bridge. Ausgehender Traffic wird per NAT über das Host-Interface geleitet.

### Bridge erstellen und konfigurieren

```bash
# Standard-Bridge anzeigen
lxc network show lxdbr0

# Neue Bridge erstellen
lxc network create meinbr0 \
    ipv4.address=192.168.100.1/24 \
    ipv4.nat=true \
    ipv6.address=none

# DNS-Server für Container
lxc network set meinbr0 dns.nameservers=1.1.1.1,8.8.8.8

# DHCP-Bereich anpassen
lxc network set meinbr0 ipv4.dhcp.ranges=192.168.100.100-192.168.100.200

# Bridge einem Profil zuweisen
lxc profile device set default eth0 network=meinbr0

# Alle Netzwerke anzeigen
lxc network list
```

### Statische IP vergeben

```bash
# Feste IP für Container
lxc config device set mycontainer eth0 ipv4.address 192.168.100.50

# Oder via DHCP-Reservation (MAC-Adresse)
lxc config show mycontainer | grep hwaddr
lxc network set meinbr0 "ipv4.dhcp.reservations.mycontainer=MAC:192.168.100.50"
```

---

## macvlan — Container direkt im LAN

macvlan gibt dem Container eine eigene MAC-Adresse und macht ihn zu einem vollwertigen Gerät im physischen Netz:

```
Physisches Netz (192.168.1.0/24)
├── Router: 192.168.1.1
├── Host (eth0): 192.168.1.10  ← Host selbst NICHT mehr erreichbar von Containern!
├── Container A (macvlan): 192.168.1.20
└── Container B (macvlan): 192.168.1.21
```

> [!WARNING]
> **macvlan-Limitation:** Host und Container können sich bei macvlan NICHT direkt erreichen. Das ist eine Kernel-Einschränkung. Für Host→Container-Kommunikation: Bridge oder ipvlan verwenden.

```bash
# macvlan NIC zu Container hinzufügen
lxc config device add mycontainer eth0 nic \
    nictype=macvlan \
    parent=eth0

# Oder als Netzwerk-Ressource
lxc network create macvlan-net \
    --type=macvlan \
    parent=eth0

lxc config device add mycontainer eth0 nic \
    network=macvlan-net
```

---

## Port-Forwarding mit Proxy-Devices

Port-Forwarding ohne Router-Konfiguration:

```bash
# HTTP-Traffic weiterleiten (Host:8080 → Container:80)
lxc config device add mycontainer http-proxy proxy \
    listen=tcp:0.0.0.0:8080 \
    connect=tcp:127.0.0.1:80

# HTTPS weiterleiten
lxc config device add mycontainer https-proxy proxy \
    listen=tcp:0.0.0.0:443 \
    connect=tcp:127.0.0.1:443

# UDP weiterleiten
lxc config device add mycontainer dns-proxy proxy \
    listen=udp:0.0.0.0:53 \
    connect=udp:127.0.0.1:53

# Proxy entfernen
lxc config device remove mycontainer http-proxy
```

---

## Netzwerk-Forward (LXD 5.x+)

Modernes Port-Forwarding über das Netzwerk-Objekt selbst:

```bash
# Forward-Regel erstellen (Port 80 auf Container-IP)
lxc network forward create lxdbr0 0.0.0.0 \
    target_address=10.10.10.50 \
    ports='[{"listen_port": "80", "target_port": "80", "protocol": "tcp"}]'

# Forward anzeigen
lxc network forward list lxdbr0

# Forward löschen
lxc network forward delete lxdbr0 0.0.0.0
```

---

## Mehrere Netzwerk-Interfaces

Container können mehrere Netzwerk-Interfaces haben:

```bash
# Zweites Interface (anderes Subnetz) hinzufügen
lxc config device add mycontainer eth1 nic \
    nictype=bridged \
    parent=management-br

# Interface mit fester MAC
lxc config device add mycontainer eth0 nic \
    nictype=bridged \
    parent=lxdbr0 \
    hwaddr=00:16:3e:12:34:56

# Netzwerk im Container konfigurieren
lxc exec mycontainer -- ip addr show
lxc exec mycontainer -- ip route show
```

---

## OVN — Open Virtual Network (Cluster)

OVN ist das empfohlene Netzwerk für LXD-Cluster:

```
OVN Provider Network
├── Northbound DB (Logical Network State)
├── Southbound DB (Physical Network State)
└── OVN Controllers (auf jedem Cluster-Node)

┌──────────────────────────────────────────────┐
│               LXD Cluster                    │
│  ┌─────────┐     ┌─────────┐    ┌─────────┐ │
│  │ Node 1  │     │ Node 2  │    │ Node 3  │ │
│  │ ovn-ctl │─────│ ovn-ctl │────│ ovn-ctl │ │
│  └─────────┘     └─────────┘    └─────────┘ │
└──────────────────────────────────────────────┘
```

```bash
# OVN installieren (auf allen Nodes)
sudo apt install ovn-host ovn-central

# OVN als Uplink-Netzwerk konfigurieren
lxc network create UPLINK \
    --type=physical \
    parent=eth0

# OVN Virtual Network erstellen
lxc network create ovn-net \
    --type=ovn \
    network=UPLINK \
    ipv4.address=10.20.0.1/24 \
    ipv4.nat=true
```

---

## DNS und Service-Discovery

```bash
# Hostname eines Containers auflösen
# Container-Name wird automatisch als DNS-Name registriert
lxc exec mycontainer -- nslookup other-container.lxd

# Custom DNS-Einträge
lxc network set lxdbr0 "dns.hosts.myapp=10.10.10.50"

# DNS-Resolver im Container prüfen
lxc exec mycontainer -- cat /etc/resolv.conf
```

---

## Firewall und Sicherheit

LXD konfiguriert automatisch iptables/nftables-Regeln:

```bash
# Container-Firewall-Regeln ansehen
sudo iptables -L FORWARD -n | grep lxdbr0

# Firewall für Container deaktivieren (nicht empfohlen)
lxc network set lxdbr0 ipv4.firewall false

# Spezifische Ports im Container blockieren
lxc exec mycontainer -- ufw enable
lxc exec mycontainer -- ufw allow 22/tcp
lxc exec mycontainer -- ufw allow 80/tcp
```

---

## Netzwerk-Diagnose

```bash
# Netzwerk-Status aller Container
lxc list --columns=nf

# Container-Netzwerk-Details
lxc info mycontainer | grep -A 20 "Network usage"

# Ping zwischen Containern
lxc exec container-a -- ping container-b.lxd

# Netzwerk-Traffic mitschneiden
lxc exec mycontainer -- tcpdump -i eth0 -n

# Interface-Details im Container
lxc exec mycontainer -- ip addr
lxc exec mycontainer -- ip route

# Verbindung von Host zum Container testen
curl http://$(lxc list mycontainer --format=json | python3 -c "import sys,json;print(json.load(sys.stdin)[0]['state']['network']['eth0']['addresses'][0]['address'])")
```

---

## Typische Setup-Szenarien

### Homelab: Alle Container im selben Subnetz

```bash
lxc network create homelabbr \
    ipv4.address=192.168.10.1/24 \
    ipv4.nat=true \
    ipv6.address=none

lxc profile device set default eth0 network=homelabbr
```

### Prod: Container direkt im LAN erreichbar

```bash
# macvlan mit automatischer DHCP vom Router
lxc network create lannet \
    --type=macvlan \
    parent=eno1

lxc launch ubuntu:24.04 webserver
lxc config device add webserver eth0 nic network=lannet
# Container bekommt IP von Router-DHCP
```

### Isolation: Separates Backend-Netz

```bash
# Frontend-Netz (mit NAT)
lxc network create frontnet ipv4.address=10.1.0.1/24 ipv4.nat=true

# Backend-Netz (kein Internet-Zugang)
lxc network create backnet ipv4.address=10.2.0.1/24 ipv4.nat=false

# Webserver: beide Netzwerke
lxc config device add webserver eth0 nic network=frontnet
lxc config device add webserver eth1 nic network=backnet

# Datenbank: nur Backend-Netz
lxc config device add database eth0 nic network=backnet
```

---

## Verwandte Notizen

- [[Profile und Konfiguration]] — Netzwerk-Devices in Profilen
- [[LXD Installation und Setup]] — Netzwerk beim Setup
- [[LXD Cluster Betrieb]] — OVN im Cluster
- [[LXD Troubleshooting]] — Netzwerk-Diagnose
