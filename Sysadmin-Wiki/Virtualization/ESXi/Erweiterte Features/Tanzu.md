---
tags: [virtualization, esxi, vsphere, vmware, tanzu, kubernetes, container, tkg]
aliases: ["Tanzu", "VMware Tanzu", "vSphere with Tanzu", "TKG"]
---

# Tanzu

> [!ABSTRACT]
> VMware Tanzu ist ein Portfolio von Produkten für Kubernetes- und Container-Management auf vSphere. Es ermöglicht das native Betreiben von Kubernetes-Workloads direkt auf dem Hypervisor — ohne separate VM-Schicht.

---

## Was ist Tanzu?

VMware Tanzu ist kein einzelnes Produkt, sondern ein Portfolio von Tools und Plattformen, das Kubernetes-Cluster auf VMware-Infrastruktur bereitstellt und verwaltet. Es wurde 2019 mit der Übernahme von Heptio und Pivotal eingeführt und ist seit vSphere 7.0 fest in die vSphere-Plattform integriert.

**Kernidee:** Kubernetes direkt auf ESXi betreiben — der Hypervisor wird selbst zum Kubernetes-Cluster. VMs und Container teilen sich dieselbe Verwaltungsebene in vCenter.

| Konzept | Bedeutung |
|---|---|
| Supervisor Cluster | K8s-Control-Plane direkt auf ESXi-Hosts aktiviert |
| vSphere Namespace | K8s-Namespace mit vSphere-Ressourcen-Policies (CPU, RAM, Storage) |
| Tanzu Kubernetes Grid (TKG) | Vollständige K8s-Cluster als VMs deployen und managen |
| Tanzu Mission Control (TMC) | Zentrale Multi-Cloud-K8s-Governance-Plattform |
| vSphere Pod | Container läuft direkt in einer leichtgewichtigen VM-Kapsel auf ESXi |

### Unterschied zu Standard-Kubernetes

| Merkmal | Standard K8s | vSphere with Tanzu |
|---|---|---|
| Infrastruktur | Beliebige VMs/Bare-Metal | ESXi-Hosts direkt |
| Storage | PersistentVolume via CSI | vSAN/VMFS via vSphere CSI |
| Netzwerk | CNI Plugin | NSX oder HAProxy Load Balancer |
| Verwaltung | kubectl + separate Tools | vCenter UI + kubectl |
| Lizenzbedarf | Open Source | VMware VCF |

---

## Varianten des Tanzu-Portfolios

| Produkt | Einsatzzweck | Voraussetzung |
|---|---|---|
| vSphere with Tanzu | K8s nativ auf ESXi-Hosts | vSAN/NSX, vCenter 7.0+ |
| TKG Service | K8s-Cluster als VMs über Supervisor | vSphere with Tanzu |
| TKG Standalone | K8s-Cluster auf beliebiger Infrastruktur | Keine vSphere-Pflicht |
| Tanzu Mission Control | Multi-Cluster-Governance, Policy-Mgmt | Beliebige K8s-Cluster |
| Tanzu Application Platform | Developer-Plattform für Cloud-native Apps | TKG oder andere K8s |

---

## Voraussetzungen

> [!IMPORTANT]
> vSphere with Tanzu erfordert zwingend einen Load Balancer (NSX oder HAProxy) sowie vSAN oder External Storage. Standard vSwitch reicht nicht aus.

| Komponente | Anforderung |
|---|---|
| vCenter | Version 7.0+ (8.0 empfohlen) |
| ESXi-Hosts | Mindestens 3 Hosts im Cluster |
| Netzwerk | NSX-T 3.0+ **oder** HAProxy als Load Balancer |
| Storage | vSAN **oder** NFS/iSCSI mit vSphere CSI |
| CPU | 4+ Kerne pro Host empfohlen |
| RAM | Mindestens 128 GB RAM pro Host (Produktion) |
| Lizenz | VCF (VMware Cloud Foundation) |

---

## Supervisor Cluster aktivieren

Der Supervisor Cluster ist der K8s-Control-Plane, der direkt auf den ESXi-Hosts läuft.

**GUI-Pfad:** `vCenter → Menü → Workload Management → Supervisor aktivieren`

1. **Cluster auswählen** — Workload Management → Get Started
2. **Netzwerk-Stack wählen** — NSX oder vSphere Networking mit HAProxy
3. **Storage-Policy auswählen** — vSAN Default Storage Policy oder Custom
4. **Management-Netzwerk konfigurieren** — IP-Bereich für Supervisor-Control-Plane-VMs (3 IPs)
5. **Workload-Netzwerk konfigurieren** — IP-Bereich für K8s-Nodes und Services
6. **Content Library verknüpfen** — für TKG-VM-Images
7. **Aktivierung starten** — dauert 15–30 Minuten

> [!NOTE]
> Nach der Aktivierung erscheinen 3 Control-Plane-VMs pro Cluster im vCenter-Inventar. Diese sind vom System verwaltet und dürfen nicht manuell verändert werden.

---

## vSphere Namespaces

Namespaces sind die Einheit der Ressourcen-Isolation in Tanzu — vergleichbar mit K8s-Namespaces, aber mit vSphere-Ressourcen-Policies verknüpft.

**GUI-Pfad:** `Workload Management → Namespaces → Namespace erstellen`

| Konfigurationsoption | Bedeutung |
|---|---|
| Speicher-Policy | Welche vSAN/Datastore-Policy für PVCs genutzt wird |
| VM-Klassen | CPU/RAM-Profile für TKG-Cluster-Nodes |
| Inhaltsquellen | Content Libraries für VM-Images |
| Berechtigungen | Welche SSO-Benutzer/Gruppen Zugriff haben |
| Ressourcen-Limits | CPU, RAM, Storage-Limits für den Namespace |

---

## TKG-Cluster deployen

Tanzu Kubernetes Grid-Cluster sind vollständige K8s-Cluster, die als VMs auf dem Supervisor-Cluster laufen.

### Login über kubectl

```bash
# kubectl vsphere Plugin installieren (wird vom Supervisor bereitgestellt)
# Download-URL im vCenter Workload Management UI

# Einloggen
kubectl vsphere login \
  --server=<supervisor-control-plane-ip> \
  --vsphere-username administrator@vsphere.local \
  --insecure-skip-tls-verify

# Context wechseln
kubectl config use-context <namespace-name>
```

### TKG-Cluster-YAML

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mein-tkg-cluster
  namespace: dev-namespace
spec:
  topology:
    class: tanzukubernetescluster
    version: v1.27.5+vmware.1-tkg.1
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
        - class: node-pool
          name: worker-pool
          replicas: 3
          variables:
            overrides:
              - name: vmClass
                value: best-effort-medium
              - name: storageClass
                value: vsan-default-storage-policy
```

```bash
# Cluster deployen
kubectl apply -f tkg-cluster.yaml

# Status prüfen
kubectl get clusters -n dev-namespace
kubectl get machines -n dev-namespace
```

---

## Workloads deployen

Nach dem Cluster-Deploy kann kubectl direkt gegen den TKG-Cluster arbeiten:

```bash
# Login zum TKG-Cluster
kubectl vsphere login \
  --server=<supervisor-ip> \
  --tanzu-kubernetes-cluster-name mein-tkg-cluster \
  --tanzu-kubernetes-cluster-namespace dev-namespace \
  --vsphere-username administrator@vsphere.local

# Standard K8s-Workflows
kubectl apply -f deployment.yaml
kubectl get pods -A

# PersistentVolume erstellen (via vSAN)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  storageClassName: vsan-default-storage-policy
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
EOF
```

---

## Monitoring

**vCenter Dashboard:** `Workload Management → Supervisor → Namespaces → Cluster`

| Monitoring-Bereich | Pfad/Befehl |
|---|---|
| Supervisor-Status | vCenter → Workload Management → Supervisors |
| TKG-Cluster-Status | `kubectl get clusters -A` |
| Node-Status | `kubectl get nodes` (im TKG-Context) |
| Pod-Status | `kubectl get pods -A` |
| Events | `kubectl get events -n <namespace>` |
| vSphere-Alarm | vCenter → Überwachung → Alarmierungen |

---

## Troubleshooting

| Problem | Ursache | Lösung |
|---|---|---|
| Supervisor-Aktivierung schlägt fehl | Netzwerk-Konfiguration fehlerhaft | NSX/HAProxy-Konfiguration prüfen |
| Control-Plane-VMs starten nicht | Zu wenig Ressourcen auf Hosts | Host-CPU/RAM prüfen |
| TKG-Cluster bleibt in „Provisioning" | Content Library nicht erreichbar | Library-Sync prüfen, Images vorhanden? |
| PVC bleibt in „Pending" | Storage-Policy nicht verfügbar | vSAN-Health, Policy-Name prüfen |
| kubectl vsphere login schlägt fehl | SSL-Zertifikat nicht vertrauenswürdig | `--insecure-skip-tls-verify` oder Zertifikat importieren |

> [!WARNING]
> Tanzu-Cluster-VMs (Control Plane + Worker Nodes) dürfen NICHT manuell in vCenter gelöscht oder verschoben werden. Änderungen nur über kubectl/Tanzu API.

---

## Verwandte Notizen

- [[ESXi Grundlagen]]
- [[vCenter]]
- [[NSX]]
- [[vSAN]]
- [[vSphere Lifecycle Manager]]
