---
tags: [kubernetes, portainer, management, dashboard, container-management]
---

# Portainer

> [!ABSTRACT]
> Portainer ist eine Web-basierte Container-Management-Plattform. Es bietet eine grafische Oberfläche zur Verwaltung von Kubernetes-Clustern, Docker-Hosts und Docker Swarm — ideal für Teams, die eine visuelle Alternative zu `kubectl` suchen.

---

## Was ist Portainer?

Portainer vereinfacht die Verwaltung von Container-Umgebungen durch eine intuitive Web-Oberfläche:

| Feature | Beschreibung |
|---------|-------------|
| Multi-Cluster | Mehrere K8s-Cluster und Docker-Hosts verwalten |
| RBAC | Rollenbasierte Zugriffskontrolle über die UI |
| Templates | Anwendungen per Klick deployen |
| Logs | Container-Logs direkt im Browser |
| Shell | Console-Zugang zu Containern |
| GitOps | Deployments aus Git-Repositories |

> [!NOTE]
> Portainer existiert als **Community Edition (CE)** und **Business Edition (BE)**. Die CE ist kostenlos und für die meisten Homelab-Anwendungen ausreichend.

---

## Portainer in Docker vs Kubernetes

| Aspekt | Docker | Kubernetes |
|--------|--------|-----------|
| Deployment | Docker Container | Helm Chart |
| Scope | Einzelner Docker-Host | Ganzes K8s-Cluster |
| Features | Container, Images, Networks | Namespaces, Deployments, Helm, RBAC |
| Zugang | Port 9000 | Ingress / NodePort |
| Agent | Portainer Agent (optional) | Edge Agent (optional) |

---

## Installation in Kubernetes

### Mit Helm

```bash
# Helm Repository hinzufügen
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

# Portainer installieren
helm upgrade --install portainer portainer/portainer \
  --namespace portainer \
  --create-namespace \
  --values portainer-values.yaml
```

### Values-Datei

```yaml
# portainer-values.yaml
replicaCount: 1

image:
  repository: portainer/portainer-ce
  tag: 2.22.0
  pullPolicy: IfNotPresent

serviceAccount:
  annotations: {}
  name: portainer-sa-clusteradmin

service:
  type: ClusterIP
  httpPort: 9000
  edgePort: 8000

ingress:
  enabled: true
  ingressClassName: traefik
  annotations:
    traefik.ingress.kubernetes.io/backend-protocol: HTTPS
  hosts:
    - host: portainer.home.soniiit.net
      paths:
        - path: "/"
  tls:
    - secretName: home-soniiit-net-selfsigned-tls
      hosts:
        - portainer.home.soniiit.net

resources: {}

persistence:
  size: "1Gi"
  annotations: {}
```

**Wichtige Parameter:**

| Parameter | Beschreibung |
|-----------|-------------|
| `replicaCount: 1` | Portainer unterstützt nur eine Replica |
| `service.type: ClusterIP` | Intern erreichbar (Ingress leitet weiter) |
| `ingress.enabled: true` | Standard-Ingress aktiviert |
| `ingressClassName: traefik` | Traefik als Ingress Controller |
| `backend-protocol: HTTPS` | Portainer spricht intern HTTPS |
| `persistence.size: "1Gi"` | Speicher für Portainer-Daten |

> [!IMPORTANT]
> Portainer verwendet intern **HTTPS** (Port 9443). Die Annotation `traefik.ingress.kubernetes.io/backend-protocol: HTTPS` ist essentiell, damit Traefik korrekt mit Portainer kommuniziert.

---

## Erster Start

1. Portainer im Browser öffnen: `https://portainer.home.soniiit.net`
2. Admin-Benutzer mit Passwort erstellen
3. „Get Started" wählen — der lokale Cluster wird automatisch erkannt
4. Cluster-Übersicht mit Namespaces, Deployments und Pods

---

## Funktionen in Kubernetes

### Cluster-Verwaltung

- **Namespaces** — Erstellen, löschen und zwischen Namespaces wechseln
- **Applications** — Deployments, StatefulSets, DaemonSets verwalten
- **Helm** — Helm Charts direkt über die UI installieren
- **ConfigMaps & Secrets** — Konfigurationen bearbeiten
- **Volumes** — PVCs und StorageClasses verwalten

### Monitoring

- **Resource Usage** — CPU und RAM Verbrauch pro Pod
- **Events** — Kubernetes Events im Dashboard
- **Logs** — Container-Logs in Echtzeit

### Deployment-Optionen

| Methode | Beschreibung |
|---------|-------------|
| Form | Anwendung über Formular deployen |
| YAML | YAML-Manifest direkt eingeben |
| Helm | Helm Chart aus Repository installieren |
| Git | Manifest aus Git-Repository deployen |
| URL | YAML von einer URL laden |

---

## Portainer für Docker

Portainer kann auch Docker-Hosts verwalten — entweder als separater Docker-Container oder als Agent auf Remote-Hosts.

### Docker-Installation

```yaml
# docker-compose.yaml
services:
  portainer:
    image: portainer/portainer-ce:2.22.0
    ports:
      - "9443:9443"
      - "9000:9000"
      - "8000:8000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: unless-stopped

volumes:
  portainer_data:
```

### Remote Docker-Hosts verwalten

1. **Portainer Agent** auf dem Remote-Host installieren:

```yaml
services:
  portainer-agent:
    image: portainer/agent:latest
    ports:
      - "9001:9001"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    restart: unless-stopped
```

2. In Portainer unter **Environments → Add Environment → Docker Agent** den Host hinzufügen.

---

## Best Practices

| Praxis | Beschreibung |
|--------|-------------|
| Starkes Admin-Passwort | Mindestens 12 Zeichen mit Sonderzeichen |
| RBAC nutzen | Teams und Rollen konfigurieren |
| TLS aktivieren | Zugriff nur über HTTPS |
| Backup | Portainer-Volume regelmäßig sichern |
| Updates | Helm Upgrade für neue Versionen |

---

## Verwandte Notizen

- [[Kubernetes Index]]
- [[Helm Package Manager]]
- [[Ingress und IngressRoutes]]
- [[Traefik Ingress Controller]]
