# Structured Output

> [!ABSTRACT]
> **Structured Output** ermöglicht es LLMs, Daten in einem garantiert validen, maschinenlesbaren Format (JSON, XML) auszugeben — statt Freitext. Mit JSON Schema, Pydantic-Modellen und Strict Mode liefern moderne LLMs 100% schema-konforme Ausgaben. Structured Output ist die Grundlage für zuverlässige AI-Pipelines und eng verwandt mit [[Function Calling und Tool Use]].

---

## Übersicht

| Aspekt | Details |
|---|---|
| **Zweck** | LLM-Output in garantiert validem Format |
| **Formate** | JSON (primär), XML, YAML |
| **Schema-Sprache** | JSON Schema, Pydantic (Python), Zod (TypeScript) |
| **Garantie** | 100% Schema-Compliance bei Strict Mode (OpenAI, Anthropic) |
| **Nutzen** | ~90% weniger Parse-Fehler in Produktions-Pipelines |

---

## Warum Structured Output?

### Das Problem mit Freitext

```
User: "Extrahiere Name und Alter aus dem Text"

❌ Ohne Structured Output (Freitext):
"Der Name ist Max Mustermann und er ist 32 Jahre alt."
→ Muss mit Regex/NLP geparst werden
→ Format variiert bei jedem Aufruf
→ Fehleranfällig bei Sonderzeichen

✅ Mit Structured Output (JSON):
{"name": "Max Mustermann", "age": 32}
→ Sofort maschinenlesbar
→ Immer gleiche Struktur
→ Typ-sicher (age ist number, nicht string)
```

### Wo Structured Output unverzichtbar ist

| Anwendung | Warum |
|---|---|
| **Datenbank-Integration** | Felder müssen exakt zum Schema passen |
| **API-Responses** | Frontend erwartet definierte Struktur |
| **Multi-Step-Pipelines** | Output von Schritt 1 = Input von Schritt 2 |
| **Datenextraktion** | Informationen aus unstrukturierten Texten |
| **UI-Generierung** | Komponenten brauchen typisierte Props |

---

## Drei Ansätze

### 1. JSON Mode (Basis)

Das LLM wird angewiesen, JSON auszugeben. **Garantiert**: Valides JSON. **Nicht garantiert**: Richtige Felder.

```python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    response_format={"type": "json_object"},
    messages=[
        {"role": "system", "content": "Antworte immer als JSON mit name und age"},
        {"role": "user", "content": "Max Mustermann, 32 Jahre"}
    ]
)
# → {"name": "Max Mustermann", "age": 32}
# ⚠️ Feld-Namen könnten variieren (Name vs name vs full_name)
```

### 2. Function/Tool Calling (Mittel)

Stärkere Struktur-Erzwingung über Tool-Schemas. Siehe [[Function Calling und Tool Use]].

```python
tools = [{
    "type": "function",
    "function": {
        "name": "extract_person",
        "parameters": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "age": {"type": "integer"}
            },
            "required": ["name", "age"]
        }
    }
}]
# → Feld-Namen sind jetzt fixiert
```

### 3. Strict Structured Output (Maximum)

**100% Schema-Compliance** — das LLM *muss* das Schema einhalten oder der Call schlägt fehl.

```python
from openai import OpenAI
from pydantic import BaseModel

class Person(BaseModel):
    name: str
    age: int
    email: str | None = None

client = OpenAI()
response = client.beta.chat.completions.parse(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Max Mustermann, 32, max@example.com"}],
    response_format=Person
)

person = response.choices[0].message.parsed
print(person.name)   # Max Mustermann
print(person.age)    # 32
print(person.email)  # max@example.com
# → 100% typ-sicher, 100% schema-konform
```

---

## Provider-Support

| Provider | JSON Mode | Tool Schema | Strict Mode | Pydantic | Notes |
|---|---|---|---|---|---|
| **OpenAI** | ✅ | ✅ | ✅ (100%) | ✅ Nativ | `.parse()` seit GPT-4o |
| **Anthropic** | ✅ | ✅ | ✅ (ab Sonnet 4.5) | Indirekt | Via Tool Use |
| **Google Gemini** | ✅ | ✅ | ✅ | Indirekt | `responseSchema` Parameter |
| **Mistral** | ✅ | ✅ | Partiell | ❌ | Client-seitige Validierung empfohlen |
| **AWS Bedrock** | ✅ | ✅ | ✅ | Indirekt | Via Converse API |
| **Open Source** | Variiert | Via Frameworks | Via Outlines/Instructor | Via Frameworks | Constrained Decoding |

---

## Pydantic als Schema-Sprache

**Pydantic** ist der De-facto-Standard für Structured Output in Python:

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum

class Priority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

class Task(BaseModel):
    title: str = Field(description="Kurzer Titel der Aufgabe")
    description: str = Field(description="Detaillierte Beschreibung")
    priority: Priority = Field(description="Prioritätsstufe")
    assignee: Optional[str] = Field(None, description="Zugewiesene Person")
    estimated_hours: float = Field(ge=0, description="Geschätzte Stunden")

class TaskList(BaseModel):
    tasks: list[Task]
    total_estimated_hours: float
```

### Zod für TypeScript

```typescript
import { z } from "zod";

const TaskSchema = z.object({
  title: z.string(),
  description: z.string(),
  priority: z.enum(["low", "medium", "high"]),
  assignee: z.string().optional(),
  estimatedHours: z.number().nonnegative()
});

const TaskListSchema = z.object({
  tasks: z.array(TaskSchema),
  totalEstimatedHours: z.number()
});
```

---

## Open-Source-Alternativen

Für LLMs ohne nativen Strict Mode gibt es Bibliotheken:

| Library | Ansatz | Beschreibung |
|---|---|---|
| **Instructor** | Pydantic + Retry | Wrapper um OpenAI/Anthropic mit automatischem Retry bei Schema-Verletzungen |
| **Outlines** | Constrained Decoding | Erzwingt Schema bereits während der Token-Generierung |
| **Guardrails AI** | Validierung + Fix | Validiert Output und versucht automatische Korrektur |
| **LMQL** | Query Language | SQL-ähnliche Sprache für strukturierte LLM-Abfragen |

### Instructor-Beispiel

```python
import instructor
from openai import OpenAI
from pydantic import BaseModel

client = instructor.from_openai(OpenAI())

class UserInfo(BaseModel):
    name: str
    age: int

user = client.chat.completions.create(
    model="gpt-4o",
    response_model=UserInfo,
    messages=[{"role": "user", "content": "Max, 32 Jahre alt"}]
)
# → UserInfo(name="Max", age=32)
# Bei Schema-Verletzung: Automatischer Retry
```

---

## Structured Output vs. Function Calling

| Aspekt | Structured Output | [[Function Calling und Tool Use\|Function Calling]] |
|---|---|---|
| **Zweck** | Daten extrahieren/formatieren | Aktionen ausführen |
| **Wer handelt** | Nur LLM (generiert Daten) | LLM + App (führt Funktion aus) |
| **Ergebnis** | Typisiertes Datenobjekt | Funktions-Aufruf + Ergebnis-Loop |
| **Beispiel** | "Extrahiere Kontaktdaten" | "Sende E-Mail an Max" |
| **Schema** | `response_format` | `tools` Parameter |
| **Beziehung** | Function Calling nutzt Structured Output intern | |

---

## Best Practices

1. **Strict Mode bevorzugen** — Wo verfügbar, immer Strict Mode / Pydantic nutzen
2. **Beschreibungen hinzufügen** — `Field(description="...")` hilft dem LLM, Felder korrekt zu füllen
3. **Enum statt String** — Für begrenzte Optionen immer Enums verwenden
4. **Optional explizit markieren** — `Optional[str]` statt Hoffen, dass das LLM `null` setzt
5. **Nested Models** — Komplexe Strukturen in Sub-Models aufteilen (besser als flache Objekte)
6. **Validierung doppeln** — Auch bei Strict Mode serverseitig validieren (Defense in Depth)
7. **Fallback definieren** — Was passiert, wenn das LLM das Schema nicht einhalten kann? (Refusal handling)

---

## Verwandte Notizen

- [[Function Calling und Tool Use]] — Aktions-basierte strukturierte Interaktion
- [[Was ist MCP]] — MCP standardisiert Tool-Schemas
- [[Guardrails und AI Safety]] — Output-Validierung als Sicherheitsschicht
- [[Agent Frameworks]] — Frameworks nutzen Structured Output für Tool-Ergebnisse
- [[Context Window und Token Management]] — Structured Output reduziert Token-Verschwendung
