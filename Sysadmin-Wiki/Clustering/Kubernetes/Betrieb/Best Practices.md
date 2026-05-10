---
tags:
  - kubernetes
  - best-practices
  - produktion
  - hardening
  - resource-management
---

# Kubernetes Best Practices

> [!ABSTRACT] TL;DR
> Gesammelte Best Practices für Kubernetes in Produktion — von Resource Management über Health Checks und Labeling bis hin zu Security, Namespace-Organisation und Deployment-Strategien. Dient als zentrale Checkliste für produktionsreife Cluster.

---

## Resource Management

### Requests und Limits setzen

```yaml
resources:
  requests:
    cpu: "100m"       # Garantierte 0.1 CPU Cores
    memory: "128Mi"   # Garantierte 128 MB RAM
  limits:
    cpu: "500m"       # Maximum 0.5 CPU Cores
    memory: "512Mi"   # Maximum 512 MB RAM — bei Überschreitung: OOMKill
```

| Parameter | Bedeutung | Empfehlung |
|-----------|-----------|------------|
| `requests.cpu` | Garantierte CPU | Durchschnittliche Nutzung |
| `limits.cpu` | Maximale CPU | 2–5x der Requests |
| `requests.memory` | Garantierte Memory | Normale Arbeitslast |
| `limits.memory` | Maximale Memory | 1.5–2x der Requests |

> [!IMPORTANT] Requests sind Pflicht in Produktion
> Ohne `requests` kann der Scheduler Pods nicht intelligent platzieren — das führt zu Overcommitment und Instabilität. Ohne `limits` kann ein einzelner Pod den gesamten Node lahmlegen.

### QoS-Klassen verstehen

| QoS-Klasse | Bedingung | OOM-Priorität |
|------------|-----------|---------------|
| **Guaranteed** | requests == limits (CPU + Memory) | Zuletzt gekillt |
| **Burstable** | requests < limits | Mittel |
| **BestEffort** | Keine requests/limits gesetzt | Zuerst gekillt |

```yaml
# Guaranteed QoS — für kritische Services
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"       # == requests
    memory: "512Mi"   # == requests
```

### Resource Quotas pro Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "20"
```

### LimitRange für Defaults

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

---

## Health Checks

### Drei Probe-Typen

| Probe | Zweck | Konsequenz bei Fehler |
|-------|-------|----------------------|
| **startupProbe** | Ist die App fertig gestartet? | Liveness/Readiness pausiert |
| **livenessProbe** | Lebt der Prozess noch? | Container wird neugestartet |
| **readinessProbe** | Kann die App Requests bearbeiten? | Pod aus Service-Endpoints entfernt |

```yaml
containers:
  - name: app
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30    # 30 × 10s = 5 Min für Startup
      periodSeconds: 10

    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 15
      timeoutSeconds: 5
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
```

> [!WARNING] Liveness-Probe nicht zu aggressiv
> Eine zu empfindliche Liveness-Probe (kurzes Timeout, wenige Retries) kann gesunde Container unnötig neustarten — besonders unter Last. Verwende `startupProbe` für langsam startende Apps statt `initialDelaySeconds`.

### Empfehlungen nach Service-Typ

| Service-Typ | startupProbe | livenessProbe | readinessProbe |
|-------------|-------------|---------------|----------------|
| Web-App | HTTP /health (60s) | HTTP /health (15s) | HTTP /ready (5s) |
| API | HTTP /status (30s) | HTTP /status (10s) | HTTP /status (5s) |
| Worker | exec check (60s) | exec check (30s) | Nicht nötig |
| Datenbank | TCP port (120s) | TCP port (30s) | exec pg_isready (10s) |

---

## Labeling-Strategie

### Empfohlene Standard-Labels

```yaml
metadata:
  labels:
    # Kubernetes-empfohlene Labels
    app.kubernetes.io/name: webapp
    app.kubernetes.io/instance: webapp-production
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: online-shop
    app.kubernetes.io/managed-by: helm

    # Eigene Labels
    environment: production
    team: platform
    cost-center: engineering
```

### Label-Konventionen

| Kategorie | Label | Werte |
|-----------|-------|-------|
| App-Identifikation | `app.kubernetes.io/name` | `webapp`, `api`, `worker` |
| Umgebung | `environment` | `production`, `staging`, `dev` |
| Team | `team` | `platform`, `backend`, `frontend` |
| Tier | `tier` | `frontend`, `backend`, `database`, `cache` |
| Release | `app.kubernetes.io/version` | `1.2.3`, `2.0.0-beta` |

### Node-Labels für Placement

```bash
# Storage-Typ
kubectl label node worker-01 storage-type=ssd
kubectl label node worker-02 storage-type=hdd

# Zone/Rack
kubectl label node worker-01 topology.kubernetes.io/zone=eu-west-1a

# Rolle
kubectl label node worker-03 node-role=gpu
```

```yaml
# Node-Affinität nutzen
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: storage-type
              operator: In
              values: ["ssd"]
```

---

## Namespace-Organisation

### Empfohlene Struktur

| Namespace | Inhalt |
|-----------|--------|
| `production` | Produktions-Workloads |
| `staging` | Staging-Umgebung |
| `monitoring` | Prometheus, Grafana, Alertmanager |
| `logging` | Loki, Promtail, Elasticsearch |
| `ingress` | Traefik/Nginx Ingress Controller |
| `cert-manager` | Zertifikatsverwaltung |
| `storage` | Longhorn, CSI-Driver |

### Network Policies pro Namespace

```yaml
# Default: Alles blockieren
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Dann gezielt freigeben
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## Pod Disruption Budgets

PDBs verhindern, dass zu viele Pods gleichzeitig durch Wartung (Drain, Upgrades) entfernt werden:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
spec:
  minAvailable: 2          # Mindestens 2 Pods müssen laufen
  # ODER:
  # maxUnavailable: 1      # Maximal 1 Pod gleichzeitig weg
  selector:
    matchLabels:
      app: webapp
```

| Strategie | Wann verwenden |
|-----------|----------------|
| `minAvailable: N` | Mindestanzahl garantieren (für HA) |
| `maxUnavailable: 1` | Maximal 1 Pod gleichzeitig weg (für Rolling Updates) |
| `maxUnavailable: "25%"` | Prozentual (für größere Deployments) |

> [!IMPORTANT] PDB + replicas beachten
> Bei `replicas: 3` und `minAvailable: 3` kann kein Node mehr gedrained werden! Setze `minAvailable` immer mindestens 1 unter `replicas`.

---

## Deployment-Strategien

### Rolling Update (Standard)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Max. 1 zusätzlicher Pod
      maxUnavailable: 0     # Kein Pod darf fehlen (Zero-Downtime)
```

### Empfehlungen

| Parameter | Wert | Effekt |
|-----------|------|--------|
| `maxSurge: 1, maxUnavailable: 0` | Zero-Downtime | Langsamer, aber sicher |
| `maxSurge: 25%, maxUnavailable: 25%` | Balanced | Standard für größere Deployments |
| `maxSurge: 0, maxUnavailable: 1` | Resource-schonend | Keine extra Kapazität nötig |

### Rollback vorbereiten

```bash
# Revision-History konfigurieren
spec:
  revisionHistoryLimit: 10    # Letzte 10 Revisionen behalten

# Rollback durchführen
kubectl rollout undo deployment/webapp
kubectl rollout undo deployment/webapp --to-revision=3

# Rollout-Status
kubectl rollout status deployment/webapp
kubectl rollout history deployment/webapp
```

---

## Security Best Practices

### Pod Security Context

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

### Security-Checkliste

| Punkt | Umsetzung |
|-------|-----------|
| ✅ Non-Root Container | `runAsNonRoot: true` |
| ✅ Read-Only Filesystem | `readOnlyRootFilesystem: true` + tmpfs |
| ✅ Keine Privilege Escalation | `allowPrivilegeEscalation: false` |
| ✅ Capabilities droppen | `capabilities.drop: ["ALL"]` |
| ✅ Network Policies | Default-Deny + explizite Freigaben |
| ✅ RBAC minimale Rechte | Least-Privilege-Prinzip |
| ✅ Image-Scanning | Trivy/Grype in CI/CD Pipeline |
| ✅ Secrets verschlüsselt | EncryptionConfiguration für etcd |
| ✅ Service Accounts einschränken | `automountServiceAccountToken: false` |
| ✅ Pod Security Standards | `restricted` Level als Standard |

---

## Logging & Monitoring

### Structured Logging

```yaml
# Annotations für Log-Parsing
metadata:
  annotations:
    fluentbit.io/parser: "json"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

### Monitoring-Labels

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: webapp-monitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: webapp
  endpoints:
    - port: metrics
      interval: 15s
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Lösung |
|-------------|---------|--------|
| `:latest` Tag | Nicht reproduzierbar | Feste Tags oder SHA-Digest |
| Keine Resource Requests | Overcommitment, OOM | Requests + Limits setzen |
| Keine Health Checks | Fehlerhafte Pods im Traffic | Liveness + Readiness Probes |
| Root-Container | Privilege Escalation | `runAsNonRoot: true` |
| Alles in `default` Namespace | Keine Isolation | Namespaces nach Funktion |
| `kubectl apply` ohne Git | Drift, nicht nachvollziehbar | GitOps verwenden |
| PVC ohne StorageClass | Manuelles PV-Management | Dynamic Provisioning |
| Keine PDB | Downtime bei Node-Drain | PDBs für alle HA-Services |
| Service Account auto-mount | Unnötige API-Zugriffe | `automountServiceAccountToken: false` |
| Kein Rollback-Plan | Stuck bei fehlerhaftem Release | `revisionHistoryLimit` + Rollback-Strategie |

---

## Produktions-Checkliste

### Cluster

- [ ] Mindestens 3 Control-Plane-Nodes (HA)
- [ ] etcd-Backups automatisiert
- [ ] Node Auto-Scaling oder genug Kapazität
- [ ] NTP synchronisiert auf allen Nodes
- [ ] Cluster-Monitoring (Prometheus + Grafana)
- [ ] Alerting konfiguriert (Alertmanager)
- [ ] Log-Aggregation (Loki oder ELK)
- [ ] Ingress Controller + TLS (Cert-Manager)
- [ ] Network Policies aktiv

### Workloads

- [ ] Resource Requests + Limits auf allen Pods
- [ ] Health Checks (Startup + Liveness + Readiness)
- [ ] Pod Disruption Budgets für HA-Services
- [ ] Security Context (Non-Root, Read-Only, Drop Caps)
- [ ] Feste Image-Tags (kein `:latest`)
- [ ] ConfigMaps/Secrets statt Env-Hardcoding
- [ ] Horizontal Pod Autoscaler wo sinnvoll
- [ ] Anti-Affinity für Spread über Nodes

### Operations

- [ ] GitOps-Workflow (ArgoCD/Flux)
- [ ] CI/CD-Pipeline mit Image-Scanning
- [ ] Rollback-Strategie dokumentiert
- [ ] Upgrade-Plan für Kubernetes-Versionen
- [ ] Disaster-Recovery-Plan getestet
- [ ] RBAC nach Least-Privilege-Prinzip
- [ ] Secret-Rotation automatisiert

---

## Verwandte Notizen

- [[Kubernetes Index]] — Gesamtübersicht
- [[Pods]] — Pod-Konfiguration im Detail
- [[Deployments und ReplicaSets]] — Deployment-Strategien
- [[RBAC]] — Rollenbasierte Zugriffskontrolle
- [[Pod Security Standards]] — Security Levels
- [[Namespaces]] — Namespace-Konzepte
- [[Labels Selektoren und Annotations]] — Labeling im Detail
- [[Monitoring und Logging]] — Monitoring-Setup
- [[Cluster Upgrades und Backups]] — Upgrade-Strategien
- [[Troubleshooting]] — Fehlersuche
