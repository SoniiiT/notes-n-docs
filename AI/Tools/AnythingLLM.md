# AnythingLLM

## Was ist AnythingLLM?

AnythingLLM ist eine All-in-One AI-Anwendung mit starkem Fokus auf [[RAG Grundlagen|RAG]] (Retrieval-Augmented Generation). Sie erlaubt es, eigene Dokumente, Webseiten und Datenbanken als Wissensbasis für LLM-Gespräche zu nutzen – in vollständig lokal betriebenen oder Cloud-basierten Konfigurationen.

AnythingLLM unterstützt eine Vielzahl von LLM-Backends ([[Ollama]], OpenAI, Anthropic, Azure, Groq, u.v.m.), verschiedene [[Embeddings|Embedding]]-Modelle und Vektor-Datenbanken, und bietet zusätzlich einen eingebauten Agenten-Modus mit Tool-Calling.

Die Anwendung ist als Docker-Container sowie als plattformübergreifende Desktop-App (Windows, macOS, Linux) verfügbar.

---

## Unterschied zu Open WebUI

| Merkmal | AnythingLLM | Open WebUI |
|---|---|---|
| Hauptfokus | RAG & Dokument-Chat | Chat-Interface à la ChatGPT |
| Workspace-Konzept | Getrennte Räume mit eigenem Kontext und eigener Wissensbasis | Getrennte Chat-Verläufe |
| RAG-Tiefe | Sehr ausgefeilt: Chunk-Strategie, Re-Ranking, hybride Suche | Einfachere Dokument-Integration |
| Vektordatenbank | Wählbar (ChromaDB, Pinecone, Weaviate, Qdrant, etc.) | Eingebaut (chroma intern) |
| Benutzerverwaltung | Multi-User mit Workspace-Berechtigungen | Multi-User mit Admin/User-Rollen |
| Agenten | Eingebaut mit konfigurierbaren Tools | Tools via Python-Skripte |
| Desktop-App | Ja (Electron-basiert) | Nein (nur Browser) |
| Komplexität | Höher, mehr Konfiguration nötig | Einfacher Einstieg |

> [!NOTE]
> AnythingLLM eignet sich besonders für Anwendungsfälle, bei denen Dokumente (PDFs, Word-Dateien, Code-Repositories, Webseiten) als Wissensbasis dienen sollen. Open WebUI ist die bessere Wahl für eine allgemeine Chat-Oberfläche mit vielen Modellen.

---

## Installation: Docker

### Schnellstart

```bash
docker run -d \
  --name anythingllm \
  --restart always \
  -p 3001:3001 \
  -v "${HOME}/.anythingllm:/app/server/storage" \
  -e STORAGE_DIR=/app/server/storage \
  mintplexlabs/anythingllm:latest
```

Nach dem Start: `http://localhost:3001`

### Docker Compose

```yaml
# docker-compose.yml
services:
  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    restart: always
    ports:
      - "3001:3001"
    environment:
      - STORAGE_DIR=/app/server/storage
      - JWT_SECRET=dein-jwt-secret-hier          # Unbedingt setzen!
      - LLM_PROVIDER=ollama
      - OLLAMA_BASE_PATH=http://host.docker.internal:11434
      - OLLAMA_MODEL_PREF=llama3.2
      - EMBEDDING_ENGINE=native
      - VECTOR_DB=lancedb                         # Eingebaute Vektordatenbank
      - AUTH_TOKEN=dein-passwort-hier             # Für Single-User-Modus
    volumes:
      - anythingllm_storage:/app/server/storage
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  anythingllm_storage:
```

```bash
docker compose up -d
docker compose logs -f anythingllm
```

> [!TIP]
> Ohne `AUTH_TOKEN` ist AnythingLLM offen zugänglich. Für lokale Nutzung ausreichend, für Server-Deployments unbedingt ein Token setzen.

### Mit externem ChromaDB

```yaml
services:
  chromadb:
    image: chromadb/chroma:latest
    container_name: chromadb
    restart: always
    volumes:
      - chromadb_data:/chroma/chroma
    ports:
      - "8000:8000"

  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    restart: always
    ports:
      - "3001:3001"
    environment:
      - STORAGE_DIR=/app/server/storage
      - VECTOR_DB=chroma
      - CHROMA_ENDPOINT=http://chromadb:8000
    volumes:
      - anythingllm_storage:/app/server/storage
    depends_on:
      - chromadb

volumes:
  chromadb_data:
  anythingllm_storage:
```

---

## Installation: Desktop-App

Die Desktop-App ist eine Electron-Anwendung, die AnythingLLM komplett eingebettet (inkl. Webserver) auf dem Desktop ausführt – ohne Docker.

### Windows

1. Installer herunterladen: [https://anythingllm.com/download](https://anythingllm.com/download)
2. `AnythingLLMDesktop.exe` ausführen und installieren
3. App starten – öffnet automatisch im Browser

### macOS

```bash
# Homebrew
brew install --cask anythingllm

# Oder: .dmg von der Website herunterladen
```

### Linux (AppImage)

```bash
# AppImage herunterladen
wget https://cdn.anythingllm.com/latest/AnythingLLMDesktop.AppImage

# Ausführbar machen und starten
chmod +x AnythingLLMDesktop.AppImage
./AnythingLLMDesktop.AppImage
```

> [!NOTE]
> Die Desktop-App speichert ihre Daten standardmäßig unter `~/Documents/AnythingLLM` (Windows/macOS) bzw. `~/.config/anythingllm-desktop/storage` (Linux). Alle Einstellungen, Workspaces und Dokumente liegen dort.

---

## LLM-Backend konfigurieren

Nach dem ersten Start öffnet sich ein Setup-Wizard. Das LLM-Backend kann jederzeit unter **Settings** → **LLM Preference** geändert werden.

### Ollama (lokal)

| Einstellung | Wert |
|---|---|
| LLM Provider | `Ollama` |
| Ollama Base URL | `http://localhost:11434` (Desktop) oder `http://host.docker.internal:11434` (Docker) |
| Ollama Model | Modell aus Dropdown wählen (z.B. `llama3.2`) |

```bash
# Sicherstellen, dass Ollama läuft und das Modell vorhanden ist
ollama list
ollama pull llama3.2
```

### OpenAI

| Einstellung | Wert |
|---|---|
| LLM Provider | `OpenAI` |
| API Key | `sk-...` |
| Model | `gpt-4o` / `gpt-4o-mini` / etc. |

### Anthropic Claude

| Einstellung | Wert |
|---|---|
| LLM Provider | `Anthropic` |
| API Key | Anthropic API-Key |
| Model | `claude-opus-4-6` / `claude-sonnet-4-6` |

### Weitere unterstützte Backends

- Azure OpenAI
- Google Gemini
- Groq
- Mistral AI
- Perplexity
- Cohere
- KoboldCPP
- LM Studio
- LiteLLM (als universeller Proxy)
- Hugging Face Inference API

---

## Embedding-Modell konfigurieren

Das Embedding-Modell wandelt Texte in Vektoren um und ist entscheidend für die RAG-Qualität. Einstellung: **Settings** → **Embedding Preference**.

| Option | Beschreibung | Empfehlung |
|---|---|---|
| `AnythingLLM Native` | Eingebautes lokales Modell | Einstieg, keine Abhängigkeiten |
| `Ollama` | Embedding-Modell via Ollama | Empfohlen für lokalen Betrieb |
| `OpenAI` | `text-embedding-3-small` / `large` | Beste Qualität, kostenpflichtig |
| `Cohere` | Cohere Embed | Gut für mehrsprachige Inhalte |
| `Azure OpenAI` | Wie OpenAI, via Azure | Enterprise-Nutzung |

**Ollama-Embedding konfigurieren:**

```bash
# Empfohlene Embedding-Modelle für Ollama
ollama pull nomic-embed-text    # 274 MB, schnell, gut
ollama pull mxbai-embed-large   # 669 MB, besser bei englischen Texten
ollama pull bge-m3              # 1.2 GB, sehr gut, mehrsprachig
```

Dann in AnythingLLM: **Embedding Provider** → `Ollama`, **Model** → `nomic-embed-text`

> [!WARNING]
> Wenn du das Embedding-Modell nach dem Anlegen von Workspaces wechselst, müssen alle Dokumente neu eingebettet werden. Die bestehenden Vektoren werden ungültig.

---

## Workspace erstellen

Workspaces sind isolierte Bereiche mit eigener Wissensbasis, eigenem LLM-Kontext und eigenen Einstellungen.

1. Linke Seitenleiste → **+ New Workspace**
2. Name vergeben (z.B. "Projektdokumentation", "Linux-Wissen")
3. Workspace auswählen → Chat-Interface öffnet sich
4. Rechts: **Settings** für Workspace-spezifische Konfiguration

### Workspace-Einstellungen

| Einstellung | Beschreibung |
|---|---|
| LLM Override | Anderes Modell als Standard nutzen |
| Prompt | System Prompt für diesen Workspace |
| Chat Mode | `chat` (mit Verlauf) oder `query` (jede Frage separat) |
| Similarity Threshold | Wie ähnlich Dokument-Chunks sein müssen (0.0–1.0) |
| Top-N Chunks | Wie viele Chunks aus der Vektordatenbank abgerufen werden |
| History Messages | Anzahl der Chat-Nachrichten im Kontext |

---

## Dokumente hochladen und als Wissensquelle nutzen (RAG)

### Dokumente in einen Workspace hochladen

1. Workspace öffnen
2. Unten links auf das **Upload**-Symbol klicken (oder Drag & Drop)
3. Datei(en) hochladen: PDF, DOCX, TXT, MD, CSV, HTML, JSON, etc.
4. Auf **Move to Workspace** klicken → Dokument wird eingebettet
5. Das Dokument erscheint nun in der Wissensbasis des Workspaces

### Unterstützte Dokumenttypen

| Format | Beschreibung |
|---|---|
| PDF | Text wird extrahiert (kein OCR für Bilder) |
| DOCX / DOC | Word-Dokumente |
| TXT, MD | Plain Text und Markdown |
| CSV | Tabellendaten |
| HTML | Webseiten (lokal gespeichert) |
| JSON | Strukturierte Daten |
| YouTube-URL | Transkript wird automatisch geladen |
| Webseiten-URL | Seite wird gecrawlt und gespeichert |
| GitHub-Repo | Code-Repository als Wissensbasis |

### RAG im Chat nutzen

Sobald Dokumente im Workspace sind, werden bei jeder Anfrage automatisch relevante Text-Chunks aus der Vektordatenbank abgerufen und dem LLM-Kontext beigefügt. Die Quellen werden unter der Antwort angezeigt.

```
Benutzer: Was sind die Installationsvoraussetzungen laut der Dokumentation?
AnythingLLM: [sucht in Vektordatenbank nach relevanten Chunks]
             → Antwortet basierend auf dem gefundenen Dokumentinhalt
             → Zeigt Quellenangabe: "dokument.pdf, Seite 3"
```

> [!TIP]
> Im `query`-Modus (Workspace-Einstellung) wird der Chat-Verlauf nicht für Folgefragen berücksichtigt. Gut für präzise Dokumentenabfragen. Im `chat`-Modus wird der gesamte Verlauf + Dokumentenkontext übergeben.

---

## Agenten in AnythingLLM

Der eingebaute Agenten-Modus erlaubt es dem LLM, Tools zu nutzen, um Aufgaben autonom zu erledigen.

### Agenten aktivieren

1. Im Chat `@agent` eingeben, um den Agenten-Modus zu starten
2. Oder: Workspace-Einstellungen → **Agent** → aktivieren

### Verfügbare eingebaute Tools

| Tool | Beschreibung |
|---|---|
| Web Search | DuckDuckGo/Google-Suche |
| Web Scraping | Webseiten-Inhalte abrufen |
| Save File | Dateien speichern |
| List Files | Dateisystem-Zugriff |
| RAG | Eigene Wissensbasis abfragen |
| Chart Creation | Diagramme generieren |
| SQL Database | Datenbankabfragen ausführen |

### Beispiel: Agenten-Anfrage

```
@agent Suche nach den neuesten Informationen zu Ollama und fasse zusammen,
welche neuen Features in der letzten Version enthalten sind.
```

Der Agent führt dann eigenständig Websuchen durch, ruft Seiten ab und synthetisiert eine Antwort.

> [!NOTE]
> Agenten funktionieren am besten mit leistungsfähigeren Modellen wie GPT-5.2, Claude Sonnet 4.6 oder Llama 4 Scout. Kleinere Modelle haben oft Probleme mit der korrekten Tool-Nutzung.

---

## API-Zugang aktivieren und nutzen

AnythingLLM bietet eine REST API, die programmatischen Zugriff auf Workspaces, Dokumente und Chats ermöglicht.

### API aktivieren

1. **Settings** → **Security** → **API Keys** → **+ Generate API Key**
2. Key kopieren und sicher speichern

### Wichtige API-Endpunkte

```bash
BASE_URL="http://localhost:3001"
API_KEY="dein-api-key"

# Workspace-Chat
curl -X POST "${BASE_URL}/api/v1/workspace/mein-workspace/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "message": "Was steht in meinen Dokumenten über Kubernetes?",
    "mode": "chat"
  }'

# Dokument hochladen
curl -X POST "${BASE_URL}/api/v1/document/upload" \
  -H "Authorization: Bearer ${API_KEY}" \
  -F "file=@/pfad/zur/datei.pdf"

# Alle Workspaces auflisten
curl -X GET "${BASE_URL}/api/v1/workspaces" \
  -H "Authorization: Bearer ${API_KEY}"

# Dokument zu Workspace hinzufügen
curl -X POST "${BASE_URL}/api/v1/workspace/mein-workspace/update-embeddings" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{"adds": ["custom-documents/datei.pdf-uuid.json"]}'
```

### Python-Beispiel

```python
import requests

class AnythingLLMClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }

    def chat(self, workspace_slug: str, message: str, mode: str = "chat") -> dict:
        url = f"{self.base_url}/api/v1/workspace/{workspace_slug}/chat"
        payload = {"message": message, "mode": mode}
        response = requests.post(url, json=payload, headers=self.headers)
        response.raise_for_status()
        return response.json()

    def upload_document(self, file_path: str) -> dict:
        url = f"{self.base_url}/api/v1/document/upload"
        headers = {"Authorization": self.headers["Authorization"]}
        with open(file_path, "rb") as f:
            files = {"file": f}
            response = requests.post(url, headers=headers, files=files)
        response.raise_for_status()
        return response.json()


# Verwendung
client = AnythingLLMClient("http://localhost:3001", "dein-api-key")
result = client.chat("mein-workspace", "Erkläre den Inhalt meiner Dokumentation.")
print(result["textResponse"])
```

---

## Datenpersistenz und Backup

Alle AnythingLLM-Daten (Einstellungen, Dokumente, Vektoren, Chat-Verläufe) liegen im Storage-Verzeichnis.

### Speicherorte

| Umgebung | Pfad |
|---|---|
| Docker (Volume) | `/app/server/storage` im Container |
| Desktop (Linux) | `~/.config/anythingllm-desktop/storage` |
| Desktop (macOS) | `~/Library/Application Support/anythingllm-desktop/storage` |
| Desktop (Windows) | `%APPDATA%\anythingllm-desktop\storage` |

### Backup (Docker)

```bash
# Manuelles Backup
docker exec anythingllm tar czf /tmp/backup.tar.gz -C /app/server storage
docker cp anythingllm:/tmp/backup.tar.gz ./anythingllm-backup-$(date +%Y%m%d).tar.gz

# Oder via Volume-Mount (wenn Bind-Mount verwendet wird)
tar czf anythingllm-backup-$(date +%Y%m%d).tar.gz ~/.anythingllm/
```

### Automatisches Backup mit Cron

```bash
# Backup-Skript anlegen
cat > /usr/local/bin/backup-anythingllm.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/backup/anythingllm"
DATE=$(date +%Y%m%d_%H%M)
mkdir -p "${BACKUP_DIR}"

docker exec anythingllm tar czf /tmp/backup.tar.gz -C /app/server storage
docker cp anythingllm:/tmp/backup.tar.gz "${BACKUP_DIR}/backup-${DATE}.tar.gz"

# Backups älter als 30 Tage löschen
find "${BACKUP_DIR}" -name "*.tar.gz" -mtime +30 -delete
EOF

chmod +x /usr/local/bin/backup-anythingllm.sh

# Cron-Job: täglich um 02:00 Uhr
echo "0 2 * * * /usr/local/bin/backup-anythingllm.sh" | crontab -
```

### Update

```bash
# Docker Compose
docker compose pull
docker compose up -d

# Einzelner Container
docker pull mintplexlabs/anythingllm:latest
docker stop anythingllm && docker rm anythingllm
# Dann erneut docker run mit identischen Parametern
```

> [!IMPORTANT]
> Das Storage-Verzeichnis enthält die SQLite-Datenbank (`anythingllm.db`) mit allen Workspaces, Chats und Einstellungen sowie den `documents/`-Ordner mit allen hochgeladenen Dateien und deren Vektoren. Dieses Verzeichnis ist das einzige, was für ein vollständiges Backup benötigt wird.
