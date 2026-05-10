---
tags: [podman, troubleshooting, debugging, fehler, rootless, selinux, netzwerk, storage, ports]
aliases: ["Podman Fehler", "Podman Debug", "Container Troubleshooting"]
---

# Podman Troubleshooting

> [!ABSTRACT]
> Systematische Fehlerdiagnose für häufige Podman-Probleme: Rootless-Setup (subuid/subgid), Netzwerk und Ports, SELinux/AppArmor, Storage und Image-Probleme, Systemd/Quadlet-Fehler und allgemeine Container-Debugging-Techniken.

---

## Allgemeine Debug-Strategie

```bash
# 1. Podman-Version und System-Info
podman version
podman info

# 2. Container-Status
podman ps -a
podman inspect container-name

# 3. Logs
podman logs container-name
podman logs --since 1h container-name

# 4. Events (Was ist wann passiert?)
podman events --since 1h

# 5. Debug-Modus
podman --log-level debug run myapp 2>&1 | head -100
```

---

## Rootless-Probleme

### subuid/subgid nicht konfiguriert

```
Error: OCI runtime error: crun: join namespaces: create: opening user namespace: can't open /proc/self/ns/user
```

```bash
# Prüfen ob Einträge existieren
cat /etc/subuid | grep $USER
cat /etc/subgid | grep $USER

# Fehlt → manuell hinzufügen
sudo usermod --add-subuids 100000-165535 $USER
sudo usermod --add-subgids 100000-165535 $USER

# Oder direkt in Dateien
echo "$USER:100000:65536" | sudo tee -a /etc/subuid
echo "$USER:100000:65536" | sudo tee -a /etc/subgid

# Migration durchführen (WICHTIG nach Änderungen!)
podman system migrate

# Prüfen
podman unshare cat /proc/self/uid_map
```

### cgroups v2 fehlt (rootless)

```
ERRO error rootless: error creating cgroup for UID ... cgroups v2 not enabled
```

```bash
# cgroups v2 prüfen
cat /sys/fs/cgroup/cgroup.controllers
# Zeigt: cpuset cpu io memory pids rdma misc  ← v2 aktiv
# Wenn leer oder Datei nicht vorhanden → v1

# cgroups v2 aktivieren (Ubuntu, kein systemd 242+)
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
sudo reboot
```

### Port unter 1024 (rootless)

```
Error: rootlessport cannot expose privileged port 80
```

```bash
# Methode 1: Anderen Port nehmen (empfohlen)
podman run -p 8080:80 nginx   # und dann mit Reverse Proxy weiterleiten

# Methode 2: Unprivileged Port-Range anpassen
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
# Dauerhaft:
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/50-podman-ports.conf
sudo sysctl -p /etc/sysctl.d/50-podman-ports.conf
```

### Container läuft nach Logout nicht mehr

```bash
# Problem: Rootless Container ohne lingering gestoppt
# Lösung:
sudo loginctl enable-linger $USER

# Prüfen
loginctl show-user $USER | grep Linger
# Linger=yes
```

---

## SELinux / AppArmor Probleme

### SELinux blockiert Volume-Zugriff

```
Error: container_linux.go:380: ... permission denied
# oder in Logs:
open /etc/nginx/conf.d/default.conf: permission denied
```

```bash
# AVC-Fehler anzeigen
sudo ausearch -m avc -ts recent

# Prüfen ob SELinux das Problem ist
sudo setenforce 0   # Temporär deaktivieren zum Testen
podman run -v /daten:/data nginx   # Wenn es jetzt geht → SELinux war es!
sudo setenforce 1   # Wieder aktivieren!

# Richtige Lösung: :z oder :Z am Volume
podman run -v /daten:/data:z nginx   # shared
podman run -v /daten:/data:Z nginx   # privat (nur dieser Container)

# Oder: SELinux-Kontext manuell setzen
sudo chcon -Rt container_file_t /daten/
sudo restorecon -Rv /daten/

# Kontext prüfen
ls -laZ /daten/
```

> [!WARNING] Nie dauerhaft deaktivieren
> `setenforce 0` ist nur für Debugging — nicht in Produktion!

---

## Netzwerk-Probleme

### Container kann Host nicht erreichen

```bash
# Host-IP im rootless Kontext
ip route | grep default
# default via 10.0.2.2   ← das ist der Host bei slirp4netns/pasta

# In neueren Podman-Versionen (pasta):
# host.containers.internal verweist auf den Host
podman run --add-host host.containers.internal:host-gateway myapp
```

### DNS-Auflösung schlägt fehl

```bash
# Im Container testen
podman exec webserver nslookup google.com
podman exec webserver cat /etc/resolv.conf

# Custom DNS setzen
podman run --dns 1.1.1.1 --dns 8.8.8.8 myapp

# Netzwerk mit DNS
podman network create --dns 1.1.1.1 my-netz
podman run --network my-netz myapp
```

### Container-zu-Container Kommunikation schlägt fehl

```bash
# Problem: Beide Container im selben Netzwerk?
podman network ls
podman inspect container1 | grep -A5 Networks
podman inspect container2 | grep -A5 Networks

# Netzwerk erstellen und Container verbinden
podman network create mynet
podman run -d --name db --network mynet postgres:16
podman run -d --name app --network mynet myapp

# Container im laufenden Betrieb verbinden
podman network connect mynet container-name

# DNS-Test zwischen Containern
podman exec app ping db
podman exec app nslookup db
```

### Port ist bereits belegt

```
Error: error while calling PortManager.AddPort(): listen tcp4 0.0.0.0:8080: bind: address already in use
```

```bash
# Wer hört auf Port 8080?
sudo ss -tlnp | grep 8080
sudo lsof -i :8080

# Anderen Port wählen
podman run -p 8090:80 nginx
```

---

## Image-Probleme

### Short-Name-Prompt nervt

```
? Please select an image:
  ▸ docker.io/library/nginx:latest
    quay.io/jitesoft/nginx:latest
    ...
```

```bash
# Immer vollständigen Image-Namen angeben:
podman pull docker.io/library/nginx:latest  # STATT: nginx:latest

# Oder Default-Registry konfigurieren:
# /etc/containers/registries.conf oder ~/.config/containers/registries.conf
```

```toml
# ~/.config/containers/registries.conf
unqualified-search-registries = ["docker.io"]
```

### Image kann nicht gepullt werden

```bash
# Netzwerk-Problem?
curl -I https://registry-1.docker.io/v2/

# Mit Debug
podman --log-level debug pull nginx:alpine 2>&1 | tail -20

# Registry-Verbindungstest
podman login docker.io

# Proxy konfigurieren (falls nötig)
export HTTPS_PROXY=http://proxy.firma.de:3128
export NO_PROXY=localhost,127.0.0.1,.firma.de
```

### Image-Layer korrumpiert

```
Error: writing blob: adding layer with blob ...: layer not known
```

```bash
# Storage-Fehler reparieren
podman system prune
podman system reset   # WARNUNG: Löscht ALLES!

# Einzelnes Image neu pullen
podman image rm nginx:alpine
podman pull nginx:alpine
```

---

## Storage-Probleme

### Kein Speicherplatz mehr (auf Image-Partition)

```bash
# Speichernutzung anzeigen
podman system df

# Aufräumen (trocken laufen lassen)
podman system prune --dry-run

# Aufräumen
podman image prune           # Ungetaggte Images
podman container prune       # Gestoppte Container
podman volume prune          # Unbenutzte Volumes
podman system prune          # Alles auf einmal

# Aggresiver: auch unbenutzte Images
podman system prune -a
```

### Container-Storage korrupt

```bash
# Storage-Verzeichnis prüfen
podman info | grep graphRoot
# ~/.local/share/containers/storage (rootless)

# Podman-Storage-Status
podman system check

# Reparaturversuch
podman system renumber

# Letzter Ausweg: Reset
podman system reset
# WARNUNG: Löscht alle Container, Images, Volumes!
```

---

## Systemd/Quadlet-Probleme

### Quadlet-Service startet nicht

```bash
# Schritt 1: Fehlermeldung lesen
systemctl --user status myapp.service
journalctl --user -u myapp.service -n 50

# Schritt 2: Quadlet-Generierung prüfen
/usr/lib/systemd/system-generators/podman-system-generator --user --dry-run

# Schritt 3: Generierte Unit-Datei prüfen
cat ~/.config/systemd/user/myapp.service

# Schritt 4: Daemon reload
systemctl --user daemon-reload
systemctl --user restart myapp.service
```

### podman.socket nicht verfügbar

```bash
# Socket aktivieren
systemctl --user enable --now podman.socket
systemctl --user status podman.socket

# Socket-Pfad prüfen
ls -la $XDG_RUNTIME_DIR/podman/podman.sock

# Falls $XDG_RUNTIME_DIR leer:
export XDG_RUNTIME_DIR=/run/user/$(id -u)
```

---

## Container-Debugging

### Im Container debuggen

```bash
# Shell in laufendem Container
podman exec -it webserver sh
podman exec -it webserver bash

# Als Root debuggen (auch wenn Container als user läuft)
podman exec -u root -it webserver sh

# Prozesse im Container
podman exec webserver ps aux
podman top webserver

# Netzwerk im Container
podman exec webserver ip addr
podman exec webserver ss -tlnp
podman exec webserver curl http://localhost:8080
```

### Container inspizieren

```bash
# Vollständige Container-Config
podman inspect webserver
podman inspect webserver | jq .

# Nur bestimmte Felder
podman inspect --format "{{.State.Status}}" webserver
podman inspect --format "{{.NetworkSettings.IPAddress}}" webserver
podman inspect --format "{{range .Mounts}}{{.Source}} → {{.Destination}}\n{{end}}" webserver
podman inspect --format "{{range .Config.Env}}{{.}}\n{{end}}" webserver

# Unterschied zwischen zwei Zuständen
podman diff webserver   # Zeigt geänderte Dateien vs. Image
```

---

## Podman-Installation reparieren

```bash
# Podman-Daten komplett zurücksetzen
podman system reset

# Konfiguration zurücksetzen
rm -rf ~/.config/containers/
# Systemweite Config:
sudo rm -f /etc/containers/containers.conf

# Speicher-Backend wechseln (overlay → vfs für Debugging)
# In ~/.config/containers/storage.conf:
```

```toml
[storage]
driver = "vfs"   # Langsamer aber ohne Kernel-Features
```

---

## Häufige Fehlermeldungen — Schnellreferenz

| Fehlermeldung | Ursache | Lösung |
|---------------|---------|--------|
| `permission denied` auf Volume | SELinux | `:z` oder `:Z` an Volume |
| `address already in use` | Port belegt | anderen Port wählen |
| `rootlessport cannot expose port 80` | Privileged Port | Port > 1024 oder sysctl anpassen |
| `can't open /proc/self/ns/user` | subuid fehlt | `usermod --add-subuids` + migrate |
| `cgroups v2 not enabled` | altes cgroups | grub-Eintrag + reboot |
| `layer not known` | Storage corrupt | `system prune` + neu pullen |
| `Name or service not known` | DNS | `--dns` Flag oder Netzwerk-DNS |
| `Linger` nicht aktiv | Container stirbt bei Logout | `loginctl enable-linger` |

---

## Verwandte Notizen

- [[Rootless Container]] — subuid/subgid Details
- [[Podman Sicherheit]] — SELinux-Konfiguration
- [[Podman Networking]] — Netzwerk-Konfiguration
- [[Systemd Integration]] — Quadlet-Debugging
