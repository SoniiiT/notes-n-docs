---
tags: [kubernetes, labels, selektoren, annotations, metadata]
---

# Labels Selektoren und Annotations

> [!ABSTRACT]
> Labels, Selektoren und Annotations sind Metadaten-Mechanismen in Kubernetes. Labels dienen zur Identifikation und Gruppierung von Ressourcen, Selektoren filtern nach Labels und Annotations speichern zusätzliche nicht-identifizierende Informationen.

---

## Warum Metadaten?

In einem Kubernetes-Cluster laufen schnell hunderte oder tausende Ressourcen (Pods, Services, Deployments). Ohne ein System zur Organisation wäre es unmöglich, den Überblick zu behalten. Kubernetes nutzt drei Arten von Metadaten:

- **Labels** — Identifikation und Gruppierung. Labels sind *die zentrale Mechanik* in Kubernetes: Services finden ihre Pods über Labels, Deployments wissen über Labels, welche Pods ihnen gehören, und mit `kubectl` filtert man nach Labels. Ohne Labels funktioniert fast nichts.

- **Selektoren** — Filter für Labels. Wenn ein Service definiert `selector: app: webapp`, leitet er Traffic nur an Pods mit dem Label `app: webapp` weiter. Selektoren sind der Klebstoff, der Ressourcen miteinander verbindet.

- **Annotations** — Zusätzliche Informationen, die *nicht* zum Filtern verwendet werden. Beispiele: Build-Nummer, Verantwortlicher, Konfiguration für Ingress Controller. Annotations sind wie Notizzettel — sie speichern alles, was nicht in Labels passt.

> [!IMPORTANT]
> Der wichtigste Unterschied: **Labels** werden für die Auswahl (Selection) verwendet, **Annotations** nicht. Ein Service kann Pods über Labels finden, aber nicht über Annotations.

---

## Labels

Labels sind **Key-Value-Paare**, die an Kubernetes-Ressourcen angehängt werden. Sie dienen zur Organisation, Gruppierung und Auswahl von Ressourcen.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
    environment: production
    tier: frontend
    version: v2.1.0
    team: platform
```

### Label-Konventionen

| Muster | Beispiel | Beschreibung |
|--------|---------|-------------|
| `app` | `webapp` | Anwendungsname |
| `environment` | `production` | Umgebung |
| `tier` | `frontend` / `backend` | Architektur-Schicht |
| `version` | `v2.1.0` | Version |
| `app.kubernetes.io/name` | `traefik` | Kubernetes-empfohlenes Label |
| `app.kubernetes.io/component` | `ingress-controller` | Komponenten-Typ |

### Labels verwalten

```bash
# Label hinzufügen
kubectl label pod webapp tier=frontend

# Label ändern (--overwrite nötig)
kubectl label pod webapp tier=backend --overwrite

# Label entfernen
kubectl label pod webapp tier-

# Labels einer Ressource anzeigen
kubectl get pods --show-labels

# Pods mit bestimmtem Label anzeigen
kubectl get pods -l app=webapp
```

---

## Selektoren

Selektoren filtern Ressourcen anhand ihrer Labels. Sie werden von Services, Deployments und anderen Controllern verwendet.

### Equality-based Selektoren

```bash
# Exakter Match
kubectl get pods -l environment=production

# Ungleich
kubectl get pods -l environment!=staging

# Mehrere Bedingungen (AND)
kubectl get pods -l app=webapp,environment=production
```

### Set-based Selektoren

```bash
# In einer Menge
kubectl get pods -l 'environment in (production, staging)'

# Nicht in einer Menge
kubectl get pods -l 'environment notin (development)'

# Label existiert
kubectl get pods -l 'tier'

# Label existiert nicht
kubectl get pods -l '!canary'
```

### Selektoren in YAML

```yaml
# Deployment Selektor (matchLabels — equality-based)
spec:
  selector:
    matchLabels:
      app: webapp
      environment: production

# Erweitert (matchExpressions — set-based)
spec:
  selector:
    matchLabels:
      app: webapp
    matchExpressions:
      - key: environment
        operator: In
        values:
          - production
          - staging
      - key: tier
        operator: Exists
```

### Selektoren in der Praxis

```yaml
# Service findet Pods über Label-Selektor
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp        # ← Findet alle Pods mit label app=webapp
  ports:
    - port: 80
      targetPort: 8080
```

---

## Annotations

Annotations speichern **zusätzliche Metadaten**, die nicht zur Identifikation oder Selektion verwendet werden. Sie sind wie Notizzettel an Ressourcen — für Tools, Controller und Menschen.

Der wichtigste Unterschied zu Labels: **Kein Service, kein Deployment und kein Controller kann über Annotations filtern.** Annotations werden stattdessen von externen Tools gelesen und interpretiert:

- **Cert-Manager** liest `cert-manager.io/cluster-issuer` auf einem Ingress und erstellt automatisch ein TLS-Zertifikat
- **Traefik** liest `traefik.ingress.kubernetes.io/backend-protocol` und weiß, ob das Backend HTTP oder HTTPS spricht
- **kubectl** speichert in `kubectl.kubernetes.io/last-applied-configuration` die letzte angewandte Konfiguration für `kubectl diff`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    description: "Frontend-Anwendung für das Portal"
    git.commit: "a1b2c3d4e5f6"
    contact: "team-platform@example.com"
    kubectl.kubernetes.io/last-applied-configuration: "..."
```

### Häufige Annotations

| Annotation | Verwendet von | Beschreibung |
|-----------|-------------|-------------|
| `cert-manager.io/cluster-issuer` | Cert-Manager | ClusterIssuer für TLS-Zertifikate |
| `traefik.ingress.kubernetes.io/backend-protocol` | Traefik | Backend-Protokoll (HTTP/HTTPS) |
| `kubernetes.io/ingress.class` | Ingress Controller | Welcher Controller zuständig ist |
| `kubectl.kubernetes.io/last-applied-configuration` | kubectl | Letzte applied Konfiguration |

### Annotations verwalten

```bash
# Annotation hinzufügen
kubectl annotate pod webapp description="Frontend App"

# Annotation ändern
kubectl annotate pod webapp description="Neue Beschreibung" --overwrite

# Annotation entfernen
kubectl annotate pod webapp description-
```

---

## Labels vs Annotations

| Aspekt | Labels | Annotations |
|--------|--------|------------|
| Zweck | Identifikation und Selektion | Zusätzliche Informationen |
| Selektierbar | Ja | Nein |
| Zeichenlimit | 63 Zeichen (Value) | 256 KB |
| Verwendet von | Services, Deployments, RBAC | Tools, Controller, Dokumentation |
| Beispiel | `app: webapp` | `git.commit: a1b2c3` |

---

## Recommended Labels

Kubernetes empfiehlt standardisierte Labels mit dem Prefix `app.kubernetes.io/`:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: traefik
    app.kubernetes.io/instance: traefik-production
    app.kubernetes.io/version: "3.6.7"
    app.kubernetes.io/component: ingress-controller
    app.kubernetes.io/part-of: networking
    app.kubernetes.io/managed-by: helm
```

---

## Verwandte Notizen

- [[Pods]]
- [[Deployments und ReplicaSets]]
- [[Services]]
- [[Namespaces]]
