---
tags: [podman, registry, skopeo, quay, docker-hub, registries.conf, image-transport, login, mirror]
aliases: ["Podman Registries", "Container Registry", "Skopeo", "Image Registry"]
---

# Podman Registries

> [!ABSTRACT]
> Podman kann mit jeder OCI-kompatiblen Registry arbeiten. Die Konfiguration erfolgt über `registries.conf`. **Skopeo** (ebenfalls aus dem Container-Tools-Ökosystem) ermöglicht erweiterte Registry-Operationen: Images zwischen Registries kopieren, Tags auflisten, Images inspizieren — ohne sie zu pullen.

---

## Populäre Registries

| Registry | URL | Beschreibung | Kosten |
|----------|-----|-------------|--------|
| Docker Hub | `docker.io` | Größte Registry, offizielle Images | Kostenlos (Rate-Limits) |
| Quay.io | `quay.io` | Red Hat, Sicherheits-Scans, OCI-nativ | Kostenlos |
| GitHub GHCR | `ghcr.io` | GitHub Container Registry | Kostenlos für öffentlich |
| Red Hat Registry | `registry.access.redhat.com` | Offizielle RHEL-Images, signiert | Kostenlos |
| Google GCR | `gcr.io` | Google Cloud, Distroless-Images | Kostenlos |
| Amazon ECR | `public.ecr.aws` | AWS public Images | Kostenlos |

---

## registries.conf — Konfiguration

```toml
# /etc/containers/registries.conf        (System, root)
# ~/.config/containers/registries.conf   (User, rootless)

# Standard-Suchreihenfolge bei Short-Names
unqualified-search-registries = ["registry.access.redhat.com", "docker.io"]

# Mirror für Docker Hub (Firmen-Proxy oder lokale Mirror)
[[registry]]
prefix = "docker.io"
insecure = false
blocked = false
location = "docker.io"

  [[registry.mirror]]
  location = "mirror.firma.de:5000"
  insecure = true

# Interne Registry (unsicher, HTTP)
[[registry]]
prefix = "intern.firma.de:5000"
insecure = true
location = "intern.firma.de:5000"

# Bestimmte Registry blockieren
[[registry]]
prefix = "untrusted.registry.example.com"
blocked = true
```

---

## Anmelden an Registries

```bash
# Docker Hub
podman login docker.io
podman login docker.io -u BENUTZERNAME -p PASSWORT

# Quay.io (empfohlen: mit Token statt Passwort)
podman login quay.io -u BENUTZERNAME --password-stdin < ~/.quay-token

# GitHub GHCR (Personal Access Token)
echo $GITHUB_TOKEN | podman login ghcr.io -u GITHUB_USERNAME --password-stdin

# Private Registry
podman login registry.firma.de:5000 -u admin -p passwort

# Mit TLS-Zertifikat-Problem (selbstsigniert)
podman login --tls-verify=false registry.intern.de:5000

# Anmeldestatus prüfen
podman login --get-login docker.io

# Abmelden
podman logout docker.io
podman logout --all   # Alle Registries
```

### Credentials speichern

```bash
# Standard: ~/.docker/config.json (kompatibel mit Docker CLI)
cat ~/.docker/config.json

# Oder: ~/.config/containers/auth.json
cat ~/.config/containers/auth.json

# Credentials-Helper installieren (sicherer als Plaintext)
# Fedora/RHEL:
sudo dnf install -y podman-docker containers-common
# credential-helper nutzen:
```

```json
// ~/.docker/config.json
{
  "credHelpers": {
    "docker.io": "pass",
    "quay.io": "pass"
  }
}
```

---

## Images pushen

```bash
# Image taggen für Registry
podman tag myapp:1.0 docker.io/USERNAME/myapp:1.0
podman tag myapp:1.0 docker.io/USERNAME/myapp:latest

# Pushen
podman push docker.io/USERNAME/myapp:1.0
podman push docker.io/USERNAME/myapp:latest

# Zu Quay.io
podman tag myapp:1.0 quay.io/USERNAME/myapp:1.0
podman push quay.io/USERNAME/myapp:1.0

# Zu GHCR
podman tag myapp:1.0 ghcr.io/ORG/myapp:1.0
podman push ghcr.io/ORG/myapp:1.0

# Zu interner Registry
podman push --tls-verify=false registry.intern.de:5000/myapp:1.0

# Manifest-Liste (Multi-Arch) pushen
podman manifest push myapp:multi-arch docker.io/USERNAME/myapp:latest
```

---

## Images suchen

```bash
# Suche in Standard-Registries
podman search nginx
podman search node --limit 10

# Nur in bestimmter Registry
podman search docker.io/nginx
podman search quay.io/prometheus/prometheus

# Filter
podman search nginx --filter stars=100
podman search nginx --filter is-official=true
podman search nginx --filter is-automated=false

# JSON-Output für Scripting
podman search nginx --format json | jq '.[].Name'
```

---

## Skopeo — Erweiterte Registry-Operationen

Skopeo ist das Spezialtool für Registry-Operationen ohne Images zu pullen.

```bash
# Installation
sudo dnf install -y skopeo    # Fedora/RHEL
sudo apt-get install -y skopeo  # Ubuntu/Debian
```

### Images inspizieren (ohne Download)

```bash
# Image-Manifest inspizieren
skopeo inspect docker://docker.io/library/nginx:latest

# Nur bestimmte Felder
skopeo inspect docker://docker.io/library/nginx:latest | jq .Architecture
skopeo inspect docker://docker.io/library/nginx:latest | jq .Layers
skopeo inspect docker://docker.io/library/nginx:latest | jq .Created

# OCI-Manifest direkt
skopeo inspect --raw docker://docker.io/library/nginx:latest | jq .
```

### Tags auflisten

```bash
# Alle Tags einer Image
skopeo list-tags docker://docker.io/library/nginx
skopeo list-tags docker://quay.io/prometheus/prometheus

# Neueste Tags (letzte 10)
skopeo list-tags docker://docker.io/library/node | jq '.Tags[-10:]'
```

### Images kopieren (ohne lokalen Download)

```bash
# Direkt zwischen Registries kopieren
skopeo copy \
  docker://docker.io/library/nginx:latest \
  docker://registry.intern.de:5000/nginx:latest

# Von Registry in lokales Verzeichnis (OCI-Layout)
skopeo copy \
  docker://docker.io/library/nginx:latest \
  oci:/tmp/nginx-oci:latest

# Von Podman-Storage in Registry
skopeo copy \
  containers-storage:myapp:latest \
  docker://quay.io/USERNAME/myapp:latest

# Multi-Arch kopieren
skopeo copy \
  --all \
  docker://docker.io/library/nginx:latest \
  docker://registry.intern.de:5000/nginx:latest
```

### Images synchronisieren (Mirror)

```bash
# Ganze Registry spiegeln
skopeo sync \
  --src docker \
  --dest docker \
  docker.io/library/nginx \
  registry.intern.de:5000/mirror/

# In lokales Verzeichnis spiegeln
skopeo sync \
  --src docker \
  --dest dir \
  docker.io/library/nginx \
  /data/mirror/
```

---

## Lokale Registry betreiben

```bash
# Registry-Container starten
podman run -d \
  --name registry \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry:Z \
  docker.io/library/registry:2

# Image pushen zur lokalen Registry
podman tag myapp:latest localhost:5000/myapp:latest
podman push --tls-verify=false localhost:5000/myapp:latest

# Image pullen
podman pull --tls-verify=false localhost:5000/myapp:latest

# Images in lokaler Registry auflisten
skopeo list-tags docker://localhost:5000/myapp
```

### Mit HTTPS und Authentifizierung

```bash
# Zertifikat erstellen
mkdir -p ~/registry/certs
openssl req -newkey rsa:4096 -nodes \
  -keyout ~/registry/certs/domain.key \
  -x509 -days 365 \
  -out ~/registry/certs/domain.crt \
  -subj "/CN=registry.intern.de"

# Registry mit HTTPS
podman run -d \
  --name registry \
  -p 5000:5000 \
  -v ~/registry/certs:/certs:Z \
  -v registry-data:/var/lib/registry:Z \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  docker.io/library/registry:2

# Zertifikat dem System hinzufügen (Fedora/RHEL)
sudo cp ~/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

---

## Image-Sicherheit mit Cosign

```bash
# Cosign installieren
sudo dnf install -y cosign

# Image signieren
cosign sign docker.io/USERNAME/myapp:latest

# Signatur prüfen
cosign verify docker.io/USERNAME/myapp:latest \
  --certificate-identity-regexp=".*" \
  --certificate-oidc-issuer-regexp=".*"
```

---

## Verwandte Notizen

- [[Container und Images]] — Image-Verwaltung lokal
- [[Containerfile]] — Images selbst bauen
- [[Podman Sicherheit]] — Image-Trust-Policies
- [[Podman CI-CD Integration]] — Registry in CI/CD-Pipelines
