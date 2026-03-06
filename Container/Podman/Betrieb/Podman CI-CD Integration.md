---
tags: [podman, ci-cd, github-actions, gitlab-ci, buildah, pipeline, registry, automation, jenkins]
aliases: ["Podman CI/CD", "Podman Pipeline", "Buildah CI", "Container CI"]
---

# Podman CI-CD Integration

> [!ABSTRACT]
> Podman und Buildah eignen sich hervorragend für CI/CD-Pipelines: rootless Build ohne Docker-Daemon, OCI-kompatible Images, direkte Registry-Integration. GitHub Actions, GitLab CI und Jenkins können alle nativ mit Podman arbeiten.

---

## Vorteile von Podman/Buildah in CI/CD

| Merkmal | Docker in CI | Podman/Buildah in CI |
|---------|:----------:|:------------------:|
| Root nötig | ✅ (Docker-Daemon) | ❌ (rootless) |
| Daemon-Socket | ✅ (security concern) | ❌ |
| DinD (Docker-in-Docker) | ✅ Nötig für Pipelines | ❌ Nicht nötig |
| Rootless Builds | ❌ | ✅ |
| Parallelisierung | eingeschränkt | ✅ |
| Image-Signierung | extra Tool | ✅ integriert |

---

## GitHub Actions

### Workflow mit Podman

```yaml
# .github/workflows/build-and-push.yml
name: Build and Push Container Image

on:
  push:
    branches: [main]
    tags: ["v*"]
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

      - name: Install Podman
        run: |
          sudo apt-get update
          sudo apt-get install -y podman buildah skopeo

      - name: Anmelden an Registry
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
          podman login ghcr.io \
            -u "${{ github.actor }}" \
            --password-stdin

      - name: Image bauen
        id: build
        run: |
          podman build \
            --label "org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}" \
            --label "org.opencontainers.image.revision=${{ github.sha }}" \
            --label "org.opencontainers.image.created=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
            .

      - name: Image pushen
        if: github.event_name != 'pull_request'
        run: |
          podman push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          podman push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

### Mit Tests und Security-Scan

```yaml
  test:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - uses: actions/checkout@v4

      - name: Tests im Container ausführen
        run: |
          podman run --rm \
            -v $(pwd):/workspace:Z \
            -w /workspace \
            docker.io/library/python:3.12-slim \
            pip install -r requirements-dev.txt && pytest

      - name: Trivy Security-Scan
        run: |
          podman run --rm \
            -v /var/run/podman/podman.sock:/var/run/docker.sock:z \
            docker.io/aquasec/trivy:latest \
            image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

---

## GitLab CI/CD

### Mit Podman im Runner

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - push

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  IMAGE_LATEST: $CI_REGISTRY_IMAGE:latest

build:
  stage: build
  image: quay.io/podman/stable:latest
  script:
    - podman login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - podman build -t "$IMAGE_TAG" -t "$IMAGE_LATEST" .
    - podman push "$IMAGE_TAG"
  rules:
    - if: $CI_COMMIT_BRANCH

test:
  stage: test
  image: quay.io/podman/stable:latest
  script:
    - podman pull "$IMAGE_TAG"
    - podman run --rm "$IMAGE_TAG" /app/run-tests.sh
  rules:
    - if: $CI_COMMIT_BRANCH

push-latest:
  stage: push
  image: quay.io/podman/stable:latest
  script:
    - podman login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - skopeo copy \
        docker://"$IMAGE_TAG" \
        docker://"$IMAGE_LATEST"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

### GitLab mit Buildah (rootless, privileged: false)

```yaml
# .gitlab-ci.yml — vollständig rootless
build-rootless:
  stage: build
  image:
    name: quay.io/buildah/stable:latest
    entrypoint: [""]
  variables:
    BUILDAH_ISOLATION: chroot
    STORAGE_DRIVER: vfs
  script:
    - buildah login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - buildah build
        --storage-driver vfs
        -t "$IMAGE_TAG"
        .
    - buildah push --storage-driver vfs "$IMAGE_TAG"
  rules:
    - if: $CI_COMMIT_BRANCH
```

---

## Buildah in CI-Skripten

```bash
#!/bin/bash
# build-and-push.sh — Buildah-basiertes CI-Skript

set -euo pipefail

REGISTRY="${REGISTRY:-ghcr.io}"
IMAGE="${IMAGE:-myorg/myapp}"
TAG="${TAG:-$(git rev-parse --short HEAD)}"

echo "Building $REGISTRY/$IMAGE:$TAG..."

# Image bauen
buildah build \
  --label "git-commit=$TAG" \
  --label "build-date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
  -t "$REGISTRY/$IMAGE:$TAG" \
  .

# Tests ausführen
echo "Running tests..."
podman run --rm "$REGISTRY/$IMAGE:$TAG" /app/tests/run.sh

# Pushen
echo "Pushing $REGISTRY/$IMAGE:$TAG..."
buildah push "$REGISTRY/$IMAGE:$TAG"

# Latest taggen wenn main-Branch
if [ "${CI_COMMIT_BRANCH:-}" = "main" ]; then
  skopeo copy \
    "docker://$REGISTRY/$IMAGE:$TAG" \
    "docker://$REGISTRY/$IMAGE:latest"
fi

echo "Done!"
```

---

## Multi-Arch Builds in CI

```yaml
# .github/workflows/multi-arch.yml
name: Multi-Arch Build

on:
  push:
    branches: [main]

jobs:
  build-multi-arch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Podman
        run: sudo apt-get install -y podman qemu-user-static

      - name: Multi-Arch Build mit Manifest
        run: |
          # AMD64 bauen
          podman build --platform linux/amd64 \
            -t myapp:amd64 .

          # ARM64 bauen (via QEMU-Emulation)
          podman build --platform linux/arm64 \
            -t myapp:arm64 .

          # Manifest-Liste erstellen
          podman manifest create myapp:latest
          podman manifest add myapp:latest myapp:amd64
          podman manifest add myapp:latest myapp:arm64

      - name: Manifest pushen
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            podman login ghcr.io -u "${{ github.actor }}" --password-stdin

          podman manifest push \
            myapp:latest \
            ghcr.io/${{ github.repository }}:latest
```

---

## Caching in CI-Pipelines

```bash
# Registry als Cache-Backend
podman build \
  --cache-from ghcr.io/myorg/myapp:buildcache \
  --cache-to ghcr.io/myorg/myapp:buildcache \
  -t ghcr.io/myorg/myapp:latest \
  .

# Layer-Cache lokal (für selbst-gehostete Runner)
podman build --layers -t myapp:latest .
```

---

## Jenkins mit Podman

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        REGISTRY = 'registry.firma.de:5000'
        IMAGE = 'myapp'
        TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('Build') {
            steps {
                sh """
                    podman build \
                      -t ${REGISTRY}/${IMAGE}:${TAG} \
                      -t ${REGISTRY}/${IMAGE}:latest \
                      .
                """
            }
        }

        stage('Test') {
            steps {
                sh "podman run --rm ${REGISTRY}/${IMAGE}:${TAG} npm test"
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'registry-credentials',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS'
                )]) {
                    sh """
                        echo \$REGISTRY_PASS | podman login \
                          -u \$REGISTRY_USER \
                          --password-stdin \
                          ${REGISTRY}
                        podman push ${REGISTRY}/${IMAGE}:${TAG}
                        podman push ${REGISTRY}/${IMAGE}:latest
                    """
                }
            }
        }
    }

    post {
        always {
            sh "podman system prune -f"  // Aufräumen
        }
    }
}
```

---

## Sicherheit in CI/CD

```bash
# Image-Scanning vor dem Push
trivy image myapp:latest --exit-code 1 --severity CRITICAL

# Oder mit Grype
grype myapp:latest

# Image-Signierung (Cosign)
cosign sign --key cosign.key ghcr.io/myorg/myapp:latest

# Signature in CI verifizieren
cosign verify \
  --key cosign.pub \
  ghcr.io/myorg/myapp:latest
```

---

## Verwandte Notizen

- [[Podman Registries]] — Registry-Authentifizierung
- [[Containerfile]] — Optimierte Containerfiles für CI
- [[Multi-Stage Builds]] — Schnelle Build-Zeiten durch Caching
- [[Podman Best Practices]] — Security-Checkliste
