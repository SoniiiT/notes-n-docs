---
tags: [kubernetes, daemonsets, statefulsets, jobs, cronjobs, workloads]
---

# DaemonSets StatefulSets und Jobs

> [!ABSTRACT]
> Neben Deployments bietet Kubernetes weitere Workload-Typen: DaemonSets für Pods auf jedem Node, StatefulSets für zustandsbehaftete Anwendungen und Jobs/CronJobs für einmalige oder wiederkehrende Aufgaben.

---

## Warum verschiedene Workload-Typen?

Ein [[Deployments und ReplicaSets|Deployment]] ist der Standard-Workload für zustandslose Anwendungen: „Starte X Kopien und verteile sie im Cluster." Aber nicht jede Anwendung passt in dieses Muster:

- **Was, wenn ein Pod auf *jedem* Node laufen muss?** — z.B. ein Log-Collector oder Monitoring Agent, der Daten vom Node selbst sammelt. Ein Deployment verteilt Pods zufällig, aber hier braucht man *genau einen pro Node*. → **DaemonSet**

- **Was, wenn Pods eine feste Identität brauchen?** — z.B. eine Datenbank mit 3 Replicas, bei der jede Instanz ihren eigenen Speicher und einen festen Hostnamen braucht (`db-0`, `db-1`, `db-2`). Ein Deployment behandelt alle Pods als austauschbar — das geht hier nicht. → **StatefulSet**

- **Was, wenn eine Aufgabe nur einmal (oder periodisch) laufen soll?** — z.B. ein Datenbank-Backup, eine Migration oder ein Report. Ein Deployment läuft dauerhaft, aber hier soll der Pod nach Abschluss der Aufgabe beendet werden. → **Job / CronJob**

---

## DaemonSet

Ein DaemonSet stellt sicher, dass **auf jedem Node** (oder einer definierten Auswahl) genau eine Kopie eines Pods läuft. Ideal für Node-level Services.

### Typische Anwendungsfälle

| Anwendung | Beispiel |
|-----------|---------|
| Log-Collection | Fluentd, Filebeat |
| Monitoring | Prometheus Node Exporter |
| Networking | kube-proxy, Flannel, Calico |
| Storage | Longhorn Manager |
| Security | CrowdSec, Falco |

### DaemonSet-Definition

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "250m"
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
```

> [!NOTE]
> Mit `tolerations` kann ein DaemonSet auch auf Control Plane Nodes laufen, die normalerweise mit einem Taint versehen sind.

```bash
# DaemonSets anzeigen
kubectl get daemonsets -A

# DaemonSet-Status
kubectl describe daemonset node-exporter -n monitoring
```

---

## StatefulSet

Ein StatefulSet verwaltet Pods mit **stabiler Identität** und **persistentem Storage**. Anders als bei Deployments, wo alle Pods austauschbar sind (jeder Pod kann jeden anderen ersetzen), bekommt bei einem StatefulSet jeder Pod eine **feste Nummer, einen festen DNS-Namen und eigenen Speicher**.

Warum ist das wichtig? Bei einer Datenbank mit 3 Replicas muss klar sein, welche Instanz der Primary ist (`db-0`), welche Replicas sind (`db-1`, `db-2`), und jede braucht ihren eigenen Speicher. Wird `db-1` neu gestartet, muss sie **denselben Speicher** und **denselben Namen** bekommen — ein Deployment kann das nicht garantieren.

### Unterschiede zu Deployments

| Aspekt | Deployment | StatefulSet |
|--------|-----------|------------|
| Pod-Namen | Zufällig (`webapp-7d9f8`) | Geordnet (`db-0`, `db-1`, `db-2`) |
| Reihenfolge | Parallel | Sequentiell (0 → 1 → 2) |
| Storage | Geteilt (optional) | Individuell pro Pod (VolumeClaimTemplate) |
| DNS | Über Service | Individuell pro Pod |
| Anwendung | Stateless Apps | Datenbanken, Message Queues |

### StatefulSet-Definition

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: default
spec:
  serviceName: postgres       # Headless Service Name
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # ← Jeder Pod bekommt ein eigenes PVC
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: longhorn
        resources:
          requests:
            storage: 10Gi
```

### Headless Service (erforderlich)

StatefulSets benötigen einen Headless Service für stabile DNS-Einträge:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None           # ← Headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

Damit erhält jeder Pod einen stabilen DNS-Namen:

```text
postgres-0.postgres.default.svc.cluster.local
postgres-1.postgres.default.svc.cluster.local
postgres-2.postgres.default.svc.cluster.local
```

---

## Job

Ein Deployment läuft dauerhaft — es sorgt dafür, dass Pods immer laufen. Aber nicht alles muss dauerhaft laufen: Datenbank-Migrationen, Batch-Verarbeitungen oder Report-Generierungen sind **einmalige Aufgaben**, die nach Abschluss beendet werden sollen.

Ein **Job** führt eine solche einmalige Aufgabe aus. Er erstellt einen oder mehrere Pods, wartet bis sie erfolgreich abgeschlossen sind, und markiert sich dann als fertig. Bei einem Fehler wird der Pod automatisch neu gestartet (bis `backoffLimit` erreicht ist).

### Einfacher Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
        - name: migration
          image: meine-app:latest
          command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
  backoffLimit: 3              # Maximal 3 Versuche bei Fehlschlag
  activeDeadlineSeconds: 300   # Timeout nach 5 Minuten
```

### Paralleler Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processing
spec:
  completions: 10     # 10 erfolgreiche Durchläufe insgesamt
  parallelism: 3      # Maximal 3 Pods gleichzeitig
  template:
    spec:
      containers:
        - name: worker
          image: batch-worker:latest
      restartPolicy: Never
```

```bash
# Jobs anzeigen
kubectl get jobs

# Job-Logs anzeigen
kubectl logs job/db-migration

# Abgeschlossene Jobs aufräumen
kubectl delete job db-migration
```

---

## CronJob

Ein **CronJob** ist ein Job, der nach einem **Zeitplan** automatisch ausgeführt wird — wie ein Cron-Daemon auf Linux, aber Kubernetes-nativ. Typische Anwendungsfälle:

- **Backups** — Tägliches Datenbank-Backup um 02:00 Uhr
- **Cleanup** — Alte Logs oder temporäre Dateien stündlich aufräumen
- **Reports** — Wöchentliche Nutzungsstatistiken generieren
- **Health Checks** — Alle 5 Minuten externe Services auf Erreichbarkeit prüfen

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"           # Täglich um 02:00 Uhr
  concurrencyPolicy: Forbid        # Kein paralleler Lauf
  successfulJobsHistoryLimit: 3    # 3 erfolgreiche Jobs behalten
  failedJobsHistoryLimit: 1        # 1 fehlgeschlagenen Job behalten
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: backup-tool:latest
              command: ["/bin/sh", "-c", "backup.sh"]
          restartPolicy: OnFailure
```

### Cron-Syntax

```text
┌───────────── Minute (0-59)
│ ┌───────────── Stunde (0-23)
│ │ ┌───────────── Tag des Monats (1-31)
│ │ │ ┌───────────── Monat (1-12)
│ │ │ │ ┌───────────── Wochentag (0-6, 0=Sonntag)
│ │ │ │ │
* * * * *

Beispiele:
  "0 * * * *"      → Jede Stunde
  "0 2 * * *"      → Täglich um 02:00
  "0 0 * * 0"      → Jeden Sonntag um Mitternacht
  "*/15 * * * *"   → Alle 15 Minuten
  "0 9-17 * * 1-5" → Stündlich Mo-Fr 09:00-17:00
```

### Concurrency Policy

| Policy | Verhalten |
|--------|----------|
| `Allow` | Mehrere Jobs können gleichzeitig laufen (Standard) |
| `Forbid` | Neuer Job wird übersprungen, wenn alter noch läuft |
| `Replace` | Alter Job wird abgebrochen, neuer wird gestartet |

```bash
# CronJobs anzeigen
kubectl get cronjobs

# CronJob manuell auslösen
kubectl create job --from=cronjob/backup manual-backup
```

---

## Verwandte Notizen

- [[Pods]]
- [[Deployments und ReplicaSets]]
- [[Persistent Volumes und Storage Classes]]
- [[Labels Selektoren und Annotations]]
