---
tags: [ai, agents, konzept]
---

# Was ist ein AI Agent?

## Definition

Ein **AI Agent** ist ein System, das ein Large Language Model (LLM) als Kern-Reasoning-Engine verwendet und autonom Aufgaben ausführt, indem es eigenständig Entscheidungen trifft, Tools einsetzt und über mehrere Schritte hinweg zielgerichtet handelt.

Der wesentliche Unterschied zu einem normalen LLM-Chat:

| Merkmal | LLM Chat | AI Agent |
|---|---|---|
| Interaktion | Einmaliger Prompt → Antwort | Mehrere Schritte, iterativ |
| Tool-Nutzung | Keine (oder manuell) | Autonom, eigenständig |
| Zielorientierung | Antwortet auf Fragen | Löst komplexe Aufgaben |
| Gedächtnis | Nur aktueller Kontext | Persistentes Memory möglich |
| Autonomie | Passiv, reaktiv | Aktiv, proaktiv |
| Fehlerkorrektur | Nicht möglich | Kann Fehler erkennen und korrigieren |

> [!IMPORTANT]
> Ein Agent ist kein "besseres ChatGPT" – er ist ein System, das eigenständig handelt. Das bedeutet auch, dass er Fehler machen kann, die sich über mehrere Schritte fortpflanzen. Aufsicht und klare Grenzen sind essenziell.

---

## Kernkomponenten eines AI Agents

```
┌─────────────────────────────────────────┐
│              AI Agent                   │
│                                         │
│  ┌──────────┐    ┌────────────────────┐ │
│  │   LLM    │◄──►│      Memory        │ │
│  │ (Brain)  │    │ (Short/Long-term)  │ │
│  └────┬─────┘    └────────────────────┘ │
│       │                                 │
│  ┌────▼─────┐    ┌────────────────────┐ │
│  │ Planning │    │       Tools        │ │
│  │  Engine  │───►│ (Actions/APIs/Code)│ │
│  └──────────┘    └────────────────────┘ │
└─────────────────────────────────────────┘
```

### 1. LLM (Language Model)
Das "Gehirn" des Agenten. Zuständig für:
- Verstehen des Ziels und des aktuellen Zustands
- Entscheidung, welche Aktion als nächste ausgeführt wird
- Interpretation von Tool-Ergebnissen
- Generierung der finalen Antwort

Gängige LLMs: GPT-5.2, Claude Sonnet 4.6, Gemini 2.5 Pro, Llama 4 Scout

### 2. Tools
Schnittstellen zur Außenwelt. Beispiele:
- `web_search` – Suche im Internet
- `read_file` / `write_file` – Dateisystem-Zugriff
- `run_code` – Python/Bash-Ausführung
- `database_query` – Datenbankabfragen
- REST-API-Aufrufe (z.B. Jira, GitHub, Slack)

### 3. Memory
Kurzzeit- und Langzeitgedächtnis (siehe Abschnitt Memory-Typen).

### 4. Planning
Die Fähigkeit, ein Ziel in Teilaufgaben zu zerlegen und diese sequenziell oder parallel auszuführen.

---

## Der Agentic Loop

Der Kern jedes AI Agents ist ein iterativer Zyklus:

```
        ┌──────────────┐
        │  Ziel/Aufgabe│
        └──────┬───────┘
               │
       ┌───────▼────────┐
   ┌──►│  PERCEPTION    │  ◄── Kontext lesen, Zustand erfassen
   │   │  (Wahrnehmen)  │
   │   └───────┬────────┘
   │           │
   │   ┌───────▼────────┐
   │   │  REASONING     │  ◄── LLM entscheidet: Was als nächstes?
   │   │  (Schlussfolgern│
   │   └───────┬────────┘
   │           │
   │   ┌───────▼────────┐
   │   │    ACTION      │  ◄── Tool aufrufen / Code ausführen
   │   │  (Handeln)     │
   │   └───────┬────────┘
   │           │
   │   ┌───────▼────────┐
   └───┤  OBSERVATION   │  ◄── Ergebnis beobachten, weiter?
       │  (Beobachten)  │
       └───────┬────────┘
               │
       ┌───────▼────────┐
       │   FERTIG?      │ ──► Ja → Antwort ausgeben
       └────────────────┘ ──► Nein → Loop wiederholen
```

### Beispiel-Trace eines Agents (Dateien analysieren):

```
Ziel: "Analysiere alle Python-Dateien im Projekt und erstelle einen Report"

Loop 1:
  Reasoning: "Ich muss zuerst wissen, welche Dateien existieren."
  Action:    list_files(path="./src")
  Observation: ["main.py", "utils.py", "models.py", "tests/test_main.py"]

Loop 2:
  Reasoning: "Ich lese jetzt main.py"
  Action:    read_file("./src/main.py")
  Observation: "def main(): ..."

Loop 3-5: [liest utils.py, models.py, test_main.py]

Loop 6:
  Reasoning: "Ich habe alle Daten. Ich schreibe jetzt den Report."
  Action:    write_file("report.md", "# Analyse-Report\n...")
  Observation: "Datei erfolgreich geschrieben"

Fertig → "Report wurde unter report.md gespeichert."
```

> [!NOTE]
> Die Anzahl der Loops ist nicht fix. Ein Agent kann 3 oder 50 Iterationen durchlaufen – je nach Komplexität der Aufgabe und Qualität des Plannings.

---

## Tool Use: Wie Agenten mit der Außenwelt interagieren

Tools werden üblicherweise als JSON-Schema definiert und dem LLM als verfügbare Funktionen präsentiert. Das LLM entscheidet dann, wann und wie es ein Tool aufruft.

### Tool-Definition (OpenAI-Format):

```json
{
  "name": "web_search",
  "description": "Sucht im Internet nach aktuellen Informationen",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Der Suchbegriff"
      },
      "num_results": {
        "type": "integer",
        "description": "Anzahl der Ergebnisse (1-10)",
        "default": 5
      }
    },
    "required": ["query"]
  }
}
```

### Tool-Aufruf durch das LLM (Function Call):

```json
{
  "tool": "web_search",
  "parameters": {
    "query": "Claude Sonnet 4.6 release date",
    "num_results": 3
  }
}
```

### Tool-Ergebnis zurück ans LLM:

```json
{
  "tool_result": "web_search",
  "content": [
    {"title": "Anthropic Blog", "url": "...", "snippet": "Claude Sonnet 4.6 was released on..."}
  ]
}
```

> [!TIP]
> Gute Tool-Descriptions sind entscheidend. Das LLM entscheidet allein anhand der Beschreibung, ob und wann es ein Tool einsetzt. Vage Beschreibungen führen zu falschen oder unnötigen Tool-Aufrufen.

### Häufige Tool-Kategorien:

| Kategorie | Beispiele |
|---|---|
| Datei-Operationen | read_file, write_file, list_dir, delete_file |
| Code-Ausführung | python_repl, bash_exec, node_exec |
| Web/HTTP | web_search, http_get, http_post |
| Datenbanken | sql_query, vector_search, redis_get |
| APIs | github_api, jira_api, slack_send |
| Kommunikation | send_email, create_ticket |
| System | get_env, run_process, read_logs |

---

## Memory-Typen

### 1. In-Context Memory (Kurzzeit)
Der aktuelle Kontext-Window des LLMs. Alles, was im aktuellen Gespräch passiert ist.

- **Vorteil:** Direkt verfügbar, kein Setup nötig
- **Nachteil:** Begrenzt (128k – 1M Token), geht beim Neustart verloren
- **Typische Nutzung:** Aktueller Gesprächsverlauf, Tool-Ergebnisse der laufenden Session

### 2. External Memory (Langzeit) – Vektordatenbank
Informationen werden als Vektoren gespeichert und per semantischer Suche abgerufen.

```python
# Beispiel: Dokument in Vektordatenbank speichern
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

vectorstore = Chroma(embedding_function=OpenAIEmbeddings())

# Dokument einpflegen
vectorstore.add_texts(
    texts=["Das Deployment erfolgt über GitHub Actions..."],
    metadatas=[{"source": "deployment_guide.md", "date": "2024-01-15"}]
)

# Semantische Suche
results = vectorstore.similarity_search("Wie deploye ich die Anwendung?", k=3)
```

- **Vorteil:** Persistiert über Sessions, skaliert unbegrenzt
- **Nachteil:** Erfordert [[Embeddings|Embedding]]-Modell, Latenz bei Abfragen
- **Typische Nutzung:** Wissensbasis, Dokumentationen, vergangene Interaktionen

### 3. Episodic Memory
Strukturierte Aufzeichnung vergangener Aktionen und deren Ergebnisse.

```json
{
  "episode_id": "ep_20240115_001",
  "task": "Bug in payment module fixen",
  "steps": [
    {"action": "read_file", "file": "payment.py", "result": "success"},
    {"action": "run_tests", "result": "3 tests failed"},
    {"action": "edit_file", "file": "payment.py", "change": "Fixed null check"},
    {"action": "run_tests", "result": "all tests passed"}
  ],
  "outcome": "success",
  "learnings": "Null-Checks bei externen API-Responses immer prüfen"
}
```

- **Vorteil:** Agent kann aus vergangenen Erfahrungen lernen
- **Nachteil:** Komplex zu implementieren, erfordert sorgfältiges Design

### 4. Semantic/Procedural Memory
Allgemeines Wissen und gelernte Prozeduren – typischerweise direkt im System-Prompt verankert.

```
System: Du bist ein DevOps-Agent. Wenn du Kubernetes-Fehler debuggst,
        gehe immer in dieser Reihenfolge vor:
        1. kubectl get events
        2. kubectl describe pod <name>
        3. kubectl logs <name>
```

---

## Typen von Agenten

### Single Agent
Ein einzelner Agent bearbeitet die gesamte Aufgabe.

```
User → [Agent mit Tools] → Ergebnis
```

**Geeignet für:** Klar abgegrenzte Aufgaben, einfache Workflows, schnelle Prototypen

### Multi-Agent (Flach)
Mehrere Agenten arbeiten parallel oder sequenziell, koordiniert von einem Orchestrator.

```
User → [Orchestrator]
              ├──► [Research Agent]  → Ergebnis 1
              ├──► [Code Agent]      → Ergebnis 2
              └──► [Writer Agent]    → Ergebnis 3
                                    ↓
                             [Orchestrator fasst zusammen]
```

**Geeignet für:** Parallelisierbare Tasks, Spezialisierung nach Domänen

### Hierarchisches Multi-Agent-System
Agenten koordinieren Agenten, die ihrerseits weitere Agenten koordinieren.

```
[Meta-Orchestrator]
    ├──► [Orchestrator A: Backend-Team]
    │         ├──► [API-Agent]
    │         └──► [DB-Agent]
    └──► [Orchestrator B: Frontend-Team]
              ├──► [React-Agent]
              └──► [CSS-Agent]
```

**Geeignet für:** Sehr komplexe Softwareprojekte, Enterprise-Workflows

> [!WARNING]
> Hierarchische Systeme sind schwer zu debuggen. Fehler können durch mehrere Ebenen propagieren, bevor sie auffallen. Beginne immer mit dem einfachsten Ansatz.

---

## Anwendungsbeispiele

| Anwendung | Beschreibung | Tools |
|---|---|---|
| Code-Assistent | Code schreiben, testen, debuggen | read_file, write_file, run_tests, bash |
| Research Agent | Web recherchieren, zusammenfassen | web_search, read_url, write_file |
| DevOps-Assistent | Deployments, Monitoring, Alerts | kubectl, docker, aws_cli, slack |
| Customer Support | Tickets analysieren, antworten | jira_api, email_send, kb_search |
| Data Analyst | Daten laden, analysieren, visualisieren | sql_query, python_repl, plot |
| Content Creator | Recherche + Artikel schreiben | web_search, image_search, write_file |

---

## Bekannte Agent-Frameworks

### LangChain / LangGraph

```python
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_openai import ChatOpenAI
from langchain.tools import tool

@tool
def search_docs(query: str) -> str:
    """Sucht in der internen Dokumentation."""
    # ... Implementierung
    return results

llm = ChatOpenAI(model="gpt-4o")
agent = create_openai_tools_agent(llm, tools=[search_docs], prompt=prompt)
executor = AgentExecutor(agent=agent, tools=[search_docs], verbose=True)

result = executor.invoke({"input": "Wie konfiguriere ich den CI/CD-Pipeline?"})
```

**Stärken:** Riesiges Ökosystem, viele Integrationen, LangGraph für komplexe Workflows
**Schwächen:** Hohe Abstraktionskomplexität, Overhead

### LlamaIndex

```python
from llama_index.core.agent import ReActAgent
from llama_index.core.tools import FunctionTool

def multiply(a: float, b: float) -> float:
    """Multipliziert zwei Zahlen."""
    return a * b

multiply_tool = FunctionTool.from_defaults(fn=multiply)
agent = ReActAgent.from_tools([multiply_tool], llm=llm, verbose=True)
response = agent.chat("Wie viel ist 7 mal 8?")
```

**Stärken:** Exzellent für RAG + Agent Kombination, gute Datenquellen-Anbindung
**Schwächen:** Fokus auf Retrieval, weniger auf komplexe Workflows

### AutoGen (Microsoft)

```python
import autogen

assistant = autogen.AssistantAgent(
    name="Assistent",
    llm_config={"model": "gpt-4o"}
)

user_proxy = autogen.UserProxyAgent(
    name="User",
    human_input_mode="NEVER",
    code_execution_config={"work_dir": "coding"}
)

user_proxy.initiate_chat(assistant, message="Schreibe ein Python-Script für Fibonacci.")
```

**Stärken:** Natürliche Multi-Agent-Konversationen, Code-Ausführung built-in
**Schwächen:** Kann in Loops stecken bleiben, Debugging schwierig

### CrewAI

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Senior Researcher",
    goal="Finde die neuesten AI-Trends",
    backstory="Erfahrener Technologie-Analyst",
    verbose=True
)

writer = Agent(
    role="Content Writer",
    goal="Schreibe einen verständlichen Artikel",
    backstory="Erfahrener Tech-Journalist"
)

research_task = Task(
    description="Recherchiere die Top-5-AI-Trends 2025",
    agent=researcher
)

writing_task = Task(
    description="Schreibe einen Artikel basierend auf der Recherche",
    agent=writer,
    context=[research_task]
)

crew = Crew(agents=[researcher, writer], tasks=[research_task, writing_task])
result = crew.kickoff()
```

**Stärken:** Einfaches API, rollenbasierte Agenten, gut für sequenzielle Workflows
**Schwächen:** Weniger Flexibilität bei komplexen Graphen

### Vergleich der Frameworks:

| Framework | Komplexität | Multi-Agent | RAG | Code-Exec | Best for |
|---|---|---|---|---|---|
| LangChain/LangGraph | Hoch | Ja (Graph) | Gut | Möglich | Komplexe Workflows |
| LlamaIndex | Mittel | Begrenzt | Exzellent | Möglich | RAG-zentriert |
| AutoGen | Mittel | Ja (Chat) | Möglich | Built-in | Konversations-Agenten |
| CrewAI | Niedrig | Ja (Rollen) | Möglich | Möglich | Schnelle Prototypen |

---

## Verwandte Notizen

- [[Multi-Agent Workflows]]
- [[Custom Instructions für AI Coding Agents]]
- [[Claude Code Konfiguration und Hooks]]
