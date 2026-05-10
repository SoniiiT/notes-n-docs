---
tags:
  - github
  - pages
  - hosting
  - static-site
  - web
aliases:
  - Pages
  - GitHub Static Hosting
---

# GitHub Pages

> [!ABSTRACT]
> GitHub Pages ist ein kostenloses statisches Web-Hosting direkt aus einem GitHub-Repository heraus. Mit optionaler Jekyll-Integration, Custom Domains und automatischem HTTPS via Let's Encrypt eignet es sich ideal für Projektdokumentation, Portfolios und Blogs.

---

## Was ist GitHub Pages?

GitHub Pages veröffentlicht statische Websites direkt aus einem GitHub-Repository. Unterstützte Inhalte sind HTML, CSS, JavaScript und Markdown — serverseitiger Code (PHP, Python, Node.js) wird **nicht** ausgeführt.

**Typische Einsatzbereiche:**

- Persönliche oder Organisations-Websites
- Projektdokumentation (Open Source)
- Technische Blogs mit Jekyll
- Portfolios für Entwickler und Designer
- Landing Pages für Software-Produkte

GitHub Pages ist für öffentliche Repositories mit dem Free-Plan vollständig kostenlos. Private Repositories benötigen mindestens GitHub Pro.

---

## Hosting-Typen

### User Site / Organization Site

Eine User Site wird aus einem Repository erstellt, dessen Name exakt dem Muster `username.github.io` entspricht.

| Eigenschaft      | Details                          |
|------------------|----------------------------------|
| Repository-Name  | `username.github.io` (exakt)     |
| URL              | `https://username.github.io`     |
| Standard-Branch  | `main` oder `master`             |
| Limit            | 1 Site pro Account               |
| Quelle           | Root oder `/docs`-Ordner         |

**Beispiel:** Nutzer `johndoe` erstellt das Repository `johndoe.github.io` → Website erreichbar unter `https://johndoe.github.io`.

Organization Sites funktionieren identisch: `myorg.github.io` → `https://myorg.github.io`.

### Project Site

Eine Project Site wird aus einem beliebigen Repository veröffentlicht. Der Inhalt wird als Unterverzeichnis der User/Org-Site bereitgestellt.

| Eigenschaft      | Details                                      |
|------------------|----------------------------------------------|
| Repository-Name  | Beliebig                                     |
| URL              | `https://username.github.io/repo-name`       |
| Branch-Optionen  | `main`, `gh-pages`, `/docs`-Ordner           |
| Limit            | Kein Limit pro Account                        |
| Konfiguration    | Repository Settings → Pages                  |

> [!NOTE]
> Der `gh-pages`-Branch ist eine historische Konvention. Moderne Projekte verwenden bevorzugt GitHub Actions, um aus `main` zu deployen.

---

## Jekyll-Integration

GitHub Pages baut Jekyll-Sites automatisch — ohne CI/CD-Aufwand. Jekyll ist ein Ruby-basierter Static Site Generator, der Markdown-Dateien in vollständige HTML-Websites umwandelt.

### Projektstruktur

```
my-site/
├── _config.yml          # Globale Konfiguration
├── _posts/              # Blog-Posts (Format: YYYY-MM-DD-titel.md)
├── _layouts/            # HTML-Layout-Templates
├── _includes/           # Wiederverwendbare HTML-Fragmente
├── _data/               # YAML/JSON-Datendateien
├── assets/
│   ├── css/
│   ├── js/
│   └── images/
├── index.md             # Startseite
└── about.md             # Weitere Seiten
```

### _config.yml Beispiel

```yaml
# Grundlegende Konfiguration
title: "Meine GitHub Pages Site"
description: "Technische Dokumentation und Blog"
url: "https://johndoe.github.io"
baseurl: ""

# Theme-Konfiguration
theme: minima

# Autor-Informationen
author:
  name: "John Doe"
  email: "john@example.com"
  github: "johndoe"

# Whitelisted Plugins (nur diese werden von GitHub Pages gebaut)
plugins:
  - jekyll-feed
  - jekyll-sitemap
  - jekyll-seo-tag
  - jekyll-relative-links

# Build-Einstellungen
markdown: kramdown
highlighter: rouge

kramdown:
  input: GFM
  syntax_highlighter: rouge

# Kollektionen
collections:
  docs:
    output: true
    permalink: /docs/:name/

# Standardwerte für Frontmatter
defaults:
  - scope:
      path: "_posts"
      type: "posts"
    values:
      layout: "post"
      author: "John Doe"

# Ausgeschlossene Dateien
exclude:
  - Gemfile
  - Gemfile.lock
  - node_modules/
  - vendor/
  - README.md
  - .github/
```

### Unterstützte Jekyll Themes

| Theme        | Beschreibung                         | `remote_theme`-Name                   |
|--------------|--------------------------------------|---------------------------------------|
| Minima       | Standard — einfach, Blog-fokussiert  | `jekyll/minima`                       |
| Cayman       | Projektdoku — grüner Header          | `pages-themes/cayman@v0.2.0`          |
| Slate        | Dunkel — technische Doku             | `pages-themes/slate@v0.2.0`           |
| Architect    | Klassisch — blaue Seitenleiste       | `pages-themes/architect@v0.2.0`       |
| Midnight     | Dunkel — Portfolios                  | `pages-themes/midnight@v0.2.0`        |
| Time Machine | Retro — Blog-Stil                    | `pages-themes/time-machine@v0.2.0`    |
| Hacker       | Terminal-Look — technische Projekte  | `pages-themes/hacker@v0.2.0`          |
| Dinky        | Kompakt — kurze Dokumentationen      | `pages-themes/dinky@v0.2.0`           |

**Beliebige Jekyll Themes via `remote_theme`:**

```yaml
# _config.yml — jedes öffentliche Jekyll-Theme nutzen
remote_theme: mmistakes/minimal-mistakes@4.26.2

plugins:
  - jekyll-remote-theme
  - jekyll-feed
  - jekyll-sitemap
```

> [!TIP]
> `remote_theme` erlaubt die Nutzung **jedes** öffentlichen Jekyll-Themes auf GitHub — nicht nur die acht offiziell unterstützten. Das Plugin `jekyll-remote-theme` muss in den Plugins gelistet sein.

---

## Custom Domains

### CNAME-Datei

Im Repository-Root eine Datei namens `CNAME` anlegen (kein `.txt`-Suffix — exakt `CNAME`):

```
www.meinedomain.de
```

> [!IMPORTANT]
> Die CNAME-Datei darf nur **eine einzige Domain** enthalten. Wird die Datei via GitHub Actions deployed, muss sie im Build-Output enthalten sein — sonst überschreibt GitHub Pages sie.

### DNS-Konfiguration: Apex Domain

Für `meinedomain.de` (ohne `www`) werden A- und AAAA-Records verwendet:

| Typ    | Host | Wert                      |
|--------|------|---------------------------|
| A      | @    | `185.199.108.153`         |
| A      | @    | `185.199.109.153`         |
| A      | @    | `185.199.110.153`         |
| A      | @    | `185.199.111.153`         |
| AAAA   | @    | `2606:50c0:8000::153`     |
| AAAA   | @    | `2606:50c0:8001::153`     |
| AAAA   | @    | `2606:50c0:8002::153`     |
| AAAA   | @    | `2606:50c0:8003::153`     |

> [!WARNING]
> Bei Apex Domains (`meinedomain.de`) **keinen** CNAME-Record verwenden — das ist DNS-technisch ungültig (CNAME darf nicht auf Root-Ebene neben anderen Records existieren). Ausschließlich A- und AAAA-Records nutzen.

### DNS-Konfiguration: Subdomain

Für `www.meinedomain.de` wird ein CNAME-Record gesetzt:

| Typ   | Host | Wert                    |
|-------|------|-------------------------|
| CNAME | www  | `username.github.io`    |

### HTTPS via Let's Encrypt

1. DNS-Records korrekt setzen (DNS-Propagation: 24–48 Stunden einplanen)
2. In **Repository Settings → Pages → Custom domain** die Domain eintragen
3. GitHub prüft die DNS-Konfiguration automatisch
4. Checkbox **Enforce HTTPS** aktivieren (erscheint nach erfolgreicher Prüfung)

> [!NOTE]
> GitHub erneuert das Let's Encrypt-Zertifikat automatisch alle 90 Tage. Kein manueller Eingriff notwendig. Bei DNS-Änderungen kann die Zertifikat-Ausstellung einige Stunden dauern.

---

## GitHub Actions für Pages Deployment

### Warum GitHub Actions?

Die native Jekyll-Methode unterliegt Einschränkungen:

- Nur **whitelisted Jekyll-Plugins** werden unterstützt
- Keine anderen Static Site Generators (Hugo, MkDocs, React etc.)
- Begrenzter Build-Prozess (kein npm, Python, Go etc.)

Mit **GitHub Actions** sind diese Einschränkungen aufgehoben — jeder Build-Prozess ist möglich.

### Deployment Flow

```mermaid
flowchart LR
    A[Push to main] --> B[Actions Trigger]
    B --> C[Build Job Install und Build]
    C --> D[Upload Artifact upload-pages-artifact]
    D --> E[Deploy Job deploy-pages]
    E --> F[GitHub Pages CDN]
    F --> G[Website live]
```

### Hugo Site Deployment — Vollständiges YAML

```yaml
# .github/workflows/pages.yml
name: Deploy Hugo Site to GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.128.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb \
            https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Install Dart Sass
        run: sudo snap install dart-sass

      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup GitHub Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Install Node.js Dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"

      - name: Build Hugo Site
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: Europe/Berlin
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload Pages Artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> [!TIP]
> Das `environment: github-pages` ist zwingend erforderlich. GitHub Pages akzeptiert nur Deployments aus dem dedizierten `github-pages`-Environment. Die Permissions `pages: write` und `id-token: write` sind ebenfalls Pflicht.

---

## Dokumentations-Frameworks für Pages

GitHub Pages ist besonders beliebt für technische Dokumentation. Folgende Frameworks werden häufig eingesetzt:

| Framework    | Sprache    | Pages nativ | Actions nötig | Besonderheit                              |
|--------------|------------|-------------|---------------|-------------------------------------------|
| Jekyll       | Ruby       | Ja          | Optional      | Eingebaut, keine Actions für einfache Sites|
| MkDocs       | Python     | Nein        | Ja            | Material-Theme sehr beliebt               |
| Docusaurus   | JavaScript | Nein        | Ja            | Facebooks Doku-Framework                  |
| Hugo         | Go         | Nein        | Ja            | Extrem schnelle Build-Zeiten              |
| Sphinx       | Python     | Nein        | Ja            | Python-Wissenschaft-Standard              |
| mdBook       | Rust       | Nein        | Ja            | Rust-Dokumentations-Standard              |
| VitePress    | Vue        | Nein        | Ja            | Vues modernes Doku-Tool                   |
| Astro        | JavaScript | Nein        | Ja            | Modern, multi-framework-fähig             |

### MkDocs mit Material Theme — GitHub Actions YAML

```yaml
# .github/workflows/mkdocs.yml
name: Deploy MkDocs to GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Cache pip packages
        uses: actions/cache@v4
        with:
          key: mkdocs-material-${{ hashFiles('requirements.txt') }}
          path: ~/.cache/pip
          restore-keys: mkdocs-material-

      - name: Install MkDocs Material
        run: pip install mkdocs-material mkdocs-git-revision-date-localized-plugin

      - name: Deploy to GitHub Pages
        run: mkdocs gh-deploy --force
```

---

## Einschränkungen

> [!WARNING]
> GitHub Pages ist ausschließlich für **statische Inhalte** konzipiert. Kein serverseitiger Code, keine Datenbanken, keine dynamischen Backend-Funktionen.

| Einschränkung              | Limit / Details                                   |
|----------------------------|---------------------------------------------------|
| Repository-Größe           | 1 GB (empfohlen)                                  |
| Deployment-Größe           | 10 GB (hart)                                      |
| Bandbreite / Monat         | 100 GB (Soft Limit)                               |
| Jekyll-Builds / Stunde     | 10 Builds                                         |
| Serverseitige Sprachen     | Nicht unterstützt                                 |
| Datenbanken                | Nicht unterstützt                                 |
| Formulare                  | Nur via Drittanbieter (Formspree, Netlify Forms)  |
| Verbotene Inhalte          | Keine Malware, kein illegaler Content             |
| Inaktivitäts-Deaktivierung | Bei dauerhafter Inaktivität möglich               |

> [!DANGER]
> Wird das **100 GB Bandbreiten-Limit** überschritten, kann GitHub die Site vorübergehend deaktivieren. GitHub sendet zuvor eine Warnung per E-Mail. Für hochfrequentierte Sites empfiehlt sich ein CDN (Cloudflare) vor GitHub Pages.

---

## Deployment Protection Rules

In **Repository Settings → Environments → github-pages** können Schutzregeln für Deployments konfiguriert werden:

| Regel                               | Beschreibung                                              |
|-------------------------------------|-----------------------------------------------------------|
| Required reviewers                  | Deployment muss von bestimmten Personen genehmigt werden  |
| Wait timer                          | Deployment verzögert um 0–43.200 Minuten                  |
| Deployment branches and tags        | Nur bestimmte Branches/Tags dürfen deployen               |
| Custom deployment protection rules  | GitHub App-basierte externe Regeln (Enterprise)           |

---

## Private Pages

| GitHub-Plan              | Private Repos mit Pages |
|--------------------------|-------------------------|
| GitHub Free              | Nicht verfügbar         |
| GitHub Pro               | Verfügbar               |
| GitHub Team              | Verfügbar               |
| GitHub Enterprise Cloud  | Verfügbar               |
| GitHub Enterprise Server | Verfügbar (admin-seitig konfigurierbar) |

> [!IMPORTANT]
> Private Pages auf **GitHub Free sind nicht möglich**. Für interne Dokumentation aus einem privaten Repository wird mindestens **GitHub Pro** ($4/Monat) benötigt. Alternativen für private Doku: Azure Static Web Apps, Confluence, Notion oder ein selbst-gehosteter MkDocs-Server.

---

## Vergleich: GitHub Pages vs. Alternativen

| Feature                  | GitHub Pages          | Azure Static Web Apps      | Netlify            | Vercel             |
|--------------------------|-----------------------|----------------------------|--------------------|--------------------|
| **Preis (Basis)**        | Kostenlos             | Kostenlos (Free Tier)      | Kostenlos (Hobby)  | Kostenlos (Hobby)  |
| **Custom Domain**        | Ja                    | Ja                         | Ja                 | Ja                 |
| **HTTPS**                | Automatisch           | Automatisch                | Automatisch        | Automatisch        |
| **CDN**                  | Global (Fastly)       | Azure CDN / Global         | Global             | Global (Edge)      |
| **CI/CD**                | GitHub Actions        | GitHub Actions / Pipelines | Eingebaut          | Eingebaut          |
| **Serverless Functions** | Nein                  | Ja (Azure Functions)       | Ja                 | Ja (Edge Fns)      |
| **Authentifizierung**    | Nein                  | Ja (Entra ID, GitHub)      | Ja (Identity)      | Ja                 |
| **Forms**                | Nein                  | Nein (nativ)               | Ja                 | Nein (nativ)       |
| **A/B Testing**          | Nein                  | Ja                         | Ja                 | Ja                 |
| **Build-Minuten/Monat**  | Unbegrenzt (Actions)  | 100 Min (Free)             | 300 Min            | Unbegrenzt         |
| **Bandbreite/Monat**     | 100 GB                | 100 GB                     | 100 GB             | 100 GB             |
| **Branch Previews**      | Nein                  | Ja                         | Ja                 | Ja                 |
| **SSR Support**          | Nein                  | Begrenzt                   | Ja                 | Ja (Next.js)       |
| **Private Sites**        | Pro/Team/Enterprise   | Ja                         | Ja                 | Ja                 |
| **SLA**                  | Kein offizielles SLA  | 99,95 %                    | 99,99 %            | 99,99 %            |
| **Beste für**            | Open-Source-Doku      | Microsoft/Azure-Projekte   | JAMstack           | Next.js / React    |

---

## Kosten im Überblick

| GitHub-Plan              | Preis                  | Pages für Public Repos | Pages für Private Repos |
|--------------------------|------------------------|------------------------|-------------------------|
| Free                     | $0                     | Ja                     | Nein                    |
| Pro                      | $4 / Monat             | Ja                     | Ja                      |
| Team                     | $4 / Nutzer / Monat    | Ja                     | Ja                      |
| Enterprise Cloud         | $21 / Nutzer / Monat   | Ja                     | Ja                      |

> [!NOTE]
> GitHub Pages an sich hat **keine zusätzlichen Kosten** — es ist im jeweiligen Plan enthalten. Zusatzkosten entstehen nur durch GitHub Actions-Minuten bei privaten Repositories, falls das monatliche Inklusiv-Kontingent überschritten wird.

---

## Wann GitHub Pages nutzen?

**GitHub Pages ist ideal wenn:**

- Das Projekt bereits auf GitHub liegt
- Einfache, statische Dokumentation oder ein Blog benötigt wird
- Kein Hosting-Budget vorhanden ist
- Jekyll ausreicht oder GitHub Actions für komplexere Builds genutzt wird
- Open-Source-Charakter und Community-Sichtbarkeit gewünscht sind

**Alternative erwägen wenn:**

- Serverless Functions / API-Backend benötigt werden → **Azure Static Web Apps / Netlify / Vercel**
- Branch-Preview-Deployments für Teams wichtig sind → **Netlify / Vercel**
- Next.js oder serverseitiges Rendering genutzt wird → **Vercel**
- Microsoft-Ökosystem, Compliance und SLA wichtig → **Azure Static Web Apps**
- Interne Dokumentation ohne öffentlichen Zugriff → **Azure Static Web Apps** oder selbst-gehostete Lösung

---

## Via GitHub.com (UI)

### GitHub Pages aktivieren

1. Öffne **[github.com](https://github.com/[org]/[repo])**
2. Navigiere zu **Settings** → **Pages**
3. Unter **Build and deployment** → Source: **Deploy from a branch** oder **GitHub Actions**
4. Branch auswählen (meist `main` oder `gh-pages`), Ordner `/` oder `/docs`
5. **Save** → URL wird angezeigt (z. B. `https://[org].github.io/[repo]`)

### Custom Domain konfigurieren

1. **Settings** → **Pages** → **Custom domain**
2. Domain eingeben → **Save**
3. DNS beim Domain-Registrar konfigurieren (CNAME oder A-Records)
4. **Enforce HTTPS** aktivieren (nach DNS-Propagation)

### Deployment-Status prüfen

1. Repository → **Actions** Tab → **pages build and deployment** Workflow
2. Status der letzten Deployments anzeigen

---

## Verwandte Notizen

- [[GitHub Grundlagen]]
- [[GitHub Actions]]
- [[GitHub Enterprise]]
