---
tags: [podman, monitoring, logging, metrics, prometheus, journald, health-check, stats, events]
aliases: ["Podman Monitoring", "Podman Logging", "Container Monitoring"]
---

# Podman Monitoring und Logging

> [!ABSTRACT]
> Podman-Container loggen standardmäßig via **journald** (auf systemd-Systemen) oder **json-file**. Für Monitoring bietet Podman `podman stats`, `podman events` und Prometheus-Metriken via podman-exporter. Health Checks überwachen Container-Zustand automatisch.

---

## Logging-Treiber

### Überblick

| Treiber | Beschreibung | Empfehlung |
|---------|-------------|-----------|
| `journald` | systemd-Journal | ✅ Standard auf RHEL/Fedora |
| `json-file` | JSON-Datei im Container-Dir | ✅ Standard auf Ubuntu |
| `k8s-file` | Kubernetes-kompatibles JSON | Für K8s-Migration |
| `none` | Kein Logging | Nur für Debugging |
| `passthrough` | Direkte Container-Ausgabe | Dev-Umgebungen |

### Logging-Treiber konfigurieren

```bash
# Standard-Treiber abfragen
podman info --format "{{.Host.LogDriver}}"

# Beim Container setzen
podman run --log-driver journald myapp
podman run --log-driver json-file myapp
podman run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 myapp

# Dauerhaft in /etc/containers/containers.conf
```

```toml
# /etc/containers/containers.conf  (System)
# ~/.config/containers/containers.conf  (User)
[containers]
log_driver = "journald"
```

---

## Container-Logs lesen

### podman logs

```bash
# Alle Logs
podman logs webserver

# Live-Logs (follow)
podman logs -f webserver

# Letzte N Zeilen
podman logs --tail 100 webserver
podman logs -n 50 webserver   # -n ist Alias für --tail

# Mit Timestamps
podman logs --timestamps webserver

# Seit Zeitpunkt
podman logs --since 1h webserver
podman logs --since "2026-03-06T10:00:00" webserver
podman logs --until 30m webserver

# Stdout/Stderr getrennt
podman logs --stdout --stderr webserver
```

### journald-Logs

```bash
# Container-Logs via journald
journalctl CONTAINER_NAME=webserver -f

# Quadlet-Service-Logs
journalctl --user -u webserver.service -f
journalctl --user -u webserver.service -n 100 --no-pager

# Zeitraum
journalctl --user -u webserver.service \
  --since "2026-03-06 10:00" \
  --until "2026-03-06 12:00"

# JSON-Format für Weiterverarbeitung
journalctl --user -u webserver.service -o json | jq .MESSAGE

# Alle Container-Logs
journalctl -t podman -f
```

---

## Health Checks

### Health Check im Run-Befehl

```bash
# Health Check definieren
podman run \
  --health-cmd "curl -sf http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-timeout 5s \
  --health-retries 3 \
  --health-start-period 10s \
  myapp

# Gesunden Container verlassen — ab wann als healthy?
# start-period: Wartezeit nach Start (Anwendung kann hochfahren)
# interval: Wie oft prüfen
# timeout: Einzelner Check darf nicht länger dauern
# retries: N Fehler = unhealthy
```

### Health Check im Containerfile

```dockerfile
FROM node:20-alpine

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

### Health-Status abfragen

```bash
# Status eines Containers
podman inspect --format "{{.State.Health.Status}}" webserver
# starting → healthy → unhealthy

# Details zu Health-Checks
podman inspect --format "{{json .State.Health}}" webserver | jq .

# Alle Container mit Health-Status
podman ps --format "table {{.Names}}\t{{.Status}}"
```

---

## Ressourcen-Monitoring

### podman stats

```bash
# Live-Statistiken aller Container
podman stats

# Nur bestimmte Container
podman stats webserver db-container

# Einmaliger Snapshot (kein Loop)
podman stats --no-stream

# Nur bestimmte Felder
podman stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# JSON-Output
podman stats --no-stream --format json | jq .
```

### Ressourcen begrenzen

```bash
# CPU begrenzen
podman run --cpus 0.5 myapp        # 50% einer CPU
podman run --cpu-shares 512 myapp  # Relative Gewichtung (default=1024)

# Memory begrenzen
podman run --memory 512m myapp
podman run --memory 512m --memory-swap 1g myapp  # inkl. Swap

# CPU-Pinning (nur bestimmte Cores)
podman run --cpuset-cpus 0,1 myapp   # nur Core 0 und 1
```

---

## Events-System

```bash
# Alle Container-Events verfolgen
podman events

# Nur bestimmte Event-Typen
podman events --filter event=start
podman events --filter event=die
podman events --filter event=health_status

# Events eines Containers
podman events --filter container=webserver

# Zeitraum
podman events --since 1h
podman events --since "2026-03-06T10:00:00" --until "2026-03-06T11:00:00"

# JSON für Scripting
podman events --format json | jq .
```

**Wichtige Event-Typen:**
| Event | Beschreibung |
|-------|-------------|
| `create` | Container erstellt |
| `start` | Container gestartet |
| `stop` | Container gestoppt |
| `die` | Container beendet (mit Exit-Code) |
| `health_status` | Health-Check-Ergebnis |
| `oom` | Out-of-Memory Kill |
| `exec` | exec-Befehl ausgeführt |

---

## Prometheus-Metriken

### podman-exporter einrichten

```bash
# podman-exporter installieren (Fedora/RHEL)
sudo dnf install -y prometheus-podman-exporter

# Service aktivieren
sudo systemctl enable --now prometheus-podman-exporter.service

# Metriken abrufen
curl http://localhost:9882/metrics | grep podman
```

### Mit Quadlet deployen

```ini
# ~/.config/containers/systemd/podman-exporter.container
[Unit]
Description=Podman Prometheus Exporter

[Container]
Image=quay.io/navidys/prometheus-podman-exporter:latest
PublishPort=9882:9882
Volume=%t/podman/podman.sock:/run/podman/podman.sock:z
Environment=CONTAINER_HOST=unix:///run/podman/podman.sock

[Service]
Restart=always

[Install]
WantedBy=default.target
```

### Wichtige Metriken

```
# Container-Anzahl
podman_containers_info{status="running"}

# CPU-Nutzung
podman_container_cpu_percent{name="webserver"}

# Memory
podman_container_mem_usage_bytes{name="webserver"}
podman_container_mem_limit_bytes{name="webserver"}

# Netzwerk
podman_container_net_input_bytes_total{name="webserver"}
podman_container_net_output_bytes_total{name="webserver"}
```

---

## Image-Inspektion

```bash
# Image-Details
podman image inspect nginx:alpine

# Nur bestimmte Felder
podman image inspect --format "{{.Size}}" nginx:alpine
podman image inspect --format "{{.Created}}" nginx:alpine

# Layers anzeigen (für Build-Optimierung)
podman history nginx:alpine

# Vulnerabilities scannen
podman scout cves nginx:latest   # Falls podman scout verfügbar
trivy image nginx:latest          # Trivy-Alternative
```

---

## System-Infos

```bash
# Allgemeine Podman-Infos
podman info
podman info --format json | jq .

# Speichernutzung anzeigen
podman system df

# Einzelner Container
podman system df -v

# Unbenutzte Ressourcen aufräumen
podman system prune
podman system prune -a --volumes   # VORSICHT: Alle ungenutzten Volumes!
```

---

## Grafana-Dashboard

```yaml
# docker-compose.yml / Quadlet für Monitoring-Stack
# Grafana + Prometheus + podman-exporter

services:
  prometheus:
    image: docker.io/prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro,z
    ports:
      - "9090:9090"

  grafana:
    image: docker.io/grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=geheim
    volumes:
      - grafana-data:/var/lib/grafana:Z
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: podman
    static_configs:
      - targets: ['localhost:9882']
```

---

## Verwandte Notizen

- [[Systemd Integration]] — journald-Logging via Quadlet
- [[Podman Sicherheit]] — Resource Limits und Isolation
- [[Podman Troubleshooting]] — Logs für Fehlerdiagnose
- [[Podman Praktische Beispiele]] — Monitoring-Stack Beispiel
