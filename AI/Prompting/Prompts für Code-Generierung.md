---
tags: [ai, prompting, code-generierung]
---

# Prompts für Code-Generierung

## Best Practices für Code-Prompts

Code-Generierung gehört zu den stärksten Anwendungsfällen von KI-Modellen – aber auch zu den fehleranfälligsten. Ohne ausreichend Kontext produziert das Modell generischen Code, der selten direkt in bestehende Architekturen passt.

### Die fünf wichtigsten Grundregeln

1. **Sprache, Framework und Version immer angeben** – nie davon ausgehen, dass das Modell die richtige wählt
2. **Bestehenden Code mitliefern** – das Modell soll ergänzen, nicht neu erfinden
3. **Constraints explizit nennen** – welche Libraries sind erlaubt, welche verboten?
4. **Ausgabeformat definieren** – nur Code? Code + Erklärung? Code + Tests?
5. **Erwartetes Verhalten beschreiben** – nicht nur "was", sondern auch Grenzfälle und Fehlerverhalten

> [!IMPORTANT]
> Generierter Code ist immer ein Ausgangspunkt, kein fertiges Produkt. Code-Review und Tests sind Pflicht, unabhängig davon, wie überzeugend der generierte Code aussieht.

---

## Sprache, Framework und Version immer angeben

Das Modell kennt viele Sprachen, Frameworks und Versionen – und wählt ohne Anweisung oft eine veraltete oder unpassende Variante.

### Schlechter Prompt (zu vage)

```
Schreibe eine Funktion, die eine Liste von Zahlen sortiert.
```

Das Modell wählt eine zufällige Sprache, möglicherweise einen veralteten Ansatz, und du bekommst keinen Mehrwert.

### Guter Prompt (präzise)

```
Python 3.12. Schreibe eine Funktion `sort_numbers`, die eine Liste von Integers
oder Floats sortiert. Anforderungen:
- Type Hints verwenden
- Ascending/Descending per Parameter steuerbar (default: ascending)
- Gibt eine neue Liste zurück (in-place Mutation vermeiden)
- PEP 8 konform
```

### Vollständiges Versions-Kontext-Beispiel

```
Stack: TypeScript 5.3, Node.js 20, Express 4.18, Prisma 5.x, PostgreSQL 16

Schreibe eine Express-Route POST /users, die:
1. Den Request-Body validiert (name: string, email: string, age: number)
2. Prüft, ob die E-Mail bereits existiert (via Prisma)
3. Den neuen User anlegt
4. Bei Erfolg 201 + das erstellte User-Objekt zurückgibt
5. Bei Validierungsfehlern 400 + strukturierte Fehlermeldung zurückgibt
6. Bei bereits existierender E-Mail 409 zurückgibt

Verwende async/await, keinen Callback-Stil. Keine zusätzlichen Validation-Libraries.
```

---

## Kontext bereitstellen

### Bestehenden Code einfügen

Wenn Code in ein bestehendes System integriert werden soll, muss das Modell die Umgebung kennen.

````
Hier ist meine bestehende User-Service-Klasse in Python:

```python
class UserService:
    def __init__(self, db: Database):
        self.db = db

    def get_user(self, user_id: int) -> Optional[User]:
        return self.db.query(User).filter(User.id == user_id).first()

    def list_users(self, limit: int = 100) -> list[User]:
        return self.db.query(User).limit(limit).all()
```

Füge eine Methode `update_user` hinzu, die:
- user_id (int) und ein dict mit den zu ändernden Feldern entgegennimmt
- Nur erlaubte Felder ändert: name, email, bio
- Den aktualisierten User zurückgibt
- UserNotFoundError wirft wenn der User nicht existiert
- Denselben Stil wie die bestehenden Methoden verwendet
````

### Architektur beschreiben

```
Ich baue ein Microservices-System mit folgender Architektur:
- API Gateway (Nginx)
- Auth-Service (Node.js + JWT)
- Product-Service (Python + FastAPI)
- Kommunikation über RabbitMQ

Schreibe einen FastAPI-Endpoint GET /products/{product_id}, der:
1. Das JWT-Token aus dem Authorization-Header validiert (der Auth-Service stellt
   eine Validierungs-Funktion bereit: validate_token(token: str) -> dict)
2. Das Produkt aus der PostgreSQL-Datenbank lädt (SQLAlchemy 2.0 ORM)
3. Bei fehlendem/ungültigem Token 401 zurückgibt
4. Bei nicht gefundenem Produkt 404 zurückgibt

Verwende FastAPI-Dependency Injection für die Token-Validierung.
```

> [!TIP]
> Je mehr relevanter Kontext du gibst, desto besser passt sich der generierte Code in deine Architektur ein. Füge relevante Interfaces, Type-Definitionen und bestehende Patterns als Referenz ein.

---

## Testfälle mit anfordern

Ein häufiger Fehler: Man lässt sich Code generieren und vergisst, gleichzeitig Tests anzufordern. Das Nachfordern von Tests ist ineffizienter als das gleichzeitige Anfordern.

### Prompt mit Tests

```
Python 3.12, pytest. Implementiere eine Funktion `parse_csv_row`, die:
- Eine CSV-Zeile als String entgegennimmt
- Ein Dict zurückgibt mit den Schlüsseln: id (int), name (str), email (str), score (float)
- ValueError wirft bei fehlenden Pflichtfeldern
- None-Werte für optionale fehlende Felder setzt (score ist optional)

Gib mir:
1. Die Implementierung (python Code-Block)
2. pytest-Tests in einem separaten Code-Block mit mindestens:
   - Happy Path (vollständige Zeile)
   - Fehlende Pflichtfelder
   - Optionales fehlendes Feld
   - Ungültige Typen (z.B. Buchstaben statt Zahl bei id)
```

### Template für Tests beim Generieren mitanfordern

```
[Deine Code-Anforderungen hier]

Ausgabestruktur:
1. Implementierung (Code-Block mit Sprachangabe)
2. Unit Tests mit pytest/jest/[passendes Framework] (separater Code-Block)
   - Mindestens: Happy Path, Fehlerfall, Grenzfall
3. Kurze Erklärung der Testabdeckung (2-3 Sätze)
```

---

## Refactoring-Prompts

### Allgemeines Refactoring

````
Python 3.12. Refaktoriere folgenden Code. Ziele:
- Lesbarkeit verbessern
- Redundanz entfernen
- PEP 8 und moderne Python-Idiome anwenden
- Keine Funktionalität ändern

Bestehender Code:
```python
def process_data(data):
    result = []
    for i in range(len(data)):
        if data[i] != None:
            if data[i] > 0:
                result.append(data[i] * 2)
    return result
```

Gib mir: refaktorierter Code + kurze Auflistung der Änderungen (Bullet Points).
````

### Refactoring auf ein bestimmtes Pattern

````
TypeScript 5.3. Refaktoriere folgenden Code von einem imperativen zu einem
funktionalen Stil. Verwende Array-Methoden (map, filter, reduce), keine for-Loops.
Typ-Inferenz beibehalten.

```typescript
function processOrders(orders: Order[]): ProcessedOrder[] {
  const result: ProcessedOrder[] = [];
  for (let i = 0; i < orders.length; i++) {
    if (orders[i].status === 'completed') {
      const processed: ProcessedOrder = {
        id: orders[i].id,
        total: orders[i].items.reduce((sum, item) => sum + item.price, 0),
        customerName: orders[i].customer.name
      };
      result.push(processed);
    }
  }
  return result;
}
```
````

### Performance-Refactoring

```
Python 3.12. Analysiere folgenden Code auf Performance-Probleme und schlage
Optimierungen vor. Der Code wird auf einem Datensatz mit ~1 Million Einträgen
ausgeführt.

[Code hier einfügen]

Ausgabe:
1. Liste der identifizierten Performance-Probleme (nummeriert, mit Begründung)
2. Optimierter Code
3. Erwartete Verbesserung (qualitativ: "O(n²) → O(n)")
```

---

## Bug-Fixing Prompts

### Fehler mit Stack Trace

````
Python 3.11, FastAPI 0.104. Folgender Code wirft den unten stehenden Fehler.
Analysiere den Fehler und gib mir den korrigierten Code.

Code:
```python
@app.get("/users/{user_id}")
async def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    return user
```

Fehler:
```
pydantic.ValidationError: 1 validation error for Response
value
  value is not a valid dict (type=type_error.dict)
```

Erkläre kurz die Ursache, dann gib den korrigierten Code.
````

### Fehler ohne klare Ursache

````
JavaScript, Node.js 20. Folgender Code verhält sich unerwartet: Die Funktion
gibt manchmal `undefined` zurück, obwohl Daten vorhanden sein sollten.

```javascript
async function getUserData(userId) {
  const user = await db.users.findOne({ id: userId });
  if (user) {
    const orders = await db.orders.find({ userId: user.id });
    user.orders = orders;
  }
  return user;
}
```

Analysiere mögliche Ursachen (liste alle plausiblen Gründe auf) und schlage
Fixes vor. Füge auch Debugging-Code hinzu, um die Ursache einzugrenzen.
````

### Systematische Fehleranalyse

```
[Sprache/Framework]. Analysiere den folgenden Code auf:
1. Logikfehler
2. Potenzielle Laufzeitfehler
3. Sicherheitsprobleme
4. Stellen, die bei unerwarteten Eingaben brechen könnten

Für jeden gefundenen Fehler:
- Beschreibe das Problem (1-2 Sätze)
- Zeige die problematische Zeile
- Gib die korrigierte Version

[Code hier einfügen]
```

> [!WARNING]
> Bei Security-relevanten Bug-Fixes (SQL Injection, XSS, Auth-Bypässe) solltest du die Ausgabe des Modells von jemandem mit Security-Kenntnissen reviewen lassen. Modelle können Security-Issues übersehen oder unvollständige Fixes vorschlagen.

---

## Code-Review Prompts

### Allgemeines Code-Review

```
Python 3.12. Führe ein Code-Review des folgenden Codes durch.

Bewertungskriterien (priorisiert nach Wichtigkeit):
1. Korrektheit: Gibt es logische Fehler?
2. Sicherheit: Gibt es Sicherheitslücken?
3. Performance: Gibt es offensichtliche Performance-Probleme?
4. Lesbarkeit: Ist der Code verständlich und gut strukturiert?
5. Tests: Sind die Edge Cases abgedeckt?

Ausgabeformat:
- Für jeden Kritikpunkt: Schweregrad [KRITISCH/MAJOR/MINOR] + Beschreibung + Verbesserungsvorschlag
- Am Ende: Gesamtbewertung (1-5 Sterne) mit kurzer Begründung

[Code hier einfügen]
```

### Review nach spezifischen Patterns

```
TypeScript. Reviewe den folgenden Code speziell auf:
- Typ-Sicherheit: Gibt es `any`, implizite `any` oder unsichere Type Assertions?
- Error-Handling: Werden alle möglichen Fehler korrekt behandelt?
- Async/Await: Gibt es unbehandelte Promises oder Race Conditions?

Für jeden Fund: Zeile + Problem + Fix.

[Code hier einfügen]
```

### PR-Review Prompt

```
Ich reviewe einen Pull Request. Folgender Code ist der Diff.
Führe ein Code-Review durch und schreibe Review-Kommentare im GitHub-Stil.

Für jeden Kommentar:
- Zeile/Bereich: [Zeilennummer oder Bereich]
- Typ: [suggestion / question / issue / nit]
- Kommentar: [dein Kommentar]

Priorisiere: Correctness > Security > Performance > Style

[Diff hier einfügen]
```

---

## Dokumentations-Prompts

### Docstrings generieren

```
Python 3.12, Google-Style Docstrings. Generiere Docstrings für alle Funktionen
in folgendem Code. Der Docstring soll enthalten:
- Kurzbeschreibung (1 Zeile)
- Args: alle Parameter mit Typ und Beschreibung
- Returns: Rückgabewert mit Typ und Beschreibung
- Raises: alle möglichen Exceptions
- Example: ein kurzes Verwendungsbeispiel

Gib den vollständigen Code mit eingefügten Docstrings zurück.

[Code hier einfügen]
```

### JSDoc generieren

```
TypeScript/JavaScript. Generiere JSDoc-Kommentare für alle exportierten
Funktionen und Klassen. Format:
- @description
- @param {Typ} name - Beschreibung
- @returns {Typ} Beschreibung
- @throws {ErrorType} Wann wird dieser Fehler geworfen?
- @example

[Code hier einfügen]
```

### README generieren

```
Erstelle eine README.md für folgendes Projekt. Das README soll enthalten:

1. **Projektname und Kurzbeschreibung** (1-2 Sätze)
2. **Features** (Bullet Points, max. 6)
3. **Requirements** (Sprache/Framework-Version, Dependencies)
4. **Installation** (nummerierte Schritte mit Code-Blöcken)
5. **Usage** (2-3 Anwendungsbeispiele mit Code)
6. **Configuration** (alle relevanten Umgebungsvariablen als Tabelle)
7. **API Reference** (falls API-Endpunkte vorhanden)
8. **License**

Projektbeschreibung:
[Kurze Beschreibung des Projekts]

Hauptdateien/Code:
[Wichtigste Code-Snippets einfügen]
```

> [!TIP]
> Für Docstrings: Gib dem Modell einen bestehenden gut-dokumentierten Docstring als Beispiel mit. Das Modell übernimmt dann genau diesen Stil, was in einem Codebase wichtig ist.

---

## Beispiel-Prompts nach Sprache

### Python

```python
# Prompt:
"""
Python 3.12, keine externen Libraries außer der Standardbibliothek.

Implementiere eine Klasse `RateLimiter` mit dem Token-Bucket-Algorithmus:
- __init__(self, rate: int, per: float): rate = Tokens pro Zeiteinheit, per = Zeiteinheit in Sekunden
- acquire(self, tokens: int = 1) -> bool: Gibt True zurück wenn Tokens verfügbar, False sonst
- Thread-safe (threading.Lock)
- Methode wait_for_token(self, tokens: int = 1): Blockiert bis Tokens verfügbar sind

Füge Type Hints und einen Google-Style Docstring für die Klasse und alle Methoden hinzu.
Schreibe danach pytest-Tests: mindestens 4 Tests (Erfolg, Limit erreicht, Wait, Thread-Safety).
"""
```

### PowerShell

```powershell
# Prompt:
"""
PowerShell 7.x. Schreibe ein Script `Get-ServiceHealth.ps1`, das:
1. Alle Windows-Services mit Status "Stopped" listet, die beim System-Start starten sollten
2. Versucht, gestoppte Services neu zu starten
3. Einen Log-Eintrag schreibt (C:\Logs\ServiceHealth.log mit Timestamp)
4. Bei Fehler eine E-Mail sendet (Send-MailMessage, SMTP-Server als Parameter)
5. Am Ende einen HTML-Report generiert

Parameter:
- SmtpServer (string, mandatory)
- AlertEmail (string, mandatory)
- LogPath (string, optional, default: C:\Logs\ServiceHealth.log)

Fehlerbehandlung: try/catch für jeden Service. Skript soll weiterlaufen wenn ein
einzelner Service-Restart fehlschlägt.
Verwende Verb-Noun Naming Convention. Kommentiere kritische Abschnitte.
"""
```

### Bash

```bash
# Prompt:
"""
Bash 5.x, Linux (Ubuntu 22.04). Schreibe ein Deployment-Script `deploy.sh`:

1. Prüft, ob alle benötigten Tools installiert sind: docker, docker-compose, curl
2. Lädt die neueste Version eines Docker-Images (als Parameter: IMAGE_NAME:TAG)
3. Führt einen Health-Check durch: wartet max. 60 Sekunden bis der Container
   auf /health antwortet (curl mit Retry-Logik, 5 Sekunden zwischen Versuchen)
4. Bei Erfolg: altes Image entfernen
5. Bei Fehler: Rollback zum vorherigen Image + Exit 1

Anforderungen:
- set -euo pipefail am Anfang
- Alle Fehler nach stderr (>&2)
- Farbige Ausgabe: grün für Erfolg, rot für Fehler, gelb für Warnungen
- Verwendung: ./deploy.sh [IMAGE_NAME] [TAG]
- --help Flag implementieren

Kommentiere jeden Abschnitt.
"""
```

### TypeScript

```typescript
// Prompt:
"""
TypeScript 5.3, Node.js 20. Kein Framework. Nur eingebaute Node.js-Module.

Implementiere einen einfachen HTTP-Server `server.ts` mit folgenden Endpoints:
- GET /health → { status: "ok", uptime: number }
- POST /transform → nimmt JSON Body { text: string, operation: "upper"|"lower"|"reverse" }
                   → gibt { result: string, originalLength: number } zurück

Anforderungen:
- Strikte TypeScript-Typen (kein any)
- Request-Body Validierung mit eigener Funktion (keine externe Library)
- Fehlerbehandlung: 400 für invalide Requests, 405 für falsche HTTP-Methoden, 404 für unbekannte Routen
- Alle Typen und Interfaces in einer separaten types.ts Datei
- Server-Port über Umgebungsvariable PORT (default: 3000)

Gib mir:
1. types.ts
2. server.ts
3. Beispiel-curl-Requests für alle Endpoints
"""
```

---

## Häufige Fehler bei Code-Prompts

### Fehler 1: Keine Versionsinformationen

**Schlecht:**
```
Schreibe einen HTTP-Client in Python.
```

**Gut:**
```
Python 3.12, httpx 0.27 (kein requests). Schreibe einen async HTTP-Client...
```

---

### Fehler 2: Fehlende Constraints

**Schlecht:**
```
Implementiere User-Authentifizierung.
```

**Gut:**
```
Python 3.12, FastAPI 0.104, keine externe Auth-Library außer python-jose für JWT.
Kein OAuth, kein SSO – nur lokale Username/Password-Authentifizierung mit JWT-Tokens.
Passwörter mit bcrypt hashen. Token-Expiry: 24 Stunden.
```

---

### Fehler 3: Zu groß auf einmal

**Schlecht:**
```
Schreibe eine komplette REST-API für ein E-Commerce-System mit allen Endpoints.
```

**Gut:**
Aufteilen in kleine, konkrete Aufgaben:
```
Schritt 1: Implementiere den Product-Service (CRUD für Products)
Schritt 2: Implementiere den Order-Service (Bestellungen anlegen)
Schritt 3: Verbinde beide Services über ein Repository Pattern
```

---

### Fehler 4: Kein Ausgabeformat definiert

**Schlecht:**
```
Erkläre mir, wie ich einen Cache in Redis implementiere.
```

**Gut:**
```
Python 3.12, redis-py 5.x. Zeige mir eine Cache-Klasse mit get/set/delete/exists.
Ausgabe: Code-Block + kurze Kommentare im Code (keine externe Erklärung).
Danach: 5-Zeilen-Zusammenfassung der wichtigsten Design-Entscheidungen.
```

---

### Fehler 5: Halluzinierte APIs nicht abfangen

> [!WARNING]
> Modelle erfinden manchmal Funktionen und Parameter, die nicht existieren. Das ist besonders gefährlich bei neueren Library-Versionen oder nischigen Frameworks. Prüfe immer: Existiert die Funktion wirklich? Sind die Parameter korrekt? Stimmt die Signatur?

**Schutzstrategie:**
```
Verwende ausschließlich Funktionen aus der offiziellen Dokumentation von [Library X].
Wenn du dir bei einer Funktion nicht sicher bist, ob sie in Version X existiert,
weise mich explizit darauf hin und ich überprüfe es.
```

---

### Fehler 6: Keine Fehlerbehandlung anfordern

**Schlecht:**
```
Schreibe eine Funktion, die eine Datei liest.
```

**Gut:**
```
Python 3.12. Schreibe eine Funktion `read_config_file(path: Path) -> dict`, die:
- Eine JSON-Konfigurationsdatei liest und als dict zurückgibt
- FileNotFoundError wirft wenn die Datei nicht existiert (mit Pfad in der Fehlermeldung)
- ValueError wirft wenn der JSON-Inhalt ungültig ist
- PermissionError weitergibt wenn keine Leseberechtigung vorhanden ist
- Alle anderen Exceptions als RuntimeError mit Original-Exception als cause wrapped
```

---

## Checkliste: Code-Prompt Quality

Bevor du einen Code-Prompt absendest:

- [ ] Sprache und Version angegeben?
- [ ] Framework und relevante Libraries mit Versionen angegeben?
- [ ] Verbotene Libraries/Patterns explizit ausgeschlossen?
- [ ] Bestehender relevanter Code als Kontext eingefügt?
- [ ] Ausgabeformat definiert (nur Code? Code + Tests? Code + Erklärung)?
- [ ] Fehlerverhalten spezifiziert?
- [ ] Grenzfälle und unerwartete Eingaben erwähnt?
- [ ] Code-Style und Konventionen angegeben (PEP 8, Type Hints, Docstrings)?
- [ ] Ist die Aufgabe klein genug für eine gute Antwort? (ggf. aufteilen)

---

## Verwandte Notizen

- [[Prompt Engineering Techniken]] — Allgemeine Techniken wie Few-Shot und Chain-of-Thought, die in Code-Prompts einfließen
- [[System Prompts schreiben]] — System Prompts für Code-Assistenten mit Stil- und Qualitätsvorgaben erstellen
- [[Custom Instructions für AI Coding Agents]] — Dauerhafte Coding-Konventionen für KI-Agenten hinterlegen
- [[AI Coding Agents Vergleich]] — Übersicht und Vergleich von KI-Coding-Tools wie Copilot, Cursor und Aider
