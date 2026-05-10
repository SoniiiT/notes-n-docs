---
tags: [kubernetes, configmaps, secrets, konfiguration, sicherheit]
---

# ConfigMaps und Secrets

> [!ABSTRACT]
> ConfigMaps speichern nicht-sensible Konfigurationsdaten, Secrets speichern sensible Daten wie Passwörter, API-Keys und TLS-Zertifikate. Beide können als Umgebungsvariablen oder Dateien in Pods eingebunden werden.

---

## Warum Konfiguration vom Code trennen?

In der klassischen Welt wird Konfiguration oft direkt in der Anwendung oder im Container-Image gespeichert — z.B. eine Datenbank-URL im Code oder eine Konfigurations-Datei, die ins Image gebacken wird. Das Problem: Jede Änderung an der Konfiguration erfordert ein neues Image und ein neues Deployment.

Kubernetes löst das mit **ConfigMaps** und **Secrets**. Statt Konfiguration ins Image zu packen, wird sie als eigene Ressource im Cluster gespeichert und zur Laufzeit in den Pod injiziert — als Umgebungsvariable oder als Datei.

> [!TIP]
> **Faustregel:** Alles, was sich zwischen Umgebungen unterscheidet (Datenbank-Host, Feature-Flags, Log-Level), gehört in eine ConfigMap oder ein Secret — **nicht** ins Image.

| Ressource | Zweck | Beispiele |
|-----------|-------|-----------|
| **ConfigMap** | Nicht-sensible Konfiguration | Datenbank-Host, Port, Log-Level, nginx.conf |
| **Secret** | Sensible Daten (Base64-kodiert) | Passwörter, API-Keys, TLS-Zertifikate |

Der Unterschied: Secrets werden Base64-kodiert gespeichert und können durch Encryption at Rest verschlüsselt werden. ConfigMaps sind Klartext. Funktional verhalten sich beide gleich — sie können als Umgebungsvariablen oder als Dateien in Pods gemountet werden.

---

## ConfigMaps

Eine ConfigMap speichert Konfigurationsdaten als Key-Value-Paare, die von der Anwendung getrennt werden.

### ConfigMap erstellen

```bash
# Aus Literalen
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=postgres \
  --from-literal=DATABASE_PORT=5432

# Aus einer Datei
kubectl create configmap nginx-config --from-file=nginx.conf

# Aus einem Verzeichnis
kubectl create configmap app-configs --from-file=config/
```

```yaml
# Als YAML-Manifest
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  DATABASE_HOST: "postgres"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    database:
      pool_size: 10
```

### ConfigMap verwenden

**Als Umgebungsvariablen:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: meine-app:latest
      # Einzelne Keys
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
      # Alle Keys auf einmal
      envFrom:
        - configMapRef:
            name: app-config
```

**Als Volume-Mount:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: meine-app:latest
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

> [!TIP]
> ConfigMaps, die als Volumes gemountet werden, werden automatisch aktualisiert, wenn sich die ConfigMap ändert. Bei Umgebungsvariablen ist ein Pod-Neustart nötig.

---

## Secrets

Secrets funktionieren wie ConfigMaps, aber für **sensible Daten** — Passwörter, API-Keys, TLS-Zertifikate, SSH-Keys. Der Unterschied ist nicht technisch groß (beide sind Key-Value-Paare), aber Kubernetes behandelt Secrets besonders:

- Secrets werden **Base64-kodiert** gespeichert (nicht verschlüsselt — Base64 ist nur Encoding!)
- Secrets können mit **Encryption at Rest** in etcd verschlüsselt werden
- Der Zugriff auf Secrets kann über [[RBAC]] separat eingeschränkt werden
- Secrets werden nur an Nodes gesendet, auf denen ein Pod läuft, der sie braucht
- Sie werden in `tmpfs` gemountet (nicht auf die Festplatte geschrieben)

> [!WARNING]
> Base64 ist **keine Verschlüsselung** — jeder kann `echo "Z2VoZWlt" | base64 -d` ausführen. Für echte Sicherheit muss **Encryption at Rest** für etcd aktiviert werden, oder man nutzt einen externen Secret-Manager wie HashiCorp Vault.

### Secret-Typen

| Typ | Beschreibung |
|-----|-------------|
| `Opaque` | Allgemeine Daten (Standard) |
| `kubernetes.io/tls` | TLS-Zertifikate (tls.crt + tls.key) |
| `kubernetes.io/dockerconfigjson` | Docker Registry Credentials |
| `kubernetes.io/basic-auth` | Basic Authentication |
| `kubernetes.io/ssh-auth` | SSH Key |

### Secret erstellen

```bash
# Aus Literalen
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=geheimes-passwort

# TLS Secret
kubectl create secret tls mein-tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Docker Registry Secret
kubectl create secret docker-registry registry-credentials \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=passwort
```

```yaml
# Als YAML-Manifest (Werte müssen Base64-kodiert sein)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
data:
  username: YWRtaW4=          # echo -n "admin" | base64
  password: Z2VoZWltZXMtcGFzc3dvcnQ=  # echo -n "geheimes-passwort" | base64
```

```yaml
# Mit stringData (Klartext, wird automatisch kodiert)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: geheimes-passwort
```

### Secret verwenden

**Als Umgebungsvariablen:**

```yaml
spec:
  containers:
    - name: app
      image: meine-app:latest
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

**Als Volume-Mount:**

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
```

---

## Praxisbeispiel: Cloudflare API Token

Aus dem Cert-Manager Setup — ein Secret für den Cloudflare API Token:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-credentials
  namespace: cert-manager
type: Opaque
stringData:
  token: DEIN-API-TOKEN-HIER
```

Dieses Secret wird vom Cert-Manager ClusterIssuer referenziert:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-credentials
              key: token
```

---

## Praxisbeispiel: TLS-Zertifikat

Self-Signed CA-Zertifikat als Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: selfsigned-ca-home
  namespace: cert-manager
type: Opaque
data:
  tls.crt: <base64-encoded-cert>   # cat ca.crt | base64 -w 0
  tls.key: <base64-encoded-key>    # cat ca.key | base64 -w 0
```

---

## Secrets verwalten

```bash
# Secrets auflisten
kubectl get secrets

# Secret-Details anzeigen (Werte sind Base64-kodiert)
kubectl get secret db-credentials -o yaml

# Secret dekodieren
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d

# Secret bearbeiten
kubectl edit secret db-credentials

# Secret löschen
kubectl delete secret db-credentials
```

---

## Best Practices

| Praxis | Beschreibung |
|--------|-------------|
| Secrets nicht in Git | YAML-Dateien mit Secrets gehören nicht ins Repository |
| External Secrets | Tools wie External Secrets Operator oder Vault für externe Secret-Verwaltung |
| RBAC einschränken | Nicht jeder User sollte Secrets lesen dürfen |
| Encryption at Rest | etcd-Verschlüsselung für Secrets aktivieren |
| Immutable Secrets | `immutable: true` für Secrets die sich nicht ändern sollen |
| Namespaces | Secrets in den Namespace der Anwendung legen |

```yaml
# Immutable Secret
apiVersion: v1
kind: Secret
metadata:
  name: static-config
type: Opaque
immutable: true
data:
  api-key: bWVpbi1hcGkta2V5
```

---

## Verwandte Notizen

- [[Pods]]
- [[Cert-Manager]]
- [[RBAC]]
- [[Persistent Volumes und Storage Classes]]
