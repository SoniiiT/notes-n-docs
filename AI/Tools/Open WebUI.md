# Open WebUI

## Was ist Open WebUI?

Open WebUI ist eine selbst-gehostete, ChatGPT-ähnliche Weboberfläche für lokale und externe LLMs. Es bietet eine vollständige Chat-Oberfläche im Stil von ChatGPT, unterstützt Ollama und OpenAI-kompatible APIs als Backend, und erweitert diese um Funktionen wie RAG (Retrieval-Augmented Generation), Dokument-Uploads, Tool-Calling, Bildgenerierung, Sprachausgabe und ein Benutzerverwaltungssystem.

Open WebUI läuft als Docker-Container oder Node.js-Anwendung und speichert alle Daten lokal in einer SQLite-Datenbank.

> [!NOTE]
> Open WebUI kommuniziert direkt mit Ollama oder einer OpenAI-kompatiblen API. Es ist kein Proxy, sondern ein vollständiges Frontend. Die Verbindung zu Ollama erfolgt serverseitig – der Browser kommuniziert nur mit Open WebUI, nicht direkt mit Ollama.

---

## Voraussetzungen

- **Docker** (empfohlen) oder Node.js 20+
- **Ollama** (optional, aber empfohlen für lokale Modelle) – [[Ollama]]
- Mindestens 2 GB RAM für den Container selbst
- Port `3000` frei (konfigurierbar)

```bash
# Docker installieren (Ubuntu/Debian)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Docker-Version prüfen
docker --version
docker compose version
```

---

## Installation via Docker (schnellstart)

### Mit Ollama auf dem gleichen Host

```bash
docker run -d \
  --name open-webui \
  --restart always \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  ghcr.io/open-webui/open-webui:main
```

### Mit NVIDIA GPU (für Bildgenerierung / Whisper im Container)

```bash
docker run -d \
  --name open-webui \
  --restart always \
  --gpus all \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  ghcr.io/open-webui/open-webui:cuda
```

> [!TIP]
> Das Tag `:main` ist der aktuelle Entwicklungsstand. Für stabilere Versionen `:stable` oder ein spezifisches Versions-Tag wie `:0.4.8` verwenden.

Nach dem Start ist Open WebUI unter `http://localhost:3000` erreichbar. Beim ersten Aufruf wird ein Admin-Konto erstellt.

---

## Installation via Docker Compose

```yaml
# docker-compose.yml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: always
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
      - WEBUI_SECRET_KEY=dein-geheimer-schluessel-hier   # Unbedingt ändern!
      - WEBUI_AUTH=true                                  # Benutzeranmeldung aktivieren
      - DEFAULT_LOCALE=de-DE
    volumes:
      - open-webui:/app/backend/data
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  open-webui:
```

```bash
# Starten
docker compose up -d

# Logs verfolgen
docker compose logs -f open-webui

# Stoppen
docker compose down

# Update (neue Version ziehen und neu starten)
docker compose pull && docker compose up -d
```

### Docker Compose mit integriertem Ollama

```yaml
# docker-compose.yml (Ollama + Open WebUI zusammen)
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: always
    volumes:
      - ollama:/root/.ollama
    ports:
      - "11434:11434"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: always
    depends_on:
      - ollama
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=dein-geheimer-schluessel-hier
    volumes:
      - open-webui:/app/backend/data

volumes:
  ollama:
  open-webui:
```

---

## Mit Ollama verbinden

### Automatische Verbindung (gleicher Host)

Wenn Ollama und Open WebUI auf dem gleichen Rechner laufen und der Docker-Container mit `--add-host=host.docker.internal:host-gateway` gestartet wurde, verbindet sich Open WebUI automatisch via `http://host.docker.internal:11434`.

### Manuelle Konfiguration im Admin-Panel

1. Open WebUI öffnen → **Admin Panel** → **Settings** → **Connections**
2. Unter **Ollama API** die URL eintragen: `http://host.docker.internal:11434`
3. **Verbindung testen** klicken
4. Speichern

### Mit externem Ollama-Server verbinden

```bash
# Umgebungsvariable beim Container-Start setzen
-e OLLAMA_BASE_URL=http://192.168.1.100:11434

# Oder in docker-compose.yml:
environment:
  - OLLAMA_BASE_URL=http://192.168.1.100:11434
```

> [!WARNING]
> Der externe Ollama-Server muss auf `0.0.0.0` lauschen (`OLLAMA_HOST=0.0.0.0:11434`). Stelle sicher, dass der Port in der Firewall des Zielservers freigegeben ist und der Zugang abgesichert ist.

---

## Mit OpenAI-kompatibler API verbinden

Open WebUI kann neben Ollama auch mit jeder OpenAI-kompatiblen API arbeiten – also z.B. der echten OpenAI-API, Azure OpenAI, Groq, Mistral, Anthropic (via Proxy) oder lokalen Alternativen wie LM Studio.

### Im Admin-Panel konfigurieren

1. **Admin Panel** → **Settings** → **Connections**
2. Unter **OpenAI API** auf **+** klicken
3. **API-URL** und **API-Key** eintragen:

| Anbieter | URL | Key |
|---|---|---|
| OpenAI | `https://api.openai.com/v1` | OpenAI API-Key |
| Groq | `https://api.groq.com/openai/v1` | Groq API-Key |
| LM Studio | `http://localhost:1234/v1` | `lm-studio` |
| Ollama (alt.) | `http://localhost:11434/v1` | `ollama` |

### Über Umgebungsvariable

```bash
-e OPENAI_API_KEY=sk-...
-e OPENAI_API_BASE_URL=https://api.openai.com/v1
```

---

## Features

### RAG (Retrieval-Augmented Generation)

Open WebUI enthält eine integrierte RAG-Pipeline:

1. **Dokument hochladen:** Im Chat auf das Büroklammer-Symbol klicken → Datei hochladen (PDF, TXT, DOCX, HTML, etc.)
2. **Wissensdatenbank anlegen:** **Workspace** → **Knowledge** → neue Collection erstellen → Dateien hinzufügen
3. **Collection im Chat nutzen:** `#` im Chat-Eingabefeld eingeben → Collection auswählen

Für Embeddings wird standardmäßig ein lokales Sentence-Transformer-Modell verwendet. In den Einstellungen kann auch ein Ollama-Embedding-Modell oder OpenAI-Embeddings konfiguriert werden.

### Modell-Auswahl

- Oben im Chat-Interface das Modell-Dropdown nutzen
- Mehrere Modelle gleichzeitig aktivieren → parallele Antworten vergleichen
- Modell-Filter: nur bestimmte Modelle für Benutzer freigeben

### System Prompts und Personas

1. **Workspace** → **Models** → **+ Neues Modell**
2. Basis-Modell auswählen (z.B. `llama3.2`)
3. System Prompt, Avatar und Name definieren
4. Als Preset speichern – erscheint dann im Modell-Dropdown

### Werkzeuge (Tools)

Open WebUI unterstützt Python-basierte Tools (ähnlich OpenAI Function Calling):

1. **Workspace** → **Tools** → **+ Neues Tool**
2. Python-Funktion definieren (z.B. Websuche, Wetterdaten, Datenbankabfragen)
3. Tool dem Modell zuweisen

### Benutzer und Rollen

- **Admin:** Vollzugriff, Systemkonfiguration
- **User:** Normaler Chat-Zugang
- **Pending:** Konto angelegt, wartet auf Freischaltung

Neue Benutzer können sich selbst registrieren (konfigurierbar) oder nur eingeladen werden.

---

## Wichtige Admin-Einstellungen

### Admin-Panel öffnen

Oben rechts auf Profilbild → **Admin Panel** oder direkt: `http://localhost:3000/admin`

### Kritische Einstellungen

| Einstellung | Pfad | Beschreibung |
|---|---|---|
| Registrierung | General → Signup | Neue Konten erlauben/sperren |
| Standard-Modell | General → Default Model | Für alle neuen Chats |
| RAG-Chunk-Size | Documents → Chunk Size | Größe der Text-Chunks für RAG |
| Embedding-Modell | Documents → Embedding Model | Für RAG-Vektorisierung |
| Bildgenerierung | Images → AUTOMATIC1111 URL | Für Stable Diffusion Integration |
| Spracherkennung | Audio → STT Engine | Whisper-Konfiguration |
| HTTPS/Proxy | – | Über Reverse Proxy lösen |

---

## Datenpersistenz: Volumes konfigurieren

Alle Daten (Chats, Benutzer, Dokumente, Einstellungen) werden im Docker-Volume `open-webui` gespeichert, das auf `/app/backend/data` im Container gemountet ist.

```bash
# Volume-Speicherort auf dem Host finden
docker volume inspect open-webui

# Backup erstellen
docker run --rm \
  -v open-webui:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/open-webui-backup.tar.gz -C / data

# Backup wiederherstellen
docker run --rm \
  -v open-webui:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/open-webui-backup.tar.gz -C /
```

Für explizites Bind-Mount (direkter Pfad auf Host):

```yaml
volumes:
  - /home/benutzer/open-webui-data:/app/backend/data
```

---

## Reverse Proxy für HTTPS-Zugang

### Nginx

```nginx
# /etc/nginx/sites-available/openwebui
server {
    listen 80;
    server_name webui.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name webui.example.com;

    ssl_certificate     /etc/letsencrypt/live/webui.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/webui.example.com/privkey.pem;

    client_max_body_size 100M;  # Für große Datei-Uploads

    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_buffering    off;                     # Wichtig für Streaming!
        proxy_read_timeout 300s;                    # Für lange Antworten
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";    # Für WebSocket
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/openwebui /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# SSL-Zertifikat mit Certbot
sudo certbot --nginx -d webui.example.com
```

### Traefik (Docker-Labels)

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.openwebui.rule=Host(`webui.example.com`)"
      - "traefik.http.routers.openwebui.entrypoints=websecure"
      - "traefik.http.routers.openwebui.tls.certresolver=letsencrypt"
      - "traefik.http.services.openwebui.loadbalancer.server.port=8080"
      - "traefik.http.middlewares.openwebui-buffer.buffering.maxRequestBodyBytes=104857600"
```

> [!TIP]
> `proxy_buffering off` in Nginx ist entscheidend, damit Streaming-Antworten (Token für Token) korrekt an den Browser weitergeleitet werden und nicht gebuffert werden.

---

## Update auf neue Version

```bash
# Docker Compose
docker compose pull
docker compose up -d

# Einzelner Container
docker pull ghcr.io/open-webui/open-webui:main
docker stop open-webui
docker rm open-webui
# Dann erneut docker run ... ausführen (Volume bleibt erhalten!)
```

> [!IMPORTANT]
> Das Docker-Volume `open-webui` enthält alle Daten und bleibt beim Container-Löschen erhalten. Es wird nicht gelöscht, wenn `docker rm` ausgeführt wird – nur `docker volume rm open-webui` würde die Daten löschen.

```bash
# Vor dem Update: Backup erstellen
docker run --rm \
  -v open-webui:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/open-webui-backup-$(date +%Y%m%d).tar.gz -C / data
```

> [!WARNING]
> Open WebUI befindet sich in aktiver Entwicklung. Beim Update auf eine neue Major-Version können sich Breaking Changes bei der Datenbankstruktur ergeben. Backup vor jedem Update ist daher dringend empfohlen.
