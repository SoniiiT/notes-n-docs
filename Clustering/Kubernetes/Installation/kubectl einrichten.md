---
tags: [kubernetes, kubectl, installation, kubeconfig, cli]
---

# kubectl einrichten

> [!ABSTRACT]
> kubectl ist das Kommandozeilen-Tool zur Interaktion mit Kubernetes-Clustern. Es kommuniziert über die Kubernetes API und ermöglicht die Verwaltung aller Cluster-Ressourcen.

---

## Installation

### Linux

```bash
# Aktuelle stabile Version herunterladen
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Ausführbar machen und installieren
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Version prüfen
kubectl version --client
```

### macOS

```bash
# Mit Homebrew
brew install kubectl

# Oder manuell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Windows

```powershell
# Mit Chocolatey
choco install kubernetes-cli

# Mit winget
winget install Kubernetes.kubectl

# Oder manuell
curl.exe -LO "https://dl.k8s.io/release/v1.35.0/bin/windows/amd64/kubectl.exe"
# kubectl.exe in den PATH verschieben
```

---

## Kubeconfig

Die Kubeconfig-Datei enthält die Verbindungsinformationen zum Cluster. Standardmäßig liegt sie unter `~/.kube/config`.

### Kubeconfig vom K3s-Server kopieren

```bash
# Auf dem K3s-Server
sudo cat /etc/rancher/k3s/k3s.yaml

# Auf den lokalen Rechner kopieren
scp user@192.168.178.11:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# Server-Adresse auf VIP ändern
sed -i 's/127.0.0.1/192.168.178.10/g' ~/.kube/config
```

### Kubeconfig-Struktur

```yaml
apiVersion: v1
kind: Config
current-context: k3s-cluster
clusters:
  - cluster:
      certificate-authority-data: <BASE64-CA-CERT>
      server: https://192.168.178.10:6443
    name: k3s-cluster
contexts:
  - context:
      cluster: k3s-cluster
      user: admin
      namespace: default
    name: k3s-cluster
users:
  - name: admin
    user:
      client-certificate-data: <BASE64-CLIENT-CERT>
      client-key-data: <BASE64-CLIENT-KEY>
```

---

## Contexts verwalten

Contexts ermöglichen die Arbeit mit mehreren Clustern:

```bash
# Aktuelle Konfiguration anzeigen
kubectl config view

# Aktuellen Context anzeigen
kubectl config current-context

# Alle Contexts auflisten
kubectl config get-contexts

# Context wechseln
kubectl config use-context production-cluster

# Context mit Namespace erstellen
kubectl config set-context dev --cluster=k3s-cluster --user=admin --namespace=development
```

### Mehrere Kubeconfigs zusammenführen

```bash
# KUBECONFIG Environment Variable mit mehreren Dateien
export KUBECONFIG=~/.kube/config:~/.kube/config-production:~/.kube/config-staging

# Zusammengeführte Config speichern
kubectl config view --flatten > ~/.kube/config-merged
```

---

## Autocompletion

### Bash

```bash
# Bash-Completion installieren
sudo apt install bash-completion

# kubectl Completion aktivieren
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Alias 'k' mit Completion
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

source ~/.bashrc
```

### Zsh

```bash
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
source ~/.zshrc
```

### PowerShell

```powershell
kubectl completion powershell | Out-String | Invoke-Expression

# Permanent in Profil speichern
kubectl completion powershell >> $PROFILE
```

---

## Nützliche Plugins

### kubectl Krew (Plugin Manager)

```bash
# Krew installieren
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Plugin installieren
kubectl krew install ctx       # Schneller Context-Wechsel
kubectl krew install ns        # Schneller Namespace-Wechsel
kubectl krew install neat      # Bereinigter YAML-Output
kubectl krew install tree      # Ressourcen-Hierarchie anzeigen
```

### Nützliche Plugins

| Plugin | Beschreibung |
|--------|-------------|
| `kubectx` / `kubens` | Schneller Context/Namespace-Wechsel |
| `k9s` | Terminal-basiertes Cluster-Dashboard |
| `stern` | Multi-Pod Log-Tailing |
| `kubecolor` | Farbiger kubectl-Output |
| `kubectl-tree` | Ressourcen-Hierarchie visualisieren |

---

## Schnellreferenz

```bash
# Verbindung testen
kubectl cluster-info
kubectl get nodes

# Konfiguration prüfen
kubectl config view --minify

# API-Versionen anzeigen
kubectl api-versions

# API-Ressourcen anzeigen
kubectl api-resources
```

---

## Verwandte Notizen

- [[K3s Cluster Installation]]
- [[kubectl Befehle und Cheatsheet]]
- [[Kubernetes Index]]
