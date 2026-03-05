# ComfyUI

## Was ist ComfyUI?

ComfyUI ist eine node-basierte, grafische Benutzeroberfläche für Stable Diffusion und andere Diffusionsmodelle. Anstatt ein einfaches Formular mit Einstellungen zu bieten, stellt ComfyUI einen Workflow-Editor bereit, in dem einzelne Verarbeitungsschritte (Nodes) visuell miteinander verbunden werden. Jeder Node repräsentiert eine konkrete Operation im Generierungsprozess – Modell laden, Text einlesen, Sampling, VAE-Dekodierung, Bild ausgeben.

Workflows werden als JSON gespeichert und können geteilt, importiert und modifiziert werden.

---

## Unterschied zu AUTOMATIC1111 / Forge

| Merkmal | ComfyUI | AUTOMATIC1111 / Forge |
|---|---|---|
| Interface-Paradigma | Node-Graph (visuell) | Formular-basiert (Tabs) |
| Einstiegshürde | Höher | Niedriger |
| Flexibilität | Sehr hoch – jeder Schritt anpassbar | Begrenzt auf Extension-Ökosystem |
| Performance | Sehr gut, inkrementelles Caching | Gut |
| Geschwindigkeit | Nur geänderte Nodes werden neu ausgeführt | Immer vollständiger Durchlauf |
| SDXL-Unterstützung | Vollständig, nativ | Vollständig, nativ |
| Flux.1-Unterstützung | Vollständig, beste Unterstützung | Begrenzt / via Extensions |
| API-Zugang | JSON-Workflow via HTTP-API | REST API (einfacher) |
| Workflow-Sharing | JSON-Dateien / Workflow als PNG-Metadaten | Schwieriger |
| Custom Nodes | Sehr aktives Ökosystem | Extensions |

> [!NOTE]
> ComfyUI ist die Referenzimplementierung für viele neue Modell-Architekturen (Flux.1, SD3, SDXL, ControlNet-Workflows). Neue Modelle werden oft zuerst für ComfyUI verfügbar. Für einfache Nutzung ist AUTOMATIC1111 zugänglicher, für präzise Kontrolle und neue Modelle ist ComfyUI die bessere Wahl.

---

## Voraussetzungen

### NVIDIA GPU (empfohlen)

| Anforderung | Minimum | Empfohlen |
|---|---|---|
| VRAM | 4 GB | 8–16 GB |
| GPU | GTX 1060 | RTX 3060+ |
| CUDA | 11.8+ | 12.1+ |
| RAM | 8 GB | 16 GB |
| Speicher | 20 GB | 100+ GB |

```bash
# CUDA-Version prüfen
nvidia-smi
nvcc --version

# Python-Version (3.10–3.12 empfohlen)
python3 --version
```

### AMD GPU (ROCm, Linux only)

ROCm 5.7+ für RX 5000-Serie und neuer. Instabiler als CUDA.

### Apple Silicon (Metal)

M1/M2/M3/M4 werden über `mps` (Metal Performance Shaders) unterstützt. Langsamere Inferenz als NVIDIA, aber Unified Memory erlaubt größere Modelle.

### CPU-only

Sehr langsam, aber möglich. Generierung dauert mehrere Minuten pro Bild.

---

## Installation: Manuell (alle Plattformen)

### 1. Repository klonen

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
```

### 2. Python-Umgebung einrichten

```bash
# Virtuelle Umgebung erstellen (empfohlen)
python3 -m venv venv
source venv/bin/activate      # Linux/macOS
# oder: venv\Scripts\activate  # Windows

# Pip aktualisieren
pip install --upgrade pip
```

### 3. PyTorch installieren

```bash
# NVIDIA CUDA 12.1
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# NVIDIA CUDA 11.8
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

# AMD ROCm 6.0 (Linux)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.0

# Apple Silicon (MPS)
pip install torch torchvision torchaudio

# CPU only
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
```

### 4. ComfyUI-Abhängigkeiten installieren

```bash
pip install -r requirements.txt
```

### 5. ComfyUI starten

```bash
python main.py

# Mit spezifischen Optionen:
python main.py --listen 0.0.0.0  # Von anderen Hosts erreichbar
python main.py --port 8188       # Standard-Port (änderbar)
python main.py --cpu             # CPU erzwingen (kein GPU)
python main.py --lowvram         # VRAM-sparender Modus (< 4 GB VRAM)
python main.py --novram          # Minimaler VRAM-Modus
python main.py --fp16            # Halbpräzision (spart VRAM)
python main.py --bf16            # BF16 (für Ampere+ / M1+)
```

ComfyUI ist dann unter `http://localhost:8188` erreichbar.

---

## Installation: Portabel (Windows, vorkompiliert)

Für Windows ohne manuelle Python/CUDA-Installation:

1. Download von [https://github.com/comfyanonymous/ComfyUI/releases](https://github.com/comfyanonymous/ComfyUI/releases)
2. `ComfyUI_windows_portable_nvidia.7z` (NVIDIA) oder `..._cpu.7z` herunterladen
3. Archiv entpacken (z.B. nach `C:\ComfyUI`)
4. `run_nvidia_gpu.bat` (oder `run_cpu.bat`) ausführen

> [!TIP]
> Die portable Version enthält bereits eine fertige Python-Umgebung mit PyTorch. Updates werden durch erneutes Herunterladen der neuesten Release oder durch `git pull` im Unterordner `ComfyUI/` eingespielt.

---

## Erste Schritte: Modell herunterladen und einrichten

### Verzeichnisstruktur

```
ComfyUI/
├── models/
│   ├── checkpoints/    ← Haupt-Modelle (SD 1.5, SDXL, Flux, etc.)
│   ├── loras/          ← LoRA-Gewichte
│   ├── vae/            ← VAE-Modelle
│   ├── controlnet/     ← ControlNet-Modelle
│   ├── clip/           ← CLIP/Text-Encoder
│   ├── unet/           ← UNet-Modelle (Flux-Format)
│   ├── embeddings/     ← Textual Inversion Embeddings
│   └── upscale_models/ ← Upscaling-Modelle (ESRGAN, etc.)
├── custom_nodes/       ← Installierte Custom Nodes
├── input/              ← Eingabebilder für img2img etc.
├── output/             ← Generierte Bilder
└── workflows/          ← Gespeicherte Workflows
```

### Modell herunterladen (Beispiel: SDXL Base)

```bash
# Direkt in das checkpoints-Verzeichnis
cd ComfyUI/models/checkpoints

# SD 1.5 (klassisch, 2 GB)
wget https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.ckpt

# SDXL Base 1.0 (6.5 GB)
wget https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors

# Mit Hugging Face CLI (empfohlen für große Modelle)
pip install huggingface_hub
huggingface-cli download stabilityai/stable-diffusion-xl-base-1.0 \
  sd_xl_base_1.0.safetensors \
  --local-dir ./
```

---

## Grundlegender Workflow: Text-to-Image

Der Standard-Workflow in ComfyUI nach dem Start:

```
[Load Checkpoint] ──────────────────────────────────────┐
                    MODEL ──────────────────────► [KSampler]
                    CLIP ─► [CLIP Text Encode (+)] ──► POSITIVE ─► [KSampler]
                    CLIP ─► [CLIP Text Encode (-)] ──► NEGATIVE ─► [KSampler]
                    VAE ─────────────────────────────────────────► [VAE Decode]

[Empty Latent Image] ──────────────────────────► LATENT ► [KSampler]
                                                           LATENT ► [VAE Decode]
                                                                     IMAGE ► [Save Image]
```

### Schritt-für-Schritt

1. ComfyUI starten, `http://localhost:8188` öffnen
2. Der Standard-Workflow ist bereits geladen
3. **Load Checkpoint** Node: Modell aus Dropdown wählen (muss in `models/checkpoints/` liegen)
4. **CLIP Text Encode (Positive Prompt)**: Positiven Prompt eingeben
5. **CLIP Text Encode (Negative Prompt)**: Negativen Prompt eingeben
6. **Empty Latent Image**: Bildgröße festlegen (z.B. 512x512 für SD1.5, 1024x1024 für SDXL)
7. **KSampler**: Steps, CFG-Scale, Sampler, Scheduler anpassen
8. **Queue Prompt** (oben rechts) oder `Ctrl+Enter`

---

## Wichtige Node-Typen

### KSampler

Führt den eigentlichen Diffusionsprozess durch.

| Parameter | Beschreibung | Typischer Wert |
|---|---|---|
| `seed` | Seed für Reproduzierbarkeit (-1 = zufällig) | beliebig |
| `steps` | Anzahl Denoising-Schritte | 20–30 |
| `cfg` | Classifier-Free Guidance (Prompt-Treue) | 5–8 für SD, 3.5 für Flux |
| `sampler_name` | Sampling-Algorithmus | `euler`, `dpmpp_2m` |
| `scheduler` | Noise-Schedule | `karras`, `exponential` |
| `denoise` | Stärke des Denoisings (1.0 = komplett) | 1.0 für txt2img |

### CLIP Text Encode

Wandelt Text-Prompts in CLIP-Embeddings um. Zwei Instanzen: eine für den positiven, eine für den negativen Prompt.

### VAE Encode / VAE Decode

- **VAE Encode:** Bild in latenten Raum komprimieren (für img2img)
- **VAE Decode:** Latentes Bild in sichtbares Pixel-Bild umwandeln

### Load Checkpoint

Lädt ein Checkpoint-Modell und gibt `MODEL`, `CLIP` und `VAE` aus.

### Load LoRA

Lädt eine LoRA und modifiziert `MODEL` und `CLIP`.

```
[Load Checkpoint] ─► MODEL ─► [Load LoRA] ─► MODEL (modifiziert) ─► [KSampler]
                    CLIP  ─► [Load LoRA] ─► CLIP  (modifiziert) ─► [CLIP Text Encode]
```

### Empty Latent Image / Load Image

- **Empty Latent Image:** Erstellt leeren latenten Raum für txt2img
- **Load Image:** Lädt Bild für img2img oder Inpainting

### Save Image / Preview Image

- **Save Image:** Speichert das generierte Bild in `output/`
- **Preview Image:** Zeigt Bild im Interface ohne Speichern

### Image Upscale (über Modell oder Pixel)

```
[Save Image] ◄── [Image Scale] ◄── [Upscale Model Loader] + [Upscale Image (using Model)]
```

---

## Empfohlene Custom Nodes

Custom Nodes erweitern ComfyUI um zusätzliche Nodes und Funktionen. Sie werden in `custom_nodes/` installiert.

### ComfyUI Manager (Node-Verwaltung)

Unverzichtbar – ermöglicht das Installieren, Aktualisieren und Deinstallieren von Custom Nodes direkt aus dem Interface.

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
```

Nach ComfyUI-Neustart erscheint ein **Manager**-Button in der Menüleiste. Dort können alle weiteren Nodes gesucht und installiert werden.

> [!IMPORTANT]
> ComfyUI Manager zuerst installieren, da danach alle anderen Nodes über die UI verwaltet werden können.

### ComfyUI-Impact-Pack

Erweiterte Nodes für Gesichtserkennung, Segmentierung (SAM), Detailer und Inpainting-Workflows.

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Impact-Pack.git
cd ComfyUI-Impact-Pack
pip install -r requirements.txt
```

Enthält u.a.:
- **FaceDetailer** – automatisches Gesichts-Upscaling und Nachbearbeitung
- **SAMDetector** – Segment Anything Model für präzise Masken
- **BboxDetector / SegmDetector** – Objekt-Erkennung und Masken-Generierung

### WAS Node Suite

Große Sammlung von Utility-Nodes für Bildbearbeitung, Masken, Texte, Dateisystem-Operationen und mehr.

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/WASasquatch/was-node-suite-comfyui.git
cd was-node-suite-comfyui
pip install -r requirements.txt
```

Enthält u.a.:
- **Image Blend**, **Image Composite** – Bilder zusammenführen
- **Text** – Dynamische Texte, Datei lesen, String-Operationen
- **Mask Expand / Feather** – Masken-Bearbeitung
- **Image Save** – Erweiterter Speichern-Node mit Metadaten

### Weitere empfohlene Custom Nodes

| Node | Beschreibung | GitHub |
|---|---|---|
| `ComfyUI-Advanced-ControlNet` | Erweiterte ControlNet-Features | `Kosinkadink/ComfyUI-Advanced-ControlNet` |
| `ComfyUI_IPAdapter_plus` | IP-Adapter für Stil-Transfer | `cubiq/ComfyUI_IPAdapter_plus` |
| `ComfyUI-AnimateDiff-Evolved` | Video/Animation-Generierung | `Kosinkadink/ComfyUI-AnimateDiff-Evolved` |
| `ComfyUI-VideoHelperSuite` | Video-Utilities | `Kosinkadink/ComfyUI-VideoHelperSuite` |
| `efficiency-nodes-comfyui` | Effizientere Nodes, weniger Boilerplate | `jags111/efficiency-nodes-comfyui` |
| `ComfyUI_Comfyroll_CustomNodes` | Sehr viele Utility-Nodes | `Suzie1/ComfyUI_Comfyroll_CustomNodes` |

---

## Wichtige Modell-Typen

| Typ | Ordner | Beschreibung |
|---|---|---|
| Checkpoint | `models/checkpoints/` | Vollständiges Modell (UNet + CLIP + VAE). `.safetensors` oder `.ckpt` |
| LoRA | `models/loras/` | Kleine Gewichts-Anpassung für Stil/Charakter. Wird auf Checkpoint addiert |
| VAE | `models/vae/` | Variational Autoencoder. Für bessere Farben / Details (oft `vae-ft-mse`) |
| ControlNet | `models/controlnet/` | Steuert Pose, Tiefe, Kanten im Bild |
| CLIP | `models/clip/` | Text-Encoder (separat bei Flux) |
| UNet | `models/unet/` | Diffusions-Netzwerk (separat bei Flux) |
| Embedding | `models/embeddings/` | Textual Inversion (`.pt` oder `.safetensors`) |
| Upscale | `models/upscale_models/` | ESRGAN, RealESRGAN für Hochskalierung |
| CLIP Vision | `models/clip_vision/` | Für IP-Adapter |

---

## Wo Modelle herunterladen

### CivitAI

[https://civitai.com](https://civitai.com) – größte Community-Plattform für Checkpoints und LoRAs.

```bash
# CivitAI Download (API-Key erforderlich für manche Modelle)
# Model-ID aus der URL entnehmen (z.B. civitai.com/models/4201)
wget "https://civitai.com/api/download/models/128713" \
  -O models/checkpoints/dreamshaper_8.safetensors

# Mit API-Key (für authentifizierte Downloads)
wget "https://civitai.com/api/download/models/128713?token=DEIN_API_KEY" \
  -O models/checkpoints/dreamshaper_8.safetensors
```

### Hugging Face

[https://huggingface.co](https://huggingface.co) – offizielle Modell-Hub von Stability AI und anderen.

```bash
pip install huggingface_hub

# SDXL Base
huggingface-cli download stabilityai/stable-diffusion-xl-base-1.0 \
  sd_xl_base_1.0.safetensors \
  --local-dir ComfyUI/models/checkpoints/

# Flux.1 Dev (Registrierung + Gated Repo erforderlich)
huggingface-cli login  # HF-Token eingeben
huggingface-cli download black-forest-labs/FLUX.1-dev \
  flux1-dev.safetensors \
  --local-dir ComfyUI/models/unet/

# VAE für SDXL
huggingface-cli download madebyollin/sdxl-vae-fp16-fix \
  diffusion_pytorch_model.safetensors \
  --local-dir ComfyUI/models/vae/ \
  --local-dir-use-symlinks False
```

> [!WARNING]
> Modelle von CivitAI haben unterschiedliche Lizenzen. Vor der kommerziellen Nutzung die jeweilige Lizenz prüfen. Flux.1 Dev z.B. ist nur für nicht-kommerzielle Nutzung lizenziert.

---

## SDXL vs. SD 1.5 vs. Flux.1 Modelle

### Stable Diffusion 1.5 (SD1.5)

- **Ausgabe-Auflösung:** 512×512 px nativ
- **Checkpoint-Größe:** ~2 GB (fp16)
- **VRAM-Bedarf:** 4–6 GB
- **Stärken:** Riesiges Ökosystem, viele LoRAs/ControlNets, schnell
- **Schwächen:** Veraltete Architektur, schlechtere Qualität bei hohen Auflösungen
- **Modell-Ordner:** `models/checkpoints/`
- **Typische Auflösung im Workflow:** 512×512 bis 768×768

### Stable Diffusion XL (SDXL)

- **Ausgabe-Auflösung:** 1024×1024 px nativ
- **Checkpoint-Größe:** ~6.5 GB (Base) + ~6 GB (Refiner, optional)
- **VRAM-Bedarf:** 8–12 GB
- **Stärken:** Deutlich bessere Qualität, Zwei-Stufen-Prozess (Base + Refiner)
- **Schwächen:** Größer, langsamer als SD1.5
- **Modell-Ordner:** `models/checkpoints/`
- **Typische Auflösung:** 1024×1024

### Flux.1

- **Ausgabe-Auflösung:** Flexibel, sehr gut bei verschiedenen Seitenverhältnissen
- **Varianten:** `Flux.1-dev` (non-commercial), `Flux.1-schnell` (Apache 2.0, schneller)
- **Checkpoint-Größe:** ~12–24 GB (je nach Quantisierung)
- **VRAM-Bedarf:** 12–24 GB (fp16), 8 GB+ mit fp8/nf4-Quantisierung
- **Stärken:** Beste Qualität aktuell, sehr gute Textdarstellung im Bild, Photorealismus
- **Schwächen:** Sehr groß, andere Architektur (kein klassischer U-Net-Stack), weniger LoRAs
- **Modell-Ordner:** `models/unet/` (UNet separat), `models/clip/` (T5 + CLIP)
- **CFG:** Flux nutzt kein klassisches CFG (cfg = 1.0), stattdessen Guidance Distillation

> [!NOTE]
> Flux.1 hat eine andere Architektur als SD1.5/SDXL. Es benötigt separate CLIP und T5-Encoder-Dateien. Ein vollständiges Flux-Setup besteht aus: `flux1-dev.safetensors` (UNet), `clip_l.safetensors`, `t5xxl_fp16.safetensors`, und einem VAE.

---

## Workflow speichern und laden

### Workflow als JSON speichern

1. Oben rechts: **Save** (oder `Ctrl+S`) → speichert als `.json`-Datei
2. Workflow wird standardmäßig im Browser-Speicher gehalten

### Workflow als PNG mit Metadaten

ComfyUI bettet den Workflow automatisch als Metadaten in alle generierten PNG-Bilder ein.

```bash
# Workflow aus PNG extrahieren (Python)
from PIL import Image
import json

img = Image.open("output/ComfyUI_00001_.png")
workflow_json = img.info.get("workflow") or img.info.get("prompt")
print(json.loads(workflow_json))
```

Um ein Bild mit eingebettetem Workflow zu laden:

1. Bild in den ComfyUI-Browser-Tab ziehen (Drag & Drop)
2. ComfyUI lädt automatisch den eingebetteten Workflow

### Workflow importieren

1. Oben rechts: **Load** → JSON-Datei wählen
2. Oder: JSON-Datei per Drag & Drop in den Canvas ziehen

> [!TIP]
> Auf [https://openart.ai/workflows](https://openart.ai/workflows) und [https://comfyworkflows.com](https://comfyworkflows.com) gibt es tausende vorgefertigte Workflows zum Download – ideal zum Lernen und als Ausgangspunkt für eigene Anpassungen.

### ComfyUI aktualisieren

```bash
cd /pfad/zu/ComfyUI
git pull

# Virtuelle Umgebung aktivieren
source venv/bin/activate

# Abhängigkeiten aktualisieren (falls neue in requirements.txt)
pip install -r requirements.txt

# Custom Nodes aktualisieren (mit ComfyUI Manager: Manager → Update All)
# Oder manuell:
for dir in custom_nodes/*/; do
  echo "Updating $dir..."
  git -C "$dir" pull 2>/dev/null || true
done
```
