---
tags: [ai, rag, langchain, llamaindex, framework]
---

# LangChain und LlamaIndex

---

## LangChain

### Was ist LangChain?

LangChain ist ein Open-Source-Framework für die Entwicklung von LLM-basierten Anwendungen. Es abstrahiert die Integration verschiedener Komponenten – LLMs, Vektordatenbanken, Tools, Agenten, Memory – hinter einer einheitlichen API.

**Kernphilosophie:** "Verkette" (chain) verschiedene Komponenten zu komplexen Pipelines. Statt direkt mit OpenAI-API, ChromaDB und Prompts herumzujonglieren, kombiniert LangChain diese in wiederverwendbaren, komponierbaren Einheiten.

```
Ohne LangChain:
Query → OpenAI Embed → Chroma Query → Ergebnisse formatieren → OpenAI Chat → Antwort
(Jeder Schritt: eigener Code, eigene Fehlerbehandlung)

Mit LangChain:
chain.invoke({"question": query})
(Gleiche Pipeline, aber als deklarativer Chain)
```

> [!NOTE]
> LangChain wird manchmal für seine Komplexität kritisiert. Für einfache RAG-Pipelines ist es manchmal "over-engineered". Für komplexe Agenten-Systeme, die viele verschiedene Tools kombinieren, ist es hingegen sehr wertvoll.

### Installation

```bash
pip install langchain langchain-openai langchain-community chromadb
# Für LCEL (wird automatisch mitinstalliert):
# langchain-core ist eine Abhängigkeit von langchain
```

### Grundkonzepte

#### Chains

Eine **Chain** ist eine Sequenz von Schritten. Mit der LangChain Expression Language (LCEL) werden sie mit `|` (Pipe-Operator) verbunden:

```
prompt | llm | output_parser
```

#### Retrievers

Ein **Retriever** ist ein Objekt, das bei gegebener Query die relevantesten Dokumente zurückgibt. Es abstrahiert die Vektordatenbank dahinter.

#### Memory

**Memory** ermöglicht Gesprächsgedächtnis – der Chatbot "erinnert sich" an frühere Nachrichten im gleichen Gespräch.

#### Agents

**Agents** lassen das LLM selbst entscheiden, welche Tools es in welcher Reihenfolge aufruft. Für Reasoning-intensive Tasks.

---

### Einfaches RAG-Beispiel mit LangChain

```python
import os
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import TextLoader, DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# --- 1. Dokumente laden ---

# Einzelne Datei:
loader = TextLoader("./firmenhandbuch.txt", encoding="utf-8")

# Oder ganzes Verzeichnis (alle .txt Dateien):
# loader = DirectoryLoader("./docs/", glob="**/*.txt", loader_cls=TextLoader)

documents = loader.load()
print(f"Dokumente geladen: {len(documents)}")


# --- 2. Chunking ---

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ". ", " ", ""]
)
chunks = splitter.split_documents(documents)
print(f"Chunks erstellt: {len(chunks)}")


# --- 3. Embeddings und Vektordatenbank ---

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db"
)

retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4}  # Top-4 Chunks zurückgeben
)


# --- 4. Prompt-Template ---

prompt = ChatPromptTemplate.from_messages([
    ("system", """Du bist ein hilfreicher Assistent für Mitarbeiterfragen.
Beantworte die Frage AUSSCHLIESSLICH basierend auf dem folgenden Kontext.
Wenn der Kontext die Antwort nicht enthält, sage das klar.

Kontext:
{context}"""),
    ("human", "{question}")
])


# --- 5. LLM ---

llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0  # Für faktische Antworten: Temperatur = 0
)


# --- 6. Chain mit LCEL zusammenbauen ---

def format_docs(docs) -> str:
    """Formatiert die Dokumente für den Kontext."""
    return "\n\n---\n\n".join(
        f"[Quelle: {doc.metadata.get('source', 'Unbekannt')}]\n{doc.page_content}"
        for doc in docs
    )

rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough()  # Query unverändert weitergeben
    }
    | prompt
    | llm
    | StrOutputParser()
)


# --- 7. Anfrage stellen ---

question = "Wie viele Tage Urlaub stehen mir zu?"
answer = rag_chain.invoke(question)
print(f"Frage: {question}")
print(f"Antwort: {answer}")


# --- 8. Mit Quellenangaben ---

# Separater Chain der Quellen zurückgibt:
from langchain_core.runnables import RunnableParallel

rag_chain_with_sources = RunnableParallel(
    answer=rag_chain,
    sources=retriever | (lambda docs: [doc.metadata.get("source") for doc in docs])
)

result = rag_chain_with_sources.invoke(question)
print(f"Antwort: {result['answer']}")
print(f"Quellen: {result['sources']}")
```

### Konversations-RAG mit Memory

```python
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferWindowMemory

# Memory: erinnert sich an die letzten 5 Nachrichten
memory = ConversationBufferWindowMemory(
    memory_key="chat_history",
    return_messages=True,
    k=5  # Letzte k Konversationspaare merken
)

conv_chain = ConversationalRetrievalChain.from_llm(
    llm=ChatOpenAI(model="gpt-4o-mini", temperature=0),
    retriever=retriever,
    memory=memory,
    verbose=False
)

# Gespräch:
print(conv_chain.invoke({"question": "Wie viele Urlaubstage habe ich?"})["answer"])
print(conv_chain.invoke({"question": "Und was gilt für Sonderurlaub?"})["answer"])
# Der zweite Aufruf kennt den Kontext des ersten dank Memory
```

### LangChain Expression Language (LCEL)

LCEL ist die moderne Art, Chains in LangChain zu definieren. Der `|`-Operator verketten Runnable-Objekte:

```python
from langchain_core.runnables import RunnableLambda, RunnableParallel, RunnableBranch

# Jedes Element ist ein Runnable:
chain = prompt | llm | StrOutputParser()

# Parallele Ausführung:
parallel = RunnableParallel(
    summary=summarize_chain,
    keywords=keyword_chain
)

# Bedingte Ausführung:
branch = RunnableBranch(
    (lambda x: "urlaub" in x.lower(), urlaub_chain),
    (lambda x: "homeoffice" in x.lower(), homeoffice_chain),
    default_chain  # Fallback
)

# Streaming (token-by-token):
for chunk in chain.stream({"question": "Erkläre RAG"}):
    print(chunk, end="", flush=True)

# Async:
import asyncio
answer = asyncio.run(chain.ainvoke({"question": "Erkläre RAG"}))

# Batch:
answers = chain.batch([
    {"question": "Frage 1"},
    {"question": "Frage 2"},
])
```

> [!TIP]
> LCEL unterstützt **Streaming, Async und Batching** automatisch für jeden Chain. Wenn ein Runnable in der Chain diese Features nicht unterstützt, fällt LCEL automatisch auf synchrone Ausführung zurück.

### LangChain: Stärken und Schwächen

**Stärken:**
- Riesiges Ökosystem (200+ Integrationen für LLMs, DBs, Tools)
- LCEL macht komplexe Pipelines deklarativ und lesbar
- LangGraph für Agenten und State-Machines (sehr mächtig)
- LangSmith für Debugging und Tracing

**Schwächen:**
- Steile Lernkurve bei komplexen Features
- Abstraktion kann bei Debugging hinderlich sein ("Magie")
- Häufige Breaking Changes zwischen Versionen
- Für simple RAG-Pipelines oft Overkill

---

## LlamaIndex

### Was ist LlamaIndex?

LlamaIndex (früher GPT Index) ist ein **Data Framework für LLM-Anwendungen**, das speziell für das Verbinden von LLMs mit externen Datenquellen entwickelt wurde. Während LangChain breiter aufgestellt ist, fokussiert sich LlamaIndex tiefer auf **Daten-Ingestion, Indexierung und Retrieval**.

**Kernphilosophie:** Daten sind der Kern. LlamaIndex bietet strukturiertere Konzepte für das Laden, Aufbereiten und Abfragen von Dokumenten. Die Trennung in `Documents → Nodes → Index → QueryEngine` ist expliziter als bei LangChain.

### Installation

```bash
pip install llama-index llama-index-llms-openai llama-index-embeddings-openai
# Für lokale Modelle:
pip install llama-index-llms-ollama llama-index-embeddings-huggingface
```

### Grundkonzepte

```
Documents                    Nodes                   Index
┌──────────────┐    Split    ┌──────┐┌──────┐      ┌─────────────────┐
│ PDF, TXT,    │  ────────►  │Node 1││Node 2│ ──►  │ VectorStoreIndex│
│ Web, DB, ... │    Chunk    └──────┘└──────┘      │ (Embeddings)    │
└──────────────┘                                    └────────┬────────┘
                                                             │
                                                    QueryEngine
                                                    ┌────────▼────────┐
                                                    │ Retrieve + LLM  │
                                                    └─────────────────┘
```

- **Document:** Rohdokument mit Inhalt und Metadaten
- **Node (TextNode):** Ein Chunk eines Dokuments mit Referenz zum Originaldokument
- **Index:** Datenstruktur über Nodes (VectorStoreIndex, SummaryIndex, etc.)
- **QueryEngine:** Kombiniert Retrieval und LLM-Antwort-Generierung

---

### Einfaches RAG-Beispiel mit LlamaIndex

```python
import os
from llama_index.core import (
    VectorStoreIndex,
    SimpleDirectoryReader,
    Settings,
    StorageContext,
    load_index_from_storage
)
from llama_index.core.node_parser import SentenceSplitter
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# --- 1. Globale Einstellungen konfigurieren ---

Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.node_parser = SentenceSplitter(chunk_size=512, chunk_overlap=50)
Settings.num_output = 512      # Max. Tokens in der Antwort
Settings.context_window = 4096 # Context-Fenster des LLMs


# --- 2. Dokumente laden ---

# Einzelnes Verzeichnis (alle unterstützten Dateien):
documents = SimpleDirectoryReader("./docs/").load_data()
print(f"Dokumente geladen: {len(documents)}")


# --- 3. Index erstellen ---

# Index aus Dokumenten erstellen (Chunking + Embedding passiert automatisch):
index = VectorStoreIndex.from_documents(documents, show_progress=True)

# Index persistieren (damit wir nicht jedes Mal neu indexieren müssen):
index.storage_context.persist(persist_dir="./llamaindex_storage")


# --- 4. Index laden (beim nächsten Start) ---

# Falls Index bereits vorhanden:
if os.path.exists("./llamaindex_storage"):
    storage_context = StorageContext.from_defaults(persist_dir="./llamaindex_storage")
    index = load_index_from_storage(storage_context)
    print("Index aus Speicher geladen")


# --- 5. QueryEngine erstellen und abfragen ---

query_engine = index.as_query_engine(
    similarity_top_k=4,        # Top-4 Chunks retrieven
    response_mode="compact",   # compact, tree_summarize, refine, simple_summarize
    verbose=False
)

response = query_engine.query("Wie viele Tage Urlaub stehen mir zu?")

print(f"Antwort: {response}")
print(f"\nQuellen ({len(response.source_nodes)} Nodes):")
for node in response.source_nodes:
    print(f"  Score: {node.score:.4f} | {node.text[:80]}...")
```

### SimpleDirectoryReader im Detail

LlamaIndex's SimpleDirectoryReader kann viele Dateiformate automatisch laden:

```python
from llama_index.core import SimpleDirectoryReader

# Alle Dateien in einem Verzeichnis:
reader = SimpleDirectoryReader(
    input_dir="./docs/",
    recursive=True,             # Unterverzeichnisse einschließen
    required_exts=[".pdf", ".txt", ".md", ".docx"],  # Nur diese Typen
    exclude=["*.tmp", "*.log"], # Dateimuster ausschließen
    filename_as_id=True         # Dateiname als Dokument-ID
)
documents = reader.load_data()

# Einzelne Dateien:
reader = SimpleDirectoryReader(input_files=["./handbuch.pdf", "./richtlinien.docx"])
documents = reader.load_data()

# Metadaten anreichern:
def add_metadata(filename: str) -> dict:
    return {
        "department": "HR" if "hr" in filename.lower() else "General",
        "file_type": filename.split(".")[-1]
    }

reader = SimpleDirectoryReader(
    input_dir="./docs/",
    file_metadata=add_metadata
)
documents = reader.load_data()

print(f"Beispiel-Metadaten: {documents[0].metadata}")
```

**Unterstützte Formate:** PDF, Word (.docx), PowerPoint (.pptx), Excel (.xlsx), Markdown, HTML, CSV, JSON, Bilder (mit OCR), Jupyter Notebooks und viele mehr.

> [!NOTE]
> Für PDF-Dateien installiere `pip install pypdf`. Für Word-Dateien: `pip install python-docx`. LlamaIndex zeigt beim Laden an, welche Reader installiert sind.

### VectorStoreIndex im Detail

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.node_parser import (
    SentenceSplitter,
    SemanticSplitterNodeParser,
    SimpleFileNodeParser
)
from llama_index.embeddings.openai import OpenAIEmbedding

# Verschiedene Node-Parser:

# 1. Sentence Splitter (Standard):
sentence_parser = SentenceSplitter(
    chunk_size=512,
    chunk_overlap=50
)

# 2. Semantic Splitter (qualitativ besser, teurer):
semantic_parser = SemanticSplitterNodeParser(
    embed_model=OpenAIEmbedding(),
    breakpoint_percentile_threshold=95
)

# Index mit manuellem Parser:
from llama_index.core.ingestion import IngestionPipeline

pipeline = IngestionPipeline(
    transformations=[
        sentence_parser,
        OpenAIEmbedding()
    ]
)
nodes = pipeline.run(documents=documents)
index = VectorStoreIndex(nodes)


# Retriever direkt verwenden (ohne QueryEngine):
retriever = index.as_retriever(similarity_top_k=5)
nodes = retriever.retrieve("Wie viele Urlaubstage?")
for node in nodes:
    print(f"[{node.score:.4f}] {node.text[:80]}")


# QueryEngine mit verschiedenen Response-Modi:
# "compact"          – Kombiniert alle Chunks in einen Prompt (Standard)
# "refine"           – Iterativ: erst Chunk 1, dann Chunk 2 als Refinement
# "tree_summarize"   – Baut einen Baum für sehr lange Kontexte
# "simple_summarize" – Einfache Zusammenfassung

engine = index.as_query_engine(
    response_mode="tree_summarize",
    similarity_top_k=6
)
```

### LlamaIndex mit externer Vektordatenbank (Qdrant)

```python
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.vector_stores.qdrant import QdrantVectorStore
from qdrant_client import QdrantClient

# Qdrant-Client:
qdrant_client = QdrantClient(url="http://localhost:6333")

# Vector Store:
vector_store = QdrantVectorStore(
    client=qdrant_client,
    collection_name="firmenhandbuch"
)

# Storage Context mit Qdrant als Backend:
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# Index erstellen (Daten landen in Qdrant):
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
    show_progress=True
)

# Später: Index aus Qdrant laden (kein lokales Speichern nötig):
index = VectorStoreIndex.from_vector_store(vector_store)
query_engine = index.as_query_engine()
```

### LlamaIndex: Stärken und Schwächen

**Stärken:**
- Sehr strukturierter Ansatz für Daten-Ingestion
- SimpleDirectoryReader liest viele Formate automatisch
- Persistenz und Caching out-of-the-box
- Bessere Abstraktion für komplexe Retrieval-Strategien (SubQuestion, HyDE, etc.)
- Übersichtliche Node-Struktur mit Metadaten und Beziehungen

**Schwächen:**
- Geringere Flexibilität bei nicht-RAG-Anwendungen
- Weniger Integrationen für externe Tools / APIs
- Agenten-Funktionalität weniger ausgereift als LangChain
- Kleinere Community als LangChain

---

## Vergleich: LangChain vs. LlamaIndex

### Das Kernproblem beider Frameworks

> [!IMPORTANT]
> Beide Frameworks abstrahieren komplexe LLM-Pipelines, aber mit unterschiedlichem Fokus:
> - **LangChain:** Flexibler Baukasten für beliebige LLM-Anwendungen
> - **LlamaIndex:** Spezialisiertes Tool für Daten-zu-LLM-Verbindungen (RAG)

### Vergleichstabelle nach Anwendungsfall

| Anwendungsfall                               | LangChain | LlamaIndex | Empfehlung                                |
| -------------------------------------------- | :-------: | :--------: | ----------------------------------------- |
| Einfaches RAG über Dokumente                 | Gut       | Sehr gut   | LlamaIndex (weniger Boilerplate)          |
| RAG über viele verschiedene Dateiformate     | Mittel    | Sehr gut   | LlamaIndex (SimpleDirectoryReader)        |
| Agenten mit Tool-Use                         | Sehr gut  | Gut        | LangChain (mehr Tools, LangGraph)         |
| Chatbot mit Memory                           | Sehr gut  | Mittel     | LangChain (ConversationChain)             |
| Komplexe Retrieval-Strategien (HyDE, etc.)   | Gut       | Sehr gut   | LlamaIndex                                |
| Multi-Step Pipelines / State-Machines        | Sehr gut  | Mittel     | LangChain + LangGraph                     |
| Daten aus SQL/APIs abrufen                   | Sehr gut  | Gut        | LangChain                                 |
| Produktions-Tracing / Monitoring             | Sehr gut  | Gut        | LangChain (LangSmith integriert)          |
| Schnelles Prototyping RAG                    | Mittel    | Sehr gut   | LlamaIndex                                |

### Detaillierter Code-Vergleich: Gleiches RAG, zwei Frameworks

```python
# ============================================================
# LANGCHAIN: RAG Pipeline
# ============================================================
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import DirectoryLoader
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# Explizit: jeder Schritt muss definiert werden
loader = DirectoryLoader("./docs/")
documents = loader.load()

splitter = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=50)
chunks = splitter.split_documents(documents)

vectorstore = Chroma.from_documents(chunks, OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

prompt = ChatPromptTemplate.from_messages([
    ("system", "Beantworte die Frage basierend auf: {context}"),
    ("human", "{question}")
])

chain = (
    {"context": retriever | (lambda docs: "\n".join(d.page_content for d in docs)),
     "question": RunnablePassthrough()}
    | prompt
    | ChatOpenAI(model="gpt-4o-mini")
    | StrOutputParser()
)

answer = chain.invoke("Wie viele Urlaubstage?")


# ============================================================
# LLAMAINDEX: Gleiches RAG, weniger Code
# ============================================================
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# Alles in 3 Zeilen:
documents = SimpleDirectoryReader("./docs/").load_data()
index = VectorStoreIndex.from_documents(documents)
answer = index.as_query_engine().query("Wie viele Urlaubstage?")
```

> [!NOTE]
> LlamaIndex ist bei reinen RAG-Pipelines deutlich kompakter. LangChain ist expliziter – was ein Vorteil ist, wenn man die Pipeline stark anpassen oder debuggen muss.

### Wann welches Framework?

```
Primärer Use Case: Dokumente durchsuchbar machen / RAG?
├── Ja → LlamaIndex (weniger Code, besseres Daten-Handling)
└── Nein → weiter...

Brauche ich komplexe Agenten mit vielen Tools?
├── Ja → LangChain + LangGraph
└── Nein → weiter...

Brauche ich Konversations-Memory / Chatbot?
├── Ja → LangChain (ConversationChain)
└── Nein → weiter...

Brauche ich Produktions-Monitoring / Tracing?
├── Ja → LangChain + LangSmith (kommerziell, sehr gut)
└── Nein → weiter...

Unsicher / Lernprojekt?
└── LlamaIndex für RAG, LangChain für alles andere
```

### Können sie kombiniert werden?

**Ja.** LlamaIndex und LangChain können gemeinsam genutzt werden:

```python
# LlamaIndex als Retriever in LangChain verwenden:
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from langchain_community.retrievers import LlamaIndexRetriever

# LlamaIndex Index erstellen:
documents = SimpleDirectoryReader("./docs/").load_data()
index = VectorStoreIndex.from_documents(documents)

# Als LangChain Retriever verwenden:
retriever = LlamaIndexRetriever(index=index)

# Jetzt in einer LangChain Pipeline:
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | ChatPromptTemplate.from_template("Context: {context}\nFrage: {question}")
    | ChatOpenAI(model="gpt-4o-mini")
    | StrOutputParser()
)
```

> [!TIP]
> In der Praxis empfiehlt sich jedoch **ein Framework je Projekt** zu wählen und nicht zu mischen. Die Kombination erhöht die Komplexität erheblich und macht Debugging schwieriger.

---

## Vollständiges Produktionsbeispiel: RAG mit LlamaIndex + Qdrant + Streaming

```python
import asyncio
from llama_index.core import VectorStoreIndex, StorageContext, Settings
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core.callbacks import CallbackManager, LlamaDebugHandler
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.vector_stores.qdrant import QdrantVectorStore
from llama_index.core.postprocessor import SimilarityPostprocessor
from qdrant_client import QdrantClient

# --- Konfiguration ---
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.node_parser = SentenceSplitter(chunk_size=512, chunk_overlap=50)

# --- Debug-Handler (optional, für Entwicklung) ---
debug_handler = LlamaDebugHandler(print_trace_on_end=True)
Settings.callback_manager = CallbackManager([debug_handler])

# --- Qdrant-Backend ---
qdrant_client = QdrantClient(url="http://localhost:6333")
vector_store = QdrantVectorStore(
    client=qdrant_client,
    collection_name="production_rag"
)
storage_context = StorageContext.from_defaults(vector_store=vector_store)


# --- Index (einmalig beim Start oder nach Daten-Update) ---
def build_index(docs_dir: str) -> VectorStoreIndex:
    from llama_index.core import SimpleDirectoryReader
    documents = SimpleDirectoryReader(docs_dir, recursive=True).load_data()
    print(f"Lade {len(documents)} Dokumente...")
    index = VectorStoreIndex.from_documents(
        documents,
        storage_context=storage_context,
        show_progress=True
    )
    return index


# --- QueryEngine mit Postprocessing ---
def create_query_engine(index: VectorStoreIndex):
    return index.as_query_engine(
        similarity_top_k=6,
        response_mode="compact",
        # Filtere Chunks mit zu niedriger Ähnlichkeit heraus:
        node_postprocessors=[
            SimilarityPostprocessor(similarity_cutoff=0.6)
        ]
    )


# --- Streaming-Anfrage ---
async def stream_answer(index, question: str):
    streaming_engine = index.as_query_engine(
        streaming=True,
        similarity_top_k=4,
        response_mode="compact"
    )
    streaming_response = await asyncio.to_thread(
        streaming_engine.query, question
    )
    print(f"\nAntwort (streaming):\n")
    streaming_response.print_response_stream()
    print(f"\n\nVerwendete Quellen:")
    for node in streaming_response.source_nodes:
        print(f"  - {node.metadata.get('file_name', 'Unbekannt')} (Score: {node.score:.3f})")


# --- Hauptprogramm ---
if __name__ == "__main__":
    # Index aufbauen oder aus Qdrant laden:
    index = VectorStoreIndex.from_vector_store(vector_store)
    engine = create_query_engine(index)

    question = "Was sind die Homeoffice-Regelungen im Unternehmen?"
    response = engine.query(question)

    print(f"Frage: {question}")
    print(f"Antwort: {response}")
    print(f"\nQuellen:")
    for src in response.source_nodes:
        print(f"  [{src.score:.3f}] {src.text[:100]}...")
```

---

## Weiterführende Links

- [[RAG Grundlagen]] – RAG-Architektur und Grundprinzipien
- [[Embeddings]] – Embedding-Modelle für LangChain und LlamaIndex
- [[Vektordatenbanken]] – ChromaDB, Qdrant und pgvector-Integration

---

## Verwandte Notizen

- [[RAG Grundlagen]] — Architektur und Prinzipien hinter RAG-Pipelines
- [[Embeddings]] — Embedding-Modelle für Vektorisierung in RAG
- [[Vektordatenbanken]] — Vektorspeicher für LangChain und LlamaIndex
- [[Agent Frameworks]] — Agentenarchitekturen als Erweiterung der Frameworks
- [[Function Calling und Tool Use]] — Werkzeugnutzung durch LLMs in Agenten-Pipelines
