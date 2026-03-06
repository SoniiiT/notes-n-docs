---
tags: [kubernetes, traefik, ingress-controller, reverse-proxy, middleware]
---

# Traefik Ingress Controller

> [!ABSTRACT]
> Traefik ist ein moderner Reverse Proxy und Ingress Controller für Kubernetes. Er unterstützt automatische Service Discovery, Let's Encrypt Zertifikate, Middleware-Chains und sowohl Standard-Ingress als auch eigene CRDs (IngressRoute).

---

## Was ist Traefik?

Traefik ist ein Cloud-nativer Edge Router, der als Ingress Controller in Kubernetes eingesetzt wird. Er leitet externen Traffic an die richtigen Services im Cluster weiter.

| Feature | Beschreibung |
|---------|-------------|
| Auto-Discovery | Erkennt automatisch neue Services |
| CRDs | IngressRoute, Middleware, TraefikService |
| TLS | Automatische TLS-Terminierung |
| Middleware | Headers, Auth, Rate Limiting, Redirect |
| Dashboard | Eingebautes Monitoring-Dashboard |
| Protokolle | HTTP, HTTPS, TCP, UDP, gRPC |

> [!NOTE]
> K3s liefert Traefik als Standard-Ingress-Controller mit. In einem Custom-Setup wird der Standard-Traefik deaktiviert (`--disable=traefik`) und eine eigene Installation mit angepassten Values durchgeführt.

---

## Installation

### Standard-Traefik in K3s deaktivieren

Bei der K3s-Installation:

```bash
curl -sfL https://get.k3s.io | sh -s - --disable=traefik
```

### Traefik mit Helm installieren

```bash
# Helm Repository hinzufügen
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

# Traefik installieren
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --values traefik-values.yaml
```

### Values-Datei

```yaml
# traefik-values.yaml
globalArguments:
  - "--global.sendanonymoususage=false"
  - "--global.checknewversion=false"

additionalArguments:
  - "--serversTransport.insecureSkipVerify=true"
  - "--log.level=INFO"

deployment:
  enabled: true
  replicas: 2
  annotations: {}
  podAnnotations: {}
  additionalContainers: []
  initContainers: []

ports:
  web:
    expose:
      default: true
    port: 8000
    exposedPort: 80
    protocol: TCP
  websecure:
    expose:
      default: true
    port: 8443
    exposedPort: 443
    protocol: TCP
    tls:
      enabled: true

ingressRoute:
  dashboard:
    enabled: false

providers:
  kubernetesCRD:
    enabled: true
    ingressClass: traefik
    allowExternalNameServices: true
  kubernetesIngress:
    enabled: true
    allowExternalNameServices: true
    publishedService:
      enabled: false

rbac:
  enabled: true

service:
  enabled: true
  type: LoadBalancer
  spec:
    loadBalancerIP: 192.168.178.20
```

**Wichtige Parameter:**

| Parameter | Beschreibung |
|-----------|-------------|
| `replicas: 2` | Zwei Traefik-Instanzen für HA |
| `insecureSkipVerify` | SSL-Verifikation zu Backend-Services überspringen |
| `loadBalancerIP` | Feste IP aus dem KubeVIP-Bereich |
| `dashboard.enabled: false` | Dashboard wird separat über IngressRoute konfiguriert |
| `kubernetesCRD.enabled` | Traefik CRDs (IngressRoute) aktivieren |
| `kubernetesIngress.enabled` | Standard Kubernetes Ingress ebenfalls unterstützen |

---

## Dashboard einrichten

### BasicAuth Credentials erstellen

```bash
# Passwort-Hash generieren
htpasswd -nb admin MeinSicheresPasswort | base64
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dashboard-credentials
  namespace: traefik
type: Opaque
data:
  users: <BASE64-ENCODED-HTPASSWD>
```

### Dashboard Middleware

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: traefik-dashboard-basicauth
  namespace: traefik
spec:
  basicAuth:
    secret: dashboard-credentials
```

### Dashboard IngressRoute

```yaml
# HTTP → HTTPS Redirect
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-http
  namespace: traefik
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.home.soniiit.net`)
      kind: Rule
      middlewares:
        - name: traefik-redirect
          namespace: traefik
      services:
        - name: api@internal
          kind: TraefikService
---
# HTTPS mit BasicAuth
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-https
  namespace: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.home.soniiit.net`)
      kind: Rule
      middlewares:
        - name: traefik-dashboard-basicauth
          namespace: traefik
      services:
        - name: api@internal
          kind: TraefikService
  tls:
    secretName: home-soniiit-net-selfsigned-tls
---
# Redirect Middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: traefik-redirect
  namespace: traefik
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

---

## Security Headers Middleware

Eine Middleware für Security Headers, die von allen Services genutzt werden kann:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: default-headers
  namespace: default
spec:
  headers:
    browserXssFilter: true
    contentTypeNosniff: true
    forceSTSHeader: true
    stsIncludeSubdomains: true
    stsPreload: true
    stsSeconds: 15552000
    referrerPolicy: no-referrer
    contentSecurityPolicy: >-
      default-src 'none';
      script-src 'self' 'unsafe-inline' 'unsafe-eval' https:;
      style-src 'self' 'unsafe-inline' https:;
      img-src 'self' data: https:;
      font-src 'self' https: data:;
      connect-src 'self' https:;
      frame-src 'self' https:;
      media-src 'self' https:;
      object-src 'none';
      frame-ancestors 'self';
      base-uri 'self';
      form-action 'self';
    customFrameOptionsValue: SAMEORIGIN
    customRequestHeaders:
      X-Forwarded-Proto: https
```

---

## EntryPoints

Traefik EntryPoints definieren die Ports, auf denen Traefik lauscht:

| EntryPoint | Port | Beschreibung |
|-----------|------|-------------|
| `web` | 80 | HTTP-Traffic |
| `websecure` | 443 | HTTPS-Traffic |

```yaml
# In der IngressRoute referenzieren
spec:
  entryPoints:
    - web         # Nur HTTP
    - websecure   # Nur HTTPS
```

---

## Service-Beispiel: Anwendung exponieren

Komplettes Beispiel, um eine Anwendung über Traefik erreichbar zu machen:

```yaml
# 1. Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: nginx:latest
          ports:
            - containerPort: 80
---
# 2. Service
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: default
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
---
# 3. IngressRoute
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: webapp-https
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`webapp.home.soniiit.net`)
      kind: Rule
      middlewares:
        - name: default-headers
      services:
        - name: webapp
          port: 80
  tls:
    secretName: home-soniiit-net-selfsigned-tls
```

---

## Traefik in Docker vs Kubernetes

| Aspekt | Docker | Kubernetes |
|--------|--------|-----------|
| Discovery | Docker Labels | Kubernetes CRDs / Ingress |
| Konfiguration | docker-compose Labels | Helm Values + YAML Manifeste |
| TLS | Traefik ACME direkt | Cert-Manager + Secrets |
| Middleware | File Provider / Labels | Kubernetes CRD Middleware |
| Dashboard | Traefik API aktivieren | IngressRoute + BasicAuth |

> [!TIP]
> In Kubernetes übernimmt der **Cert-Manager** die TLS-Zertifikatsverwaltung statt Traefiks eingebautem ACME. Das ist flexibler und trennt die Verantwortlichkeiten sauber.

---

## Troubleshooting

```bash
# Traefik Pods prüfen
kubectl get pods -n traefik

# Traefik Logs
kubectl logs -n traefik -l app.kubernetes.io/name=traefik

# IngressRoutes anzeigen
kubectl get ingressroutes -A

# Middlewares anzeigen
kubectl get middlewares -A

# Service-Typ und IP prüfen
kubectl get svc -n traefik
```

---

## Verwandte Notizen

- [[Kubernetes Index]]
- [[Ingress und IngressRoutes]]
- [[Cert-Manager]]
- [[Helm Package Manager]]
- [[Services]]
