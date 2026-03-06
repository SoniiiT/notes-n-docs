---
tags: [ai, tools, ollama, self-hosting]
---

# Ollama

## Was ist Ollama?

Ollama ist ein lokaler LLM-Runner, der es ermöglicht, Large Language Models direkt auf dem eigenen Rechner auszuführen – ohne Cloud-Abhängigkeit, ohne API-Kosten, ohne Datenweitergabe an Dritte. Ollama abstrahiert den gesamten Modell-Download, die Quantisierung und die Inferenz in ein einfaches CLI-Tool und stellt gleichzeitig eine OpenAI-kompatible REST API bereit.

Intern nutzt Ollama `llama.cpp` als Inferenz-Backend, unterstützt GPU-Beschleunigung via CUDA (NVIDIA) und ROCm (AMD) sowie Metal (Apple Silicon), und verwaltet Modelle in einem eigenen Format (`.gguf`-basiert mit Modelfile-Metadaten).

> [!NOTE]
> Ollama läuft als Hintergrunddienst und stellt eine HTTP-API auf `localhost:11434` bereit. CLI-Befehle kommunizieren intern mit diesem Dienst.

---

## Installation

### Linux

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Das Skript installiert den `ollama`-Binary nach `/usr/local/bin/ollama`, legt einen systemd-Service an und startet ihn automatisch.

> [!TIP]
> Für eine manuelle Installation ohne Root-Rechte kann der Binary direkt von [https://github.com/ollama/ollama/releases](https://github.com/ollama/ollama/releases) heruntergeladen werden.

```bash
# Manuell
sudo curl -L https://github.com/ollama/ollama/releases/latest/download/ollama-linux-amd64 \
  -o /usr/local/bin/ollama
sudo chmod +x /usr/local/bin/ollama
```

### macOS

**Option 1: Homebrew**
```bash
brew install ollama
```

**Option 2: .dmg (empfohlen für Desktop-Nutzung)**

Download unter [https://ollama.com/download](https://ollama.com/download) – das installiert eine macOS-App mit Menüleisten-Icon und automatischem Start.

Anschließend Dienst starten:
```bash
ollama serve
```

### Windows

1. Installer herunterladen: [https://ollama.com/download/windows](https://ollama.com/download/windows)
2. `OllamaSetup.exe` ausführen
3. Ollama startet automatisch als Hintergrunddienst

> [!WARNING]
> Auf Windows wird CUDA nativ unterstützt. Alternativ kann CUDA auch über WSL2 genutzt werden.

---

## Wichtige Befehle

### Modell starten (interaktiver Chat)

```bash
ollama run llama3.2
```

Beim ersten Aufruf wird das Modell automatisch heruntergeladen. Der Befehl öffnet eine interaktive REPL. Beenden mit `/bye` oder `Ctrl+D`.

### Modell herunterladen (ohne Start)

```bash
ollama pull llama3.2
ollama pull mistral
ollama pull codellama:13b
```

### Alle heruntergeladenen Modelle anzeigen

```bash
ollama list
```

Ausgabe:
```
NAME                    ID              SIZE      MODIFIED
llama3.2:latest         a80c4f17acd5    2.0 GB    2 hours ago
mistral:latest          f974a74358d6    4.1 GB    1 day ago
codellama:13b           9f438cb9cd58    7.4 GB    3 days ago
```

### Modell löschen

```bash
ollama rm mistral
```

### Laufende Modelle anzeigen

```bash
ollama ps
```

Zeigt alle aktuell im Speicher gehaltenen Modelle sowie GPU/CPU-Auslastung:
```
NAME              ID              SIZE      PROCESSOR    UNTIL
llama3.2:latest   a80c4f17acd5    3.8 GB    100% GPU     4 minutes from now
```

### Modellinformationen anzeigen

```bash
ollama show llama3.2
```

Gibt aus: Architektur, Parameter-Anzahl, Quantisierung, System Prompt, Template, Lizenz.

### Modell mit Prompt (nicht-interaktiv)

```bash
ollama run llama3.2 "Erkläre mir Docker in 3 Sätzen."
```

### Eingabe aus Datei/Pipe

```bash
cat mein_dokument.txt | ollama run llama3.2 "Fasse diesen Text zusammen:"
```

---

## Empfohlene Modelle

| Modell | Parameter | Größe (Q4) | VRAM-Bedarf | Verwendung |
|---|---|---|---|---|
| `llama3.2:3b` | 3B | ~2 GB | 4 GB | Schnell, ressourcenschonend |
| `llama3.2:latest` | 3B | ~2.0 GB | 4 GB | Guter Allrounder |
| `llama3.1:70b` | 70B | ~40 GB | 48 GB | Sehr leistungsstark |
| `mistral:latest` | 7B | ~4.1 GB | 8 GB | Gut für Instruktionen |
| `codellama:13b` | 13B | ~7.4 GB | 16 GB | Code-Generierung |
| `deepseek-coder-v2` | 16B | ~9 GB | 16 GB | Code, sehr präzise |
| `phi3:mini` | 3.8B | ~2.3 GB | 4 GB | Microsoft, effizient |
| `gemma2:9b` | 9B | ~5.4 GB | 8 GB | Google, gute Qualität |
| `qwen2.5:14b` | 14B | ~8.2 GB | 12 GB | Mehrsprachig, stark |
| `nomic-embed-text` | – | ~274 MB | 1 GB | [[Embeddings]] für [[RAG Grundlagen|RAG]] |

> [!TIP]
> Mit `ollama run <modell>:latest` wird immer die empfohlene Standard-Quantisierung geladen. Mit z.B. `:q8_0` oder `:fp16` kann explizit eine bestimmte Quantisierung gewählt werden.

---

## Ollama als Systemdienst (systemd)

Die Linux-Installation via curl-Skript legt den systemd-Service automatisch an. Manuelle Verwaltung:

```bash
# Status prüfen
sudo systemctl status ollama

# Dienst starten / stoppen / neustarten
sudo systemctl start ollama
sudo systemctl stop ollama
sudo systemctl restart ollama

# Autostart aktivieren / deaktivieren
sudo systemctl enable ollama
sudo systemctl disable ollama

# Logs anzeigen
sudo journalctl -u ollama -f
```

Die Unit-Datei liegt unter `/etc/systemd/system/ollama.service`. Inhalt für manuelle Anlagen:

```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

[Install]
WantedBy=default.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ollama
```

---

## REST API

Ollama stellt eine HTTP-API auf Port `11434` bereit, die teilweise OpenAI-kompatibel ist.

### Endpunkte

| Methode | Endpunkt | Beschreibung |
|---|---|---|
| `POST` | `/api/generate` | Text-Completion (Streaming) |
| `POST` | `/api/chat` | Chat-Completion mit Verlauf |
| `POST` | `/api/embeddings` | Embeddings generieren |
| `POST` | `/api/pull` | Modell herunterladen |
| `POST` | `/api/push` | Modell hochladen |
| `POST` | `/api/create` | Modell aus Modelfile erstellen |
| `DELETE` | `/api/delete` | Modell löschen |
| `GET` | `/api/tags` | Alle Modelle auflisten |
| `GET` | `/api/ps` | Laufende Modelle |
| `POST` | `/api/show` | Modellinformationen |

### curl-Beispiele

**Einfache Text-Completion:**
```bash
curl http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "prompt": "Was ist der Unterschied zwischen TCP und UDP?",
    "stream": false
  }'
```

**Chat mit Verlauf:**
```bash
curl http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "messages": [
      { "role": "user", "content": "Hallo! Wer bist du?" },
      { "role": "assistant", "content": "Ich bin ein KI-Assistent." },
      { "role": "user", "content": "Kannst du mir Python erklären?" }
    ],
    "stream": false
  }'
```

**Embeddings:**
```bash
curl http://localhost:11434/api/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nomic-embed-text",
    "prompt": "Der schnelle braune Fuchs"
  }'
```

**Modelle auflisten:**
```bash
curl http://localhost:11434/api/tags
```

### Python-Beispiel

```python
import requests
import json

def chat_with_ollama(model: str, message: str) -> str:
    url = "http://localhost:11434/api/chat"
    payload = {
        "model": model,
        "messages": [{"role": "user", "content": message}],
        "stream": False
    }
    response = requests.post(url, json=payload)
    response.raise_for_status()
    return response.json()["message"]["content"]

# OpenAI-kompatibler Endpunkt (via ollama)
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # Pflichtfeld, wird nicht geprüft
)

response = client.chat.completions.create(
    model="llama3.2",
    messages=[
        {"role": "user", "content": "Erkläre Kubernetes in einem Satz."}
    ]
)
print(response.choices[0].message.content)
```

> [!NOTE]
> Die OpenAI-kompatible API ist unter `/v1/` erreichbar und unterstützt `/v1/chat/completions`, `/v1/models` und `/v1/embeddings`. Damit kann Ollama als Drop-in-Ersatz für viele OpenAI-kompatible Tools verwendet werden.

---

## Umgebungsvariablen

| Variable | Standard | Beschreibung |
|---|---|---|
| `OLLAMA_HOST` | `127.0.0.1:11434` | IP und Port, auf dem Ollama lauscht |
| `OLLAMA_MODELS` | `~/.ollama/models` | Pfad zum Modell-Verzeichnis |
| `OLLAMA_KEEP_ALIVE` | `5m` | Wie lange Modelle im VRAM bleiben |
| `OLLAMA_NUM_PARALLEL` | `1` | Anzahl paralleler Anfragen |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | Max. gleichzeitig geladene Modelle |
| `OLLAMA_DEBUG` | `false` | Ausführliches Logging |
| `OLLAMA_FLASH_ATTENTION` | `0` | Flash Attention aktivieren (experimentell) |
| `CUDA_VISIBLE_DEVICES` | alle | Welche GPUs genutzt werden sollen |

### Beispiel: Ollama auf allen Interfaces lauschen lassen

```bash
# Temporär (nur aktuelle Shell)
export OLLAMA_HOST=0.0.0.0:11434
ollama serve

# Dauerhaft via systemd override
sudo systemctl edit ollama
```

Im Editor folgenden Inhalt eintragen:
```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_MODELS=/data/ollama/models"
Environment="OLLAMA_KEEP_ALIVE=10m"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

> [!WARNING]
> `OLLAMA_HOST=0.0.0.0` macht die API für alle Netzwerkinterfaces zugänglich. Im lokalen Netz oder Server-Umfeld unbedingt mit einer Firewall oder einem Reverse Proxy absichern – Ollama hat keine eingebaute Authentifizierung.

---

## GPU-Nutzung

### NVIDIA (CUDA)

```bash
# CUDA-Version prüfen
nvidia-smi

# Ollama-Logs auf GPU-Nutzung prüfen
sudo journalctl -u ollama | grep -i gpu
```

Ollama erkennt NVIDIA-GPUs automatisch, wenn CUDA-Treiber installiert sind. Kein zusätzliches Setup erforderlich.

```bash
# Spezifische GPU auswählen
CUDA_VISIBLE_DEVICES=1 ollama run llama3.2
```

### AMD (ROCm)

```bash
# ROCm-Version prüfen
rocminfo | grep "Agent"

# Für ROCm-Unterstützung: Ollama mit ROCm-Variante
# Das curl-Installationsskript erkennt ROCm automatisch
```

> [!NOTE]
> ROCm wird ab AMD RX 6000-Serie (RDNA2+) und Radeon Pro W-Serie unterstützt. Ältere GPUs (RDNA1 und früher) haben nur eingeschränkte oder inoffizielle Unterstützung und laufen ggf. nur auf CPU.

### Apple Silicon (Metal)

Auf macOS mit M1/M2/M3/M4 wird Metal automatisch genutzt – kein Setup erforderlich. Der Unified Memory erlaubt es, sehr große Modelle effizient zu laden.

### CPU-only Fallback

Wenn keine GPU erkannt wird, läuft Ollama automatisch auf der CPU. Deutlich langsamer, aber funktional.

```bash
# Erzwingen, dass nur CPU genutzt wird
OLLAMA_NUM_GPU=0 ollama run llama3.2
```

---

## Modelfile: Eigenes Modell mit System Prompt

Ein Modelfile definiert Basis-Modell, System Prompt, Template und Parameter für ein angepasstes Modell.

```dockerfile
# Datei: Modelfile
FROM llama3.2

# System Prompt
SYSTEM """
Du bist ein erfahrener Linux-Systemadministrator.
Antworte immer auf Deutsch.
Gib präzise, ausführliche technische Antworten.
Nutze Codeblöcke für Befehle und Konfigurationsdateien.
"""

# Parameter anpassen
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER num_ctx 8192
PARAMETER stop "<|eot_id|>"
```

```bash
# Modell aus Modelfile erstellen
ollama create mein-linux-assistent -f ./Modelfile

# Modell testen
ollama run mein-linux-assistent

# Im Ollama Hub veröffentlichen (optionales Konto erforderlich)
ollama push benutzername/mein-linux-assistent
```

### Verfügbare Modelfile-Parameter

| Parameter | Beschreibung |
|---|---|
| `temperature` | Kreativität (0.0 = deterministisch, 1.0 = kreativ) |
| `top_p` | Nucleus Sampling Schwellenwert |
| `top_k` | Top-K Sampling |
| `num_ctx` | Kontextfenstergröße in Tokens |
| `num_predict` | Max. generierte Tokens (-1 = unbegrenzt) |
| `repeat_penalty` | Strafe für Wiederholungen |
| `seed` | Fixer Seed für reproduzierbare Ausgaben |
| `stop` | Stop-Sequenz(en) |

---

## Troubleshooting

### Ollama startet nicht

```bash
# Läuft der Dienst?
sudo systemctl status ollama

# Port bereits belegt?
sudo ss -tlnp | grep 11434

# Logs prüfen
sudo journalctl -u ollama -n 50 --no-pager
```

### Modell-Download schlägt fehl

```bash
# Netzwerkverbindung testen
curl -I https://ollama.com

# Proxy setzen falls nötig
export HTTPS_PROXY=http://proxy.example.com:8080
ollama pull llama3.2
```

### GPU wird nicht erkannt

```bash
# NVIDIA: Treiber-Status
nvidia-smi

# CUDA-Libraries prüfen
ldconfig -p | grep libcuda

# Ollama muss nach Treiberinstallation neu gestartet werden
sudo systemctl restart ollama
```

### Out of Memory (OOM)

```bash
# Kleinere Quantisierung wählen
ollama pull llama3.2:3b         # Kleines Modell
ollama pull llama3.1:8b-q4_0   # Explizit Q4-Quantisierung

# Kontextfenster verringern
ollama run llama3.2 --num-ctx 2048

# Keep-Alive auf 0 setzen, damit Modell sofort aus VRAM entladen wird
OLLAMA_KEEP_ALIVE=0 ollama serve
```

### Langsame Antworten

```bash
# Prüfen, ob GPU genutzt wird
ollama ps  # Spalte "PROCESSOR" zeigt GPU/CPU

# Modell ist zu groß für VRAM → teilweise CPU-Offloading
# Lösung: Kleineres Modell oder mehr VRAM

# Flash Attention aktivieren (experimentell, schneller bei großem Kontext)
OLLAMA_FLASH_ATTENTION=1 ollama serve
```

### API nicht erreichbar von anderen Hosts

```bash
# Ollama lauscht standardmäßig nur auf 127.0.0.1
# Ändern via Umgebungsvariable:
OLLAMA_HOST=0.0.0.0:11434 ollama serve

# Firewall-Regel prüfen
sudo ufw allow 11434/tcp
```

> [!IMPORTANT]
> Ollama-Modelle werden unter `~/.ollama/models` gespeichert. Bei mehreren großen Modellen kann dies schnell viel Speicherplatz belegen. Mit `OLLAMA_MODELS` kann ein anderes Verzeichnis (z.B. auf einer separaten Festplatte) angegeben werden.

---

## Verwandte Notizen

- [[Lokale LLMs]] — Übersicht und Vergleich lokal betriebener Sprachmodelle
- [[Open WebUI]] — Chat-Oberfläche, die Ollama als Backend nutzt
- [[AnythingLLM]] — RAG-fokussierte Anwendung mit Ollama-Integration
- [[Fine-Tuning Grundlagen]] — Grundlagen zum Anpassen und Trainieren von Modellen
- [[Modell Übersicht]] — Vergleich verfügbarer LLM-Architekturen und Modelle
