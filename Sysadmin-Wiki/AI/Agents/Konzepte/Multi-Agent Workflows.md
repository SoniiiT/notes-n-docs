---
tags: [ai, agents, konzept, multi-agent]
---

# Multi-Agent Workflows

## Warum Multi-Agent?

Ein einzelner Agent stößt schnell an Grenzen: Der Kontext-Window füllt sich, die Aufgabe ist zu komplex für eine lineare Abarbeitung, oder verschiedene Teile erfordern unterschiedliches Spezialwissen. Multi-Agent-Systeme lösen diese Probleme durch drei Kernprinzipien:

### 1. Parallelisierung
Unabhängige Teilaufgaben können gleichzeitig ausgeführt werden.

```
Single Agent (sequenziell):          Multi-Agent (parallel):

Aufgabe A: 30s                        Aufgabe A: 30s ──►
Aufgabe B: 25s          →             Aufgabe B: 25s ──►  Gesamt: 30s
Aufgabe C: 20s                        Aufgabe C: 20s ──►
─────────────
Gesamt: 75s
```

### 2. Spezialisierung
Jeder Agent hat einen eigenen System-Prompt, optimiert für seine spezifische Aufgabe.

| Spezialist-Agent | System-Prompt-Fokus |
|---|---|
| Security Reviewer | OWASP Top 10, sichere Coding-Patterns, Schwachstellen |
| Performance Analyst | Big-O-Analyse, Profiling, Caching-Strategien |
| Documentation Writer | Klare Sprache, API-Dokumentation, Beispiele |
| Test Engineer | Edge Cases, Testabdeckung, Mocking-Strategien |

### 3. Kontext-Trennung
Jeder Agent hat nur den Kontext, den er für seine Aufgabe braucht. Kein "Rauschen" durch irrelevante Informationen.

> [!NOTE]
> Multi-Agent-Systeme sind kein Allheilmittel. Die Koordination kostet Zeit und Tokens. Für einfache Aufgaben ist ein gut konfigurierter Single-Agent immer die bessere Wahl.

---

## Das Orchestrator-Subagent-Muster

Das häufigste Multi-Agent-Muster: Ein **Orchestrator** koordiniert mehrere spezialisierte **Subagenten**.

```
┌─────────────────────────────────────────────────────┐
│                   ORCHESTRATOR                       │
│  - Versteht das Gesamtziel                          │
│  - Zerlegt in Teilaufgaben                          │
│  - Delegiert an Subagenten                          │
│  - Integriert Ergebnisse                            │
└──────┬──────────────┬─────────────────┬─────────────┘
       │              │                 │
       ▼              ▼                 ▼
┌──────────┐   ┌──────────┐   ┌──────────────┐
│ Subagent │   │ Subagent │   │   Subagent   │
│    A     │   │    B     │   │      C       │
│ Research │   │  Coding  │   │   Testing    │
└──────────┘   └──────────┘   └──────────────┘
```

### Implementierung (konzeptionell):

```python
import asyncio
from anthropic import Anthropic

client = Anthropic()

def run_subagent(role: str, system_prompt: str, task: str) -> str:
    """Führt einen spezialisierten Subagenten aus."""
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        system=system_prompt,
        messages=[{"role": "user", "content": task}]
    )
    return response.content[0].text

def orchestrator(main_task: str) -> str:
    """Orchestrator zerlegt Aufgabe und koordiniert Subagenten."""

    # Schritt 1: Orchestrator plant
    plan_response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="Du bist ein Orchestrator. Zerlege die Aufgabe in parallele Subtasks.",
        messages=[{"role": "user", "content": main_task}]
    )

    # Schritt 2: Subagenten sequenziell ausführen
    results = {
        "research": run_subagent(
            "Researcher",
            "Du bist ein Research-Spezialist. Sammle alle relevanten Fakten.",
            f"Recherchiere für: {main_task}"
        ),
        "implementation": run_subagent(
            "Developer",
            "Du bist ein Senior Developer. Schreibe produktionsreifen Code.",
            f"Implementiere: {main_task}"
        ),
        "tests": run_subagent(
            "QA Engineer",
            "Du bist ein QA-Experte. Schreibe umfassende Tests.",
            f"Teste: {main_task}"
        )
    }

    # Schritt 3: Orchestrator integriert Ergebnisse
    final_response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system="Integriere die Ergebnisse der Subagenten zu einer kohärenten Lösung.",
        messages=[{
            "role": "user",
            "content": f"Ergebnisse:\n{results}\n\nIntegriere diese zu einer Gesamtlösung."
        }]
    )

    return final_response.content[0].text
```

> [!TIP]
> Der Orchestrator sollte ein leistungsfähiges Modell (z.B. Claude Opus) sein, während Subagenten oft mit einem schnelleren, günstigeren Modell (z.B. Claude Haiku) auskommen – besonders für wiederholende Tasks.

---

## Parallele Agenten

Für voneinander unabhängige Aufgaben, die gleichzeitig ausgeführt werden können.

```
Input
  │
  ├──► Agent 1 (asyncio.Task) ──► Result 1 ──►
  ├──► Agent 2 (asyncio.Task) ──► Result 2 ──► Aggregator ──► Final Output
  └──► Agent 3 (asyncio.Task) ──► Result 3 ──►
```

### Implementierung mit asyncio:

```python
import asyncio
import anthropic

async def analyze_component(client: anthropic.AsyncAnthropic, component: str, code: str) -> dict:
    """Analysiert eine Code-Komponente aus einer spezifischen Perspektive."""

    prompts = {
        "security": f"Analysiere diesen Code auf Sicherheitsprobleme:\n```\n{code}\n```",
        "performance": f"Analysiere die Performance dieses Codes:\n```\n{code}\n```",
        "maintainability": f"Bewerte die Wartbarkeit dieses Codes:\n```\n{code}\n```",
    }

    response = await client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=1024,
        system=f"Du bist ein {component}-Experte. Analysiere präzise und gib konkrete Empfehlungen.",
        messages=[{"role": "user", "content": prompts[component]}]
    )

    return {
        "component": component,
        "analysis": response.content[0].text
    }

async def parallel_code_review(code: str) -> list[dict]:
    """Führt parallele Code-Reviews aus verschiedenen Perspektiven durch."""
    client = anthropic.AsyncAnthropic()

    # Alle 3 Analysen gleichzeitig starten
    tasks = [
        analyze_component(client, "security", code),
        analyze_component(client, "performance", code),
        analyze_component(client, "maintainability", code),
    ]

    # Auf alle warten
    results = await asyncio.gather(*tasks)
    return list(results)

# Ausführung
code_to_review = """
def get_user(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return db.execute(query)
"""

results = asyncio.run(parallel_code_review(code_to_review))
for result in results:
    print(f"\n=== {result['component'].upper()} ===")
    print(result['analysis'])
```

---

## Sequenzielle Agenten

Der Output von Agent A wird als Input für Agent B verwendet.

```
Input
  │
  ▼
Agent 1 (Researcher)
  │ Output: Rohe Daten / Fakten
  ▼
Agent 2 (Analyst)
  │ Output: Strukturierte Analyse
  ▼
Agent 3 (Writer)
  │ Output: Fertiges Dokument
  ▼
Final Output
```

### Implementierung (Pipeline-Pattern):

```python
from dataclasses import dataclass
from anthropic import Anthropic

@dataclass
class PipelineStage:
    name: str
    system_prompt: str
    input_template: str

def run_pipeline(initial_input: str, stages: list[PipelineStage]) -> dict:
    """Führt eine sequenzielle Agent-Pipeline aus."""
    client = Anthropic()
    results = {"input": initial_input}
    current_input = initial_input

    for stage in stages:
        print(f"Stage: {stage.name}...")

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system=stage.system_prompt,
            messages=[{
                "role": "user",
                "content": stage.input_template.format(input=current_input)
            }]
        )

        current_input = response.content[0].text
        results[stage.name] = current_input

    return results

# Beispiel: Blog-Post-Erstellungs-Pipeline
pipeline = [
    PipelineStage(
        name="research",
        system_prompt="Du bist ein Research-Spezialist. Sammle Fakten und Quellen.",
        input_template="Recherchiere das Thema: {input}"
    ),
    PipelineStage(
        name="outline",
        system_prompt="Du bist ein Content-Stratege. Erstelle eine strukturierte Gliederung.",
        input_template="Erstelle eine Blog-Post-Gliederung basierend auf:\n{input}"
    ),
    PipelineStage(
        name="draft",
        system_prompt="Du bist ein erfahrener Tech-Autor. Schreibe verständlich und präzise.",
        input_template="Schreibe einen Blog-Post nach dieser Gliederung:\n{input}"
    ),
    PipelineStage(
        name="review",
        system_prompt="Du bist ein Editor. Verbessere Sprache, Struktur und Verständlichkeit.",
        input_template="Überarbeite und optimiere diesen Blog-Post:\n{input}"
    )
]

result = run_pipeline("Kubernetes Networking für Einsteiger", pipeline)
print(result["review"])  # Finaler, überarbeiteter Blog-Post
```

---

## Kommunikation zwischen Agenten

Agenten können auf verschiedene Arten miteinander kommunizieren:

### 1. Direkter Message-Pass (einfachste Methode)
Output eines Agenten wird direkt als Input des nächsten übergeben.

### 2. Shared Memory / Blackboard
Alle Agenten lesen und schreiben in einen gemeinsamen Zustandsspeicher.

```python
from typing import Any

class SharedWorkspace:
    """Gemeinsamer Arbeitsbereich für alle Agenten."""

    def __init__(self):
        self.artifacts: dict = {}     # Ergebnisse
        self.messages: list = []      # Nachrichten zwischen Agenten
        self.status: dict = {}        # Status pro Aufgabe
        self.errors: list = []        # Fehlerliste

    def post_result(self, agent_name: str, task_id: str, result: Any):
        self.artifacts[f"{agent_name}_{task_id}"] = result
        self.status[task_id] = "completed"

    def get_result(self, agent_name: str, task_id: str) -> Any:
        return self.artifacts.get(f"{agent_name}_{task_id}")

    def send_message(self, from_agent: str, to_agent: str, content: str):
        self.messages.append({
            "from": from_agent,
            "to": to_agent,
            "content": content,
            "timestamp": datetime.now().isoformat()
        })

    def get_messages_for(self, agent_name: str) -> list:
        return [m for m in self.messages if m["to"] == agent_name]

workspace = SharedWorkspace()
```

### 3. Event-basierte Kommunikation
Agenten reagieren auf Events, die andere Agenten auslösen.

```python
import asyncio
from typing import Callable

class AgentEventBus:
    """Event-Bus für Agenten-Kommunikation."""

    def __init__(self):
        self._handlers: dict[str, list[Callable]] = {}

    def subscribe(self, event_type: str, handler: Callable):
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)

    async def publish(self, event_type: str, data: dict):
        handlers = self._handlers.get(event_type, [])
        await asyncio.gather(*[handler(data) for handler in handlers])

event_bus = AgentEventBus()

# Research-Agent veröffentlicht Ergebnisse
async def research_agent(topic: str):
    result = await do_research(topic)
    await event_bus.publish("research_complete", {"topic": topic, "data": result})

# Writer-Agent reagiert auf Research-Ergebnisse
async def writer_agent(event_data: dict):
    article = await write_article(event_data["data"])
    await event_bus.publish("article_draft_ready", {"article": article})

event_bus.subscribe("research_complete", writer_agent)
```

---

## Fehlerbehandlung in Multi-Agent Systemen

> [!WARNING]
> Fehler in Multi-Agent-Systemen sind schwerer zu debuggen als in Single-Agent-Systemen. Ein Fehler in einem frühen Schritt kann sich durch alle nachfolgenden Agenten fortpflanzen.

### Fehlertypen und Strategien:

| Fehlertyp | Beschreibung | Strategie |
|---|---|---|
| Timeout | Subagent antwortet nicht | Retry mit Exponential Backoff |
| Halluzination | Agent gibt falsches Ergebnis | Validation-Agent als Prüfschicht |
| Loop | Agenten rufen sich gegenseitig endlos auf | Max-Iteration-Counter |
| Kontext-Verlust | Agent vergisst vorherigen Kontext | Explizite Kontext-Übergabe |
| Ressourcen-Erschöpfung | Zu viele parallele Calls | Rate Limiting + Queue |

### Robuste Fehlerbehandlung:

```python
import asyncio
from typing import Optional
from dataclasses import dataclass, field

@dataclass
class AgentResult:
    success: bool
    data: Optional[str] = None
    error: Optional[str] = None
    retries: int = 0

async def run_agent_with_retry(
    agent_fn: callable,
    task: str,
    max_retries: int = 3,
    backoff_base: float = 2.0
) -> AgentResult:
    """Führt einen Agenten mit automatischem Retry aus."""

    for attempt in range(max_retries + 1):
        try:
            result = await agent_fn(task)
            return AgentResult(success=True, data=result, retries=attempt)

        except asyncio.TimeoutError:
            if attempt < max_retries:
                wait_time = backoff_base ** attempt
                print(f"Timeout. Retry {attempt + 1}/{max_retries} in {wait_time}s...")
                await asyncio.sleep(wait_time)
            else:
                return AgentResult(
                    success=False,
                    error=f"Timeout nach {max_retries} Versuchen",
                    retries=attempt
                )

        except Exception as e:
            return AgentResult(success=False, error=str(e), retries=attempt)

async def safe_parallel_execution(tasks: list[tuple[callable, str]]) -> list[AgentResult]:
    """Führt mehrere Agenten parallel aus, toleriert Einzelfehler."""

    coroutines = [
        run_agent_with_retry(agent_fn, task)
        for agent_fn, task in tasks
    ]

    # return_exceptions=True verhindert, dass ein Fehler alle abbricht
    results = await asyncio.gather(*coroutines, return_exceptions=True)

    # Exceptions in AgentResult umwandeln
    processed = []
    for r in results:
        if isinstance(r, Exception):
            processed.append(AgentResult(success=False, error=str(r)))
        else:
            processed.append(r)

    return processed
```

### Validation-Agent Pattern:

```python
async def validated_pipeline(task: str) -> str:
    """Pipeline mit eingebautem Validation-Agent."""

    # Schritt 1: Hauptagent generiert Ergebnis
    draft = await main_agent(task)

    # Schritt 2: Validation-Agent prüft das Ergebnis
    validation_prompt = f"""
    Prüfe dieses Ergebnis auf Korrektheit und Vollständigkeit:

    Aufgabe: {task}
    Ergebnis: {draft}

    Antworte mit:
    - VALID: [Ergebnis ist korrekt und vollständig]
    - INVALID: [Begründung was falsch/unvollständig ist]
    """

    validation = await validator_agent(validation_prompt)

    if validation.startswith("VALID"):
        return draft
    else:
        # Schritt 3: Korrektur-Agent verbessert das Ergebnis
        correction_prompt = f"""
        Das Ergebnis hat Probleme: {validation}

        Ursprüngliche Aufgabe: {task}
        Fehlerhaftes Ergebnis: {draft}

        Erstelle eine korrigierte Version.
        """
        return await correction_agent(correction_prompt)
```

---

## Beispiel: Parallele Dokumentations-Generierung

Ein reales Szenario: Automatische Generierung von API-Dokumentation für ein Softwareprojekt.

```python
import asyncio
import anthropic
from pathlib import Path

async def document_module(
    client: anthropic.AsyncAnthropic,
    module_path: str,
    module_code: str
) -> dict:
    """Dokumentiert ein einzelnes Modul."""

    response = await client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=2048,
        system="""Du bist ein technischer Dokumentations-Experte.
        Erstelle präzise, vollständige API-Dokumentation im Google-Style.
        Dokumentiere alle öffentlichen Funktionen, Klassen und Parameter.""",
        messages=[{
            "role": "user",
            "content": f"Dokumentiere dieses Python-Modul:\n\nPfad: {module_path}\n\n```python\n{module_code}\n```"
        }]
    )

    return {
        "module": module_path,
        "documentation": response.content[0].text
    }

async def generate_project_docs(project_path: str) -> dict:
    """Generiert Dokumentation für alle Module parallel."""

    client = anthropic.AsyncAnthropic()

    # Alle Python-Dateien finden
    modules = {}
    for py_file in Path(project_path).rglob("*.py"):
        if not py_file.name.startswith("_"):
            modules[str(py_file)] = py_file.read_text()

    print(f"Dokumentiere {len(modules)} Module parallel...")

    # Alle Module gleichzeitig dokumentieren
    tasks = [
        document_module(client, path, code)
        for path, code in modules.items()
    ]

    results = await asyncio.gather(*tasks)

    # Orchestrator-Phase: Zusammenfassenden Index erstellen
    module_summaries = "\n\n".join([
        f"## {r['module']}\n{r['documentation'][:500]}..."
        for r in results
    ])

    index_response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system="Erstelle einen übersichtlichen Dokumentations-Index.",
        messages=[{
            "role": "user",
            "content": f"Erstelle einen Index für diese Modul-Dokumentationen:\n\n{module_summaries}"
        }]
    )

    return {
        "modules": {r["module"]: r["documentation"] for r in results},
        "index": index_response.content[0].text
    }

# Ausführung
docs = asyncio.run(generate_project_docs("./src"))
```

---

## Beispiel: Code Review mit spezialisierten Agenten

```python
import asyncio
import anthropic
from typing import NamedTuple

class ReviewResult(NamedTuple):
    reviewer: str
    findings: list[str]
    severity: str  # "critical", "warning", "info"
    score: int     # 1-10

REVIEWER_CONFIGS = {
    "security": {
        "model": "claude-sonnet-4-6",
        "system": """Du bist ein Application Security Expert (OWASP, CVEs).
        Fokus: SQL Injection, XSS, SSRF, unsichere Deserialisierung, Krypto-Schwächen.
        Antworte im JSON-Format: {"findings": [...], "severity": "...", "score": N}""",
    },
    "performance": {
        "model": "claude-haiku-4-5",
        "system": """Du bist ein Performance Engineer.
        Fokus: Algorithmus-Komplexität, N+1-Queries, unnötige Allokationen, Blocking I/O.
        Antworte im JSON-Format: {"findings": [...], "severity": "...", "score": N}""",
    },
    "architecture": {
        "model": "claude-sonnet-4-6",
        "system": """Du bist ein Software Architect.
        Fokus: SOLID-Prinzipien, Design Patterns, Kopplung, Kohäsion, Testbarkeit.
        Antworte im JSON-Format: {"findings": [...], "severity": "...", "score": N}""",
    },
    "style": {
        "model": "claude-haiku-4-5",
        "system": """Du bist ein Code Quality Expert.
        Fokus: Naming, Kommentare, Komplexität, Duplikationen, Clean Code.
        Antworte im JSON-Format: {"findings": [...], "severity": "...", "score": N}""",
    }
}

async def run_reviewer(
    client: anthropic.AsyncAnthropic,
    reviewer_name: str,
    config: dict,
    code: str
) -> ReviewResult:
    """Führt einen spezialisierten Code-Reviewer aus."""

    import json

    response = await client.messages.create(
        model=config["model"],
        max_tokens=1024,
        system=config["system"],
        messages=[{
            "role": "user",
            "content": f"Reviewe diesen Code:\n\n```\n{code}\n```"
        }]
    )

    try:
        result = json.loads(response.content[0].text)
        return ReviewResult(
            reviewer=reviewer_name,
            findings=result.get("findings", []),
            severity=result.get("severity", "info"),
            score=result.get("score", 5)
        )
    except json.JSONDecodeError:
        return ReviewResult(
            reviewer=reviewer_name,
            findings=[response.content[0].text],
            severity="info",
            score=5
        )

async def multi_agent_code_review(code: str) -> dict:
    """Führt einen umfassenden Code Review mit spezialisierten Agenten durch."""

    client = anthropic.AsyncAnthropic()

    # Alle Reviewer parallel ausführen
    tasks = [
        run_reviewer(client, name, config, code)
        for name, config in REVIEWER_CONFIGS.items()
    ]

    print("Code Review läuft (4 Agenten parallel)...")
    results = await asyncio.gather(*tasks)

    # Orchestrator: Zusammenfassung erstellen
    all_findings = []
    for result in results:
        for finding in result.findings:
            all_findings.append(f"[{result.reviewer.upper()}] {finding}")

    summary_response = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="Erstelle eine priorisierte Code-Review-Zusammenfassung.",
        messages=[{
            "role": "user",
            "content": f"Priorisiere diese Findings:\n" + "\n".join(all_findings)
        }]
    )

    return {
        "individual_reviews": [r._asdict() for r in results],
        "overall_score": sum(r.score for r in results) / len(results),
        "critical_issues": [r for r in results if r.severity == "critical"],
        "summary": summary_response.content[0].text
    }

# Beispiel-Aufruf
problematic_code = """
def get_user_data(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    result = db.execute(query)
    return pickle.loads(result['data'])
"""

review = asyncio.run(multi_agent_code_review(problematic_code))
print(f"Overall Score: {review['overall_score']:.1f}/10")
print(f"\nSummary:\n{review['summary']}")
```

---

## Bekannte Multi-Agent Frameworks

### AutoGen (Microsoft)

AutoGen modelliert Multi-Agent-Systeme als Konversationen zwischen Agenten.

```python
import autogen

config_list = [{"model": "gpt-4o", "api_key": "..."}]

# Spezialisierte Agenten definieren
planner = autogen.AssistantAgent(
    name="Planner",
    system_message="""Du bist ein erfahrener Projekt-Planer.
    Erstelle strukturierte Aufgabenpläne und delegiere an Spezialisten.""",
    llm_config={"config_list": config_list}
)

coder = autogen.AssistantAgent(
    name="Senior_Developer",
    system_message="""Du bist ein Senior Software Engineer.
    Du schreibst sauberen, gut dokumentierten Code.""",
    llm_config={"config_list": config_list}
)

reviewer = autogen.AssistantAgent(
    name="Code_Reviewer",
    system_message="""Du bist ein Code Reviewer.
    Prüfe auf Bugs, Security Issues und Code Quality.""",
    llm_config={"config_list": config_list}
)

# User Proxy (führt Code aus)
user_proxy = autogen.UserProxyAgent(
    name="User",
    human_input_mode="NEVER",
    code_execution_config={
        "work_dir": "coding",
        "use_docker": True
    }
)

# Gruppen-Chat: Alle Agenten diskutieren
group_chat = autogen.GroupChat(
    agents=[user_proxy, planner, coder, reviewer],
    messages=[],
    max_round=12
)

manager = autogen.GroupChatManager(
    groupchat=group_chat,
    llm_config={"config_list": config_list}
)

user_proxy.initiate_chat(
    manager,
    message="Erstelle eine REST API für ein Todo-System mit Tests."
)
```

**Stärken:** Natürliche Gruppen-Konversationen, Code-Ausführung, flexibles Routing
**Schwächen:** Konversationen können sich wiederholen, schwer zu kontrollieren

### CrewAI

CrewAI fokussiert auf rollenbasierte Agenten in strukturierten Workflows.

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import SerperDevTool, FileWriterTool

search_tool = SerperDevTool()
write_tool = FileWriterTool()

# Crew definieren
researcher = Agent(
    role="Senior Research Analyst",
    goal="Finde präzise und aktuelle Informationen zu technischen Themen",
    backstory="15 Jahre Erfahrung in Tech-Research bei führenden Analysehäusern",
    tools=[search_tool],
    verbose=True,
    llm="claude-sonnet-4-6"
)

architect = Agent(
    role="Solution Architect",
    goal="Entwirf skalierbare, robuste Systemarchitekturen",
    backstory="Ex-Google SRE mit Expertise in verteilten Systemen",
    verbose=True,
    llm="claude-sonnet-4-6"
)

writer = Agent(
    role="Technical Writer",
    goal="Schreibe klare, verständliche technische Dokumentation",
    backstory="Spezialist für Developer Experience und API-Dokumentation",
    tools=[write_tool],
    verbose=True,
    llm="claude-haiku-4-5"
)

# Tasks mit Abhängigkeiten
research_task = Task(
    description="Recherchiere aktuelle Best Practices für {topic}",
    expected_output="Strukturierter Report mit 5+ Quellen",
    agent=researcher
)

architecture_task = Task(
    description="Erstelle eine Systemarchitektur basierend auf der Recherche",
    expected_output="Architektur-Diagramm als ASCII + Begründung der Entscheidungen",
    agent=architect,
    context=[research_task]  # Verwendet Research-Ergebnisse
)

documentation_task = Task(
    description="Erstelle vollständige Dokumentation für die Architektur",
    expected_output="Markdown-Datei mit Einleitung, Architektur, Setup-Guide",
    agent=writer,
    context=[research_task, architecture_task],
    output_file="architecture_docs.md"
)

# Crew zusammenstellen
crew = Crew(
    agents=[researcher, architect, writer],
    tasks=[research_task, architecture_task, documentation_task],
    process=Process.sequential,  # oder Process.hierarchical
    verbose=True
)

result = crew.kickoff(inputs={"topic": "Event-Driven Architecture mit Kafka"})
```

**Stärken:** Sauberes API, rollenbasiert, integrierte Tool-Unterstützung
**Schwächen:** Weniger flexibel bei dynamischen Workflows

### LangGraph

LangGraph modelliert Agenten-Workflows als Graphen mit explizitem State-Management.

```python
from typing import TypedDict, Annotated, Sequence
from langgraph.graph import StateGraph, END
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import BaseMessage, HumanMessage
import operator

class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    research_done: bool
    draft_ready: bool
    approved: bool

model = ChatAnthropic(model="claude-sonnet-4-6")

def research_node(state: AgentState) -> AgentState:
    """Research-Agent Node."""
    response = model.invoke([
        HumanMessage(content=f"Recherchiere: {state['messages'][-1].content}")
    ])
    return {
        "messages": [response],
        "research_done": True
    }

def write_node(state: AgentState) -> AgentState:
    """Writer-Agent Node."""
    response = model.invoke([
        HumanMessage(content=f"Schreibe basierend auf: {state['messages']}")
    ])
    return {
        "messages": [response],
        "draft_ready": True
    }

def review_node(state: AgentState) -> AgentState:
    """Review-Agent Node."""
    response = model.invoke([
        HumanMessage(content=f"Überprüfe: {state['messages'][-1].content}")
    ])
    approved = "APPROVED" in response.content
    return {
        "messages": [response],
        "approved": approved
    }

def should_revise(state: AgentState) -> str:
    """Conditional Edge: Überarbeitung notwendig?"""
    return END if state["approved"] else "write"

# Graph aufbauen
workflow = StateGraph(AgentState)

workflow.add_node("research", research_node)
workflow.add_node("write", write_node)
workflow.add_node("review", review_node)

workflow.set_entry_point("research")
workflow.add_edge("research", "write")
workflow.add_edge("write", "review")
workflow.add_conditional_edges("review", should_revise)

app = workflow.compile()

result = app.invoke({
    "messages": [HumanMessage(content="Schreibe einen Artikel über Kubernetes")],
    "research_done": False,
    "draft_ready": False,
    "approved": False
})
```

**Stärken:** Explizites State-Management, bedingte Edges, Loops möglich, sehr flexibel
**Schwächen:** Steilere Lernkurve, mehr Boilerplate-Code

### Framework-Vergleich:

| Kriterium | AutoGen | CrewAI | LangGraph |
|---|---|---|---|
| Lernkurve | Mittel | Niedrig | Hoch |
| Workflow-Typ | Konversation | Rollen/Tasks | Graph/State |
| Flexibilität | Hoch | Mittel | Sehr hoch |
| Code-Ausführung | Built-in | Via Tools | Via Tools |
| State Management | Implizit | Implizit | Explizit |
| Debugging | Schwierig | Einfach | Gut (Visualisierung) |
| Best for | Konversations-Agenten | Schnelle Prototypen | Komplexe Workflows |

---

## Verwandte Notizen

- [[Was ist ein AI Agent]]
- [[Custom Instructions für AI Coding Agents]]
- [[Claude Code Konfiguration und Hooks]]
