---
tags: [kubernetes, k3s, installation, ansible, cluster, homelab]
---

# K3s Cluster Installation

> [!ABSTRACT]
> Anleitung zur Installation eines K3s-Clusters — sowohl manuell per Shell-Befehle als auch automatisiert mit Ansible. Basierend auf einem Multi-Master-Setup mit 3 Server-Nodes und 3 Agent-Nodes, inklusive KubeVIP für High Availability.

---

## Voraussetzungen

### Hardware-Anforderungen

| Rolle | Min. CPU | Min. RAM | Min. Disk | Empfohlen |
|-------|---------|---------|----------|-----------|
| Server (Control Plane) | 2 vCPU | 2 GB | 20 GB | 4 vCPU, 4 GB RAM |
| Agent (Worker) | 1 vCPU | 1 GB | 20 GB | 2+ vCPU, 4+ GB RAM |

### Netzwerk-Planung

| Komponente | Beispiel-IP | Beschreibung |
|-----------|------------|-------------|
| VIP (Virtual IP) | 192.168.178.10 | Floating IP für HA API-Zugang |
| Server 01 | 192.168.178.11 | Erster Control Plane Node |
| Server 02 | 192.168.178.12 | Zweiter Control Plane Node |
| Server 03 | 192.168.178.13 | Dritter Control Plane Node |
| Agent 01 | 192.168.178.20 | Erster Worker Node |
| Agent 02 | 192.168.178.21 | Zweiter Worker Node |
| Agent 03 | 192.168.178.22 | Dritter Worker Node |
| LB IP-Range | 192.168.178.30-39 | IP-Bereich für LoadBalancer Services |

### Software-Voraussetzungen

- Ubuntu 22.04 / 24.04 (empfohlen) oder andere Linux-Distribution
- SSH-Zugang zu allen Nodes
- Root-Rechte auf allen Nodes
- Netzwerkverbindung zwischen allen Nodes

---

## Manuelle Installation

### Schritt 1: System vorbereiten (alle Nodes)

```bash
# System aktualisieren
sudo apt update && sudo apt upgrade -y

# Swap deaktivieren (Kubernetes-Anforderung)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Schritt 2: Ersten Server installieren

Der erste Server initialisiert das Cluster mit `--cluster-init` (aktiviert embedded etcd für HA):

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" INSTALL_K3S_CHANNEL=latest sh -s - \
  --disable=servicelb \
  --disable=traefik \
  --disable=local-storage \
  --tls-san=192.168.178.10 \
  --cluster-init \
  --kube-controller-manager-arg bind-address=0.0.0.0 \
  --kube-proxy-arg metrics-bind-address=0.0.0.0 \
  --kube-scheduler-arg bind-address=0.0.0.0 \
  --etcd-expose-metrics true \
  --kubelet-arg containerd=/run/k3s/containerd/containerd.sock
```

**Parameter erklärt:**

| Parameter | Beschreibung |
|-----------|-------------|
| `--disable=servicelb` | Standard-LB deaktivieren (KubeVIP wird verwendet) |
| `--disable=traefik` | Standard-Traefik deaktivieren (eigene Installation) |
| `--disable=local-storage` | Local Path deaktivieren (Longhorn wird verwendet) |
| `--tls-san=<VIP>` | VIP als SAN im TLS-Zertifikat hinzufügen |
| `--cluster-init` | Embedded etcd für HA-Cluster aktivieren |
| `--etcd-expose-metrics` | etcd-Metriken für Monitoring bereitstellen |

### Schritt 3: Node-Token abrufen

```bash
# Token vom ersten Server abrufen
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Schritt 4: Weitere Server hinzufügen

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" INSTALL_K3S_CHANNEL=latest sh -s - \
  --server=https://192.168.178.11:6443 \
  --token=<NODE-TOKEN> \
  --disable=servicelb \
  --disable=traefik \
  --disable=local-storage \
  --kube-controller-manager-arg bind-address=0.0.0.0 \
  --kube-proxy-arg metrics-bind-address=0.0.0.0 \
  --kube-scheduler-arg bind-address=0.0.0.0 \
  --etcd-expose-metrics true \
  --kubelet-arg containerd=/run/k3s/containerd/containerd.sock
```

### Schritt 5: Agents hinzufügen

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" INSTALL_K3S_CHANNEL=latest sh -s - \
  --server=https://192.168.178.10:6443 \
  --token=<NODE-TOKEN>
```

> [!NOTE]
> Agents verbinden sich über die **VIP** (nicht die IP eines einzelnen Servers), damit bei einem Server-Ausfall die Verbindung bestehen bleibt.

### Schritt 6: Kubeconfig kopieren

```bash
# Kubeconfig vom Server kopieren
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Server-Adresse auf VIP ändern
sed -i 's/127.0.0.1/192.168.178.10/g' ~/.kube/config

# Cluster testen
kubectl get nodes
```

---

## Installation mit Ansible

### Ansible Inventory

```yaml
all:
  hosts:
    k3s-soniiit-server01:
      ansible_host: 192.168.178.11
    k3s-soniiit-server02:
      ansible_host: 192.168.178.12
    k3s-soniiit-server03:
      ansible_host: 192.168.178.13
    k3s-soniiit-agent01:
      ansible_host: 192.168.178.20
    k3s-soniiit-agent02:
      ansible_host: 192.168.178.21
    k3s-soniiit-agent03:
      ansible_host: 192.168.178.22

  children:
    k3s_cluster:
      vars:
        kubernetes_vip: 192.168.178.10
        kubernetes_interface: eth0
        kubernetes_range_global: 192.168.178.30-192.168.178.39
      children:
        k3s_servers:
          hosts:
            k3s-soniiit-server01:
            k3s-soniiit-server02:
            k3s-soniiit-server03:
        k3s_agents:
          hosts:
            k3s-soniiit-agent01:
            k3s-soniiit-agent02:
            k3s-soniiit-agent03:
```

### Installations-Playbook

Das folgende Playbook installiert den gesamten K3s-Cluster inklusive KubeVIP:

````yaml
- name: Update und Upgrade
  hosts: all
  become: true
  tasks:
    - name: Update packages with apt
      when: ansible_pkg_mgr == 'apt'
      ansible.builtin.apt:
        update_cache: true

    - name: Upgrade packages with apt
      when: ansible_pkg_mgr == 'apt'
      ansible.builtin.apt:
        upgrade: dist

- name: Install Kubernetes Cluster
  hosts: k3s_cluster
  become: true
  tasks:
    - name: Install K3s auf Server 01
      ansible.builtin.shell:
        cmd: >-
          curl -sfL https://get.k3s.io |
          INSTALL_K3S_EXEC="server" INSTALL_K3S_CHANNEL=latest sh -s -
          --disable=servicelb --disable=traefik --disable=local-storage
          --tls-san={{ kubernetes_vip }}
          --cluster-init
          --kube-controller-manager-arg bind-address=0.0.0.0
          --kube-proxy-arg metrics-bind-address=0.0.0.0
          --kube-scheduler-arg bind-address=0.0.0.0
          --etcd-expose-metrics true
          --kubelet-arg containerd=/run/k3s/containerd/containerd.sock
      when: inventory_hostname == "k3s-soniiit-server01"

    - name: KubeVIP RBAC installieren
      ansible.builtin.shell:
        cmd: kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
      when: inventory_hostname == "k3s-soniiit-server01"

    - name: KubeVIP DaemonSet erstellen
      ansible.builtin.shell:
        cmd: |
          export VIP="{{ kubernetes_vip }}"
          export INTERFACE="{{ kubernetes_interface }}"
          KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
          alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
          kube-vip manifest daemonset \
            --interface $INTERFACE \
            --address $VIP \
            --inCluster --taint \
            --controlplane --services --arp \
            --leaderElection > /var/lib/rancher/k3s/server/manifests/kube-vip.yaml
      when: inventory_hostname == "k3s-soniiit-server01"

    - name: KubeVIP Cloud Controller installieren
      ansible.builtin.shell:
        cmd: |
          kubectl apply -f https://raw.githubusercontent.com/kube-vip/kube-vip-cloud-provider/main/manifest/kube-vip-cloud-controller.yaml
          kubectl create configmap -n kube-system kubevip --from-literal range-global="{{ kubernetes_range_global }}"
      when: inventory_hostname == "k3s-soniiit-server01"

    - name: Node Token abrufen
      ansible.builtin.shell: cat /var/lib/rancher/k3s/server/node-token
      register: k3s_token
      when: inventory_hostname == "k3s-soniiit-server01"

    - name: K3s auf weiteren Servern installieren
      ansible.builtin.shell:
        cmd: >-
          curl -sfL https://get.k3s.io |
          INSTALL_K3S_EXEC="server" INSTALL_K3S_CHANNEL=latest sh -s -
          --server=https://{{ hostvars['k3s-soniiit-server01']['ansible_host'] }}:6443
          --token={{ hostvars['k3s-soniiit-server01']['k3s_token'].stdout }}
          --disable=servicelb --disable=traefik --disable=local-storage
          --kube-controller-manager-arg bind-address=0.0.0.0
          --kube-proxy-arg metrics-bind-address=0.0.0.0
          --kube-scheduler-arg bind-address=0.0.0.0
          --etcd-expose-metrics true
          --kubelet-arg containerd=/run/k3s/containerd/containerd.sock
      when: inventory_hostname != "k3s-soniiit-server01" and inventory_hostname in groups['k3s_servers']

    - name: K3s Agents installieren
      ansible.builtin.shell:
        cmd: >-
          curl -sfL https://get.k3s.io |
          INSTALL_K3S_EXEC="agent" INSTALL_K3S_CHANNEL=latest sh -s -
          --server=https://{{ kubernetes_vip }}:6443
          --token={{ hostvars['k3s-soniiit-server01']['k3s_token'].stdout }}
      when: inventory_hostname in groups['k3s_agents']

    - name: Kubeconfig kopieren
      ansible.builtin.copy:
        src: /etc/rancher/k3s/k3s.yaml
        dest: /home/{{ ansible_user }}/config
        remote_src: true
      when: inventory_hostname == "k3s-soniiit-server01"
````

### Ausführung

```bash
# Gesamtes Cluster installieren
ansible-playbook -i inventory/inventory.yaml playbooks/kubernetes/install-kubernetes.yaml

# Nur neue Server hinzufügen
ansible-playbook -i inventory/inventory.yaml playbooks/kubernetes/install-server.yaml

# Nur neue Agents hinzufügen
ansible-playbook -i inventory/inventory.yaml playbooks/kubernetes/install-agent.yaml
```

---

## K3s Deinstallation

### Manuell

```bash
# Auf Agent-Nodes
sudo /usr/local/bin/k3s-agent-uninstall.sh

# Auf Server-Nodes
sudo /usr/local/bin/k3s-uninstall.sh
```

### Mit Ansible

```yaml
- name: Uninstall Kubernetes
  hosts: all
  become: true
  tasks:
    - name: Remove K3s Agents
      ansible.builtin.shell: /usr/local/bin/k3s-agent-uninstall.sh
      when: inventory_hostname in groups['k3s_agents']

    - name: Remove K3s Servers
      ansible.builtin.shell: /usr/local/bin/k3s-uninstall.sh
      when: inventory_hostname in groups['k3s_servers']
```

---

## Nach der Installation

Nach der K3s-Installation folgen diese Schritte:

1. **KubeVIP einrichten** → [[KubeVIP und High Availability]]
2. **Traefik installieren** → [[Traefik Ingress Controller]]
3. **Cert-Manager installieren** → [[Cert-Manager]]
4. **Longhorn installieren** → [[Longhorn]]
5. **Portainer installieren** → [[Portainer]]

```bash
# Cluster-Status prüfen
kubectl get nodes -o wide

# Alle Pods anzeigen
kubectl get pods -A

# Cluster-Info
kubectl cluster-info
```

---

## Verwandte Notizen

- [[Kubernetes Index]]
- [[Kubernetes Distributionen]]
- [[KubeVIP und High Availability]]
- [[kubectl einrichten]]
- [[Helm Package Manager]]
