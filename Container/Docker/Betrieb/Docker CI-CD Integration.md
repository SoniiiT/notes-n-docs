---
tags:
  - docker
  - ci-cd
  - github-actions
  - gitlab-ci
  - automation
  - buildx
---

# Docker CI/CD Integration

> [!ABSTRACT] TL;DR
> Docker-Images automatisiert in CI/CD-Pipelines bauen, testen und deployen — mit GitHub Actions und GitLab CI. Dazu: Image-Tagging-Strategien, Registry-Authentifizierung, Build-Caching und automatisierte Image-Updates mit Dependabot/Renovate.

---

## Image-Tagging-Strategien

### Tagging-Konventionen

| Strategie | Beispiel | Verwendung |
|-----------|---------|------------|
| Semantic Versioning | `app:1.2.3` | Release-Builds |
| Git SHA (kurz) | `app:a1b2c3d` | Exakte Zuordnung zum Commit |
| Branch-Name | `app:main`, `app:develop` | Aktuelle Branch-Version |
| Datum | `app:2026-03-06` | Tägliche Builds |
| `latest` | `app:latest` | Nur für Entwicklung — **nie in Produktion** |
| Kombination | `app:1.2.3-a1b2c3d` | Version + Commit-SHA |

> [!WARNING] `:latest` nie in Produktion
> Das Tag `:latest` ist nicht automatisch die neueste Version — es ist einfach der Standard-Tag wenn keiner angegeben wird. In Produktion immer feste Tags verwenden, um reproduzierbare Deployments sicherzustellen.

### Empfohlenes Tagging-Schema

```bash
# Format: <registry>/<organisation>/<app>:<version>
ghcr.io/myorg/myapp:1.2.3         # Semantic Version
ghcr.io/myorg/myapp:1.2.3-a1b2c3d # Version + SHA
ghcr.io/myorg/myapp:main           # Branch (mutable)
ghcr.io/myorg/myapp:sha-a1b2c3d   # Commit SHA (immutable)
```

---

## GitHub Actions

### Einfacher Docker Build & Push

```yaml
# .github/workflows/docker-build.yml
name: Docker Build

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
            type=ref,event=branch

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### Build mit Layer-Caching

```yaml
# Optimiert mit GitHub Actions Cache
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

> [!TIP] `cache-from: type=gha`
> GitHub Actions Cache (`type=gha`) speichert Docker Build-Layer im GitHub Actions Cache. Bei nachfolgenden Builds werden unveränderte Layer aus dem Cache geladen — das beschleunigt Builds erheblich (oft 50–80% schneller).

### Multi-Architektur Build in GitHub Actions

```yaml
jobs:
  build-multiarch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push (Multi-Arch)
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Build mit Tests

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Test Image
        run: |
          docker build --target test -t myapp:test .
          docker run --rm myapp:test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and Push Production Image
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### Security Scanning mit Trivy

```yaml
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Image
        run: docker build -t myapp:scan .

      - name: Run Trivy Scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:scan
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy Results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## GitLab CI

### Einfacher Docker Build

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - deploy

variables:
  IMAGE_NAME: ${CI_REGISTRY_IMAGE}

build:
  stage: build
  image: docker:27
  services:
    - docker:27-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u ${CI_REGISTRY_USER} -p ${CI_REGISTRY_PASSWORD} ${CI_REGISTRY}
  script:
    - docker build
        --cache-from ${IMAGE_NAME}:latest
        -t ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}
        -t ${IMAGE_NAME}:latest
        .
    - docker push ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}
    - docker push ${IMAGE_NAME}:latest

test:
  stage: test
  image: ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}
  script:
    - npm test  # oder pytest, go test, etc.

scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL,HIGH
        ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}
  allow_failure: true
```

### Build mit Kaniko (ohne Docker-in-Docker)

```yaml
# Sicherer: Kaniko braucht keinen privilegierten Container
build-kaniko:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]
  script:
    - |
      /kaniko/executor \
        --context "${CI_PROJECT_DIR}" \
        --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
        --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}" \
        --destination "${CI_REGISTRY_IMAGE}:latest" \
        --cache=true \
        --cache-repo="${CI_REGISTRY_IMAGE}/cache"
```

> [!NOTE] Kaniko vs. Docker-in-Docker
> Kaniko baut Images **ohne Docker Daemon** — direkt im Container-Userspace. Das ist sicherer als Docker-in-Docker (DinD), da kein privilegierter Container benötigt wird. Ideal für Kubernetes-basierte CI/CD-Runner.

---

## Build-Caching-Strategien

### Lokaler Cache

```bash
# BuildKit mit lokalem Cache-Verzeichnis
docker buildx build \
    --cache-from type=local,src=/tmp/docker-cache \
    --cache-to type=local,dest=/tmp/docker-cache,mode=max \
    -t myapp:latest .
```

### Registry-basierter Cache

```bash
# Cache-Layer in einer separaten Registry-Referenz speichern
docker buildx build \
    --cache-from type=registry,ref=registry.example.com/myapp:cache \
    --cache-to type=registry,ref=registry.example.com/myapp:cache,mode=max \
    -t registry.example.com/myapp:1.0.0 \
    --push .
```

### Dependency-Caching im Dockerfile

```dockerfile
# Dependencies zuerst kopieren — ändert sich selten
COPY package.json package-lock.json ./
RUN npm ci

# Quellcode danach — ändert sich häufig
COPY src/ ./src/
RUN npm run build
```

> [!TIP] Layer-Reihenfolge = Cache-Effizienz
> Docker cached Layer von oben nach unten. Sobald sich ein Layer ändert, werden alle darunterliegenden Layer neu gebaut. Dependencies (selten geändert) daher **immer** vor dem Quellcode (häufig geändert) kopieren.

---

## Automatisierte Image-Updates

### Dependabot (GitHub)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "myorg/devops"
    labels:
      - "docker"
      - "dependencies"
```

### Renovate

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "docker": {
    "enabled": true,
    "pinDigests": true
  },
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["major"],
      "labels": ["docker-major"]
    }
  ]
}
```

---

## Registry-Authentifizierung

### Verschiedene Registries

```bash
# GitHub Container Registry (GHCR)
echo "${GITHUB_TOKEN}" | docker login ghcr.io -u USERNAME --password-stdin

# Docker Hub
echo "${DOCKERHUB_TOKEN}" | docker login -u USERNAME --password-stdin

# GitLab Container Registry
docker login registry.gitlab.com -u ${CI_REGISTRY_USER} -p ${CI_REGISTRY_PASSWORD}

# AWS ECR
aws ecr get-login-password --region eu-central-1 | \
    docker login --username AWS --password-stdin 123456789.dkr.ecr.eu-central-1.amazonaws.com

# Google Artifact Registry
gcloud auth configure-docker europe-west3-docker.pkg.dev
```

### Credentials sicher speichern

```bash
# Docker Credential Store (statt Klartext in ~/.docker/config.json)
# macOS:
docker-credential-osxkeychain store
# Linux:
docker-credential-pass store
# Windows:
docker-credential-wincred store
```

---

## Vollständige Pipeline-Vorlage

```yaml
# .github/workflows/complete-docker-pipeline.yml
name: Complete Docker Pipeline

on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-dockerfile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

  build-and-test:
    needs: lint-dockerfile
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Build Test Image
        uses: docker/build-push-action@v5
        with:
          context: .
          target: test
          load: true
          tags: ${{ env.IMAGE_NAME }}:test
          cache-from: type=gha

      - name: Run Tests
        run: docker run --rm ${{ env.IMAGE_NAME }}:test

  security-scan:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Build Image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true
          tags: ${{ env.IMAGE_NAME }}:scan

      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:scan
          exit-code: '1'
          severity: 'CRITICAL'

  publish:
    needs: [build-and-test, security-scan]
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/setup-qemu-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
            type=ref,event=branch

      - name: Build and Push (Multi-Arch)
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Verwandte Notizen

- [[Docker Images]] — Image-Konzepte und Layer-System
- [[Docker Registries]] — Registry-Typen und -Verwaltung
- [[Dockerfile]] — Dockerfile-Referenz
- [[Dockerfile Beispiele]] — Sprachspezifische Dockerfiles
- [[Docker Multi-Arch Builds]] — Multi-Architektur mit buildx
- [[Multi-Stage Builds]] — Build-Optimierung
- [[Docker Security und Hardening]] — Image-Scanning und Sicherheit
- [[Swarm CI-CD Integration]] — CI/CD für Docker Swarm
