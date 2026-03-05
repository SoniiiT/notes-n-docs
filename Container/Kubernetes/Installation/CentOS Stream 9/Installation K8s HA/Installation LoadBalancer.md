# Zusammenfassung

Ein LoadBalancer für Kubernetes sorgt für die Instandhaltung des Clusters. Sollte ein Controlplane ausfallen, dann für durch den LoadBalancer der Traffic auf die anderen Controlplane Nodes weitergeleitet. Hier ist es möglich mehrere LoadBalancer einzurichten, um diesen auch vor Ausfällen zu schützen. Die minimale Konfiguration wäre hierbei 2, jedoch empfohlen wären 3+ LoadBalancer, damit bei einem größeren Ausfall weiterhin das Cluster eine Verbindung aufbauen kann.

# Kernel Vorbereitung

Auf den LoadBalancer wird auch Kubernetes installiert, um die Zertifikate auf den LoadBalancer zu speichern. Jedoch werden die LoadBalancer nicht teil des Clusters.

Um den Kernel für die Installation von Kubernetes vorzubereiten, muss der Kernel ein wenig angepasst werden. Damit wird das Netzwerk und Containerisierung verbessert.

```shell title:shell
sudo modprobe overlay
sudo modprobe br_netfilter
```

Dann folgt der Befehl, um die Module im Boot zu starten:

```shell title:shell
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

Mit dem folgenden Befehl werden Sachen wie Netzwerk bridging, iptables Unterstützung und IPv4 forwarding ermöglicht:

```shell title:shell
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Mit dem folgenden Befehl wird die Module jetzt aktiviert, um mit der Einrichtung von LoadBalancer fortzufahren:

```shell title:shell
sudo sysctl --system
```
# Namensänderung der Hosts

Sollte man die Hosts nicht beim Einrichten einen Namen gegeben haben, kann man dies wie folgt machen:

```shell title:shell
hostnamectl set-hostname k8s-firmennamen-cp01
```
Der Name wird beim nächsten Booten angewendet.

# Hosts in die Hosts Datei einschreiben

Damit man über Kürzel die Hosts erreichen kann, sollte man die Hosts mit ihrer IP in die Hosts Datei hineinschreiben.

Dies kann man wie folgt machen:
```shell title:shell
sudo nano /etc/hosts
```

Dann kann man in der Datei folgendes schreiben:

```shell title:shell
ip-adresse k8s-firmennamen-cp01
ip-adresse k8s-firmennamen-cp02
usw.
```
# Installation Containerd

Es könnte sein, dass Podman auf dem Server installiert ist. Um zu verhindern, dass irgendwelche Komplikation zustande kommen.

Dafür kann man folgenden Befehl verwenden:

```shell title:shell
if [[ $(cat /etc/redhat-release) == 'CentOS Stream release 9' ]]; then
  sudo dnf remove -y podman
fi
```

Nachdem Podman von System entfernt wurde, kann man mit der Installation von Containerd beginnen.

Dafür muss das Repository hinzugefügt werden, da Streamline 9 nicht Containerd in ihrer Repository vorhanden haben.

```shell title:shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
Als Nächstes wird Containerd mit folgendem Befehl installiert:

```shell title:shell
sudo dnf install -y containerd.io
```
Nach der Installation kann Containerd konfiguriert werden:

```shell title:shell
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

Da Kubernetes bei der Initialisierung eine Konfigurationsdatei von Containerd nicht mag, da es nicht die CRI (Container Runtime) starten kann. Kann mit folgendem Befehl die Konfiguration entfernt werden:

```shell title:shell
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
```
# SELinux in den Permissive Mode versetzten

Um die Sicherheit des Clusters zu erhöhen, kann folgender Befehl verwendet werden, um SELinux in den Permissive Mode zu versetzten:

```shell title:shell
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
# Firewall

Beim LoadBalancer muss man die Firewall deaktivieren, da sie nicht mit mehreren IPs zurechtkommt.

```shell title:shell
sudo systemctl disable firewalld
sudo systemctl stop firewalld
```
# Deaktivierung SWAP

Nun ist wichtig, dass SWAP deaktiviert wird. Dafür kann folgender Befehl benutzt werden:

```shell title:shell
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a
```

# Installation Kubernetes

Nun kann man das Repository von Kubernetes installieren. Dafür muss folgender Befehl eingegeben werden:

```shell title:shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
Nach dem Einrichten des Repositorys kann man mit der Installation von Kubernetes beginnen:

```shell title:shell
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```
Nachdem dies erfolgt ist, kann der LoadBalancer eingerichtet werden.

# Installation HAProxy

HAProxy wird für das Loadbalancing von internen Anfragen benötigt.

## Installation

Um HAProxy auf dem Cluster zu installieren, muss folgender Befehl verwendet werden:
```shell title:shell
sudo dnf install haproxy -y
```
## Einrichtung von HAProxy

Nach der Installation kann HAProxy eingerichtet werden.

Ich empfehle immer ein Backup zu erstellen, sollte man versehentlich etwas gelöscht haben:

```shell title:shell
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```
Nun kann man die Konfigurationsdatei bearbeiten:

```shell title:shell
sudo nano /etc/haproxy/haproxy.cfg
```
Es sollte folgendes in der Konfigurationsdatei enthalten sein:

```plaintext title:haproxy.cfg
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
  server k8s-firmennamen-cp01 ip-adresse:6443 check fall 3 rise 2
```

Nachdem dies gemacht wurde, kann man mit den folgenden Befehlen HAProxy starten:
```shell title:shell
setsebool -P haproxy_connect_any=1
sudo systemctl start haproxy.service
sudo systemctl enable haproxy.service
sudo systemctl status haproxy.service
```

Sollte im Status Failed stehen, sollte man die Konfigurationsdatei nochmal anschauen. Sollte im Fehler drin stehen, dass es die Nodes nicht erreichen konnte. Ist das nicht so schlimm da, das Cluster noch nicht erstellt wurde.

# Installation Keepalived

Nun kann man die Installation von Keepalived durchführen. Keepalived ist dafür verantwortlich, die virtuelle IP-Adresse bereitzustellen. Die IP-Adresse kann zu jedem LoadBalancer springen, sollte einer ausfallen.

## Installation

Um Keepalived zu installieren, sollte man diesen Befehl ausführen:
```shell title:shell
sudo dnf install keepalived -y
```
## Einrichtung eines Skripts

Das folgende Skript überprüft, ob das Cluster antwortet:
```shell title:shell
sudo nano /etc/keepalived/check_apiserver.sh
```

```shell title:check_apiserver.sh
#!/bin/sh

errorExit() {
  echo "*** $@" 1%3E&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q vip-adresse; then
  curl --silent --max-time 2 --insecure https://vip-adresse:6443/ -o /dev/null || errorExit "Error GET https://vip-adresse:6443/"
fi
```

Damit Keepalived das Skript ausführen kann. Benötigt es einen extra User und das Skript muss auch ausführbar gemacht werden:
```shell title:shell
sudo useradd -r keepalived_script
sudo chmod +x /etc/keepalived/check_apiserver.sh
```

Nun kann die Konfiguration von Keepalived gemacht werden. Auch hier empfehle ich ein Backup von der vorhanden Konfigurationsdatei zu erstellen:
```shell title:shell
sudo cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
```

Nachdem das Backup erstellt wurde, kann man mit der Konfiguration beginnen. Hier sollte dann folgende folgendes stehen:
```shell title:shell
sudo nano /etc/keepalived/keepalived.conf
```

```plaintext title:keepalived.conf
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
    interface intenet-interface #ip a
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        $ip_virtual
    }
    track_script {
        check_apiserver
    }
    unicast_src_ip loadbalancer-ip        #Immer die IP-Adresse von dem Loadbalancer auf dem es konfiguriert wird
    unicast_peer {
        loadbalancer-ip                   #Immer die IP-Adresse von den anderen LoadBalancer
    }
}
```

Nachdem nun die Konfigurationsdatei bearbeitet wurde, kann man Keepalived starten:
```shell title:shell
sudo systemctl start keepalived
sudo systemctl enable keepalived
sudo systemctl status keepalived
```

Sollte im Status Fehler vorhanden sein, sollte man die Konfigurationsdatei überprüfen. Sollte hier der Backupslave als Status stehen. Ist es nicht so schlimm, da das Cluster noch nicht existiert.