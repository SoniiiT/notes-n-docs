---
tags: [lxd, troubleshooting, fehler, diagnose, debugging]
aliases: ["LXD Fehlersuche", "LXD Probleme"]
---

# LXD Troubleshooting

> [!ABSTRACT]
> Systematische Fehlerdiagnose für LXD: Container-Startprobleme, Netzwerk-Fehler, Storage-Probleme, Cluster-Ausfälle und Performance-Engpässe. Mit Diagnose-Befehlen und konkreten Lösungen.

---

## Grundlegende Diagnose-Befehle

```bash
# ── Systemstatus ─────────────────────────────────────────

# LXD-Daemon-Status
systemctl status snap.lxd.daemon
journalctl -u snap.lxd.daemon -n 100 --no-pager

# LXD-Version und API-Erreichbarkeit
lxd --version
lxc info

# Alle Container und Status
lxc list -c nsLf4

# Container-Details
lxc info mycontainer

# Container-Log (letzte Start/Stop-Events)
lxc info --show-log mycontainer

# ── Debug-Modus ──────────────────────────────────────────

# Befehl mit Debug-Output
lxc --debug list
lxc --debug launch ubuntu:24.04 test

# Daemon im Debug-Modus (temporär)
sudo snap stop lxd
sudo lxd --debug &
```

---

## Container startet nicht

### Symptom: Container bleibt in "STOPPED"

```bash
# 1. Log prüfen
lxc info --show-log mycontainer

# 2. Daemon-Log nach Fehlern durchsuchen
journalctl -u snap.lxd.daemon | grep -E "ERROR|WARN|mycontainer"

# 3. Container manuell mit Debug starten
lxc start mycontainer
sleep 2
lxc info mycontainer
```

### Häufige Ursachen und Lösungen

**Storage-Volume nicht vorhanden:**
```bash
# Error: "Failed to mount..."
# Storage Pool prüfen
lxc storage info default
lxc storage volume list default

# Volume neu erstellen falls beschädigt
lxc storage volume delete default container/mycontainer
# Container-Daten gehen verloren — vorher exportieren!
```

**UID-Mapping Konflikte:**
```bash
# Error: "Failed to set up id mapping"
# Prüfen ob subuid/subgid korrekt konfiguriert
cat /etc/subuid | grep root
cat /etc/subgid | grep root
# Muss enthalten: root:1000000:1000000000

# Falls fehlend
echo "root:1000000:1000000000" | sudo tee -a /etc/subuid
echo "root:1000000:1000000000" | sudo tee -a /etc/subgid
sudo systemctl restart snap.lxd.daemon
```

**AppArmor-Fehler:**
```bash
# Error: "AppArmor profile ... could not be loaded"
# AppArmor-Status prüfen
sudo aa-status

# AppArmor neu laden
sudo systemctl reload apparmor
sudo snap restart lxd
```

---

## Netzwerk-Probleme

### Container hat keine IP-Adresse

```bash
# 1. Bridge-Status prüfen
lxc network show lxdbr0
lxc network list-leases lxdbr0

# 2. DHCP läuft?
sudo systemctl status systemd-networkd
# oder
lxc exec mycontainer -- systemctl status systemd-networkd

# 3. Interface im Container
lxc exec mycontainer -- ip addr
lxc exec mycontainer -- ip link

# 4. DHCP manuell im Container anfordern
lxc exec mycontainer -- dhclient eth0

# 5. dnsmasq-Log (LXD nutzt dnsmasq für DHCP)
journalctl -u snap.lxd.daemon | grep dnsmasq
```

**Lösung: Bridge-Konfiguration zurücksetzen:**
```bash
lxc network set lxdbr0 ipv4.address 10.10.10.1/24
lxc network set lxdbr0 ipv4.dhcp true
lxc network set lxdbr0 ipv4.nat true
sudo snap restart lxd
```

### Container kann nicht nach außen kommunizieren

```bash
# 1. NAT-Regel prüfen
sudo iptables -t nat -L POSTROUTING -n | grep lxdbr0

# 2. IP-Forwarding aktiv?
cat /proc/sys/net/ipv4/ip_forward
# Muss: 1

# 3. Falls 0:
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/lxd.conf
sudo sysctl -p /etc/sysctl.d/lxd.conf

# 4. Firewall-Regeln prüfen (ufw)
sudo ufw status
# Falls ufw aktiv ist und LXD-Traffic blockiert:
sudo ufw allow in on lxdbr0
sudo ufw route allow in on lxdbr0
sudo ufw route allow out on lxdbr0
```

### DNS-Auflösung funktioniert nicht

```bash
# Im Container testen
lxc exec mycontainer -- nslookup google.com
lxc exec mycontainer -- cat /etc/resolv.conf

# dnsmasq-Konfiguration prüfen
lxc network show lxdbr0 | grep dns

# DNS-Server für Bridge setzen
lxc network set lxdbr0 dns.nameservers=1.1.1.1,8.8.8.8

# dnsmasq neu starten
sudo snap restart lxd
```

---

## Storage-Probleme

### Storage Pool voll

```bash
# Pool-Belegung prüfen
lxc storage info default

# Größte Volumes finden
lxc storage volume list default
lxc storage volume info default container/mycontainer

# ZFS: Detaillierte Belegung
sudo zfs list -r mypool -s used

# Alte Snapshots aufräumen
lxc info mycontainer | grep -A 20 Snapshots
lxc delete mycontainer/old-snap

# Image-Cache leeren (gibt oft GB frei)
lxc image list
lxc image delete <fingerprint>
```

### ZFS-Probleme

```bash
# ZFS Pool-Status
sudo zpool status

# ZFS-Fehler (I/O errors, checksum errors)
sudo zpool status -v

# ZFS Pool reparieren (scrub)
sudo zpool scrub mypool

# ZFS Dataset reparieren
sudo zfs repair mypool

# LXD-ZFS-Pool zurücksetzen (VORSICHT: Datenverlust!)
sudo zpool destroy lxdpool
lxc storage delete default
lxc storage create default zfs size=50GiB
```

### Volume kann nicht gemountet werden

```bash
# Error: "Failed to mount ... no such file or directory"
# 1. Volume-Existenz prüfen
lxc storage volume list default | grep mycontainer

# 2. ZFS/btrfs Dataset prüfen
sudo zfs list | grep mycontainer
# oder
sudo btrfs subvolume list /var/snap/lxd/common/lxd/storage-pools/default

# 3. Manuell aushängen und neu mounten
lxc stop mycontainer --force
lxc start mycontainer
```

---

## Performance-Probleme

### Container ist langsam

```bash
# Ressourcenverbrauch prüfen
lxc top
lxc info mycontainer | grep -A 10 Resources

# CPU-Stealer (andere Container stehlen CPU-Zeit)
lxc exec mycontainer -- cat /proc/stat

# I/O-Wait
lxc exec mycontainer -- iostat -x 1

# Memory-Druck
lxc exec mycontainer -- free -h
lxc exec mycontainer -- cat /proc/meminfo | grep -E "MemAvailable|SwapUsed"

# Limits erhöhen
lxc config set mycontainer limits.cpu 4
lxc config set mycontainer limits.memory 8GiB
```

### Langsame Snapshots / Klone

```bash
# Mit dir-Backend: normal (vollständige Kopie)
# Mit ZFS/btrfs: sollte schnell sein (<1s)

# Backend prüfen
lxc storage show default | grep driver

# Falls dir-Backend → Wechsel zu ZFS empfohlen
# Migration:
lxc stop mycontainer
lxc copy mycontainer --storage zfs-pool mycontainer-new
lxc delete mycontainer
lxc move mycontainer-new mycontainer
```

---

## Cluster-Probleme

### Node zeigt "OFFLINE"

```bash
# Cluster-Status
lxc cluster list

# Netzwerk-Konnektivität zum Node
ping 192.168.1.11
curl -k https://192.168.1.11:8443

# Daemon auf dem Node neustarten
ssh node2 "sudo snap restart lxd"

# Node-Log auf dem ausgefallenen Node
ssh node2 "journalctl -u snap.lxd.daemon -n 50 --no-pager"
```

### Quorum verloren

```bash
# Nur 1 von 3 Nodes erreichbar
# 1. Auf dem verbleibenden Node:
sudo lxd cluster recover-from-quorum-loss

# 2. Danach Cluster neu starten
sudo snap restart lxd

# 3. Cluster-Status prüfen
lxc cluster list
```

### Container kann nicht migriert werden

```bash
# Error: "Migration failed"
# 1. Shared Storage?
lxc storage list  # Storage pool muss cluster-weit sein (Ceph)

# 2. Zielsystem hat genug Ressourcen?
lxc info --resources
ssh node2 "lxc info --resources"

# 3. Verbindung zwischen Nodes?
lxc query /1.0/cluster/members

# 4. Mit --stateless migrieren (kein Live-Migration)
lxc stop mycontainer
lxc move mycontainer --target node2
```

---

## Daemon-Probleme

### LXD-Daemon startet nicht

```bash
# Status und Fehlerlog
systemctl status snap.lxd.daemon
journalctl -u snap.lxd.daemon --since "10 min ago"

# Häufige Ursachen:
# 1. Port 8443 belegt
sudo ss -tlnp | grep 8443

# 2. Datenbank beschädigt
ls -la /var/snap/lxd/common/lxd/database/

# 3. Snap-Konflikte
snap list | grep lxd

# Reset versuchen (VORSICHT: Konfiguration geht verloren)
sudo snap stop lxd
sudo lxd init --auto
sudo snap start lxd
```

### API nicht erreichbar

```bash
# Unix-Socket testen
sudo -u root lxc info

# Socket-Pfad
ls -la /var/snap/lxd/common/lxd/unix.socket

# Benutzer in lxd-Gruppe?
groups $USER
# Falls nicht:
sudo usermod -aG lxd $USER
newgrp lxd
```

---

## Schnell-Diagnose Checkliste

```
Container-Problem:
  [ ] lxc info --show-log <container>
  [ ] journalctl -u snap.lxd.daemon | grep <container>
  [ ] lxc config show <container> --expanded

Netzwerk-Problem:
  [ ] lxc exec <container> -- ip addr
  [ ] lxc network list-leases lxdbr0
  [ ] sudo iptables -t nat -L POSTROUTING -n

Storage-Problem:
  [ ] lxc storage info default
  [ ] sudo zpool status (bei ZFS)
  [ ] lxc storage volume list default

Cluster-Problem:
  [ ] lxc cluster list
  [ ] ping zwischen Nodes
  [ ] journalctl auf ausgefallenen Nodes
```

---

## Nützliche Diagnose-Einzeiler

```bash
# Alle Container mit ihrem Storage-Pool
lxc list --format=json | python3 -c "
import sys,json
data = json.load(sys.stdin)
for c in data:
    print(c['name'], c.get('expanded_devices', {}).get('root', {}).get('pool','?'))
"

# Container mit meisten CPU-Ressourcen
lxc top --sort cpu

# Alle Volumes und ihre Größen
lxc storage volume list default --format=table

# LXD-Fehler der letzten Stunde
journalctl -u snap.lxd.daemon --since "1h ago" | grep -E "ERROR|CRIT|WARN"
```

---

## Verwandte Notizen

- [[LXD CLI Cheatsheet]] — Diagnose-Befehle
- [[LXD Cluster Betrieb]] — Cluster-spezifische Probleme
- [[Storage Pools]] — Storage-Diagnose
- [[LXD Netzwerk]] — Netzwerk-Diagnose
- [[LXD Sicherheit]] — Sicherheitsfehler
