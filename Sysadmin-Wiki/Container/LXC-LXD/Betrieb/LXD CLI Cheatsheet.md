---
tags: [lxd, cli, cheatsheet, befehle, lxc]
aliases: ["lxc Cheatsheet", "LXD Befehle"]
---

# LXD CLI Cheatsheet

> [!ABSTRACT]
> Vollständige Referenz aller wichtigen `lxc`-Befehle, kategorisiert nach Anwendungsfall. Der `lxc`-Client kommuniziert mit dem LXD-Daemon über die REST-API.

---

## Container-Lifecycle

```bash
# ── Erstellen und Starten ──────────────────────────────────

# Container erstellen und starten
lxc launch ubuntu:24.04 mycontainer

# Container erstellen (nicht starten)
lxc init ubuntu:24.04 mycontainer

# VM erstellen und starten
lxc launch ubuntu:24.04 myvm --vm

# Mit Profil starten
lxc launch ubuntu:24.04 mycontainer --profile webserver

# Mit sofortiger Konfiguration
lxc launch ubuntu:24.04 mycontainer \
    -c limits.cpu=2 \
    -c limits.memory=2GiB

# ── Status ────────────────────────────────────────────────

# Alle Container anzeigen
lxc list

# Nur laufende Container
lxc list --filter status=running

# Bestimmte Spalten anzeigen
lxc list --columns=nstL4

# Als JSON
lxc list --format=json

# Container-Details
lxc info mycontainer

# ── Start / Stop / Restart ───────────────────────────────

lxc start mycontainer
lxc stop mycontainer
lxc stop mycontainer --force   # Sofort (wie kill)
lxc restart mycontainer
lxc pause mycontainer          # Einfrieren (SIGSTOP)
lxc resume mycontainer         # Fortsetzen

# Alle Container stoppen
lxc stop --all

# ── Löschen ───────────────────────────────────────────────

lxc delete mycontainer                    # (Container muss gestoppt sein)
lxc delete mycontainer --force            # Auch laufende Container
lxc delete mycontainer/snap0              # Snapshot löschen
```

---

## Shell und Befehle ausführen

```bash
# Shell im Container öffnen
lxc exec mycontainer -- bash
lxc exec mycontainer -- /bin/sh

# Einzel-Befehl ausführen
lxc exec mycontainer -- apt update
lxc exec mycontainer -- systemctl status nginx

# Als bestimmten Benutzer
lxc exec mycontainer --user 1000 -- bash
lxc exec mycontainer --user www-data -- bash

# Mit Umgebungsvariable
lxc exec mycontainer --env DEBIAN_FRONTEND=noninteractive -- bash

# Interaktiver Modus erzwingen
lxc exec mycontainer -t -- bash

# Nicht-interaktiv (für Scripts)
lxc exec mycontainer -n -- apt upgrade -y
```

---

## Dateien übertragen

```bash
# Datei in Container kopieren
lxc file push localfile.txt mycontainer/tmp/

# Verzeichnis rekursiv
lxc file push -r ./configs/ mycontainer/etc/myapp/

# Datei aus Container kopieren
lxc file pull mycontainer/etc/nginx/nginx.conf ./

# Verzeichnis aus Container
lxc file pull -r mycontainer/var/log/ ./container-logs/

# Datei direkt bearbeiten
lxc file edit mycontainer/etc/nginx/nginx.conf
```

---

## Konfiguration

```bash
# ── Anzeigen ─────────────────────────────────────────────

lxc config show mycontainer              # Container-Konfiguration
lxc config show mycontainer --expanded   # inkl. Profil-Werte

# ── Setzen ───────────────────────────────────────────────

lxc config set mycontainer limits.cpu 2
lxc config set mycontainer limits.memory 4GiB
lxc config set mycontainer boot.autostart true

# ── Lesen ────────────────────────────────────────────────

lxc config get mycontainer limits.cpu

# ── Zurücksetzen ─────────────────────────────────────────

lxc config unset mycontainer limits.cpu

# ── Bearbeiten (Editor) ──────────────────────────────────

lxc config edit mycontainer

# ── Devices ─────────────────────────────────────────────

lxc config device add mycontainer mydata disk source=/mnt/data path=/data
lxc config device list mycontainer
lxc config device show mycontainer mydata
lxc config device remove mycontainer mydata
```

---

## Snapshots

```bash
# Snapshot erstellen
lxc snapshot mycontainer snap0
lxc snapshot mycontainer snap-$(date +%Y%m%d)

# Snapshots anzeigen
lxc info mycontainer | grep -A 20 Snapshots:
lxc list --columns=Ns

# Snapshot wiederherstellen (Container wird gestoppt!)
lxc restore mycontainer snap0

# Snapshot zu neuem Container
lxc copy mycontainer/snap0 restored-container

# Snapshot löschen
lxc delete mycontainer/snap0
```

---

## Images

```bash
# ── Suchen ───────────────────────────────────────────────

lxc image list ubuntu:
lxc image list ubuntu: 24.04
lxc image list images: debian
lxc image alias list ubuntu: | grep 24.04

# ── Herunterladen ────────────────────────────────────────

lxc image copy ubuntu:24.04 local: --alias my-ubuntu

# ── Lokal verwalten ──────────────────────────────────────

lxc image list                           # Lokal gecachte Images
lxc image info local:my-ubuntu           # Image-Details
lxc image delete local:my-ubuntu         # Image löschen
lxc image refresh                        # Alle Images aktualisieren

# ── Exportieren / Importieren ────────────────────────────

lxc image export ubuntu:24.04 ./export-dir/
lxc image import ./export-dir/*.tar.xz --alias imported-image

# ── Container zu Image ───────────────────────────────────

lxc stop mycontainer
lxc publish mycontainer --alias myapp-image description="My App"
```

---

## Profile

```bash
# Profile anzeigen
lxc profile list
lxc profile show default

# Profil erstellen
lxc profile create webserver

# Profil bearbeiten
lxc profile edit webserver

# Profil kopieren
lxc profile copy default mein-profil

# Gerät zum Profil hinzufügen
lxc profile device add default root disk path=/ pool=default

# Container-Profile verwalten
lxc profile assign mycontainer default,webserver
lxc profile add mycontainer webserver
lxc profile remove mycontainer webserver

# Profil löschen
lxc profile delete webserver
```

---

## Netzwerk

```bash
# ── Anzeigen ─────────────────────────────────────────────

lxc network list
lxc network show lxdbr0
lxc network info lxdbr0

# ── Erstellen ────────────────────────────────────────────

lxc network create meinbr0 ipv4.address=10.0.0.1/24 ipv4.nat=true

# ── Konfigurieren ────────────────────────────────────────

lxc network set lxdbr0 ipv4.address 10.10.0.1/24
lxc network edit lxdbr0

# ── DHCP-Lease anzeigen ──────────────────────────────────

lxc network list-leases lxdbr0

# ── Löschen ──────────────────────────────────────────────

lxc network delete meinbr0
```

---

## Storage

```bash
# ── Pools ────────────────────────────────────────────────

lxc storage list
lxc storage show default
lxc storage info default
lxc storage create mypool zfs source=/dev/sdb
lxc storage delete mypool

# ── Volumes ──────────────────────────────────────────────

lxc storage volume list default
lxc storage volume create default mydata
lxc storage volume delete default mydata
lxc storage volume copy default/mydata default/mydata-backup
lxc storage volume snapshot default/mydata snap0

# ── Volume exportieren ───────────────────────────────────

lxc storage volume export default mydata ./mydata-backup.tar.gz
lxc storage volume import ./mydata-backup.tar.gz default mydata-restored
```

---

## Kopieren und Migrieren

```bash
# Container kopieren (lokal)
lxc copy mycontainer mycontainer-clone

# Container auf Remote kopieren
lxc copy mycontainer remote-server:

# Container auf Remote mit neuem Namen
lxc copy mycontainer remote-server:new-name

# Container migrieren (Original wird gelöscht)
lxc move mycontainer remote-server:

# In anderen Storage-Pool verschieben
lxc move mycontainer mycontainer --storage other-pool
```

---

## Remote-Verwaltung

```bash
# Remotes anzeigen
lxc remote list

# Remote hinzufügen
lxc remote add myserver https://192.168.1.100:8443

# Standard-Remote wechseln
lxc remote switch myserver

# Remote entfernen
lxc remote remove myserver

# Auf Remote-Container zugreifen
lxc list myserver:
lxc exec myserver:mycontainer -- bash
```

---

## Monitoring und Logs

```bash
# ── Ressourcenverbrauch ──────────────────────────────────

lxc info mycontainer                     # CPU, RAM, Netzwerk-Stats
lxc top                                  # Live-Übersicht (wie top)

# ── Logs ─────────────────────────────────────────────────

lxc info --show-log mycontainer          # Container-Log (Start/Stop)

# LXD-Daemon Log
journalctl -u snap.lxd.daemon -f
journalctl -u snap.lxd.daemon --since "1 hour ago"

# Container-Systemlog
lxc exec mycontainer -- journalctl -f
lxc exec mycontainer -- journalctl -u nginx

# ── Events ───────────────────────────────────────────────

lxc monitor                              # Alle Events live
lxc monitor --type=lifecycle             # Nur Lifecycle-Events
```

---

## Cluster-Befehle

```bash
# Cluster-Status
lxc cluster list
lxc cluster show node1

# Container auf bestimmtem Node starten
lxc launch ubuntu:24.04 mycontainer --target node2

# Container zwischen Nodes migrieren
lxc move mycontainer --target node3

# Node evakuieren (vor Wartung)
lxc cluster evacuate node2
lxc cluster restore node2

# Node entfernen
lxc cluster remove node3
```

---

## Nützliche Einzeiler

```bash
# Alle Container-IPs anzeigen
lxc list --format=csv -c n,4 | grep -v "^,"

# Container erstellen, Befehl ausführen, löschen
lxc launch ubuntu:24.04 temp-ct && \
    lxc exec temp-ct -- bash -c "apt update && apt install -y python3" && \
    lxc stop temp-ct && lxc delete temp-ct

# Alle gestoppten Container löschen
lxc list --format=csv -c n,s | grep STOPPED | cut -d, -f1 | xargs lxc delete

# Snapshot aller laufenden Container
for ct in $(lxc list --format=csv -c n,s | grep RUNNING | cut -d, -f1); do
    lxc snapshot $ct "auto-$(date +%Y%m%d)"
done

# Container-Konfiguration sichern
lxc list --format=csv -c n | while read ct; do
    lxc config show $ct --expanded > backup-$ct.yaml
done
```

---

## Verwandte Notizen

- [[LXD Installation und Setup]] — Erste Schritte
- [[Profile und Konfiguration]] — Konfigurationsdetails
- [[LXD Netzwerk]] — Netzwerk-Befehle
- [[Storage Pools]] — Storage-Befehle
- [[LXD Troubleshooting]] — Diagnose-Befehle
