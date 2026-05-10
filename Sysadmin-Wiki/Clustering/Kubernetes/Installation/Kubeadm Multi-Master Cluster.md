---
tags: [kubernetes, kubeadm, multi-master, installation, ha-cluster]
---

# Kubeadm Multi-Master Cluster

> [!ABSTRACT]
> Anleitung zur Installation eines Kubernetes-Clusters mit kubeadm auf Ubuntu. Dieses Setup umfasst mehrere Control Plane Nodes mit HAProxy und Keepalived als LoadBalancer für High Availability.

---

## Architektur

```text
                    ┌─────────────────┐
                    │   Virtual IP    │
                    │ 192.168.178.10  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────┴──┐  ┌───────┴────┐  ┌──────┴─────┐
    │ LB Node 1  │  │ LB Node 2  │  │ LB Node 3  │
    │ HAProxy +  │  │ HAProxy +  │  │ HAProxy +  │
    │ Keepalived │  │ Keepalived │  │ Keepalived │
    └─────────┬──┘  └───────┬────┘  └──────┬─────┘
              │              │              │
    ┌─────────┴──┐  ┌───────┴────┐  ┌──────┴─────┐
    │   CP 01    │  │   CP 02    │  │   CP 03    │
    │ kubeadm    │  │ kubeadm    │  │ kubeadm    │
    └────────────┘  └────────────┘  └────────────┘
              │              │              │
    ┌─────────┴──┐  ┌───────┴────┐  ┌──────┴─────┐
    │ Worker 01  │  │ Worker 02  │  │ Worker 03  │
    └────────────┘  └────────────┘  └────────────┘
```

> [!NOTE]
> Im Vergleich zu K3s benötigt kubeadm separate LoadBalancer-Nodes und mehr manuelle Konfiguration. Die Kubernetes-Distribution K3s löst dies mit KubeVIP deutlich einfacher.

---

## Schritt 1: Alle Nodes vorbereiten

Die folgenden Schritte müssen auf **allen Nodes** ausgeführt werden (LoadBalancer, Control Plane und Worker):

### Swap deaktivieren

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Kernel-Module laden

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

# Module persistent machen
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

# Netzwerk-Parameter setzen
sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT

sudo sysctl --system
```

### Firewall deaktivieren

```bash
sudo ufw disable
```

### Container Runtime installieren

```bash
# Docker als Container Runtime
sudo apt install -y docker.io
```

### Kubernetes-Pakete installieren

```bash
# GPG Key und Repository hinzufügen
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Kubernetes-Tools installieren
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

> [!WARNING]
> `apt-mark hold` verhindert unbeabsichtigte Upgrades der Kubernetes-Pakete. Version-Upgrades sollten immer kontrolliert durchgeführt werden.

---

## Schritt 2: LoadBalancer einrichten

Auf den LoadBalancer-Nodes werden HAProxy und Keepalived installiert.

### HAProxy installieren

```bash
sudo apt-get install -y haproxy

# Backup der Konfiguration
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

HAProxy-Konfiguration am Ende der Datei hinzufügen:

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

```text
frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
  server k8s-cp01 192.168.178.11:6443 check fall 3 rise 2
  server k8s-cp02 192.168.178.12:6443 check fall 3 rise 2
  server k8s-cp03 192.168.178.13:6443 check fall 3 rise 2
```

```bash
# HAProxy starten
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

### Keepalived installieren

```bash
sudo apt-get install -y keepalived
```

Health-Check-Skript erstellen:

```bash
sudo nano /etc/keepalived/check_apiserver.sh
```

```bash
#!/bin/sh
errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 192.168.178.10; then
  curl --silent --max-time 2 --insecure https://192.168.178.10:6443/ -o /dev/null || errorExit "Error GET https://192.168.178.10:6443/"
fi
```

```bash
sudo useradd -r keepalived_script
sudo chmod +x /etc/keepalived/check_apiserver.sh
```

Keepalived konfigurieren:

```bash
sudo nano /etc/keepalived/keepalived.conf
```

```text
global_defs {
  enable_script_security
}

vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        192.168.178.10
    }
    track_script {
        check_apiserver
    }
    unicast_src_ip 192.168.178.5     # Eigene IP dieses LB
    unicast_peer {
        192.168.178.6                 # IPs der anderen LBs
        192.168.178.7
    }
}
```

```bash
sudo systemctl restart keepalived
sudo systemctl enable keepalived
```

> [!TIP]
> Die `priority` sollte auf dem primären LoadBalancer höher sein (z.B. 100) und auf den sekundären niedriger (z.B. 99, 98). Alle Nodes auf `state BACKUP` zu setzen ist empfohlen — der Node mit der höchsten Priorität wird automatisch Master.

---

## Schritt 3: Cluster initialisieren

Auf dem **ersten Control Plane Node**:

```bash
sudo kubeadm init \
  --control-plane-endpoint="192.168.178.10:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16
```

Nach erfolgreicher Initialisierung:

```bash
# Kubeconfig für den User einrichten
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### CNI Plugin installieren

```bash
# Flannel installieren
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Oder Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## Schritt 4: Weitere Control Plane Nodes hinzufügen

Der `kubeadm init` Output enthält den Join-Befehl:

```bash
sudo kubeadm join 192.168.178.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>
```

---

## Schritt 5: Worker Nodes hinzufügen

```bash
sudo kubeadm join 192.168.178.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

```bash
# Neuen Token generieren (falls abgelaufen)
kubeadm token create --print-join-command
```

---

## Vergleich: kubeadm vs K3s

| Aspekt | kubeadm | K3s |
|--------|---------|-----|
| LoadBalancer | HAProxy + Keepalived (extern) | KubeVIP (integriert) |
| Setup-Aufwand | Hoch (viele manuelle Schritte) | Niedrig (ein Befehl) |
| CNI | Manuell installieren | Flannel integriert |
| Storage | Manuell konfigurieren | Local Path integriert |
| Ressourcenbedarf | Hoch | Niedrig |
| Kontrolle | Maximum | Standard Kubernetes Features |

---

## Verwandte Notizen

- [[Kubernetes Index]]
- [[K3s Cluster Installation]]
- [[KubeVIP und High Availability]]
- [[kubectl einrichten]]
