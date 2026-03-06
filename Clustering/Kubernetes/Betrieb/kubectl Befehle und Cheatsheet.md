---
tags: [kubernetes, kubectl, befehle, cheatsheet, cli]
---

# kubectl Befehle und Cheatsheet

> [!ABSTRACT]
> Übersicht der wichtigsten kubectl-Befehle für die tägliche Arbeit mit Kubernetes — von grundlegenden Operationen über Debugging bis hin zu fortgeschrittenen Techniken.

---

## Grundlegende Befehle

### Ressourcen anzeigen

```bash
# Pods anzeigen
kubectl get pods                    # Im aktuellen Namespace
kubectl get pods -A                 # In allen Namespaces
kubectl get pods -n kube-system     # In einem bestimmten Namespace
kubectl get pods -o wide            # Mit zusätzlichen Infos (Node, IP)
kubectl get pods -o yaml            # Vollständiges YAML
kubectl get pods -o json            # Als JSON
kubectl get pods --show-labels      # Mit Labels
kubectl get pods -w                 # Watch-Modus (live Updates)

# Andere Ressourcen
kubectl get nodes
kubectl get services
kubectl get deployments
kubectl get namespaces
kubectl get ingress -A
kubectl get pvc
kubectl get secrets
kubectl get configmaps
kubectl get events --sort-by='.lastTimestamp'
```

### Ressourcen erstellen und verwalten

```bash
# Aus YAML-Datei erstellen/aktualisieren
kubectl apply -f manifest.yaml
kubectl apply -f verzeichnis/

# Aus YAML löschen
kubectl delete -f manifest.yaml

# Direkt erstellen
kubectl create namespace production
kubectl create deployment nginx --image=nginx:latest --replicas=3
kubectl create service clusterip nginx --tcp=80:80

# Löschen
kubectl delete pod mein-pod
kubectl delete deployment webapp
kubectl delete namespace test
```

### Details anzeigen

```bash
# Detaillierte Informationen
kubectl describe pod mein-pod
kubectl describe node worker-01
kubectl describe deployment webapp
kubectl describe service webapp

# YAML einer laufenden Ressource
kubectl get deployment webapp -o yaml
```

---

## Debugging

### Logs

```bash
# Container-Logs anzeigen
kubectl logs mein-pod
kubectl logs mein-pod -c container-name     # Bestimmter Container
kubectl logs mein-pod --previous             # Vorheriger Container (nach Crash)
kubectl logs mein-pod -f                     # Follow (live)
kubectl logs mein-pod --tail=100             # Letzte 100 Zeilen
kubectl logs mein-pod --since=1h             # Letzte Stunde

# Logs eines Deployments (alle Pods)
kubectl logs deployment/webapp

# Logs mit Label-Selektor
kubectl logs -l app=webapp --all-containers
```

### Shell-Zugang

```bash
# Shell in einem Pod öffnen
kubectl exec -it mein-pod -- /bin/bash
kubectl exec -it mein-pod -- /bin/sh        # Für Alpine-basierte Images
kubectl exec -it mein-pod -c container -- /bin/bash  # Bestimmter Container

# Einzelnen Befehl ausführen
kubectl exec mein-pod -- cat /etc/config/app.yaml
kubectl exec mein-pod -- env
kubectl exec mein-pod -- ls -la /data
```

### Port-Forwarding

```bash
# Pod-Port auf lokalen Rechner weiterleiten
kubectl port-forward pod/mein-pod 8080:80
kubectl port-forward deployment/webapp 8080:80
kubectl port-forward service/webapp 8080:80

# Auf allen Interfaces (nicht nur localhost)
kubectl port-forward --address 0.0.0.0 pod/mein-pod 8080:80
```

### Debug-Container

```bash
# Ephemeral Debug-Container starten
kubectl debug -it mein-pod --image=busybox --target=container-name

# Debug-Pod auf einem Node starten
kubectl debug node/worker-01 -it --image=ubuntu
```

### DNS-Debugging

```bash
# DNS-Test aus einem Pod
kubectl run dnsutils --image=tutum/dnsutils -it --rm -- nslookup kubernetes
kubectl run dnsutils --image=busybox -it --rm -- nslookup webapp.default.svc.cluster.local
```

### Netzwerk-Debugging

```bash
# Connectivity-Test
kubectl run curl-test --image=curlimages/curl -it --rm -- curl http://webapp:80

# Alle Endpoints eines Services
kubectl get endpoints webapp
```

---

## Scaling und Updates

```bash
# Skalieren
kubectl scale deployment webapp --replicas=5

# Autoscaling
kubectl autoscale deployment webapp --min=2 --max=10 --cpu-percent=80

# Rolling Update
kubectl set image deployment/webapp webapp=webapp:v2

# Rollout-Status
kubectl rollout status deployment/webapp

# Rollout-Historie
kubectl rollout history deployment/webapp

# Rollback
kubectl rollout undo deployment/webapp
kubectl rollout undo deployment/webapp --to-revision=2

# Rollout pausieren / fortsetzen
kubectl rollout pause deployment/webapp
kubectl rollout resume deployment/webapp
```

---

## Labels und Annotations

```bash
# Labels setzen
kubectl label pod mein-pod app=webapp
kubectl label node worker-01 disktype=ssd

# Labels entfernen
kubectl label pod mein-pod app-

# Nach Labels filtern
kubectl get pods -l app=webapp
kubectl get pods -l 'environment in (production, staging)'

# Annotations setzen
kubectl annotate pod mein-pod description="Mein Pod"
```

---

## Konfiguration und Secrets

```bash
# ConfigMap erstellen
kubectl create configmap app-config --from-literal=KEY=VALUE
kubectl create configmap app-config --from-file=config.yaml

# Secret erstellen
kubectl create secret generic db-secret --from-literal=password=geheim
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key

# Secret dekodieren
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

---

## Cluster-Informationen

```bash
# Cluster-Info
kubectl cluster-info
kubectl get nodes -o wide

# API-Ressourcen
kubectl api-resources
kubectl api-versions

# Resource-Verbrauch (Metrics Server nötig)
kubectl top nodes
kubectl top pods
kubectl top pods -A --sort-by=memory

# Events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n kube-system
```

---

## Fortgeschrittene Techniken

### JSONPath

```bash
# Bestimmte Felder extrahieren
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Custom Columns

```bash
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName,IP:.status.podIP'
```

### Dry-Run

```bash
# YAML generieren ohne zu erstellen
kubectl create deployment webapp --image=nginx --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip webapp --tcp=80:80 --dry-run=client -o yaml > service.yaml
```

### Diff

```bash
# Änderungen vor apply anzeigen
kubectl diff -f manifest.yaml
```

### Ressourcen kopieren

```bash
# Secret von einem Namespace in einen anderen kopieren
kubectl get secret my-secret -n source -o yaml | \
  sed 's/namespace: source/namespace: target/' | \
  kubectl apply -f -
```

---

## Nützliche Aliase

```bash
# In ~/.bashrc oder ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias ke='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
```

---

## Verwandte Notizen

- [[kubectl einrichten]]
- [[Kubernetes Index]]
- [[Pods]]
- [[Deployments und ReplicaSets]]
- [[Troubleshooting]]
