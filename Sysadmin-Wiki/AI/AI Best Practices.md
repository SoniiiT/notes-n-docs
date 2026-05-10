---
tags:
  - ai
  - best-practices
  - llm
  - agents
  - produktion
  - kosten
---

# AI Best Practices

> [!ABSTRACT] TL;DR
> Gesammelte Best Practices für den produktiven Einsatz von KI, LLMs und Agents — von der Modellwahl über Prompt-Optimierung und Kosten-Management bis hin zu Sicherheit, Reliability und Agent-Design. Dient als zentrale Referenz für den effektiven AI-Einsatz.

---

## Modellwahl

### Modell nach Aufgabe wählen

| Aufgabe | Empfohlene Kategorie | Beispiele |
|---------|---------------------|-----------|
| Code-Generierung | Code-spezialisiert | Claude Sonnet, GPT-4o, Codestral |
| Textzusammenfassung | Allzweck (günstig) | GPT-4o-mini, Claude Haiku, Gemini Flash |
| Analyse/Reasoning | Reasoning-Modelle | Claude Opus, o1, Gemini Pro |
| Übersetzung | Allzweck | GPT-4o, Claude Sonnet |
| Bild-Analyse | Multimodal | GPT-4o, Claude Sonnet, Gemini Pro |
| Einfache Klassifikation | Klein und schnell | Phi-3 Mini, Llama 3.1 8B |
| Lokales Development | Open Source | Llama 3.1, Mistral, CodeLlama |

### Entscheidungsbaum

```mermaid
graph TD
    A{Datenschutz kritisch?} -->|Ja| B{GPU vorhanden?}
    A -->|Nein| C{Budget?}
    B -->|Ja| D[Lokales Modell via Ollama]
    B -->|Nein| E[Kleine quantisierte Modelle]
    C -->|Hoch| F[Claude Opus / GPT-4o]
    C -->|Mittel| G[Claude Sonnet / GPT-4o]
    C -->|Niedrig| H[GPT-4o-mini / Claude Haiku]

    style D fill:#2ecc71,color:#fff
    style E fill:#f39c12,color:#fff
    style F fill:#9b59b6,color:#fff
    style G fill:#3498db,color:#fff
    style H fill:#1abc9c,color:#fff
```

### Cloud vs. Lokal

| Kriterium | Cloud-API | Lokales Modell (Ollama) |
|-----------|----------|------------------------|
| Qualität | Höchste Modelle verfügbar | Gut, aber hinter GPT-4/Claude |
| Latenz | 1–5s (netzwerkabhängig) | 0.1–2s (hardwareabhängig) |
| Datenschutz | Daten werden gesendet | Alles lokal |
| Kosten | Pay-per-Token | Nur Strom + Hardware |
| Verfügbarkeit | Abhängig vom Provider | Immer verfügbar |
| Skalierung | Sofort | Durch Hardware begrenzt |

---

## Prompt Engineering

### Grundprinzipien

| Prinzip | Beschreibung | Beispiel |
|---------|-------------|---------|
| **Spezifisch sein** | Klare, eindeutige Anweisungen | "Fasse in 3 Sätzen zusammen" statt "Fasse zusammen" |
| **Kontext geben** | Relevante Informationen mitliefern | Rolle, Zielgruppe, Format definieren |
| **Beispiele zeigen** | Few-Shot statt Zero-Shot | 2–3 Input-Output-Beispiele |
| **Format vorgeben** | Gewünschte Struktur definieren | "Antworte als JSON mit den Feldern..." |
| **Constraints setzen** | Grenzen definieren | "Maximal 200 Wörter, keine Code-Beispiele" |

### System-Prompt-Struktur

```text
# Rolle
Du bist ein [Rolle] mit Expertise in [Bereich].

# Aufgabe
Deine Aufgabe ist es, [konkrete Aufgabe].

# Regeln
- Regel 1: [Spezifische Anweisung]
- Regel 2: [Format-Vorgabe]
- Regel 3: [Einschränkung]

# Kontext
[Relevante Hintergrundinformationen]

# Ausgabe-Format
[Gewünschtes Format beschreiben]
```

### Temperature-Guide

| Temperature | Verhalten | Verwendung |
|------------|-----------|------------|
| `0.0` | Deterministisch, konsistent | Code, Fakten, Klassifikation |
| `0.3` | Leicht kreativ, meist konsistent | Zusammenfassungen, Analyse |
| `0.7` | Kreativ, variantenreich | Brainstorming, Marketing-Texte |
| `1.0` | Sehr kreativ, unvorhersagbar | Kreatives Schreiben, Ideenfindung |

---

## Kosten-Optimierung

### Kosten-Bewusstsein

| Modell | Input (pro 1M Tokens) | Output (pro 1M Tokens) |
|--------|----------------------|------------------------|
| GPT-4o | ~$2.50 | ~$10.00 |
| GPT-4o-mini | ~$0.15 | ~$0.60 |
| Claude Sonnet | ~$3.00 | ~$15.00 |
| Claude Haiku | ~$0.25 | ~$1.25 |
| Gemini Pro | ~$1.25 | ~$5.00 |
| Gemini Flash | ~$0.075 | ~$0.30 |

> [!TIP] Preise ändern sich häufig
> Diese Preise sind Richtwerte. Aktuelle Preise immer auf den Provider-Websites prüfen. Generell gilt: Preise sinken über Zeit, Modelle werden günstiger.

### Spar-Strategien

| Strategie | Beschreibung | Ersparnis |
|-----------|-------------|-----------|
| **Modell-Routing** | Einfache Tasks → günstiges Modell | 60–80% |
| **Prompt-Caching** | Wiederholte System-Prompts cachen | 50–90% |
| **Batching** | Batch-API für nicht-eilige Anfragen | ~50% |
| **Prompt-Optimierung** | Kürzere Prompts, weniger Kontext | 20–40% |
| **Response-Limits** | `max_tokens` begrenzen | 10–30% |
| **Lokale Modelle** | Ollama für Development/Testing | 100% API |
| **Structured Output** | JSON statt Freitext → weniger Output-Tokens | 15–25% |

### Modell-Routing Beispiel

```python
def choose_model(task_type: str, complexity: str) -> str:
    """Wählt das kosteneffizienteste Modell für die Aufgabe."""
    if complexity == "simple":
        return "gpt-4o-mini"         # Günstig
    elif task_type == "code":
        return "claude-sonnet"        # Bestes Code-Modell
    elif task_type == "reasoning":
        return "claude-opus"          # Bestes Reasoning
    else:
        return "gpt-4o"              # Allzweck
```

---

## Agent-Design

### Zuverlässige Agents bauen

| Prinzip | Umsetzung |
|---------|-----------|
| **Klare Tool-Beschreibungen** | JSON-Schema mit Beispielen und Constraints |
| **Iteration begrenzen** | Max-Iterations-Limit setzen (z.B. 25) |
| **Fehlerbehandlung** | Retry-Logik mit Exponential Backoff |
| **Kontext-Management** | Alte Nachrichten zusammenfassen |
| **Guardrails** | Input-/Output-Validation |
| **Observability** | Alle Tool-Calls loggen |
| **Fallback-Strategie** | Alternative Modelle bei Ausfall |

### Tool-Design

```json
{
  "name": "search_database",
  "description": "Durchsucht die Datenbank nach Einträgen. Verwende spezifische Suchbegriffe für bessere Ergebnisse. Gibt maximal 10 Treffer zurück.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Suchbegriff (mind. 3 Zeichen)"
      },
      "limit": {
        "type": "integer",
        "default": 5,
        "maximum": 10,
        "description": "Maximale Anzahl Ergebnisse"
      }
    },
    "required": ["query"]
  }
}
```

> [!IMPORTANT] Tool-Beschreibungen sind Prompts
> Die `description` eines Tools ist das Wichtigste — der Agent entscheidet basierend darauf, **wann** und **wie** er das Tool nutzt. Investiere Zeit in klare, präzise Beschreibungen mit Beispielen und Constraints.

### Retry-Strategie

```python
import time

def call_with_retry(func, max_retries=3, base_delay=1):
    """Exponential Backoff für API-Calls."""
    for attempt in range(max_retries):
        try:
            return func()
        except RateLimitError:
            delay = base_delay * (2 ** attempt)
            print(f"Rate Limit — warte {delay}s...")
            time.sleep(delay)
        except APIError as e:
            if e.status_code >= 500:
                time.sleep(base_delay)
            else:
                raise
    raise Exception("Max retries exceeded")
```

---

## Sicherheit und Safety

### Prompt Injection verhindern

| Maßnahme | Beschreibung |
|----------|-------------|
| **System-Prompt schützen** | Anweisung "Ignoriere keine vorherigen Regeln" |
| **Input-Sanitization** | Benutzereingaben escapen/filtern |
| **Output-Validation** | Antworten auf unerwünschte Inhalte prüfen |
| **Sandboxing** | Agent-Tools mit minimalen Berechtigungen |
| **Rate Limiting** | Maximale Anfragen pro User begrenzen |

### Daten und Datenschutz

| Empfehlung | Umsetzung |
|-----------|-----------|
| Keine Secrets in Prompts | API-Keys, Passwörter nie im Kontext senden |
| PII minimieren | Personenbezogene Daten anonymisieren |
| Data Residency beachten | EU-Endpoints bei europäischen Daten |
| Lokale Modelle für sensible Daten | Ollama, vLLM für interne Dokumente |
| Audit-Trail | Alle Interaktionen loggen (ohne PII) |

### Content-Safety

```python
# Output-Check vor Auslieferung
def validate_response(response: str) -> bool:
    """Prüft LLM-Antwort auf problematische Inhalte."""
    blocklist = ["password:", "api_key:", "secret:"]
    for term in blocklist:
        if term.lower() in response.lower():
            return False
    return True
```

---

## RAG Best Practices

| Aspekt | Empfehlung |
|--------|-----------|
| **Chunk-Größe** | 500–1000 Tokens pro Chunk, mit 100–200 Token Overlap |
| **Embedding-Modell** | `text-embedding-3-small` (OpenAI) oder `nomic-embed-text` (lokal) |
| **Top-K** | 3–5 relevante Chunks pro Query |
| **Reranking** | Cross-Encoder nach initialer Suche für bessere Relevanz |
| **Metadaten** | Quelle, Datum, Autor in Chunks speichern |
| **Hybrid Search** | Keyword (BM25) + Semantic Search kombinieren |

---

## Anti-Patterns

| Anti-Pattern | Problem | Lösung |
|-------------|---------|--------|
| Riesige System-Prompts | Hohe Kosten, langsam | Prompt auf Kernregeln reduzieren |
| Kein Kontext-Management | Context-Overflow | Alte Nachrichten zusammenfassen |
| Temperature für Code-Tasks | Inkonsistente Ergebnisse | `temperature: 0` für Code |
| Blind API-Keys teilen | Sicherheitsrisiko | Pro-Anwendung eigene Keys |
| Keine Retry-Logik | Fragile Anwendung | Exponential Backoff implementieren |
| `:latest` als Modell | Unvorhersehbare Änderungen | Feste Modell-Versionen pinnen |
| Kein Kosten-Tracking | Budget-Überraschungen | Monitoring mit Limits einrichten |
| Agent ohne Iteration-Limit | Endlosschleifen, Kosten-Explosion | Max-Iterations und Timeout setzen |
| Output ohne Validation | Unsichere Inhalte | Output-Filter implementieren |

---

## Checkliste für Produktionseinsatz

### Vor dem Go-Live

- [ ] **Modell gewählt**: Richtige Balance Qualität/Kosten/Latenz
- [ ] **Prompts getestet**: Mit Edge Cases und adversarialen Inputs
- [ ] **Rate Limits beachtet**: Backoff-Strategie implementiert
- [ ] **Kosten-Limit gesetzt**: Budget-Alerts konfiguriert
- [ ] **Error Handling**: Retry-Logik, Fallback-Modell
- [ ] **Monitoring**: Latenz, Token-Verbrauch, Fehlerrate
- [ ] **Safety-Filter**: Input- und Output-Validation
- [ ] **Datenschutz**: Keine PII in Prompts, lokale Modelle bei Bedarf
- [ ] **Audit-Trail**: Alle Interaktionen nachvollziehbar
- [ ] **Fallback**: Was passiert bei API-Ausfall?

---

## Verwandte Notizen

- [[AI Index]] — Gesamtübersicht
- [[AI Troubleshooting]] — Fehlersuche bei AI-Tools
- [[LLM Grundlagen]] — LLM-Architektur und Konzepte
- [[Prompting Grundlagen]] — Prompt Engineering Techniken
- [[RAG Grundlagen]] — Retrieval Augmented Generation
- [[Ollama]] — Lokale LLM-Inferenz
- [[AI Observability]] — Monitoring und Logging für AI
