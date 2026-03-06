---
tags:
  - docker
  - dockerfile
  - beispiele
  - multi-stage
  - praxis
---

# Dockerfile Beispiele

> [!ABSTRACT] TL;DR
> Produktionsreife Dockerfile-Beispiele für die gängigsten Programmiersprachen und Frameworks — jeweils mit Multi-Stage Build, minimaler Image-Größe, Non-Root-User und Health Check. Alle Beispiele sind direkt verwendbar und folgen aktuellen Best Practices.

---

## Allgemeine Prinzipien

Jedes produktionsreife Dockerfile sollte diese Elemente enthalten:

| Prinzip | Umsetzung |
|---------|-----------|
| Multi-Stage Build | Build-Dependencies nicht im finalen Image |
| Minimales Base Image | `alpine` oder `distroless` wo möglich |
| Non-Root User | `USER` Direktive mit eigenem Benutzer |
| `.dockerignore` | Unnötige Dateien ausschließen |
| Health Check | `HEALTHCHECK` für Container-Überwachung |
| Feste Versionen | Base Image und Dependency-Versionen pinnen |
| Layer-Optimierung | Abhängigkeiten vor Quellcode kopieren |

---

## Node.js / TypeScript

### Express / Fastify API

````dockerfile
# --- Build Stage ---
FROM node:22-alpine AS build

WORKDIR /app

# Dependencies zuerst (Cache-Optimierung)
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

# TypeScript kompilieren
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# Nur Production-Dependencies
RUN npm ci --omit=dev --ignore-scripts

# --- Production Stage ---
FROM node:22-alpine AS production

RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -q --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
````

> [!TIP] `tini` als Init-System
> `tini` sorgt für korrekte Signal-Weiterleitung (SIGTERM) und Zombie-Process-Reaping. Ohne Init-System werden Signale nicht korrekt an die Node.js-Anwendung weitergeleitet.

### `.dockerignore` für Node.js

```text
node_modules
dist
.git
.github
*.md
.env*
.vscode
coverage
.nyc_output
Dockerfile*
docker-compose*
```

---

## Python

### FastAPI / Flask

````dockerfile
# --- Build Stage ---
FROM python:3.12-slim AS build

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt ./
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# --- Production Stage ---
FROM python:3.12-slim AS production

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

COPY --from=build /install /usr/local
COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
````

### Django mit Gunicorn

````dockerfile
FROM python:3.12-slim AS build

WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim AS production

RUN groupadd -r django && useradd -r -g django django

WORKDIR /app

COPY --from=build /install /usr/local
COPY --chown=django:django . .

RUN python manage.py collectstatic --noinput

USER django

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')" || exit 1

CMD ["gunicorn", "config.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--timeout", "120"]
````

---

## Go

### Go API (minimales Image)

````dockerfile
# --- Build Stage ---
FROM golang:1.23-alpine AS build

WORKDIR /app

# Dependencies zuerst (Cache)
COPY go.mod go.sum ./
RUN go mod download

# Kompilieren — statisch gelinkt
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -o /app/server \
    ./cmd/server

# --- Production Stage ---
FROM gcr.io/distroless/static-debian12 AS production

COPY --from=build /app/server /server

EXPOSE 8080

USER nonroot:nonroot

ENTRYPOINT ["/server"]
````

> [!NOTE] Distroless für Go
> Go-Binaries sind statisch gelinkt und benötigen keine Runtime. `distroless/static` enthält nur CA-Zertifikate und `/etc/passwd` — das Image ist nur ~2 MB groß. Für Debugging kann `distroless/static:debug` mit einer Shell verwendet werden.

### Alternative: Scratch-Image

````dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-w -s" -o /server ./cmd/server

# Leerstes Base Image — kleinstmögliches Image
FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
````

---

## Java

### Spring Boot mit Maven

````dockerfile
# --- Build Stage ---
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# Maven-Wrapper und pom.xml zuerst (Cache)
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B

# Quellcode kompilieren
COPY src ./src
RUN ./mvnw package -DskipTests -B

# JAR extrahieren für Layer-Optimierung
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /extracted

# --- Production Stage ---
FROM eclipse-temurin:21-jre-alpine AS production

RUN addgroup -S spring && adduser -S spring -G spring

WORKDIR /app

# Spring Boot Layer einzeln kopieren (besseres Caching)
COPY --from=build --chown=spring:spring /extracted/dependencies/ ./
COPY --from=build --chown=spring:spring /extracted/spring-boot-loader/ ./
COPY --from=build --chown=spring:spring /extracted/snapshot-dependencies/ ./
COPY --from=build --chown=spring:spring /extracted/application/ ./

USER spring

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
````

> [!TIP] Spring Boot Layer-Extraktion
> Spring Boot 3.x unterstützt `layertools` für optimales Docker-Caching. Dependencies (selten geändert) und Application-Code (häufig geändert) werden in separate Layer aufgeteilt.

### Gradle-Variante

````dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts ./
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon
COPY src ./src
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine AS production
RUN addgroup -S spring && adduser -S spring -G spring
WORKDIR /app
COPY --from=build --chown=spring:spring /app/build/libs/*.jar app.jar
USER spring
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
````

---

## .NET

### ASP.NET Core API

````dockerfile
# --- Build Stage ---
FROM mcr.microsoft.com/dotnet/sdk:9.0-alpine AS build

WORKDIR /src

# Restore zuerst (Cache)
COPY *.csproj ./
RUN dotnet restore

# Kompilieren und publishen
COPY . .
RUN dotnet publish -c Release -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# --- Production Stage ---
FROM mcr.microsoft.com/dotnet/aspnet:9.0-alpine AS production

RUN addgroup -S dotnet && adduser -S dotnet -G dotnet

WORKDIR /app

COPY --from=build --chown=dotnet:dotnet /app/publish .

USER dotnet

EXPOSE 8080

ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -q --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
````

---

## Rust

### Rust Web-Service

````dockerfile
# --- Build Stage ---
FROM rust:1.80-alpine AS build

RUN apk add --no-cache musl-dev

WORKDIR /app

# Dependency-Cache: leeres Projekt mit Cargo.toml
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src

# Tatsächlichen Quellcode kompilieren
COPY src ./src
RUN touch src/main.rs && cargo build --release

# --- Production Stage ---
FROM alpine:3.20 AS production

RUN apk add --no-cache ca-certificates tini
RUN addgroup -S app && adduser -S app -G app

COPY --from=build /app/target/release/myapp /usr/local/bin/myapp

USER app

EXPOSE 8080

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["myapp"]
````

---

## Nginx (Static Sites / SPA)

### React / Vue / Angular SPA

````dockerfile
# --- Build Stage ---
FROM node:22-alpine AS build

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# --- Production Stage ---
FROM nginx:1.27-alpine AS production

# Eigene Nginx-Konfiguration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Build-Artefakte kopieren
COPY --from=build /app/dist /usr/share/nginx/html

# Non-Root Nginx
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown nginx:nginx /var/run/nginx.pid

USER nginx

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -q --spider http://localhost:8080/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
````

### Nginx-Konfiguration für SPA

```nginx
# nginx.conf
server {
    listen 8080;
    root /usr/share/nginx/html;
    index index.html;

    # SPA Routing — alle Routen auf index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Statische Assets cachen
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Kein Caching für index.html
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }
}
```

---

## Image-Größen-Vergleich

| Sprache | Naive Variante | Multi-Stage | Optimiert |
|---------|---------------|-------------|-----------|
| Node.js | ~1.1 GB | ~180 MB | ~120 MB (alpine) |
| Python | ~900 MB | ~150 MB | ~80 MB (slim) |
| Go | ~800 MB | ~15 MB | ~2 MB (scratch) |
| Java | ~700 MB | ~300 MB | ~200 MB (jre-alpine) |
| .NET | ~600 MB | ~200 MB | ~120 MB (alpine) |
| Rust | ~1.5 GB | ~20 MB | ~10 MB (alpine) |
| Nginx SPA | ~200 MB | ~45 MB | ~25 MB (alpine) |

---

## Verwandte Notizen

- [[Dockerfile]] — Dockerfile-Referenz und Direktiven
- [[Multi-Stage Builds]] — Multi-Stage Build Konzepte
- [[Docker Images]] — Image-Verwaltung und -Optimierung
- [[Docker Best Practices]] — Produktions-Checkliste
- [[Docker CI-CD Integration]] — Images in CI/CD-Pipelines bauen
- [[Docker Multi-Arch Builds]] — Multi-Architektur-Builds mit buildx
- [[Docker Compose Beispiele]] — Compose-basierte Beispiele
