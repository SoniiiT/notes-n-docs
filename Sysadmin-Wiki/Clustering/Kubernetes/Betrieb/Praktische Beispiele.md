---
tags:
  - kubernetes
  - beispiele
  - yaml
  - deployment
  - praxis
---

# Kubernetes Praktische Beispiele

> [!ABSTRACT] TL;DR
> Produktionsreife YAML-Manifeste und Deployment-Beispiele — vom einfachen Pod über eine Full-Stack-Anwendung bis hin zu CronJobs, StatefulSets und Canary Deployments. Alle Beispiele enthalten Health Checks, Resource Limits und Security Context.

---

## Beispiel 1: Full-Stack Web-Anwendung

Frontend (Nginx SPA) + Backend API (Node.js) + PostgreSQL Datenbank.

### Namespace und Grundstruktur

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
  labels:
    environment: production
```

### PostgreSQL (StatefulSet)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: webapp
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: webapp
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "$(POSTGRES_USER)"]
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "$(POSTGRES_USER)"]
            initialDelaySeconds: 5
            periodSeconds: 5
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: longhorn
        resources:
          requests:
            storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: webapp
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

### Backend API (Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
        tier: backend
        version: v1
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      automountServiceAccountToken: false
      containers:
        - name: api
          image: ghcr.io/myorg/api:1.2.3
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string
            - name: NODE_ENV
              value: "production"
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: api-config
                  key: log-level
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          startupProbe:
            httpGet:
              path: /health
              port: http
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            periodSeconds: 5
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api

---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: webapp
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 8080
      targetPort: http
      name: http

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: webapp
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

### Frontend (Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
      automountServiceAccountToken: false
      containers:
        - name: nginx
          image: ghcr.io/myorg/frontend:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            periodSeconds: 5
          volumeMounts:
            - name: cache
              mountPath: /var/cache/nginx
            - name: run
              mountPath: /var/run
      volumes:
        - name: cache
          emptyDir: {}
        - name: run
          emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: webapp
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 8080
      targetPort: 8080
```

### Ingress (Traefik IngressRoute)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: webapp
  namespace: webapp
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: frontend
          port: 8080
    - match: Host(`app.example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api
          port: 8080
      middlewares:
        - name: strip-api-prefix
  tls:
    certResolver: letsencrypt

---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: strip-api-prefix
  namespace: webapp
spec:
  stripPrefix:
    prefixes:
      - /api
```

### Secrets und ConfigMap

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: webapp
type: Opaque
stringData:
  username: webapp_user
  password: HIER_SICHERES_PASSWORT
  connection-string: "postgresql://webapp_user:PASSWORT@postgres:5432/webapp?sslmode=disable"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: webapp
data:
  log-level: "info"
  cors-origins: "https://app.example.com"
```

### HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: webapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

### Deployment-Befehle

```bash
# Namespace und Secrets zuerst
kubectl apply -f namespace.yaml
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml

# Datenbank
kubectl apply -f postgres-statefulset.yaml

# Backend und Frontend
kubectl apply -f api-deployment.yaml
kubectl apply -f frontend-deployment.yaml

# Ingress und Autoscaling
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml

# Status prüfen
kubectl get all -n webapp
```

---

## Beispiel 2: CronJob — Datenbank-Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: webapp
spec:
  schedule: "0 3 * * *"          # Täglich um 03:00
  concurrencyPolicy: Forbid       # Keine parallelen Jobs
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 600   # Max. 10 Minuten
      template:
        spec:
          restartPolicy: OnFailure
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
          containers:
            - name: backup
              image: postgres:16-alpine
              command:
                - /bin/sh
                - -c
                - |
                  FILENAME="backup-$(date +%Y%m%d-%H%M%S).sql.gz"
                  pg_dump -h postgres -U $PGUSER $PGDATABASE | gzip > /backups/$FILENAME
                  echo "Backup erstellt: $FILENAME ($(du -h /backups/$FILENAME | cut -f1))"
                  # Alte Backups löschen (älter als 30 Tage)
                  find /backups -name "backup-*.sql.gz" -mtime +30 -delete
              env:
                - name: PGUSER
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: username
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: password
                - name: PGDATABASE
                  value: webapp
              volumeMounts:
                - name: backup-storage
                  mountPath: /backups
              resources:
                requests:
                  cpu: "100m"
                  memory: "256Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
```

---

## Beispiel 3: Canary Deployment

Schrittweises Rollout einer neuen Version mit Traffic-Splitting.

### Stable Deployment (aktuelle Version)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-stable
  namespace: webapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api
      track: stable
  template:
    metadata:
      labels:
        app: api
        track: stable
        version: v1
    spec:
      containers:
        - name: api
          image: ghcr.io/myorg/api:1.2.3
          ports:
            - containerPort: 8080
```

### Canary Deployment (neue Version)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-canary
  namespace: webapp
spec:
  replicas: 1                     # 1 von 5 = 20% Traffic
  selector:
    matchLabels:
      app: api
      track: canary
  template:
    metadata:
      labels:
        app: api
        track: canary
        version: v2
    spec:
      containers:
        - name: api
          image: ghcr.io/myorg/api:2.0.0
          ports:
            - containerPort: 8080
```

### Service routet zu beiden

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: webapp
spec:
  selector:
    app: api            # Matcht BEIDE: stable + canary
  ports:
    - port: 8080
```

```bash
# Canary schrittweise skalieren
kubectl scale deployment api-canary --replicas=2 -n webapp   # 40%
kubectl scale deployment api-canary --replicas=4 -n webapp   # 50%

# Stable Deployment auf neue Version umstellen
kubectl set image deployment/api-stable api=ghcr.io/myorg/api:2.0.0 -n webapp

# Canary entfernen
kubectl delete deployment api-canary -n webapp
```

---

## Beispiel 4: Init Container + Sidecar

### Migration vor App-Start (Init) + Log-Shipping (Sidecar)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-with-init
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      initContainers:
        # Init 1: Warten auf Datenbank
        - name: wait-for-db
          image: busybox:1.36
          command: ['sh', '-c', 'until nc -z postgres 5432; do echo "Warte auf DB..."; sleep 2; done']

        # Init 2: Migration ausführen
        - name: migrate
          image: ghcr.io/myorg/api:1.2.3
          command: ['node', 'migrate.js']
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string

      containers:
        # Hauptcontainer
        - name: api
          image: ghcr.io/myorg/api:1.2.3
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app

        # Sidecar: Log-Shipping
        - name: log-shipper
          image: grafana/promtail:3.1.0
          args:
            - -config.file=/etc/promtail/config.yml
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
              readOnly: true
            - name: promtail-config
              mountPath: /etc/promtail
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"

      volumes:
        - name: app-logs
          emptyDir: {}
        - name: promtail-config
          configMap:
            name: promtail-config
```

---

## Beispiel 5: Network Policy

```yaml
# Default Deny in Namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: webapp
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

---
# Frontend darf nur vom Ingress erreicht werden
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      tier: frontend
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress
  egress:
    - {}   # Frontend darf alles nach außen

---
# API darf nur von Frontend und Ingress erreicht werden
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-api
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - port: 5432
    - to:   # DNS erlauben
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
```

---

## Verwandte Notizen

- [[Kubernetes Index]] — Gesamtübersicht
- [[Pods]] — Pod-Konfiguration
- [[Deployments und ReplicaSets]] — Deployment-Strategien
- [[DaemonSets StatefulSets und Jobs]] — StatefulSets und CronJobs
- [[Services]] — Service-Typen
- [[ConfigMaps und Secrets]] — Konfigurationsmanagement
- [[Persistent Volumes und Storage Classes]] — Storage
- [[Best Practices]] — Produktions-Checkliste
- [[Ingress und IngressRoutes]] — Ingress-Konfiguration
- [[Troubleshooting]] — Fehlersuche
