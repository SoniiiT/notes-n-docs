---
tags: [podman, networking, netavark, pasta, bridge, dns, port-forwarding, rootless, aardvark]
aliases: ["Podman Netzwerk", "Podman Network", "Netavark", "Pasta Networking"]
---

# Podman Networking

> [!ABSTRACT]
> Podman nutzt zwei Netzwerk-Stacks: **Netavark** (Rust-basiert) für rootful Container mit Aardvark-DNS für automatische Namensauflösung, und **Pasta** (seit Podman 5.0) für rootless Container mit verbesserter Performance gegenüber dem Vorgänger slirp4netns. Beide Stacks sind CNI-frei und OCI-konform.

---

## Zwei Netzwerk-Stacks: Rootful vs. Rootless

```
Rootful Container   → Netavark + Aardvark-DNS  (Kernel-Networking, volle Features)
Rootless Container  → Pasta                     (User-Space-Networking, sichere Isolation)
```

Warum zwei Stacks? Rootless Container dürfen aus Sicherheitsgründen keine Kernel-Netzwerk-Interfaces direkt erstellen. Pasta implementiert das Netzwerk vollständig im User-Space.

---

## Netavark — Rootful Networking

**Netavark** ist seit Podman 4.0 der Netzwerk-Backend für rootful Container (Ersatz für CNI/cni-plugins).

- In **Rust** geschrieben — memory-safe, performant
- Erstellt Linux-Bridge-Interfaces im Kernel
- Standard-Netzwerk: `podman` (Bridge `10.88.0.0/16`)
- Arbeitet mit **Aardvark-DNS** für Container-Namensauflösung zusammen

```
Host
├── eth0 (physisch)
└── podman bridge (10.88.0.0/16)
    ├── Container A (10.88.0.2)
    └── Container B (10.88.0.3)  ← erreicht A via DNS-Name
```

---

## Pasta — Rootless Networking

**Pasta** (basiert auf `passt.one`) ist seit Podman 5.0 der Standard-Netzwerk-Stack für rootless Container.

**Vorteile gegenüber slirp4netns (Vorgänger):**

| Eigenschaft | Pasta (Podman 5+) | slirp4netns (Podman 4) |
|-------------|-------------------|------------------------|
| **Durchsatz** | Höher (näher an nativer Geschwindigkeit) | Niedriger |
| **Latenz** | Geringer | Höher |
| **IPv6** | ✅ Out-of-the-box | ⚠️ Eingeschränkt |
| **VPN-Kompatibilität** | ✅ Gut | ⚠️ Problematisch |
| **Implementierung** | User-Space, separater Prozess | User-Space, separater Prozess |

---

## Netzwerke verwalten

```bash
# Vorhandene Netzwerke anzeigen
podman network ls
# NAME      ID            DRIVER    OPTIONS
# podman    2f259bab93aa  bridge    (Standard-Netzwerk)

# Neues Netzwerk erstellen
podman network create mein-netz

# Mit spezifischem Subnetz
podman network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  mein-netz

# Mit IPv6
podman network create \
  --ipv6 \
  --subnet fd00::/64 \
  mein-netz-v6

# Netzwerk inspizieren
podman network inspect mein-netz

# Netzwerk löschen
podman network rm mein-netz

# Ungenutzte Netzwerke löschen
podman network prune
```

---

## Container mit Netzwerken verbinden

```bash
# Beim Start
podman run -d --network mein-netz --name web nginx:alpine

# Nachträglich verbinden
podman network connect mein-netz container-name

# Trennen
podman network disconnect mein-netz container-name

# Mehrere Netzwerke beim Start
podman run -d \
  --network mein-netz \
  --network anderes-netz \
  --name multi-net-container nginx
```

---

## DNS zwischen Containern

**Aardvark-DNS** ermöglicht automatische Container-Namensauflösung innerhalb desselben Netzwerks:

```bash
# Netzwerk erstellen
podman network create app-netz

# Container mit Namen starten
podman run -d --network app-netz --name datenbank postgres:16
podman run -d --network app-netz --name backend myapp:latest

# Backend kann Datenbank via DNS-Name erreichen:
# hostname: datenbank
# Kein manuelles /etc/hosts nötig!
```

```bash
# DNS-Auflösung testen
podman exec backend ping -c3 datenbank
podman exec backend nslookup datenbank
```

> [!NOTE] Standard-Netzwerk hat kein DNS
> Das eingebaute `podman`-Standardnetzwerk hat **kein DNS**. Für Container-zu-Container-Kommunikation via Namen immer ein benanntes Netzwerk erstellen.

---

## Port-Forwarding

```bash
# Einzelner Port
podman run -p 8080:80 nginx

# Mehrere Ports
podman run -p 8080:80 -p 8443:443 nginx

# Nur auf localhost binden (sicherer)
podman run -p 127.0.0.1:8080:80 nginx

# UDP
podman run -p 5353:5353/udp dns-server

# Beliebiger Host-Port (OS wählt)
podman run -p 80 nginx   # OS wählt freien Port
podman port container-name   # zugewiesenen Port anzeigen

# Alle Port-Mappings anzeigen
podman port container-name
```

> [!WARNING] Rootless und Ports < 1024
> Rootless Container können keine Ports < 1024 binden. Lösungen:
> ```bash
> # Option 1: Höheren Host-Port mappen
> podman run -p 8080:80 nginx  # Host:8080 → Container:80
>
> # Option 2: Kernel-Parameter anpassen
> sudo sysctl net.ipv4.ip_unprivileged_port_start=80
> # Dann: podman run -p 80:80 nginx
> ```

---

## Macvlan-Netzwerk

Container bekommt eine eigene MAC-Adresse und IP-Adresse im physischen Netzwerk — verhält sich wie ein physisches Gerät im LAN.

```bash
# Macvlan-Netzwerk erstellen
podman network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --ip-range 192.168.1.200/29 \
  -o parent=eth0 \
  macvlan-netz

# Container starten
podman run -d \
  --network macvlan-netz \
  --name pihole \
  --ip 192.168.1.200 \
  pihole/pihole:latest
```

> [!WARNING] Macvlan-Einschränkung
> Der Host kann **nicht** direkt mit Macvlan-Containern kommunizieren (Kernel-Einschränkung). Für Host-zu-Container-Kommunikation ein zusätzliches Bridge-Interface (`macvlan shim`) einrichten oder einen normalen Bridge-Container verwenden.

---

## Host-Networking

```bash
# Container nutzt Host-Netzwerk direkt (kein Namespace)
podman run --network host nginx

# Container lauscht auf Port 80 des Hosts ohne Port-Mapping
```

> [!WARNING] Sicherheitsrisiko
> `--network host` deaktiviert die Netzwerk-Isolation vollständig. Container hat direkten Zugriff auf alle Host-Interfaces. Nur verwenden wenn absolut nötig (z.B. Monitoring-Tools die Host-Netzwerk-Daten benötigen).

---

## Netzwerk für Pods

Port-Mappings bei Pods immer am Pod definieren, nicht an einzelnen Containern:

```bash
# Netzwerk erstellen
podman network create webapp-netz

# Pod mit Netzwerk und Ports
podman pod create \
  --name webapp \
  --network webapp-netz \
  -p 8080:80 \
  -p 8443:443

# Container im Pod (kein eigenes Port-Mapping!)
podman run -d --pod webapp --name frontend nginx:alpine
podman run -d --pod webapp --name api myapi:latest
```

---

## Firewall-Integration (firewalld)

Auf RHEL/Fedora integriert sich Netavark automatisch mit firewalld:

```bash
# Podman-Zonen anzeigen
firewall-cmd --get-zones | tr ' ' '\n' | grep -i podman

# Podman-spezifische Firewall-Regeln
firewall-cmd --list-all --zone=netavark

# Manuell Port öffnen (falls Podman-Regeln fehlen)
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

---

## Netzwerk-Diagnose

```bash
# Container-IP anzeigen
podman inspect --format "{{.NetworkSettings.IPAddress}}" container-name

# Alle Container-IPs
podman inspect --format "{{.Name}}: {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}" \
  $(podman ps -q)

# Netzwerk-Statistiken
podman stats --no-stream container-name

# Netzwerk im Container debuggen
podman exec container-name ip addr
podman exec container-name ip route
podman exec container-name curl -v http://anderer-container/

# DNS-Test
podman exec container-name nslookup datenbank
```

---

## Verwandte Notizen

- [[Rootless Container]] — Rootless-Einschränkungen im Netzwerk
- [[Pods]] — Netzwerk-Sharing in Pods
- [[Podman Sicherheit]] — Netzwerk-Isolation und Firewall
- [[Podman CLI Cheatsheet]] — Netzwerk-Befehle Referenz
