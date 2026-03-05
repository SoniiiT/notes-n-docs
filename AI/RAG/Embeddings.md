# Embeddings

## Was sind Embeddings?

Ein **Embedding** ist eine numerische Darstellung von Text (oder anderen Daten) als hochdimensionaler Vektor. Die zentrale Idee: **semantisch ähnliche Texte erzeugen ähnliche Vektoren**.

```
"König"     → [0.24, -0.87, 0.12, 0.65, ...]   (z.B. 1536 Zahlen)
"Monarch"   → [0.23, -0.84, 0.14, 0.63, ...]   ← ähnlich!
"Urlaubstag" → [-0.45, 0.23, -0.67, 0.12, ...]  ← sehr verschieden
```

Der Abstand zwischen zwei Vektoren im mehrdimensionalen Raum entspricht dem **semantischen Abstand** zwischen den Texten. Das ermöglicht:
- Semantische Suche (finde ähnliche Dokumente)
- Clustering (gruppiere ähnliche Texte)
- Klassifikation (ordne Texte Kategorien zu)

### Bedeutung als Geometrie

In der Realität haben Embedding-Vektoren **768–3072 Dimensionen** — für Menschen nicht visualisierbar. Das Prinzip lässt sich aber auf 2D herunterbrechen:

```
        Tier
         ↑
  Katze  •  • Hund
         |
         |        • Auto
  ───────┼──────────→ Technik
         |
         |  • Computer
```

Semantisch verwandte Begriffe (Katze, Hund) liegen nah beieinander, unverwandte (Katze, Computer) weit auseinander. In höheren Dimensionen funktioniert das gleiche Prinzip — nur mit deutlich feineren Unterscheidungen.

**Ähnlichkeit als Zahl** — Cosine Similarity zwischen Wort-Paaren:

| Wort-Paar | Cosine Similarity | Interpretation |
|---|---|---|
| "Katze" ↔ "Hund" | 0.92 | Sehr ähnlich (beides Haustiere) |
| "Katze" ↔ "Tiger" | 0.85 | Ähnlich (beides Katzenartige) |
| "Katze" ↔ "Auto" | 0.15 | Kaum Zusammenhang |
| "König" ↔ "Königin" | 0.89 | Ähnlich (gleiche Rolle, anderes Geschlecht) |

### Vektor-Arithmetik

Embedding-Vektoren können **semantische Beziehungen** durch Arithmetik abbilden. Das berühmte Beispiel aus Word2Vec:

```
König - Mann + Frau ≈ Königin
```

| Wort | Vektor (vereinfacht, 5D) | Kodierte Merkmale |
|---|---|---|
| "König" | [0.8, 0.2, 0.9, -0.1, 0.5] | Mensch, männlich, Macht |
| "Königin" | [0.8, 0.2, 0.9, 0.7, 0.5] | Mensch, weiblich, Macht |
| "Mann" | [0.7, 0.1, 0.1, -0.1, 0.3] | Mensch, männlich |
| "Frau" | [0.7, 0.1, 0.1, 0.7, 0.3] | Mensch, weiblich |

```
  [0.8, 0.2, 0.9, -0.1, 0.5]   (König)
- [0.7, 0.1, 0.1, -0.1, 0.3]   (Mann abziehen)
+ [0.7, 0.1, 0.1,  0.7, 0.3]   (Frau addieren)
= [0.8, 0.2, 0.9,  0.7, 0.5]   ≈ Königin
```

Das zeigt: Embeddings kodieren keine Wörter, sondern **Konzepte und deren Beziehungen**.

> [!IMPORTANT]
> Embeddings sind **keine Magie** – sie sind komprimierte statistische Repräsentationen, die aus riesigen Textmengen gelernt wurden. Sie verstehen keine Bedeutung im menschlichen Sinne, aber sie kodieren statistische Zusammenhänge zwischen Wörtern und Konzepten extrem gut.

---

## Wie Embeddings funktionieren (vereinfacht)

Ein Embedding-Modell ist ein neuronales Netz (meist Transformer-basiert), das einen Text einliest und einen festen Vektor ausgibt.

```
Input:  "Der Urlaubsanspruch beträgt 30 Tage"
         ↓
[Tokenizer: Text → Token-IDs]
         ↓
[Transformer-Layer (mehrfach)]
[Self-Attention, Feed-Forward]
         ↓
[Pooling: alle Token-Vektoren → 1 Vektor]
(z.B. Mean-Pooling oder [CLS]-Token)
         ↓
Output: [0.23, -0.45, 0.12, 0.67, ...] (1536 Zahlen)
```

**Trainiert wird** das Modell darauf, dass Texte mit ähnlicher Bedeutung ähnliche Vektoren bekommen – zum Beispiel durch Contrastive Learning auf Paaren von ähnlichen und unähnlichen Texten.

---

## Similarity-Metriken

### Cosine Similarity

Misst den **Winkel** zwischen zwei Vektoren. Ist unabhängig von der Vektorgröße (Magnitude).

```
cosine_similarity(A, B) = (A · B) / (|A| × |B|)

Wertebereich: -1 (entgegengesetzt) bis +1 (identisch)
Typischer RAG-Bereich: 0.7 – 1.0 = relevant
```

```python
import numpy as np

def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Wenn Vektoren bereits normalisiert (Länge = 1):
# cosine_similarity = np.dot(a, b)  ← viel schneller!
```

**Wann verwenden:** Standard für die meisten RAG-Anwendungen. Empfohlen wenn Embeddings nicht normalisiert sind.

### Dot Product (Inneres Produkt)

Bei **normalisierten** Vektoren identisch mit Cosine Similarity. Ohne Normalisierung berücksichtigt es auch die Vektorgröße.

```python
dot_product = np.dot(a, b)
# Bei normalisierten Vektoren: dot_product == cosine_similarity
```

**Wann verwenden:** Wenn Vektoren normalisiert sind (schneller, kein Divisionsoverhead). Pinecone und viele andere Datenbanken empfehlen normalisierte Vektoren + Dot Product.

### Euclidean Distance (L2)

Misst den **direkten Abstand** im mehrdimensionalen Raum.

```
euclidean_distance(A, B) = sqrt(Σ(Aᵢ - Bᵢ)²)

Wertebereich: 0 (identisch) bis ∞ (sehr verschieden)
```

```python
euclidean_distance = np.linalg.norm(a - b)
similarity = 1 / (1 + euclidean_distance)  # Umrechnung in 0-1 Bereich
```

**Wann verwenden:** Bei nicht-normalisierten Vektoren oder wenn die Magnitude semantisch relevant ist. Für RAG meist schlechter als Cosine Similarity.

### Vergleich der Metriken

| Metrik            | Berücksichtigt Magnitude | Wertebereich     | Empfehlung für RAG |
| ----------------- | ------------------------ | ---------------- | ------------------- |
| Cosine Similarity | Nein                     | -1 bis +1        | Ja (Standard)       |
| Dot Product       | Ja                       | -∞ bis +∞        | Ja (wenn normalisiert) |
| Euclidean (L2)    | Ja                       | 0 bis ∞          | Eher nein           |

> [!TIP]
> Normalisiere deine Embeddings vor dem Speichern: `embedding = embedding / np.linalg.norm(embedding)`. Dann sind Cosine Similarity und Dot Product identisch, aber Dot Product ist schneller zu berechnen.

---

## Embedding-Modelle im Vergleich

### OpenAI text-embedding-3-small / large

```python
from openai import OpenAI

client = OpenAI(api_key="sk-...")

response = client.embeddings.create(
    input="Der Urlaubsanspruch beträgt 30 Tage.",
    model="text-embedding-3-small"   # oder "text-embedding-3-large"
)
embedding = response.data[0].embedding
print(f"Dimensionen: {len(embedding)}")  # 1536 (small) / 3072 (large)
```

**text-embedding-3-small:**
- 1536 Dimensionen (reduzierbar auf weniger)
- $0.02 / 1M Tokens
- Ausgezeichnete Balance aus Kosten und Qualität
- Unterstützt Matryoshka Representation Learning (MRL) – Dimensionen reduzierbar

**text-embedding-3-large:**
- 3072 Dimensionen
- $0.13 / 1M Tokens
- Höchste Qualität im OpenAI-Portfolio
- Sinnvoll nur für sehr anspruchsvolle Anwendungsfälle

> [!NOTE]
> Mit OpenAI's **Matryoshka Representation Learning (MRL)** kann man die Dimensionen reduzieren, ohne das Modell neu zu laden. Nützlich für Speicheroptimierung bei kleinen Performance-Einbußen.

```python
# Dimensionen reduzieren mit MRL:
response = client.embeddings.create(
    input="...",
    model="text-embedding-3-small",
    dimensions=512  # Statt 1536 – spart Speicher, minimale Qualitätseinbuße
)
```

### Sentence Transformers (Open-Source)

Eine Python-Bibliothek mit hunderten vortrainierten Modellen. Vollständig lokal, keine API-Kosten.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-small-en-v1.5")
embeddings = model.encode(
    ["Text 1", "Text 2"],
    normalize_embeddings=True,  # Normalisierung direkt beim Encoding
    batch_size=32,              # Batch-Größe für Effizienz
    show_progress_bar=True
)
```

**Wichtige Modelle:**
- `all-MiniLM-L6-v2` – Klein, schnell, gute Qualität (384 Dim)
- `all-mpnet-base-v2` – Bessere Qualität, etwas langsamer (768 Dim)
- `BAAI/bge-small-en-v1.5` – Sehr gute Qualität, klein (384 Dim)
- `BAAI/bge-large-en-v1.5` – Top-Qualität lokal (1024 Dim)
- `intfloat/multilingual-e5-large` – Mehrsprachig, sehr gut (1024 Dim)

### Voyage AI (Anthropic-Empfehlung)

Voyage AI wird von Anthropic für Claude-basierte RAG-Systeme empfohlen, da die Modelle für Retrieval-Aufgaben optimiert sind.

```python
import voyageai

vo = voyageai.Client(api_key="pa-...")

# Dokumente embedden:
doc_embeddings = vo.embed(
    texts=["Dokument 1...", "Dokument 2..."],
    model="voyage-3",
    input_type="document"  # "document" für Chunks, "query" für Suchanfragen
).embeddings

# Query embedden:
query_embedding = vo.embed(
    texts=["Suchanfrage"],
    model="voyage-3",
    input_type="query"
).embeddings[0]
```

> [!NOTE]
> Voyage AI unterscheidet zwischen `input_type="document"` und `input_type="query"`. Diese Asymmetrie verbessert die Retrieval-Qualität messbar – Queries und Dokumente werden unterschiedlich optimiert.

**Voyage-Modelle:**
- `voyage-3` – Bestes Allround-Modell (1024 Dim, $0.06/1M Tokens)
- `voyage-3-lite` – Günstiger, etwas schlechter ($0.02/1M Tokens)
- `voyage-code-3` – Spezialisiert für Code-Retrieval
- `voyage-finance-2` – Spezialisiert für Finanzdomäne

### Nomic Embed

Nomic AI's Open-Source-Modell mit sehr gutem Qualität-Kosten-Verhältnis.

```python
from sentence_transformers import SentenceTransformer

# Lokale Nutzung:
model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5", trust_remote_code=True)

# Wichtig: Präfix für Retrieval-Tasks hinzufügen:
documents = ["search_document: " + doc for doc in docs]
queries = ["search_query: " + q for q in queries]

doc_embeddings = model.encode(documents, normalize_embeddings=True)
query_embedding = model.encode(queries, normalize_embeddings=True)
```

**Besonderheit:** Unterstützt MRL (Matryoshka Representation Learning) – Dimensionen 64 bis 768 wählbar. Gute Wahl für lokale Systeme.

### BGE (BAAI General Embeddings)

Von der Beijing Academy of Artificial Intelligence entwickelt. Oft ganz oben auf MTEB-Benchmarks.

```python
from sentence_transformers import SentenceTransformer

# BGE-Modelle brauchen einen Prompt-Prefix für Queries:
model = SentenceTransformer("BAAI/bge-large-en-v1.5")

# Queries:
query_prefix = "Represent this sentence for searching relevant passages: "
query_embedding = model.encode(query_prefix + "Wie viele Urlaubstage?")

# Dokumente (kein Prefix nötig):
doc_embedding = model.encode("§5: 30 Tage Jahresurlaub.")
```

---

## Lokale Embeddings mit Sentence Transformers

```python
from sentence_transformers import SentenceTransformer, util
import numpy as np
import json
import os

class LocalEmbeddingSystem:
    """Einfaches lokales Embedding-System mit Caching."""

    def __init__(self, model_name: str = "BAAI/bge-small-en-v1.5", cache_path: str = "./embedding_cache.json"):
        self.model = SentenceTransformer(model_name)
        self.cache_path = cache_path
        self.cache = self._load_cache()
        print(f"Modell geladen: {model_name} ({self.model.get_sentence_embedding_dimension()} Dim)")

    def _load_cache(self) -> dict:
        if os.path.exists(self.cache_path):
            with open(self.cache_path) as f:
                return json.load(f)
        return {}

    def _save_cache(self):
        with open(self.cache_path, "w") as f:
            json.dump(self.cache, f)

    def embed(self, texts: list[str], use_cache: bool = True) -> np.ndarray:
        results = {}
        to_encode = []

        for text in texts:
            if use_cache and text in self.cache:
                results[text] = np.array(self.cache[text])
            else:
                to_encode.append(text)

        if to_encode:
            embeddings = self.model.encode(
                to_encode,
                normalize_embeddings=True,
                batch_size=32,
                show_progress_bar=len(to_encode) > 100
            )
            for text, emb in zip(to_encode, embeddings):
                results[text] = emb
                if use_cache:
                    self.cache[text] = emb.tolist()
            if use_cache:
                self._save_cache()

        return np.array([results[text] for text in texts])

    def similarity(self, text1: str, text2: str) -> float:
        embeddings = self.embed([text1, text2])
        return float(util.cos_sim(embeddings[0], embeddings[1]))

    def find_most_similar(self, query: str, corpus: list[str], top_k: int = 5) -> list[tuple[float, str]]:
        corpus_embeddings = self.embed(corpus)
        query_embedding = self.embed([query])[0]

        scores = util.cos_sim(query_embedding, corpus_embeddings)[0]
        top_results = scores.topk(min(top_k, len(corpus)))

        return [(float(score), corpus[idx]) for score, idx in zip(top_results.values, top_results.indices)]


# --- Verwendung ---
if __name__ == "__main__":
    system = LocalEmbeddingSystem()

    corpus = [
        "§5 Urlaubsanspruch: 30 Tage Jahresurlaub.",
        "§6 Urlaub muss 4 Wochen vorher angemeldet werden.",
        "§7 Sonderurlaub bei Heirat: 2 Tage.",
        "§12 Homeoffice: Bis zu 3 Tage pro Woche.",
        "§15 Überstunden werden durch Freizeit ausgeglichen.",
    ]

    results = system.find_most_similar("Wie viele Tage Urlaub bekomme ich?", corpus, top_k=3)

    print("Suchergebnisse:")
    for score, text in results:
        print(f"  [{score:.4f}] {text}")

    # Direkter Ähnlichkeitsvergleich:
    sim = system.similarity("Urlaub", "Ferien")
    print(f"\nÄhnlichkeit 'Urlaub' und 'Ferien': {sim:.4f}")
```

---

## Embeddings mit OpenAI API

```python
from openai import OpenAI
import numpy as np
import hashlib
import json
import os
from typing import Optional

class OpenAIEmbeddings:
    """OpenAI Embeddings mit Caching und Batch-Processing."""

    def __init__(
        self,
        model: str = "text-embedding-3-small",
        dimensions: Optional[int] = None,
        cache_dir: str = "./openai_embedding_cache"
    ):
        self.client = OpenAI()
        self.model = model
        self.dimensions = dimensions
        self.cache_dir = cache_dir
        os.makedirs(cache_dir, exist_ok=True)

    def _cache_key(self, text: str) -> str:
        content = f"{self.model}:{self.dimensions}:{text}"
        return hashlib.sha256(content.encode()).hexdigest()

    def _load_from_cache(self, key: str) -> Optional[list[float]]:
        path = os.path.join(self.cache_dir, f"{key}.json")
        if os.path.exists(path):
            with open(path) as f:
                return json.load(f)
        return None

    def _save_to_cache(self, key: str, embedding: list[float]):
        path = os.path.join(self.cache_dir, f"{key}.json")
        with open(path, "w") as f:
            json.dump(embedding, f)

    def embed(self, texts: list[str], batch_size: int = 100) -> list[np.ndarray]:
        """Embeddet eine Liste von Texten mit Caching und Batching."""
        results = {}
        to_embed = []

        # Cache prüfen
        for text in texts:
            key = self._cache_key(text)
            cached = self._load_from_cache(key)
            if cached is not None:
                results[text] = np.array(cached)
            else:
                to_embed.append(text)

        # In Batches embedden
        for i in range(0, len(to_embed), batch_size):
            batch = to_embed[i:i + batch_size]
            print(f"Embedde Batch {i//batch_size + 1}/{(len(to_embed) + batch_size - 1)//batch_size}...")

            kwargs = {"input": batch, "model": self.model}
            if self.dimensions:
                kwargs["dimensions"] = self.dimensions

            response = self.client.embeddings.create(**kwargs)

            for text, data in zip(batch, response.data):
                embedding = np.array(data.embedding)
                # Normalisieren:
                embedding = embedding / np.linalg.norm(embedding)
                results[text] = embedding
                self._save_to_cache(self._cache_key(text), embedding.tolist())

        return [results[text] for text in texts]

    def embed_single(self, text: str) -> np.ndarray:
        return self.embed([text])[0]


# --- Verwendung ---
if __name__ == "__main__":
    embedder = OpenAIEmbeddings(
        model="text-embedding-3-small",
        dimensions=512  # Optional: Dimensionen reduzieren (MRL)
    )

    docs = [
        "RAG kombiniert Retrieval mit Sprachmodellen.",
        "Vektordatenbanken speichern Embeddings für schnelle Suche.",
        "Die Küche ist gestern neu gestrichen worden.",  # Irrelevant
    ]

    doc_embeddings = embedder.embed(docs)
    query_embedding = embedder.embed_single("Wie funktioniert RAG?")

    similarities = [
        float(np.dot(query_embedding, doc_emb))
        for doc_emb in doc_embeddings
    ]

    for sim, doc in sorted(zip(similarities, docs), reverse=True):
        print(f"[{sim:.4f}] {doc[:70]}")
```

---

## Dimensionalität: Was bedeutet sie?

Die **Dimensionalität** gibt an, aus wie vielen Zahlen ein Embedding-Vektor besteht.

```
384 Dimensionen:  [0.12, -0.87, 0.34, ..., 0.56]  (384 Zahlen)
1536 Dimensionen: [0.12, -0.87, 0.34, ..., 0.56]  (1536 Zahlen)
3072 Dimensionen: [0.12, -0.87, 0.34, ..., 0.56]  (3072 Zahlen)
```

### Auswirkungen der Dimensionalität

| Aspekt               | Niedrige Dim (384)        | Hohe Dim (3072)              |
| -------------------- | ------------------------- | ----------------------------|
| Speicherbedarf       | 384 × 4 Bytes = 1.5 KB    | 3072 × 4 Bytes = 12 KB       |
| Berechnungsgeschw.   | Schnell                   | Langsamer                    |
| Ausdrucksstärke      | Begrenzt                  | Höher                        |
| Qualität             | Gut                       | Besser (meist)               |
| 1M Vektoren RAM      | ~1.5 GB                   | ~12 GB                       |

> [!NOTE]
> Hohe Dimensionalität bedeutet **nicht automatisch** bessere Qualität. Gut trainierte 384-dimensionale Modelle (wie BGE-small) übertreffen schlecht trainierte 1536-dimensionale Modelle. Das Training und die Daten sind entscheidender als die Dimensionszahl.

---

## Wann welches Embedding-Modell?

```
Ist Datenschutz / lokale Ausführung erforderlich?
├── Ja → Sentence Transformers (lokal)
│   ├── Deutsch / Mehrsprachig? → intfloat/multilingual-e5-large
│   ├── Englisch, schnell? → BAAI/bge-small-en-v1.5
│   └── Englisch, beste Qualität? → BAAI/bge-large-en-v1.5
└── Nein → API-Modell...

Verwendest du Claude als LLM?
├── Ja → Voyage AI (voyage-3) – Anthropic-Empfehlung
└── Nein → weiter...

Brauchst du maximale Einfachheit / OpenAI-Ökosystem?
├── Ja → text-embedding-3-small (gutes P/L-Verhältnis)
└── Nein → weiter...

Höchste Qualität ohne Preislimit?
└── Voyage-3 oder text-embedding-3-large
```

---

## Embeddings cachen

API-Kosten entstehen beim **Erstellen** der Embeddings, nicht beim Speichern oder Suchen. Das Einbetten der Dokument-Chunks ist ein einmaliger Aufwand – danach nur noch Query-Embeddings (günstig).

### Kostenbeispiel

```
1.000 Dokumente × 500 Tokens/Dokument = 500.000 Tokens
OpenAI text-embedding-3-small: $0.02 / 1M Tokens
Einmalkosten: 500.000 / 1.000.000 × $0.02 = $0.01 (1 Cent!)

Queries: 1.000 Queries/Tag × 50 Tokens = 50.000 Tokens/Tag
Kosten pro Tag: 50.000 / 1.000.000 × $0.02 = $0.001 (0,1 Cent/Tag)
```

> [!TIP]
> Caching bei Embeddings lohnt sich hauptsächlich für **Queries, die sich wiederholen** (z.B. in einer FAQ). Für Dokument-Chunks genügt ein einmaliges Indexieren und persistentes Speichern in der Vektordatenbank.

### Query-Embedding-Cache

```python
import hashlib
import numpy as np
from functools import lru_cache

# Einfacher In-Memory-Cache:
@lru_cache(maxsize=1000)
def embed_query_cached(query: str) -> tuple:
    """Cached Query-Embeddings (tuple weil lru_cache kein numpy array cacht)."""
    embedding = embedder.embed_single(query)
    return tuple(embedding.tolist())

# Verwendung:
query = "Wie viele Tage Urlaub bekomme ich?"
embedding = np.array(embed_query_cached(query))  # Zweiter Aufruf: aus Cache
```

---

## Modell-Vergleichstabelle

| Modell                          | Dim   | Geschwindigkeit | Kosten           | Sprachen    | Besonderheit                       |
| ------------------------------- | ----- | --------------- | ---------------- | ----------- | ---------------------------------- |
| `text-embedding-3-small`        | 1536  | Schnell (API)   | $0.02/1M Tokens  | Mehrsprachig | MRL, gutes P/L                    |
| `text-embedding-3-large`        | 3072  | Mittel (API)    | $0.13/1M Tokens  | Mehrsprachig | Beste OpenAI-Qualität              |
| `voyage-3`                      | 1024  | Schnell (API)   | $0.06/1M Tokens  | Mehrsprachig | Anthropic-Empfehlung für Claude   |
| `voyage-3-lite`                 | 512   | Sehr schnell    | $0.02/1M Tokens  | Mehrsprachig | Günstige Voyage-Option             |
| `BAAI/bge-small-en-v1.5`        | 384   | Sehr schnell    | Kostenlos (lokal)| Englisch    | Beste kleine lokale Option         |
| `BAAI/bge-large-en-v1.5`        | 1024  | Mittel (lokal)  | Kostenlos (lokal)| Englisch    | Top lokale Qualität                |
| `all-MiniLM-L6-v2`              | 384   | Sehr schnell    | Kostenlos (lokal)| Englisch    | Benchmark-Standard                 |
| `intfloat/multilingual-e5-large`| 1024  | Mittel (lokal)  | Kostenlos (lokal)| 100+ Sprachen | Beste mehrsprachige lokale Option |
| `nomic-embed-text-v1.5`         | 768   | Schnell (lokal) | Kostenlos (lokal)| Englisch    | MRL-Support, Apache 2.0            |

> [!TIP]
> Für **deutschsprachige Dokumente** empfiehlt sich `intfloat/multilingual-e5-large` (lokal) oder `voyage-3` (API). Deutsche Texte werden von rein englischen Modellen merklich schlechter eingebettet.

---

## Weiterführende Links

- [[RAG Grundlagen]] – Wie Embeddings in RAG eingesetzt werden
- [[Vektordatenbanken]] – Embeddings speichern und suchen
- [[LangChain und LlamaIndex]] – Embeddings in Frameworks verwenden
