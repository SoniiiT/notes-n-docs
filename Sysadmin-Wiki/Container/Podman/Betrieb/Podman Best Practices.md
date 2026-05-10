---
tags: [podman, best-practices, sicherheit, rootless, image, layer, secrets, production, empfehlungen]
aliases: ["Podman Best Practices", "Podman Empfehlungen", "Container Best Practices"]
---

# Podman Best Practices

> [!ABSTRACT]
> Gesammelte Empfehlungen für produktiven Podman-Einsatz: von der Image-Erstellung über Rootless-Deployment bis zu Secrets und Monitoring. Diese Best Practices sind das Destillat aus den anderen Podman-Dokumenten — eine Checkliste für Production-Ready Container.

---

## 1. Rootless-First

```bash
# IMMER rootless bevorzugen (kein sudo!)
podman run --name webserver -d nginx

# Root nur wenn wirklich nötig (z.B. CAP_NET_ADMIN für Netzwerk-Tools)
sudo podman run --cap-add CAP_NET_ADMIN network-tool

# lingering aktivieren damit Container nach Logout weiterläuft
sudo loginctl enable-linger $USER
```

> [!TIP]
> Rootless ist in Podman der Standard — nutze es. Root-Container haben Zugriff auf das Host-System; rootless Container sind durch User Namespaces isoliert.

---

## 2. Minimale Basis-Images

```dockerfile
# ❌ Schlecht — aufgeblähtes Basis-Image
FROM ubuntu:latest         # 80 MB compressed
FROM debian:latest         # 55 MB compressed

# ✅ Gut — minimal
FROM alpine:3.19           # 3 MB compressed
FROM debian:12-slim        # 22 MB compressed
FROM gcr.io/distroless/static:nonroot   # 2 MB, kein Shell!

# ✅ Für Sprachen: offizielle *-alpine oder *-slim Varianten
FROM node:20-alpine
FROM python:3.12-slim
FROM eclipse-temurin:21-jre-alpine
```

---

## 3. Vollständige Image-Referenzen

```bash
# ❌ Schlecht — löst interaktiven Prompt aus
podman pull nginx
podman run nginx

# ✅ Gut — vollständiger Registry-Pfad
podman pull docker.io/library/nginx:alpine
podman run docker.io/library/nginx:1.25-alpine

# In Containerfiles:
FROM docker.io/library/node:20-alpine  # vollständig
```

---

## 4. Nie als Root im Container laufen

```dockerfile
# ❌ Root im Container
FROM alpine
CMD ["nginx"]  # läuft als root

# ✅ Eigener User
FROM alpine
RUN adduser -D -u 1001 appuser
USER 1001
CMD ["myapp"]

# ✅ Oder UID direkt (portabler)
USER 65532   # nonroot bei distroless
```

```bash
# Laufzeit: User überschreiben
podman run --user 1001:1001 myapp
podman run --user nobody myapp
```

---

## 5. Keine Secrets in Images oder Umgebungsvariablen

```bash
# ❌ Passwort im Containerfile
RUN curl -u admin:geheim123 https://intern.de/pkg.tar.gz

# ❌ Passwort in Umgebungsvariable (sichtbar in podman inspect!)
podman run -e DB_PASSWORD=geheim123 myapp

# ✅ Podman Secrets
echo "geheim123" | podman secret create db-password -
podman run --secret db-password,type=env,target=DB_PASSWORD myapp

# ✅ Aus Datei lesen (Kubernetes-Kompatibel)
podman run --secret db-password,type=mount,target=/run/secrets/db myapp
```

---

## 6. Immer spezifische Tags — nie :latest

```dockerfile
# ❌ :latest — nicht reproduzierbar!
FROM nginx:latest

# ✅ Spezifischer Tag oder SHA
FROM docker.io/library/nginx:1.25-alpine
FROM docker.io/library/nginx@sha256:abc123...  # unveränderlich
```

```bash
# SHA eines Images ermitteln
podman image inspect nginx:alpine --format "{{.Digest}}"
# sha256:abc123...
```

---

## 7. Multi-Stage Builds für kleine Images

```dockerfile
# Builder-Stage (groß)
FROM golang:1.22 AS builder
WORKDIR /build
COPY . .
RUN go build -o app .

# Laufzeit-Stage (minimal)
FROM alpine:3.19
COPY --from=builder /build/app /usr/local/bin/app
CMD ["app"]
# Ergebnis: ~15 MB statt ~1,2 GB
```

---

## 8. Layer-Anzahl minimieren

```dockerfile
# ❌ Viele Layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN rm -rf /var/lib/apt/lists/*

# ✅ Ein Layer
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl wget \
 && rm -rf /var/lib/apt/lists/*
```

---

## 9. Health Checks definieren

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD curl -sf http://localhost:8080/health || exit 1
```

```bash
# Laufzeit
podman run \
  --health-cmd "pg_isready -U postgres" \
  --health-interval 10s \
  postgres:16
```

---

## 10. Read-Only Container

```bash
# Root-FS read-only → schreibende Angriffe sofort scheitern
podman run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /run:rw \
  -v mydata:/data:Z \
  myapp
```

---

## 11. Ressourcen begrenzen

```bash
# CPU und Memory immer begrenzen (verhindert Container-DoS)
podman run \
  --memory 512m \
  --memory-swap 512m \     # kein Swap
  --cpus 0.5 \
  myapp

# In Quadlet:
[Container]
PodmanArgs=--memory=512m --cpus=0.5
```

---

## 12. Netzwerk isolieren

```bash
# Eigenes Netzwerk statt Standard-Brücke
podman network create myapp-net

# Nur benötigte Verbindungen
podman run --network myapp-net --name db postgres:16
podman run --network myapp-net --name app myapp

# Internet deaktivieren für interne Services
podman network create --internal intern-netz
podman run --network intern-netz --name db postgres:16
```

---

## 13. .containerignore pflegen

```
node_modules/
.git/
.env
*.log
tests/
docs/
dist/
```

---

## 14. Systemd/Quadlet für Autostart

```ini
# ~/.config/containers/systemd/myapp.container
[Container]
Image=docker.io/myapp:1.0
AutoUpdate=registry
HealthCmd=curl -sf http://localhost:8080/health

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now myapp.service
```

---

## 15. Images regelmäßig aktualisieren

```bash
# Automatisches Update via Quadlet
[Container]
AutoUpdate=registry

# Timer aktivieren
systemctl --user enable --now podman-auto-update.timer

# Manuell
podman auto-update
```

---

## Checkliste für Production-Deployments

```
Containerfile:
☐ Vollständige Image-Referenzen (kein Short-Name)
☐ Spezifischer Tag (kein :latest)
☐ Multi-Stage Build
☐ Non-Root USER definiert
☐ HEALTHCHECK definiert
☐ .containerignore vorhanden
☐ Keine Secrets im Image

Laufzeit:
☐ Rootless (kein sudo)
☐ --read-only wenn möglich
☐ --cap-drop all + nur benötigte Capabilities
☐ Memory und CPU begrenzt
☐ Eigenes Netzwerk (kein default bridge)
☐ SELinux-Labels (:z/:Z) an allen Volumes
☐ Secrets via podman secret (nicht in ENV)

Betrieb:
☐ Quadlet-Service mit Restart=always
☐ loginctl enable-linger aktiviert
☐ AutoUpdate=registry konfiguriert
☐ Logging via journald
☐ Health-Monitoring eingerichtet
```

---

## Verwandte Notizen

- [[Containerfile]] — Containerfile nach Best Practices schreiben
- [[Rootless Container]] — Rootless im Detail
- [[Podman Sicherheit]] — Security-Features im Detail
- [[Systemd Integration]] — Quadlet-Deployment
