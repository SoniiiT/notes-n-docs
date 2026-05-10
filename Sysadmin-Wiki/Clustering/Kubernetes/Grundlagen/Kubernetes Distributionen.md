---
tags: [kubernetes, distributionen, k3s, kubeadm, microk8s, rke2]
---

# Kubernetes Distributionen

> [!ABSTRACT]
> Kubernetes gibt es in vielen Varianten — von leichtgewichtigen Edge-Distributionen wie K3s über Standard-Installationen mit kubeadm bis hin zu Enterprise-Plattformen wie OpenShift. Die Wahl der Distribution hängt von Anwendungsfall, Ressourcen und Expertise ab.

---

## Übersicht der Distributionen

| Distribution | Betreiber | Zielgruppe | Gewicht |
|-------------|----------|-----------|---------|
| **kubeadm** | Kubernetes (CNCF) | Standard-Installationen | Schwer |
| **K3s** | SUSE/Rancher | Edge, IoT, Homelab | Leicht |
| **MicroK8s** | Canonical | Entwickler, Edge | Leicht |
| **RKE2** | SUSE/Rancher | Enterprise, Compliance | Mittel |
| **OpenShift** | Red Hat | Enterprise | Schwer |
| **Talos Linux** | Sidero Labs | Immutable Infrastructure | Mittel |
| **k0s** | Mirantis | Einfache Installation | Leicht |

---

## K3s — Leichtgewichtiges Kubernetes

K3s ist eine zertifizierte Kubernetes-Distribution, die als **einzelnes Binary** unter 100 MB bereitgestellt wird. Entwickelt von Rancher Labs (jetzt SUSE), ist K3s ideal für ressourcenbeschränkte Umgebungen.

### Merkmale

| Eigenschaft | Details |
|-------------|---------|
| Binary-Größe | < 100 MB |
| RAM-Verbrauch | < 512 MB (Server), < 256 MB (Agent) |
| Datenbank | SQLite (Single Node) oder etcd (HA) |
| Container Runtime | containerd (eingebettet) |
| CNI | Flannel (Standard) |
| Ingress | Traefik (Standard, deaktivierbar) |
| Load Balancer | ServiceLB (ehemals Klipper, deaktivierbar) |
| Storage | Local Path Provisioner (Standard) |

### Installation

```bash
# Server (Control Plane) installieren
curl -sfL https://get.k3s.io | sh -

# Server mit deaktivierten Standardkomponenten
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - \
  --disable=servicelb \
  --disable=traefik \
  --disable=local-storage

# Agent (Worker Node) installieren
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" sh -s - \
  --server=https://<SERVER-IP>:6443 \
  --token=<NODE-TOKEN>
```

### Vorteile

- Extrem einfache Installation (ein Befehl)
- Geringer Ressourcenverbrauch
- Alles in einem Binary (containerd, Flannel, CoreDNS)
- CNCF-zertifiziert — vollständig kompatibel mit Kubernetes
- Ideal für Homelab, Edge und IoT

> [!TIP]
> K3s ist die empfohlene Distribution für Homelab-Umgebungen. Es bietet die volle Kubernetes-Funktionalität bei einem Bruchteil des Ressourcenbedarfs.

---

## kubeadm — Standard Kubernetes

kubeadm ist das **offizielle Tool** der Kubernetes-Community zur Installation und Verwaltung von Kubernetes-Clustern.

### Merkmale

| Eigenschaft | Details |
|-------------|---------|
| Typ | Vanilla Kubernetes |
| Datenbank | etcd (extern oder stacked) |
| Container Runtime | containerd, CRI-O |
| CNI | Muss separat installiert werden |
| Ingress | Muss separat installiert werden |

### Installation

```bash
# Voraussetzungen installieren (Ubuntu)
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Cluster initialisieren
sudo kubeadm init \
  --control-plane-endpoint=<VIP>:6443 \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Worker Node hinzufügen
sudo kubeadm join <VIP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Vorteile

- Offizielles Kubernetes-Tool
- Volle Kontrolle über alle Komponenten
- Breite Community-Unterstützung
- Ideal zum Lernen der Kubernetes-Interna

### Nachteile

- Mehr manueller Aufwand (CNI, Ingress, Storage separat)
- Höherer Ressourcenverbrauch
- Komplexeres Setup

---

## MicroK8s — Snap-basiertes Kubernetes

MicroK8s wird von Canonical (Ubuntu) entwickelt und als Snap-Paket verteilt.

### Merkmale

| Eigenschaft | Details |
|-------------|---------|
| Installation | `snap install microk8s` |
| Add-ons | Aktivierbar per `microk8s enable <addon>` |
| CNI | Calico (Standard) |
| Storage | hostpath (Standard), OpenEBS (Add-on) |

```bash
# Installation
sudo snap install microk8s --classic

# Add-ons aktivieren
microk8s enable dns dashboard storage ingress

# Cluster-Status prüfen
microk8s status
```

---

## RKE2 — Ranger Kubernetes Engine 2

RKE2, auch bekannt als „RKE Government", kombiniert die Einfachheit von K3s mit der Compliance-Fähigkeit von kubeadm.

### Merkmale

| Eigenschaft | Details |
|-------------|---------|
| Fokus | FIPS-140-2 Compliance, CIS Benchmarks |
| Container Runtime | containerd |
| CNI | Canal (Flannel + Calico) |
| Ingress | NGINX Ingress Controller (Standard) |

```bash
# Server Installation
curl -sfL https://get.rke2.io | sh -
systemctl enable --now rke2-server.service

# Agent Installation
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable --now rke2-agent.service
```

---

## Managed Kubernetes (Cloud)

Cloud-Provider bieten vollständig verwaltete Kubernetes-Dienste:

| Service | Provider | Besonderheiten |
|---------|----------|---------------|
| **GKE** | Google Cloud | Autopilot-Modus, GKE Enterprise |
| **EKS** | AWS | Fargate Integration, Karpenter Autoscaler |
| **AKS** | Microsoft Azure | Azure AD Integration, Azure Arc |
| **DOKS** | DigitalOcean | Einfach und günstig |
| **LKE** | Linode/Akamai | Budget-freundlich |

### Vorteile von Managed Kubernetes

- Kein Betrieb der Control Plane nötig
- Automatische Updates und Patches
- Integration mit Cloud-Services (IAM, Storage, Networking)
- SLA und Support

### Nachteile

- Kosten (Control Plane + Node-Kosten)
- Vendor Lock-in Risiko
- Begrenzte Konfigurierbarkeit der Control Plane

---

## Vergleichsmatrix

| Feature | K3s | kubeadm | MicroK8s | RKE2 | Managed |
|---------|-----|---------|----------|------|---------|
| Setup-Einfachheit | ⭐⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Ressourcenverbrauch | ⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐ | — |
| Enterprise-Features | ⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| Community / Docs | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| HA-Support | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| Edge / IoT | ⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐ | ⭐ |
| Homelab | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐ |

---

## Verwandte Notizen

- [[Was ist Kubernetes]]
- [[Kubernetes Architektur]]
- [[K3s Cluster Installation]]
- [[Kubeadm Multi-Master Cluster]]
