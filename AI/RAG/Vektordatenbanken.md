# Vektordatenbanken

## Was sind Vektordatenbanken?

Eine Vektordatenbank ist eine Datenbank, die speziell für das **Speichern und schnelle Durchsuchen von hochdimensionalen Vektoren** ([[Embeddings]]) optimiert ist. Während relationale Datenbanken nach exakten Werten suchen (`WHERE name = 'Max'`), sucht eine Vektordatenbank nach **semantischer Ähnlichkeit** (`Finde die 5 ähnlichsten Vektoren zu diesem Query-Vektor`).

### Das technische Kernproblem: ANN-Suche

Bei 1 Million gespeicherten Vektoren mit je 1536 Dimensionen wäre ein brute-force Vergleich jedes Mal prohibitiv langsam. Vektordatenbanken verwenden **Approximate Nearest Neighbor (ANN)**-Algorithmen:

#### HNSW (Hierarchical Navigable Small World)

Der derzeit dominierende Algorithmus. Baut einen mehrschichtigen Graphen, in dem Vektoren verbunden sind:

```
Layer 2 (grob):    A ——————————————— E
Layer 1 (mittel):  A —— B —— C —— D — E
Layer 0 (fein):    A - B - B2- C - C2- D - D2- E
```

- **Einfügen:** O(log n) – sehr schnell
- **Suche:** O(log n) – annähernd exakt, sehr schnell
- **Nachteil:** Hoher RAM-Verbrauch (Graph muss im Speicher liegen)

#### IVF (Inverted File Index)

Teilt Vektoren in **Cluster** auf (Voronoi-Zellen). Bei der Suche werden nur relevante Cluster durchsucht.

```
Cluster 1: [v1, v4, v8, ...]
Cluster 2: [v2, v5, v9, ...]
Cluster 3: [v3, v6, v7, ...]
→ Query landet in Cluster 2 → nur dort suchen
```

> [!NOTE]
> Für die meisten RAG-Anwendungen ist **HNSW** die beste Wahl. IVF eignet sich besser für sehr große Datensätze (>10M Vektoren), wo HNSW zu viel RAM verbraucht.

---

## Vergleichstabelle der wichtigsten Vektordatenbanken

| Datenbank    | Typ            | Deployment       | Skalierung | HNSW | Hybrid Search | Lizenz       | Besonderheit                     |
| ------------ | -------------- | ---------------- | ---------- | ---- | ------------- | ------------ | -------------------------------- |
| **ChromaDB** | Spezialisiert  | Lokal / Server   | Mittel     | Ja   | Nein          | Apache 2.0   | Einfachste API, ideal für Prototypen |
| **Qdrant**   | Spezialisiert  | Docker / Cloud   | Hoch       | Ja   | Ja            | Apache 2.0   | Produktionsreif, Rust-Performance |
| **Weaviate** | Spezialisiert  | Docker / Cloud   | Hoch       | Ja   | Ja            | BSD-3        | Integriertes Vectorizer-Modul   |
| **Pinecone** | Spezialisiert  | Managed Cloud    | Sehr hoch  | Ja   | Ja            | Proprietär   | Vollständig verwaltet, kein Ops  |
| **Milvus**   | Spezialisiert  | Docker / K8s     | Sehr hoch  | Ja   | Ja            | Apache 2.0   | Enterprise-Scale, Alibaba-backed |
| **pgvector** | PostgreSQL-Ext | Bestehende PG-DB | Mittel     | Ja   | Nein          | Open Source  | Keine neue Infrastruktur nötig   |

> [!TIP]
> **Faustregel:** Prototyp/lokal → ChromaDB. Produktion → Qdrant. Bereits PostgreSQL vorhanden → pgvector. Kein eigenes Ops-Team → Pinecone.

---

## ChromaDB

ChromaDB ist die einsteigerfreundlichste Vektordatenbank. Sie ist vollständig in Python geschrieben und kann sowohl in-memory, als lokale persistente Datenbank oder als Client-Server betrieben werden.

### Installation

```bash
pip install chromadb
```

### Grundlegende Operationen

```python
import chromadb
from chromadb.utils import embedding_functions

# --- 1. Client erstellen ---

# In-Memory (für Tests, keine Persistenz):
client = chromadb.Client()

# Persistente lokale Datenbank:
client = chromadb.PersistentClient(path="./chroma_db")

# Client-Server-Modus (Server läuft separat):
# client = chromadb.HttpClient(host="localhost", port=8000)


# --- 2. Embedding-Funktion konfigurieren ---

# Standard: Chroma's eigene all-MiniLM-L6-v2 (lokal, kostenlos)
default_ef = embedding_functions.DefaultEmbeddingFunction()

# OpenAI Embeddings:
openai_ef = embedding_functions.OpenAIEmbeddingFunction(
    api_key="sk-...",
    model_name="text-embedding-3-small"
)


# --- 3. Collection erstellen oder laden ---

collection = client.get_or_create_collection(
    name="firmenhandbuch",
    embedding_function=default_ef,
    metadata={"hnsw:space": "cosine"}  # Distanzmetrik: cosine, l2, ip
)


# --- 4. Dokumente hinzufügen ---

documents = [
    "§5 Urlaubsanspruch: Jeder Mitarbeiter hat Anspruch auf 30 Tage Jahresurlaub.",
    "§6 Urlaubsplanung: Urlaub ist mindestens 4 Wochen im Voraus anzumelden.",
    "§7 Sonderurlaub: Bei Heirat stehen dem Mitarbeiter 2 Tage Sonderurlaub zu.",
    "§8 Überstunden: Überstunden sind durch Freizeit oder Vergütung auszugleichen.",
]

metadatas = [
    {"section": "urlaub", "paragraph": 5},
    {"section": "urlaub", "paragraph": 6},
    {"section": "urlaub", "paragraph": 7},
    {"section": "arbeitszeit", "paragraph": 8},
]

ids = ["doc_5", "doc_6", "doc_7", "doc_8"]

collection.add(
    documents=documents,
    metadatas=metadatas,
    ids=ids
)

print(f"Collection enthält {collection.count()} Dokumente")


# --- 5. Suchen ---

results = collection.query(
    query_texts=["Wie viele Tage Urlaub habe ich?"],
    n_results=3,
    # Optional: Metadata-Filter
    where={"section": "urlaub"},
    # Optional: Was soll zurückgegeben werden?
    include=["documents", "metadatas", "distances"]
)

for i, doc in enumerate(results["documents"][0]):
    distance = results["distances"][0][i]
    meta = results["metadatas"][0][i]
    print(f"[Score: {1 - distance:.3f}] §{meta['paragraph']}: {doc[:80]}...")


# --- 6. Dokumente aktualisieren und löschen ---

# Aktualisieren (upsert überschreibt bei existierender ID):
collection.upsert(
    documents=["§5 Urlaubsanspruch: Jeder Mitarbeiter hat Anspruch auf 32 Tage (aktualisiert)."],
    ids=["doc_5"]
)

# Einzelnes Dokument löschen:
collection.delete(ids=["doc_8"])

# Nach Metadaten löschen:
collection.delete(where={"section": "arbeitszeit"})
```

### Wann ChromaDB verwenden?

- Lokale Entwicklung und Prototypen
- Kleinere Datensätze (bis ~100.000 Dokumente)
- Kein separater Infrastruktur-Aufwand gewünscht
- Eingebettet in Python-Anwendungen

> [!WARNING]
> ChromaDB ist nicht für horizontale Skalierung ausgelegt. Für Produktionssysteme mit hohem Durchsatz oder mehreren Millionen Vektoren sollte Qdrant oder Milvus verwendet werden.

---

## Qdrant

Qdrant ist eine in Rust geschriebene Vektordatenbank, die für Produktionseinsatz entwickelt wurde. Sie bietet exzellente Performance, Hybrid Search, Payload-Filter und eine REST- sowie gRPC-API.

### Installation via Docker

```bash
# Einzelner Container (lokal):
docker run -p 6333:6333 -p 6334:6334 \
    -v $(pwd)/qdrant_storage:/qdrant/storage \
    qdrant/qdrant

# Mit Docker Compose:
# docker-compose.yml:
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"   # REST API
      - "6334:6334"   # gRPC
    volumes:
      - ./qdrant_storage:/qdrant/storage
    environment:
      - QDRANT__SERVICE__API_KEY=my-secret-key  # Optional
```

```bash
pip install qdrant-client sentence-transformers
```

### Python-Beispiel: Vollständige RAG-Pipeline mit Qdrant

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct,
    Filter, FieldCondition, MatchValue,
    SearchRequest, SparseVector, SparseVectorParams,
    NamedVector
)
from sentence_transformers import SentenceTransformer
import uuid

# --- 1. Client und Embedding-Modell initialisieren ---

client = QdrantClient(
    url="http://localhost:6333",
    # api_key="my-secret-key",  # Falls gesetzt
    timeout=30
)

# Für In-Memory (Tests ohne Docker):
# client = QdrantClient(":memory:")

model = SentenceTransformer("BAAI/bge-small-en-v1.5")
VECTOR_DIM = model.get_sentence_embedding_dimension()  # 384


# --- 2. Collection erstellen ---

COLLECTION_NAME = "firmenhandbuch"

# Bestehende Collection löschen falls vorhanden:
if client.collection_exists(COLLECTION_NAME):
    client.delete_collection(COLLECTION_NAME)

client.create_collection(
    collection_name=COLLECTION_NAME,
    vectors_config=VectorParams(
        size=VECTOR_DIM,
        distance=Distance.COSINE,  # COSINE, DOT, EUCLID, MANHATTAN
    )
)

print(f"Collection '{COLLECTION_NAME}' erstellt (Dimensionen: {VECTOR_DIM})")


# --- 3. Dokumente einbetten und speichern ---

documents = [
    {
        "id": str(uuid.uuid4()),
        "text": "§5 Urlaubsanspruch: Jeder Mitarbeiter hat Anspruch auf 30 Tage Jahresurlaub.",
        "section": "urlaub",
        "paragraph": 5
    },
    {
        "id": str(uuid.uuid4()),
        "text": "§6 Urlaubsplanung: Urlaub ist mindestens 4 Wochen im Voraus anzumelden.",
        "section": "urlaub",
        "paragraph": 6
    },
    {
        "id": str(uuid.uuid4()),
        "text": "§7 Sonderurlaub: Bei Heirat stehen dem Mitarbeiter 2 Tage Sonderurlaub zu.",
        "section": "urlaub",
        "paragraph": 7
    },
    {
        "id": str(uuid.uuid4()),
        "text": "§12 Homeoffice: Mitarbeiter können bis zu 3 Tage pro Woche im Homeoffice arbeiten.",
        "section": "arbeitszeit",
        "paragraph": 12
    },
]

# Texte einbetten
texts = [doc["text"] for doc in documents]
embeddings = model.encode(texts, normalize_embeddings=True)

# Punkte erstellen und hochladen
points = [
    PointStruct(
        id=doc["id"],
        vector=embedding.tolist(),
        payload={
            "text": doc["text"],
            "section": doc["section"],
            "paragraph": doc["paragraph"]
        }
    )
    for doc, embedding in zip(documents, embeddings)
]

client.upsert(
    collection_name=COLLECTION_NAME,
    points=points
)

print(f"{len(points)} Dokumente gespeichert")


# --- 4. Suchen ---

query = "Wie viele Tage Urlaub bekomme ich?"
query_embedding = model.encode([query], normalize_embeddings=True)[0]

# Einfache Vektorsuche:
results = client.search(
    collection_name=COLLECTION_NAME,
    query_vector=query_embedding.tolist(),
    limit=3,
    with_payload=True,
    score_threshold=0.5  # Nur Ergebnisse mit Score >= 0.5
)

print(f"\nSuchergebnisse für: '{query}'")
for result in results:
    print(f"  Score: {result.score:.4f} | §{result.payload['paragraph']}: {result.payload['text'][:80]}...")


# --- 5. Suche mit Metadata-Filter ---

filtered_results = client.search(
    collection_name=COLLECTION_NAME,
    query_vector=query_embedding.tolist(),
    query_filter=Filter(
        must=[
            FieldCondition(
                key="section",
                match=MatchValue(value="urlaub")
            )
        ]
    ),
    limit=3
)

print(f"\nGefilterte Suche (nur 'urlaub'-Sektion):")
for result in filtered_results:
    print(f"  Score: {result.score:.4f} | {result.payload['text'][:80]}...")


# --- 6. Collection-Informationen ---

info = client.get_collection(COLLECTION_NAME)
print(f"\nCollection-Info:")
print(f"  Vektoren: {info.vectors_count}")
print(f"  Status: {info.status}")
```

### Wann Qdrant verwenden?

- Produktionssysteme mit Anforderungen an Stabilität
- Datensätze mit Millionen von Vektoren
- Hybrid Search (Dense + Sparse) benötigt
- Komplexe Metadata-Filter erforderlich
- Mehrere Teams / Services greifen auf dieselbe DB zu

---

## pgvector

pgvector ist eine PostgreSQL-Extension, die Vektoroperationen direkt in einer bestehenden PostgreSQL-Datenbank ermöglicht. Ideal, wenn bereits PostgreSQL im Einsatz ist.

### Installation

```bash
# PostgreSQL Extension installieren (Ubuntu/Debian):
sudo apt install postgresql-16-pgvector

# Oder via Docker:
docker run -p 5432:5432 \
    -e POSTGRES_PASSWORD=password \
    -e POSTGRES_DB=ragdb \
    ankane/pgvector

# Python-Client:
pip install psycopg2-binary pgvector sqlalchemy
```

### SQL-Grundoperationen

```sql
-- Extension aktivieren (einmalig pro Datenbank):
CREATE EXTENSION IF NOT EXISTS vector;

-- Tabelle mit Vektor-Spalte erstellen:
CREATE TABLE documents (
    id          SERIAL PRIMARY KEY,
    content     TEXT NOT NULL,
    section     VARCHAR(100),
    paragraph   INTEGER,
    embedding   vector(1536),  -- Dimensionen passend zum Embedding-Modell
    created_at  TIMESTAMP DEFAULT NOW()
);

-- HNSW-Index erstellen (schnelle Suche):
CREATE INDEX ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- IVFFlat-Index (Alternative, weniger RAM):
-- CREATE INDEX ON documents
--     USING ivfflat (embedding vector_cosine_ops)
--     WITH (lists = 100);

-- Dokument einfügen (Vektor als Python-Liste übergeben):
INSERT INTO documents (content, section, paragraph, embedding)
VALUES (
    '§5 Urlaubsanspruch: Jeder Mitarbeiter hat 30 Tage Urlaub.',
    'urlaub',
    5,
    '[0.12, -0.34, 0.56, ...]'  -- 1536-dimensionaler Vektor
);

-- Similarity Search mit Cosine Distance:
-- (1 - cosine_distance) = cosine_similarity
SELECT
    id,
    content,
    section,
    1 - (embedding <=> '[0.23, -0.87, ...]') AS similarity
FROM documents
ORDER BY embedding <=> '[0.23, -0.87, ...]'  -- <=> = cosine distance
LIMIT 5;

-- Andere Distanzoperatoren:
-- <-> = Euclidean distance (L2)
-- <#> = Negative inner product (Dot Product, negiert)
-- <=> = Cosine distance

-- Similarity Search mit Filter:
SELECT id, content, 1 - (embedding <=> '[...]') AS similarity
FROM documents
WHERE section = 'urlaub'
ORDER BY embedding <=> '[...]'
LIMIT 3;
```

### Python-Beispiel mit SQLAlchemy und pgvector

```python
from sqlalchemy import create_engine, Column, Integer, String, Text, select
from sqlalchemy.orm import declarative_base, Session
from pgvector.sqlalchemy import Vector
from sentence_transformers import SentenceTransformer
import numpy as np

# --- Setup ---
DATABASE_URL = "postgresql://postgres:password@localhost:5432/ragdb"
engine = create_engine(DATABASE_URL)
Base = declarative_base()
model = SentenceTransformer("BAAI/bge-small-en-v1.5")
VECTOR_DIM = 384


# --- Model definieren ---
class Document(Base):
    __tablename__ = "documents"

    id = Column(Integer, primary_key=True)
    content = Column(Text, nullable=False)
    section = Column(String(100))
    paragraph = Column(Integer)
    embedding = Column(Vector(VECTOR_DIM))


# Tabellen erstellen:
Base.metadata.create_all(engine)


# --- Dokumente einfügen ---
def add_documents(texts: list[str], sections: list[str], paragraphs: list[int]):
    embeddings = model.encode(texts, normalize_embeddings=True)

    with Session(engine) as session:
        for text, section, paragraph, embedding in zip(texts, sections, paragraphs, embeddings):
            doc = Document(
                content=text,
                section=section,
                paragraph=paragraph,
                embedding=embedding.tolist()
            )
            session.add(doc)
        session.commit()
    print(f"{len(texts)} Dokumente hinzugefügt")


# --- Suchen ---
def search(query: str, top_k: int = 5, section_filter: str = None):
    query_embedding = model.encode([query], normalize_embeddings=True)[0]

    with Session(engine) as session:
        stmt = (
            select(
                Document.id,
                Document.content,
                Document.section,
                Document.paragraph,
                # Cosine Similarity = 1 - Cosine Distance
                (1 - Document.embedding.cosine_distance(query_embedding)).label("similarity")
            )
            .order_by(Document.embedding.cosine_distance(query_embedding))
            .limit(top_k)
        )

        if section_filter:
            stmt = stmt.where(Document.section == section_filter)

        results = session.execute(stmt).fetchall()

    return results


# --- Verwendung ---
add_documents(
    texts=[
        "§5 Urlaubsanspruch: 30 Tage Jahresurlaub.",
        "§6 Urlaubsplanung: 4 Wochen im Voraus anmelden.",
        "§12 Homeoffice: Bis zu 3 Tage pro Woche möglich.",
    ],
    sections=["urlaub", "urlaub", "arbeitszeit"],
    paragraphs=[5, 6, 12]
)

results = search("Wie viele Urlaubstage?", top_k=3, section_filter="urlaub")
for row in results:
    print(f"[{row.similarity:.4f}] §{row.paragraph}: {row.content}")
```

### Wann pgvector verwenden?

- PostgreSQL ist bereits in der Infrastruktur vorhanden
- Team kennt SQL gut
- Keine zusätzliche Infrastruktur gewünscht
- ACID-Transaktionen mit Vektordaten werden benötigt (z.B. atomares Update von Text + Embedding)
- Datenmenge ist überschaubar (bis ~1M Vektoren)

> [!NOTE]
> pgvector unterstützt seit v0.5.0 auch HNSW-Indizes (vorher nur IVFFlat). Die Suchgeschwindigkeit ist vergleichbar mit spezialisierten Datenbanken bei moderaten Datenmengen.

---

## Auswahlkriterien: Wann welche Datenbank?

```
Hast du bereits PostgreSQL?
├── Ja → pgvector (kein neues System, direkte SQL-Integration)
└── Nein → weiter...

Ist es ein Prototyp / lokale Entwicklung?
├── Ja → ChromaDB (pip install, keine Konfiguration nötig)
└── Nein → weiter...

Brauchst du Hybrid Search (Dense + Sparse)?
├── Ja → Qdrant oder Weaviate
└── Nein → weiter...

Willst du keine eigene Infrastruktur betreiben?
├── Ja → Pinecone (vollständig managed)
└── Nein → weiter...

Millionen von Vektoren / Enterprise-Scale?
├── Ja → Milvus oder Qdrant
└── Nein → Qdrant (gutes Allround-Paket)
```

### Kostenvergleich (Cloud/Managed)

| Anbieter         | Free Tier      | Kosten (ca.)                  | Notizen                    |
| ---------------- | -------------- | ----------------------------- | -------------------------- |
| ChromaDB Cloud   | 1M Embeddings  | $0.20 / 1M weitere            | Relativ neu                |
| Qdrant Cloud     | 1GB RAM        | ~$25 / Monat (4GB RAM)        | Fair pricing               |
| Pinecone         | 2M Vektoren    | $0.096 / 1M Vektoren/Monat    | Teuer bei Scale            |
| Weaviate Cloud   | 14 Tage Trial  | Ab $25 / Monat                | Sandbox dauerhaft kostenlos |
| pgvector (Supabase) | 500MB DB    | Ab $25 / Monat                | Postgres-as-a-Service      |

> [!TIP]
> Für die meisten RAG-Projekte ist **Self-hosted Qdrant auf einem VPS** (z.B. Hetzner, ~$5-15/Monat) das beste Preis-Leistungs-Verhältnis. Ein einzelner Server mit 8GB RAM reicht für mehrere Millionen Vektoren.

---

## Weiterführende Links

- [[RAG Grundlagen]] – RAG-Architektur und Retrieval-Methoden
- [[Embeddings]] – Embedding-Modelle und deren Dimensionen
- [[LangChain und LlamaIndex]] – Integration in RAG-Frameworks
