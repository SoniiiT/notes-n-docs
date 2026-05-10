---
tags: [lxd, monitoring, logging, prometheus, grafana, ressourcen]
aliases: ["LXD Monitoring", "LXD Logging"]
---

# LXD Monitoring und Logging

> [!ABSTRACT]
> LXD bietet eingebaute Monitoring-Tools (`lxc top`, `lxc info`) sowie eine Prometheus-kompatible Metrics-API. Für umfassendes Monitoring lässt sich LXD in Prometheus + Grafana integrieren. Logs werden pro Container via systemd-journald verwaltet.

---

## Eingebautes Monitoring

### lxc top — Live-Ressourcenübersicht

```bash
# Live-Übersicht aller Container (wie top)
lxc top

# Beispiel-Ausgabe:
# +------------------+-------+------+--------+--------+----------+---------+
# | NAME             | CPU%  | MEM  | MEM%   | NET↑   | NET↓    | DISK    |
# +------------------+-------+------+--------+--------+----------+---------+
# | nginx-prod       | 0.2%  | 128M | 1.6%   | 1.2MB  | 45.6MB  | 4.2GB   |
# | postgres-01      | 1.4%  | 512M | 6.4%   | 240B   | 1.2KB   | 12.1GB  |
# | pihole           | 0.1%  | 64M  | 0.8%   | 8.1MB  | 3.4MB   | 1.1GB   |
# +------------------+-------+------+--------+--------+----------+---------+

# Nach CPU sortieren
lxc top --sort cpu

# Nur bestimmte Container
lxc top --project default
```

### lxc info — Ressourcen-Snapshot

```bash
# Ressourcenverbrauch eines Containers
lxc info mycontainer

# Nur Netzwerk-Statistiken
lxc info mycontainer | grep -A 10 "Network usage"

# Host-Ressourcen anzeigen
lxc info --resources
```

**Beispiel-Ausgabe `lxc info nginx-prod`:**
```
Name: nginx-prod
Status: Running
Type: container
...
Resources:
  CPU usage (in seconds): 1234
  Memory usage: 128.00MiB
  Memory usage (peak): 256.00MiB
  Network usage:
    eth0: RX: 45.60MiB TX: 1.20MiB
```

---

## Prometheus-Integration

LXD 5.x+ bietet eine eingebaute Prometheus-Metrics-API:

### Metrics-Endpoint aktivieren

```bash
# Metrics auf lokalem Port 9090 bereitstellen
lxc config set core.metrics_address :9090

# TLS-Authentifizierung (empfohlen für Produktion)
lxc config set core.metrics_address :9090
lxc config set core.metrics_authentication true

# Metrics-Zertifikat für Prometheus hinzufügen
lxc config trust add prometheus.crt --type=metrics

# Test: Metrics abrufen
curl http://localhost:9090/1.0/metrics
```

### Prometheus-Konfiguration

```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'lxd'
    metrics_path: '/1.0/metrics'
    scheme: http
    static_configs:
      - targets: ['localhost:9090']
    # Mit TLS:
    # scheme: https
    # tls_config:
    #   cert_file: /etc/prometheus/lxd-prometheus.crt
    #   key_file: /etc/prometheus/lxd-prometheus.key
    #   insecure_skip_verify: false
```

```bash
# Prometheus installieren (als LXD-Container)
lxc launch ubuntu:24.04 prometheus-ct
lxc exec prometheus-ct -- bash -c "
    apt update && apt install -y prometheus
    systemctl enable prometheus
    systemctl start prometheus
"

# Konfiguration einkopieren
lxc file push prometheus.yml prometheus-ct/etc/prometheus/prometheus.yml
lxc exec prometheus-ct -- systemctl restart prometheus
```

### Verfügbare LXD-Metriken

| Metrik | Beschreibung |
|--------|-------------|
| `lxd_cpu_seconds_total` | CPU-Zeit pro Container |
| `lxd_memory_MemTotal_bytes` | Gesamt-RAM pro Container |
| `lxd_memory_MemFree_bytes` | Freier RAM pro Container |
| `lxd_network_receive_bytes_total` | Netzwerk-Eingang |
| `lxd_network_transmit_bytes_total` | Netzwerk-Ausgang |
| `lxd_disk_read_bytes_total` | Disk-Lesezugriffe |
| `lxd_disk_written_bytes_total` | Disk-Schreibzugriffe |
| `lxd_instances_total` | Anzahl Container/VMs |
| `lxd_go_goroutines` | Daemon-interne Goroutines |

---

## Grafana Dashboard

```bash
# Grafana als Container (einfaches Setup)
lxc launch ubuntu:24.04 grafana-ct
lxc config device add grafana-ct http-proxy proxy \
    listen=tcp:0.0.0.0:3000 \
    connect=tcp:127.0.0.1:3000
lxc exec grafana-ct -- bash -c "
    apt install -y apt-transport-https software-properties-common
    wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
    echo 'deb https://packages.grafana.com/oss/deb stable main' | \
        tee /etc/apt/sources.list.d/grafana.list
    apt update && apt install -y grafana
    systemctl enable grafana-server
    systemctl start grafana-server
"
```

**Empfohlene Grafana-Dashboards für LXD:**
- Dashboard ID **19975** — LXD Metrics (community)
- Dashboard ID **13946** — Linux Host Overview (für Container-Innenperspektive)

---

## Log-Management

### LXD-Daemon-Logs

```bash
# Alle LXD-Logs
journalctl -u snap.lxd.daemon

# Letzte 100 Zeilen
journalctl -u snap.lxd.daemon -n 100

# Live-Stream
journalctl -u snap.lxd.daemon -f

# Fehler filtern
journalctl -u snap.lxd.daemon -p err

# Zeitraum eingrenzen
journalctl -u snap.lxd.daemon --since "2026-03-06 10:00" --until "2026-03-06 11:00"

# Lifecycle-Events (Container start/stop)
journalctl -u snap.lxd.daemon | grep -E "started|stopped|created|deleted"
```

### Container-Start/Stop-Logs

```bash
# Letzter Start-Log eines Containers
lxc info --show-log mycontainer

# Typische Ausgabe:
# 2026-03-06T10:00:00.000Z lxd mycontainer 20000 Starting container process
# 2026-03-06T10:00:01.234Z lxd mycontainer 20000 Container started
```

### Logs innerhalb der Container

```bash
# Systemd-Journal im Container
lxc exec mycontainer -- journalctl -n 50
lxc exec mycontainer -- journalctl -f  # live
lxc exec mycontainer -- journalctl -u nginx  # spezifischer Dienst

# Traditionelle Log-Dateien
lxc exec mycontainer -- tail -f /var/log/syslog
lxc exec mycontainer -- tail -f /var/log/nginx/access.log

# Logs aus Container auf den Host kopieren
lxc file pull -r mycontainer/var/log/ ./container-logs/
```

---

## Alerting

### Einfaches Shell-Monitoring

```bash
#!/bin/bash
# lxd-check.sh — einfaches Monitoring-Script

THRESHOLD_MEM=80  # Warnung ab 80% RAM
THRESHOLD_CPU=90  # Warnung ab 90% CPU

for ct in $(lxc list --format=csv -c n,s | grep RUNNING | cut -d, -f1); do
    # RAM-Nutzung prüfen
    mem_total=$(lxc info $ct | grep "Memory usage:" | awk '{print $3}' | tr -d 'MiB')
    mem_peak=$(lxc info $ct | grep "Memory usage (peak)" | awk '{print $4}' | tr -d 'MiB')
    
    echo "$ct: Mem: ${mem_total}MiB, Peak: ${mem_peak}MiB"
done

# Als Cron-Job (stündlich)
# 0 * * * * /usr/local/bin/lxd-check.sh >> /var/log/lxd-monitor.log 2>&1
```

### Prometheus Alertmanager (Regeln)

```yaml
# /etc/prometheus/rules/lxd-alerts.yml
groups:
  - name: lxd
    rules:
      - alert: LXDContainerHighMemory
        expr: (lxd_memory_MemTotal_bytes - lxd_memory_MemFree_bytes) 
              / lxd_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} hoher RAM-Verbrauch"
          description: "RAM-Nutzung: {{ $value | printf \"%.0f\" }}%"

      - alert: LXDContainerDown
        expr: lxd_instances_total{status="RUNNING"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Keine laufenden LXD-Container"
```

---

## cAdvisor (Alternative)

cAdvisor kann LXD-Container wie Docker-Container überwachen:

```bash
# cAdvisor als Container
lxc launch ubuntu:24.04 cadvisor-ct
lxc config device add cadvisor-ct web proxy \
    listen=tcp:0.0.0.0:8080 \
    connect=tcp:127.0.0.1:8080

lxc exec cadvisor-ct -- bash -c "
    wget https://github.com/google/cadvisor/releases/latest/download/cadvisor
    chmod +x cadvisor
    ./cadvisor --port=8080 &
"
```

> [!NOTE]
> cAdvisor erkennt LXD-Container automatisch via `/sys/fs/cgroup`. Für vollständige LXD-Metriken ist der native Prometheus-Endpoint jedoch genauer.

---

## Monitoring-Architektur (Empfehlung)

```
LXD Host
├── LXD Metrics API (:9090)
│       ↓
├── Prometheus Container (scrape alle 15s)
│       ↓
├── Grafana Container (Dashboard :3000)
│       ↓
└── Alertmanager Container → E-Mail / Slack / PagerDuty
```

```bash
# Alle Monitoring-Container mit Auto-Start
lxc config set prometheus-ct boot.autostart true
lxc config set prometheus-ct boot.autostart.priority 10
lxc config set grafana-ct boot.autostart true
lxc config set grafana-ct boot.autostart.priority 5
```

---

## Verwandte Notizen

- [[LXD CLI Cheatsheet]] — lxc top und lxc info
- [[LXD Best Practices]] — Ressourcenlimits setzen
- [[LXD Troubleshooting]] — Diagnose bei Problemen
- [[LXD Cluster Betrieb]] — Cluster-Monitoring
