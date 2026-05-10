---
tags:
  - azure
  - compute
  - container-registry
  - acr
  - docker
  - images
aliases:
  - ACR
  - Azure Container Registry ACR
  - Docker Registry Azure
---

# Azure Container Registry

> [!ABSTRACT] TL;DR
> Azure Container Registry (ACR) ist eine private, vollständig verwaltete Container-Registry für Docker- und OCI-kompatible Images. ACR integriert sich nativ mit AKS, App Service, Container Apps und Container Instances. Premium-SKU bietet Geo-Replikation, Private Link und Content Trust.

---

## Was ist ACR?

ACR ist Azures private Container Registry — vergleichbar mit Docker Hub, aber privat und vollständig in Azure integriert. ACR speichert Docker-Images, Helm-Charts und OCI-Artefakte.

**Kernfunktionen:**
- Private Docker-kompatible Registry
- Integration mit Entra ID und RBAC
- Geo-Replikation in mehrere Azure-Regionen (Premium)
- Integrierter Image-Build via ACR Tasks
- Vulnerability Scanning mit Microsoft Defender for Containers
- Webhooks für automatisierte Workflows

---

## SKUs

| SKU | Storage | Geo-Replikation | Private Link | Content Trust | Webhooks | Anwendungsfall |
|-----|---------|----------------|--------------|---------------|---------|----------------|
| **Basic** | 10 GiB | Nein | Nein | Nein | 2 | Dev/Test, Lernumgebungen |
| **Standard** | 100 GiB | Nein | Nein | Nein | 10 | Kleine bis mittlere Produktion |
| **Premium** | 500 GiB | Ja | Ja | Ja | 500 | Enterprise, Multi-Region, Compliance |

> [!TIP] Premium für Produktion
> In Produktionsumgebungen ist **Premium** empfohlen: Private Link verhindert öffentlichen Zugriff, Geo-Replikation reduziert Pull-Latenz für Multi-Region-Deployments.

---

## Image Push und Pull

### Docker CLI

```bash
# Bei ACR anmelden (via Azure CLI — empfohlen)
az acr login --name myregistry

# Image taggen und pushen
docker build -t myapp:v1.0.0 .
docker tag myapp:v1.0.0 myregistry.azurecr.io/myapp:v1.0.0
docker push myregistry.azurecr.io/myapp:v1.0.0

# Image pullen
docker pull myregistry.azurecr.io/myapp:v1.0.0
```

### az acr Import (von anderen Registries)

```bash
# Image von Docker Hub in ACR importieren
az acr import \
  --name myregistry \
  --source docker.io/library/nginx:1.25-alpine \
  --image nginx:1.25-alpine

# Image aus einer anderen ACR importieren
az acr import \
  --name myregistry-target \
  --source myregistry-source.azurecr.io/myapp:v1.0.0 \
  --image myapp:v1.0.0 \
  --registry /subscriptions/{subId}/resourceGroups/rg-source/providers/Microsoft.ContainerRegistry/registries/myregistry-source

# Image von GitHub Container Registry importieren
az acr import \
  --name myregistry \
  --source ghcr.io/myorg/myapp:latest \
  --image myapp:latest \
  --username $GITHUB_USER \
  --password $GITHUB_TOKEN
```

---

## Geo-Replikation (Premium)

Repliziert Images in mehrere Azure-Regionen — Client-Pulls werden automatisch zur nächsten Replica geleitet.

```bash
# Replikation in weitere Region hinzufügen
az acr replication create \
  --registry myregistry \
  --location eastus \
  --resource-group rg-acr

# Alle Replikationen anzeigen
az acr replication list \
  --registry myregistry \
  --output table
```

**Vorteile der Geo-Replikation:**
- AKS in East US und West Europe pullen aus der jeweiligen lokalen Replica
- Keine Egress-Kosten für Pulls innerhalb der Region
- Resilient gegen regionalen ACR-Ausfall

---

## Webhooks

Webhooks senden HTTP-Callbacks bei bestimmten ACR-Ereignissen.

```bash
# Webhook für Push-Events erstellen
az acr webhook create \
  --registry myregistry \
  --name wh-deploy-trigger \
  --uri "https://mycicd.example.internal/hooks/acr-push" \
  --actions push \
  --scope myapp:*  # Nur für myapp-Image

# Webhook testen
az acr webhook ping \
  --registry myregistry \
  --name wh-deploy-trigger

# Webhook-Events anzeigen
az acr webhook list-events \
  --registry myregistry \
  --name wh-deploy-trigger \
  --output table
```

---

## ACR Tasks

ACR Tasks ermöglichen Image-Builds direkt in ACR — kein lokaler Docker-Daemon nötig.

### Quick Task (ad-hoc Build)

```bash
# Image in ACR bauen und direkt pushen (einmalig)
az acr build \
  --registry myregistry \
  --image myapp:v1.0.0 \
  --file Dockerfile \
  .

# Image aus GitHub-Repository bauen
az acr build \
  --registry myregistry \
  --image myapp:latest \
  https://github.com/myorg/myrepo.git
```

### Automatisierter Task (Git-Commit-Trigger)

```bash
# Task erstellen: Build bei jedem Git-Push auf main
az acr task create \
  --registry myregistry \
  --name task-build-myapp \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/myorg/myrepo.git#main \
  --file Dockerfile \
  --git-access-token $GITHUB_TOKEN

# Task manuell ausführen
az acr task run \
  --registry myregistry \
  --name task-build-myapp

# Task-Logs anzeigen
az acr task logs \
  --registry myregistry \
  --name task-build-myapp
```

### Base Image Update Task

```bash
# Task erstellen: Automatisch neu bauen wenn Base-Image updated wird
az acr task create \
  --registry myregistry \
  --name task-auto-rebuild \
  --image myapp:latest \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --base-image-trigger-enabled true \
  --git-access-token $GITHUB_TOKEN
```

---

## Image-Scanning mit Microsoft Defender

```bash
# Defender for Containers auf ACR aktivieren
az security pricing create \
  --name ContainerRegistry \
  --tier Standard

# Scan-Ergebnisse anzeigen
az security sub-assessment list \
  --resource-id $(az acr show --name myregistry --query id -o tsv) \
  --query "[?contains(id, 'containerRegistry')]" \
  --output table
```

**Scan-Ergebnisse im Portal:**
- Azure Portal → Microsoft Defender for Cloud → Recommendations → Container image vulnerability findings

---

## Content Trust (Image Signing)

```bash
# Content Trust aktivieren (nur Premium)
az acr config content-trust update \
  --registry myregistry \
  --status enabled

# Image signieren (via Docker Notary)
export DOCKER_CONTENT_TRUST=1
export DOCKER_CONTENT_TRUST_SERVER=https://myregistry.azurecr.io

docker push myregistry.azurecr.io/myapp:v1.0.0
# → Signatur wird automatisch erstellt und in ACR gespeichert

# Nur signierte Images erlauben (via Azure Policy)
# Policy: "Container registries should have image signature enabled"
```

---

## ACR + Managed Identity

### AKS AcrPull-Rolle

```bash
# AKS berechtigen, Images ohne imagePullSecret zu pullen
az aks update \
  --resource-group rg-aks \
  --name aks-prod \
  --attach-acr myregistry

# Prüfen ob Rolle zugewiesen ist
az role assignment list \
  --scope $(az acr show --name myregistry --query id -o tsv) \
  --query "[?roleDefinitionName=='AcrPull']" \
  --output table
```

### App Service + ACR

```bash
# System-assigned Identity für App Service aktivieren
az webapp identity assign \
  --resource-group rg-prod \
  --name myapp-prod

# AcrPull-Rolle zuweisen
WEBAPP_PRINCIPAL=$(az webapp identity show -g rg-prod -n myapp-prod --query principalId -o tsv)
ACR_ID=$(az acr show --name myregistry --query id -o tsv)
az role assignment create \
  --assignee $WEBAPP_PRINCIPAL \
  --role AcrPull \
  --scope $ACR_ID

# App Service auf Container-Deployment umstellen
az webapp config container set \
  --resource-group rg-prod \
  --name myapp-prod \
  --docker-custom-image-name myregistry.azurecr.io/webapp:latest \
  --docker-registry-server-url https://myregistry.azurecr.io
```

---

## Private Endpoint (Premium)

```bash
# Private Endpoint für ACR erstellen
az network private-endpoint create \
  --resource-group rg-prod \
  --name pe-acr-prod \
  --vnet-name vnet-prod \
  --subnet snet-private-endpoints \
  --private-connection-resource-id $(az acr show --name myregistry --query id -o tsv) \
  --group-id registry \
  --connection-name conn-acr-prod

# DNS-Eintrag für Private Endpoint erstellen
az network private-dns zone create \
  --resource-group rg-prod \
  --name "privatelink.azurecr.io"

az network private-dns link vnet create \
  --resource-group rg-prod \
  --zone-name "privatelink.azurecr.io" \
  --name dns-link-vnet-prod \
  --virtual-network vnet-prod \
  --registration-enabled false

# Öffentlichen Zugriff deaktivieren (nur Private Link)
az acr update \
  --name myregistry \
  --public-network-enabled false
```

---

## Token-basierter Zugriff (Scope Maps)

Granulare Pull/Push-Berechtigungen ohne Entra ID-Identitäten — ideal für CI/CD-Systeme.

```bash
# Scope Map erstellen (nur Pull für production-Images)
az acr scope-map create \
  --registry myregistry \
  --name sm-readonly-production \
  --repository myapp content/read metadata/read

# Token mit Scope Map erstellen
az acr token create \
  --registry myregistry \
  --name token-ci-readonly \
  --scope-map sm-readonly-production

# Token-Passwort generieren
az acr token credential generate \
  --registry myregistry \
  --name token-ci-readonly \
  --password1
```

---

## ACR Purge: Automatisches Löschen alter Images

```bash
# Sofortiges Löschen (acr run-basierter Purge-Task)
az acr run \
  --registry myregistry \
  --cmd "acr purge \
    --filter 'myapp:.*' \
    --untagged \
    --ago 30d" \
  /dev/null

# Geplanter Purge-Task (täglich um 01:00)
az acr task create \
  --name acr-purge-task \
  --registry myregistry \
  --schedule "0 1 * * *" \
  --context /dev/null \
  --cmd "acr purge \
    --filter 'myapp:.*' \
    --untagged \
    --ago 30d \
    --keep 10" \
  --timeout 3600
```

> [!WARNING] Untagged Images
> Images ohne Tags belegen Storage, werden aber von `docker images` oder dem ACR-Portal nicht immer angezeigt. Regelmäßige Purge-Tasks sind wichtig um Storage-Kosten zu kontrollieren.

---

## Quarantine Policy (Preview)

```bash
# Quarantine Policy aktivieren (nur Premium)
az acr config content-trust update \
  --registry myregistry \
  --status enabled

# Images werden nach Push in Quarantäne gestellt
# Erst nach bestandenem Scan werden sie freigegeben
# Freigegebene Images haben das Tag "quarantine/passed"
```

---

## Kosten

| Komponente | Basic | Standard | Premium |
|-----------|-------|----------|---------|
| **Registry** (monatlich) | ~5 € | ~20 € | ~55 € |
| **Storage** | ~0,10 €/GiB | ~0,10 €/GiB | ~0,10 €/GiB |
| **Build-Minuten** | ~0,0001 €/s | ~0,0001 €/s | ~0,0001 €/s |
| **Geo-Replication** | - | - | ~55 € pro Replica |
| **Webhooks** | 2 inkl. | 10 inkl. | 500 inkl. |

---

## Via Azure Portal (UI)

> [!TIP] Schnelleinstieg
> Das Portal ermöglicht die Verwaltung von Registrierungen, Repositories und Zugriffsrichtlinien ohne CLI.

### Container Registry erstellen

1. **portal.azure.com** → **Container Registries** → **+ Erstellen**
2. **Grundlagen**-Tab:
   - Abonnement & Ressourcengruppe
   - **Registrierungsname**: 5–50 Zeichen (global eindeutig, z. B. `meinregistry`)
   - **Region**: z. B. `West Europe`
   - **SKU**: Basic, Standard oder Premium (für Geo-Replikation & Private Endpoints)
3. **Überprüfen + erstellen** → **Erstellen**

### Images und Repositories verwalten

1. Container Registry → **Repositories**
2. Repository auswählen → Tags und Manifeste einsehen
3. Tag auswählen → **Löschen** oder **In ACR-Tasks verwenden**

### Zugriffssteuerung

1. Container Registry → **Zugriffssteuerung (IAM)**
2. **+ Rollenzuweisung hinzufügen**: `AcrPush` für CI/CD-Dienste, `AcrPull` für AKS/Container Apps

### Geo-Replikation (Premium)

1. Container Registry → **Replikationen** → **+ Hinzufügen**
2. Zielregion auswählen → **Erstellen**

### ACR Tasks (Build-Pipeline)

1. Container Registry → **Tasks** → **+ Hinzufügen**
2. **Schnellaufgabe** oder **Aufgabe mit Trigger** konfigurieren
3. Quell-Repository (GitHub), Dockerfile-Pfad und Basis-Image-Trigger festlegen

---

## Vollständige az acr Befehlsreferenz

```bash
# Registry erstellen
az acr create \
  --resource-group rg-prod \
  --name myregistry \
  --sku Premium \
  --location westeurope \
  --admin-enabled false

# Repository-Liste anzeigen
az acr repository list --name myregistry --output table

# Image-Tags anzeigen
az acr repository show-tags \
  --name myregistry \
  --repository myapp \
  --orderby time_desc \
  --output table

# Image-Manifest anzeigen
az acr manifest list-metadata \
  --registry myregistry \
  --name myapp

# Tag entfernen (Image bleibt als untagged)
az acr repository delete \
  --name myregistry \
  --image myapp:old-tag \
  --yes

# Gesamten Repository löschen
az acr repository delete \
  --name myregistry \
  --repository myapp-deprecated \
  --yes

# ACR-Diagnose ausführen
az acr check-health \
  --name myregistry \
  --ignore-errors \
  --yes
```

---

## Verwandte Notizen
- [[Azure Kubernetes Service]]
- [[Azure Container Apps]]
- [[Azure Container Instances]]
- [[Azure App Service]]
- [[Azure RBAC]]
