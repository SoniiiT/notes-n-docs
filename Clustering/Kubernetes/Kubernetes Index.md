---
tags: [moc, index, kubernetes, k8s]
aliases: [Kubernetes Übersicht, K8s Index]
---

# ☸️ Kubernetes Index

> [!ABSTRACT]
> Übersicht aller Kubernetes-Themen in dieser Wissensdatenbank — von Grundlagen und Konzepten über Installation und Cluster-Komponenten bis hin zu Betrieb und Troubleshooting.

---

## 📚 Grundlagen

Einstieg in die Welt der Container-Orchestrierung.

| Thema | Notiz |
|-------|-------|
| Was ist Kubernetes? | [[Was ist Kubernetes]] |
| Architektur | [[Kubernetes Architektur]] |
| Vergleich mit Docker | [[Kubernetes vs Docker und Docker Swarm]] |
| Distributionen | [[Kubernetes Distributionen]] |

---

## 🧩 Konzepte

Kubernetes-Ressourcen und API-Objekte im Detail.

### Workloads

| Thema | Notiz |
|-------|-------|
| Pods | [[Pods]] |
| Deployments & ReplicaSets | [[Deployments und ReplicaSets]] |
| DaemonSets, StatefulSets, Jobs | [[DaemonSets StatefulSets und Jobs]] |

### Netzwerk

| Thema | Notiz |
|-------|-------|
| Kubernetes Netzwerk | [[Kubernetes Netzwerk]] |
| Services | [[Services]] |
| Ingress & IngressRoutes | [[Ingress und IngressRoutes]] |

### Konfiguration & Storage

| Thema | Notiz |
|-------|-------|
| ConfigMaps & Secrets | [[ConfigMaps und Secrets]] |
| Persistent Volumes | [[Persistent Volumes und Storage Classes]] |

### Organisation & Sicherheit

| Thema | Notiz |
|-------|-------|
| Namespaces | [[Namespaces]] |
| RBAC | [[RBAC]] |
| Pod Security Standards | [[Pod Security Standards]] |
| Labels & Annotations | [[Labels Selektoren und Annotations]] |

### Erweiterte Konzepte

| Thema | Notiz |
|-------|-------|
| CRDs & Operators | [[CRDs und Operators]] |
| Gateway API | [[Gateway API]] |
| Service Mesh | [[Service Mesh]] |

---

## 🔧 Installation & Setup

Cluster aufsetzen — von K3s bis Multi-Master.

| Thema | Notiz |
|-------|-------|
| K3s Cluster (Ansible + manuell) | [[K3s Cluster Installation]] |
| kubeadm Multi-Master | [[Kubeadm Multi-Master Cluster]] |
| High Availability mit KubeVIP | [[KubeVIP und High Availability]] |
| kubectl einrichten | [[kubectl einrichten]] |
### Helm

| Thema | Notiz |
|-------|-------|
| Helm Package Manager | [[Helm Package Manager]] |

---

## ⚙️ Cluster-Komponenten

Wichtige Add-ons und Services für den produktiven Betrieb.

| Komponente | Funktion | Notiz |
|-----------|----------|-------|
| KubeVIP | Virtual IP & LoadBalancer (Bare-Metal) | [[KubeVIP]] |
| Cert-Manager | TLS-Zertifikate (Let's Encrypt) | [[Cert-Manager]] |
| Longhorn | Verteilter Block-Storage | [[Longhorn]] |
| Traefik | Ingress Controller & Reverse Proxy | [[Traefik Ingress Controller]] |
| Portainer | Container-Management UI | [[Portainer]] |

---

## 🛠️ Betrieb

Tägliche Arbeit mit dem Cluster.

| Thema | Notiz |
|-------|-------|
| kubectl Cheatsheet | [[kubectl Befehle und Cheatsheet]] |
| Monitoring & Logging | [[Monitoring und Logging]] |
| Cluster Upgrades & Backups | [[Cluster Upgrades und Backups]] |
| GitOps (ArgoCD & Flux) | [[GitOps]] |
| Troubleshooting | [[Troubleshooting]] |

---

## Verwandte Notizen

- [[Was ist Kubernetes]]
