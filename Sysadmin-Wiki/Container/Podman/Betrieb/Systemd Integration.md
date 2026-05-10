---
tags: [podman, systemd, quadlet, service, autostart, journald, socket-activation, auto-update]
aliases: ["Podman Systemd", "Quadlet", "Podman Service", "podman generate systemd"]
---

# Systemd Integration

> [!ABSTRACT]
> Podman bietet First-Class systemd-Integration via **Quadlet** (empfohlen ab Podman 4.4) — deklarative `.container`/`.pod`/`.network`/`.volume`-Dateien die von systemd als Units verwaltet werden. `podman generate systemd` ist der ältere Weg und in Podman 5.x deprecated. Container loggen automatisch ins systemd-Journal.

---

## Zwei Wege: Quadlet vs. generate systemd

| Methode | Podman-Version | Status | Empfehlung |
|---------|---------------|--------|-----------|
| **Quadlet** | 4.4+ | Aktiv | ✅ Für alle Neuinstallationen |
| `podman generate systemd` | 3.x–4.x | Deprecated in 5.x | ⚠️ Nur für Legacy-Systeme |

---

## Quadlet — Der moderne Weg

Quadlet ist ein systemd-Generator. Er liest `.container`-Dateien aus definierten Verzeichnissen und erstellt daraus automatisch systemd-Units.

### Speicherorte

```
# Rootless (User)
~/.config/containers/systemd/        ← bevorzugt
~/.config/systemd/user/              ← alternativ

# Rootful (System)
/etc/containers/systemd/             ← bevorzugt
/usr/share/containers/systemd/       ← für Pakete
```

### Grundstruktur einer .container-Datei

```ini
# ~/.config/containers/systemd/myapp.container

[Unit]
Description=Meine Anwendung
After=network-online.target
Wants=network-online.target

[Container]
# --- Pflichtfelder ---
Image=docker.io/library/nginx:alpine

# --- Port-Mapping ---
PublishPort=8080:80
PublishPort=8443:443

# --- Volumes ---
Volume=%h/config:/etc/nginx/conf.d:ro,z
Volume=myapp-data.volume:/data:Z

# --- Netzwerk ---
Network=myapp.network

# --- Umgebung ---
Environment=APP_ENV=production
Environment=LOG_LEVEL=info
EnvironmentFile=%h/.config/myapp/env

# --- Labels ---
Label=app=myapp
Label=version=1.0

# --- Auto-Update ---
AutoUpdate=registry

# --- Weitere Optionen ---
Hostname=myapp
User=1000:1000
ReadOnly=true
Tmpfs=/tmp:size=64m

[Service]
Restart=always
RestartSec=5s
TimeoutStartSec=60

[Install]
WantedBy=default.target
```

### Quadlet aktivieren

```bash
# Daemon neu laden (nötig nach jeder Dateiänderung)
systemctl --user daemon-reload

# Service aktivieren und starten
systemctl --user enable --now myapp.service

# Nur starten (ohne Enable = kein Autostart)
systemctl --user start myapp.service

# Status
systemctl --user status myapp.service

# Logs
journalctl --user -u myapp.service -f
journalctl --user -u myapp.service -n 50 --no-pager
journalctl --user -u myapp.service --since "1 hour ago"
```

---

## Quadlet: .network-Datei

```ini
# ~/.config/containers/systemd/myapp.network
[Unit]
Description=Netzwerk für myapp

[Network]
Subnet=172.21.0.0/16
Gateway=172.21.0.1
Label=app=myapp
DNS=1.1.1.1,8.8.8.8

[Install]
WantedBy=default.target
```

---

## Quadlet: .volume-Datei

```ini
# ~/.config/containers/systemd/myapp-data.volume
[Unit]
Description=Daten-Volume für myapp

[Volume]
Label=app=myapp
```

---

## Quadlet: .pod-Datei

```ini
# ~/.config/containers/systemd/webapp.pod
[Unit]
Description=Webapp Pod

[Pod]
PublishPort=8080:80
Network=webapp.network

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/webapp-frontend.container
[Unit]
Description=Frontend Container
After=webapp-pod.service

[Container]
Image=docker.io/library/nginx:alpine
Pod=webapp.pod
Volume=%h/html:/usr/share/nginx/html:ro,z
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
# Pod-Service starten (startet auch Container)
systemctl --user daemon-reload
systemctl --user enable --now webapp-pod.service
systemctl --user start webapp-frontend.service
```

---

## Auto-Update mit Quadlet

```ini
# In .container-Datei eintragen:
[Container]
AutoUpdate=registry   # Prüft ob neueres Image in Registry verfügbar
# oder:
AutoUpdate=local      # Prüft auf lokale Image-Änderungen
```

```bash
# Auto-Update-Timer aktivieren (prüft täglich)
systemctl --user enable --now podman-auto-update.timer

# Status des Timers
systemctl --user list-timers podman-auto-update.timer

# Manuell aktualisieren
podman auto-update

# Dry-Run (nur anzeigen was aktualisiert werden würde)
podman auto-update --dry-run
```

---

## Socket Activation (Docker-API-Kompatibilität)

```bash
# Podman Socket aktivieren (User)
systemctl --user enable --now podman.socket

# Socket-Pfad
ls -la $XDG_RUNTIME_DIR/podman/podman.sock

# Docker-Tools auf Podman umleiten
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock

# Dauerhaft in .bashrc/.zshrc
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock' >> ~/.bashrc

# Testen
curl --unix-socket $XDG_RUNTIME_DIR/podman/podman.sock http://v1.0/version
```

---

## podman generate systemd (Legacy)

> [!WARNING] Deprecated in Podman 5.x
> `podman generate systemd` ist in Podman 5.x als deprecated markiert. Für neue Setups Quadlet verwenden.

```bash
# Aus laufendem Container (rootless)
podman generate systemd \
  --name \
  --files \
  --restart-policy=always \
  webserver

# Erzeugt: container-webserver.service

# Service installieren
mkdir -p ~/.config/systemd/user/
cp container-webserver.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now container-webserver.service
```

---

## Journald-Logging

Quadlet-Container loggen automatisch ins systemd-Journal:

```bash
# Container-Logs via journald
journalctl --user -u myapp.service -f

# Zeitraum
journalctl --user -u myapp.service --since "2026-03-06 10:00" --until "2026-03-06 12:00"

# JSON-Output für Tools
journalctl --user -u myapp.service -o json | jq .MESSAGE

# Alle Podman-Events
journalctl -t podman -f

# Logs nach Container-Name (journald-Treiber)
journalctl CONTAINER_NAME=myapp -f
```

---

## Restart-Policies

```ini
[Service]
Restart=always            # Immer neustarten (auch bei normalem Exit)
Restart=on-failure        # Nur bei Fehler (Exit-Code != 0)
Restart=on-abnormal       # Bei Signal, Watchdog oder Timeout
Restart=unless-stopped    # Immer außer wenn manuell gestoppt

RestartSec=5s             # Wartezeit zwischen Neustarts
StartLimitBurst=5         # Max. 5 Neustarts
StartLimitIntervalSec=30  # ...innerhalb von 30 Sekunden
```

---

## Quadlet-Variablen Referenz

| Variable | Wert |
|----------|------|
| `%h` | Home-Verzeichnis des Users |
| `%u` | Benutzername |
| `%U` | User-ID (UID) |
| `%t` | `$XDG_RUNTIME_DIR` (z.B. `/run/user/1000`) |
| `%T` | Temporäres Verzeichnis (`/tmp`) |

---

## Verwandte Notizen

- [[Podman Compose]] — Compose-Dateien vs. Quadlet im Vergleich
- [[Rootless Container]] — lingering für Auto-Start nach Reboot
- [[Podman Monitoring und Logging]] — journald-Logs und Events
- [[Podman Praktische Beispiele]] — Vollständige Quadlet-Beispiele
