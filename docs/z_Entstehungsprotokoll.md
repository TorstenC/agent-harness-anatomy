# "Making of" der Dokumentation zum Agent Harness

## GPT-4.1

Hier ist, wie du das Agent Harness in deinem Workspace analysieren und dokumentieren kannst:

---

### 1. Überblick verschaffen

- Die Projektstruktur findest du im README.md unter `## Directory Structure`.
- Der Haupteinstiegspunkt ist main.tsx.

### 2. Einstiegspunkte finden

- main.tsx ist der zentrale Einstiegspunkt und initialisiert die CLI, das UI und die wichtigsten Abläufe.
- Die Tools (wie das Agent Harness) findest du unter tools, z.B. AgentTool.tsx.

### 3. Top-Down-Analyse

- Die Hauptfunktion ist `main()` in main.tsx, die den Programmablauf steuert.
- Agenten werden über das AgentTool (src/tools/AgentTool/AgentTool.tsx) gestartet.
- Die Tool-Definitionen und das Agenten-Handling sind in Tool.ts und loadAgentsDir.ts zu finden.

### 4. Code-Kommentare und Docstrings nutzen

- Viele Funktionen und Typen sind mit JSDoc-Kommentaren versehen, z.B. in loadAgentsDir.ts.

### 5. Unit Tests anschauen

- Suche nach Tests im `tests/`-Verzeichnis oder in den jeweiligen Komponentenordnern (nicht im Auszug enthalten, ggf. im Gesamtprojekt prüfen).

### 6. Schlüsselkomponenten identifizieren

- **AgentTool**: Startet und verwaltet Subagenten (src/tools/AgentTool/AgentTool.tsx).
- **AgentDefinition**: Typdefinition für Agenten (src/tools/AgentTool/loadAgentsDir.ts).
- **ToolUseContext**: Kontext für Tool-Aufrufe (src/Tool.ts).
- **Agent Harness**: Die Logik zum Starten, Verwalten und Kommunizieren mit Agenten ist im AgentTool und den zugehörigen Utils (src/tools/AgentTool/agentToolUtils.ts).

### 7. Ablaufdiagramme und Übersichten erstellen

- **Ablauf Agent-Start:**
  1. User wählt/erstellt Agent im UI (src/components/agents/).
  2. AgentTool wird aufgerufen (src/tools/AgentTool/AgentTool.tsx).
  3. AgentDefinition wird geladen (src/tools/AgentTool/loadAgentsDir.ts).
  4. Subagent wird als Prozess oder Thread gestartet.
  5. Kommunikation erfolgt über ToolUseContext.

### 8. Dokumentationsstruktur festlegen

**Vorschlag:**

1. **Einleitung & Zweck**
   - Was ist das Agent Harness? (Subagenten-Management und -Orchestrierung)
2. **Architekturübersicht**
   - Hauptmodule und deren Zusammenspiel
3. **Hauptkomponenten**
   - AgentTool, AgentDefinition, ToolUseContext, UI-Komponenten
4. **Typische Abläufe**
   - Starten eines Agenten, Kommunikation, Beenden
5. **Erweiterungsmöglichkeiten**
   - Eigene Agenten definieren (src/components/agents/AgentEditor.tsx)
6. **API-Referenz**
   - Wichtige Typen und Funktionen (siehe oben)

---

**Tipp:** Nutze die "Gehe zu Definition" Funktion in VS Code, um z.B. von einem Agenten-Typ direkt zur Implementierung zu springen.

---

> Erstelle bitte das erste Kapitel "Einleitung & Zweck" in `Einleitung_&_Zweck.md`.

## Claude Opus 4.6 (1)

Ich werde zunächst die wichtigsten Quelldateien analysieren, um eine fundierte Einleitung zu schreiben.

Created 3 todos

- Read [main.tsx](../src/main.tsx#1-1), lines 1 to 210
- Read [Tool.ts](../src/Tool.ts#1-1), lines 1 to 200
- Read [tools.ts](../src/tools.ts#1-1), lines 1 to 200
- Read [commands.ts](../src/commands.ts#1-1), lines 1 to 200
- Read [QueryEngine.ts](../src/QueryEngine.ts#1-1), lines 1 to 200
- Read [context.ts](../src/context.ts)

![Pasted image](./assets/Pasted_image_1.png)  
Nun lese ich noch die Architekturkomponenten, um ein vollständiges Bild zu bekommen:

- Read [setup.ts](../src/setup.ts#1-1), lines 1 to 100
- Read [query.ts](../src/query.ts#1-1), lines 1 to 100
- Read [cost-tracker.ts](../src/cost-tracker.ts#1-1), lines 1 to 100
- Read [tools](../src/tools)
- Read [services](../src/services)
- Read [coordinator](../src/coordinator)
- Read [coordinatorMode.ts](../src/coordinator/coordinatorMode.ts#1-1), lines 1 to 100
- Read [replLauncher.tsx](../src/replLauncher.tsx)
- Read [init.ts](../src/entrypoints/init.ts#1-1), lines 1 to 60
- Read [hooks](../src/hooks)
- Read [plugins](../src/plugins)
- Read [skills](../src/skills)

Starting (3/3) *Einleitung & Zweck schreiben*

Jetzt habe ich genug Kontext. Ich schreibe das vollständige erste Kapitel:

- Using "Replace String in File"

Die Lint-Warnungen betreffen nur Markdown-Formatierungsdetails (Tabellen-Spacing und Heading-Abstände). Ich korrigiere die wichtigsten:

- Read [1_Einleitung_&_Zweck](1_Einleitung_&_Zweck.md)

Ich behebe nun die Markdown-Lint-Warnungen:

- Using "Replace String in File"
- Using "Replace String in File"
- Using "Replace String in File"
- Using "Replace String in File"
- Using "Replace String in File"
- Checked [1_Einleitung_&_Zweck](1_Einleitung_&_Zweck.md), no problems found

Completed (3/3) *Einleitung & Zweck schreiben*

Das Kapitel **"1. Einleitung & Zweck"** ist vollständig in Einleitung_&_Zweck.md erstellt. Es enthält folgende Abschnitte:

| Abschnitt                            | Inhalt                                                                                                                               |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| **1.1 Was ist Claude Code?**         | Definition als Agent Harness, Hintergrund des Leaks, Kennzahlen                                                                      |
| **1.2 Zweck dieser Dokumentation**   | Vier Kernziele der Dokumentation                                                                                                     |
| **1.3 Was ist ein Agent Harness?**   | Konzepterklärung mit ASCII-Architekturdiagramm, Zuordnung zu konkreten Quellcode-Modulen (main.tsx, query.ts, QueryEngine.ts, tools) |
| **1.4 Kernfähigkeiten im Überblick** | Acht Fähigkeitsbereiche: Dateisystem, Shell, Code-Suche, Web, Multi-Agent, IDE-Bridge, Plugins/Skills, Slash-Commands                |
| **1.5 Technologiestack**             | Tabellarische Übersicht aller verwendeten Technologien                                                                               |
| **1.6 Lesehinweise**                 | Geplante Kapitelstruktur der Gesamtdokumentation                                                                                     |

Alle Inhalte basieren auf der tatsächlichen Analyse der Quellcode-Dateien (main.tsx, Tool.ts, tools.ts, commands.ts, QueryEngine.ts, query.ts, context.ts, coordinatorMode.ts, etc.).

- Made changes.

---

> Magst du in diesem Stil bitte ein nächstes Kapitel erstellen?  
> Welches hilft dir beim Schreiben der darauffolgenden Kapitel am besten?

## Claude Opus 4.6 (2)

Ich werde mit **Kapitel 2 – Architekturübersicht** beginnen, da es die Grundlage für alle Folgekapitel bildet. Dazu muss ich die tatsächlichen Modul-Abhängigkeiten und Datenflüsse im Quellcode gründlich analysieren.

- Created 6 todos
- Starting (2/6) *Query-Lifecycle analysieren*
- Starting (3/6) *State-Management analysieren*  

![Pasted image](./assets/Pasted_image_2.png)  
Ich setze die Analyse fort und schreibe Kapitel 2. Lasse mich zuerst die noch fehlenden Architekturkomponenten untersuchen.

- Starting (1/2) *Verbleibende Architektur-Analyse*
- Searched text for `async function run` (`**/src/main.tsx`), 1 result
- Now let me look at the main run function to understand the full startup flow into the REPL:
- Searched text for `launchRepl` (`**/src/main.tsx`), 8 results
- Now I have comprehensive understanding. Let me write Kapitel 2.
- Starting (2/2) *Kapitel 2 schreiben*
- Using "Replace String in File"
- Checked [](../docs/2_Architektur%C3%BCbersicht.md), no problems found
- Completed (2/2) *Kapitel 2 schreiben*

Kapitel 2 ist fertig – **keine Lint-Fehler**. Hier die Zusammenfassung der Abschnitte:

| Abschnitt | Inhalt |
| --- | --- |
| **2.1 Schichtenmodell** | 9-Schichten-Diagramm von CLI-Entrypoint bis Infrastruktur |
| **2.2 Modulübersicht** | Tabellarische Zuordnung aller ~25 Top-Level-Verzeichnisse zu Schichten und Verantwortlichkeiten |
| **2.3 Startup-Ablauf** | Vollständiger Boot-Trace `main()` → `init()` → `setup()` → `launchRepl()` mit paralleler Prefetch-Strategie |
| **2.4 Query-Lifecycle** | Agentic-Loop-Diagramm mit Kontext-Management-Kaskade (6-stufige Komprimierung) und Tool-Ausführungsmodell (parallel vs. seriell + Streaming) |
| **2.5 State-Management** | Zweigeteiltes Modell: globaler Singleton (state.ts) + reaktiver Pub/Sub-Store (store.ts) |
| **2.6 Berechtigungsarchitektur** | 5-stufige Permission-Pipeline (Config → Mode → Classifier → Hooks → User-Prompt) mit Permission-Modes-Tabelle |
| **2.7 Externe Integrationen** | MCP (4 Transports, Auth, Discovery), IDE-Bridge (Architekturdiagramm), Plugin-System, Skill-System |
| **2.8 Feature-Flag-Architektur** | Build-Time Feature-Gates via `bun:bundle` mit ~30 Flags und Dead-Code-Elimination |
| **2.9 Datenfluss-Gesamtbild** | ASCII-Diagramm des End-to-End-Datenflusses User → REPL → QueryEngine → API → Tools → Bridge |

Alle Inhalte basieren auf der direkten Analyse von main.tsx, init.ts, setup.ts, query.ts, QueryEngine.ts, toolOrchestration.ts, StreamingToolExecutor.ts, state.ts, AppStateStore.ts, store.ts, PermissionContext.ts, useCanUseTool.tsx, claude.ts, autoCompact.ts, bridgeMain.ts und client.ts.

- Made changes.

---

> Magst du eine gute README.md erstellen?  
> Zweck des Repos ist, zu dokumentieren und zu verstehen, wie ein Agent Harness aus einem reinen LLM-Textgenerator einen handlungsfähigen Software-Engineering-Agenten macht.

## Claude Opus 4.6 (3)

Guter Plan! Das ist pragmatischer – GitHub WebUI erstellt dir direkt `main`-Branch, Lizenz und eine initiale README.

Ich bereite in der Zwischenzeit die README.md vor, damit du sie direkt ins neue Repo kopieren kannst:

- Read [README.md](./README.md#1-1), lines 1 to 50
- Read [.gitignore](../.gitignore)
- Read [docs](.)

Die README ist bereits gut vorbereitet! Ich ergänze sie um die fehlenden Abschnitte (Technologiestack-Kurzreferenz, Repo-Struktur, Lizenz/Disclaimer, Mitmach-Hinweis):

Read [README.md](./README.md#50-50), lines 50 to 113

Die README ist bereits vollständig und gut strukturiert. Sie hat alles, was ein gutes Repo braucht. Ich nehme nur kleine Verfeinerungen vor:

- Using "Replace String in File"
- Checked [README.md](../README.md), no problems found

Die README ist fertig und lint-sauber. Hier die **Dateien, die ins neue GitHub-Repo gehören**:

```text
.
├── README.md                        ← Projektbeschreibung (fertig)
├── .gitignore                       ← src/** ausgeschlossen + Link zum Original
└── docs/
    ├── 1_Einleitung_&_Zweck.md      ← Kapitel 1 (fertig)
    ├── 2_Architekturübersicht.md    ← Kapitel 2 (fertig)
    └── 3_Hauptkomponenten.md        ← Kapitel 3 (Platzhalter)
```

**Vorgehen:**

1. Erstelle ein neues Repo auf GitHub (z.B. `agent-harness-anatomy` oder wie du es nennen möchtest)
2. Kopiere diese 5 Dateien (README.md, .gitignore, docs/\*) dorthin
3. Den src-Ordner **nicht** mitkopieren — der ist durch .gitignore bewusst ausgeschlossen, der Link zum Originalcode steht in der .gitignore

Sobald du das Repo erstellt und geklont hast, können wir dort mit Kapitel 3 weitermachen.

- Made changes.

---

> Möchtest du (von dir für dich im Follow Up Thread) noch etwas am Ende des "Making of" ergänzen?  
> Du könntest dir z.B. noch eine kurze Reflexion über die Herausforderungen und Erkenntnisse beim Schreiben der Dokumentation ergänzen.

## Claude Opus 4.6 (4) – Reflexion & Arbeitsstand

### Wie die Dokumentation entstanden ist

Die zwei Kapitel entstanden in einer einzigen Session mit folgendem Vorgehen:

1. **Breit lesen, schmal schreiben.** Für Kapitel 1 habe ich 6 Kerndateien gelesen (`main.tsx`, `Tool.ts`, `tools.ts`, `commands.ts`, `QueryEngine.ts`, `context.ts`) plus Verzeichnislistings. Für Kapitel 2 kamen ~15 weitere Dateien hinzu (`query.ts`, `setup.ts`, `init.ts`, `toolOrchestration.ts`, `StreamingToolExecutor.ts`, `bootstrap/state.ts`, `AppStateStore.ts`, `store.ts`, `PermissionContext.ts`, `useCanUseTool.tsx`, `claude.ts`, `autoCompact.ts`, `bridgeMain.ts`, `coordinatorMode.ts`, `mcp/client.ts`). Insgesamt habe ich ca. 5.000 Zeilen Quellcode direkt gelesen – weniger als 1% der Codebasis, aber gezielt die Dateien, die das Skelett definieren.

2. **Top-Down statt Bottom-Up.** Ich bin vom Einstiegspunkt (`main.tsx`) aus den Aufrufketten gefolgt: `main()` → `run()` → `preAction` → `init()` → `setup()` → `launchRepl()`. Dann von der REPL aus: `QueryEngine` → `query()` → `callModel()` → `runTools()` → `toolExecution`. Das ergab die 9 Schichten und den Query-Lifecycle fast von selbst.

3. **Diagramme als Denkwerkzeug.** Die ASCII-Diagramme sind nicht nur Illustration – sie waren das primäre Mittel, um die Architektur zu verstehen. Erst wenn ein Flussdiagramm widerspruchsfrei zeichenbar war, war das Verständnis ausreichend.

### Was gut funktioniert hat

- **Die Code-Kommentare sind ausgezeichnet.** Fast jede wichtige Designentscheidung ist im Code dokumentiert – warum `startKeychainPrefetch()` vor den Imports läuft, warum `bootstrap/state.ts` ein Singleton ist, warum `partitionToolCalls` zwischen read-only und mutable unterscheidet. Das hat die Analyse enorm beschleunigt.
- **Konsistente Muster.** Das Feature-Flag-Pattern (`feature('X') ? require(...) : null`) ist überall identisch, was das Verständnis nach dem ersten Vorkommen trivial macht. Ebenso die Tool-Struktur (ein Verzeichnis pro Tool mit eigenem Prompt, eigenen Konstanten).
- **Der zweigeteilte State.** Die Trennung in `bootstrap/state.ts` (globaler Singleton, prozessweit) und `state/store.ts` (reaktiver Pub/Sub, UI-getrieben) ist ein ungewöhnliches aber cleveres Muster – es vermeidet React-Abhängigkeiten in der tiefen Infrastruktur.

### Was herausfordernd war

- **Zirkuläre Abhängigkeiten.** Der Code hat viele `/* eslint-disable @typescript-eslint/no-require-imports */`-Blöcke und Lazy-Requires, die Import-Zyklen brechen. Das macht das Nachverfolgen von Aufrufketten schwieriger, weil die echte Abhängigkeit erst zur Laufzeit sichtbar wird.
- **Die Größe von `main.tsx`.** Mit 4.684 Zeilen ist `main.tsx` das größte File und enthält die gesamte CLI-Definition, Commander.js-Setup, und mehrere Startup-Pfade (interaktiv, print-mode, SSH, Direct-Connect, Assistant-Mode). Das macht es schwer, den "happy path" vom Spezialfall zu trennen.
- **Feature-Gate-Explosion.** ~30 Feature Flags bedeuten theoretisch 2³⁰ Build-Kombinationen. Beim Lesen muss man ständig entscheiden, welche Gates "an" sind, um den relevanten Codepfad zu finden.

### Arbeitsstand für den nächsten Thread

| Was | Status | Nächster Schritt |
| --- | --- | --- |
| Kapitel 1 – Einleitung & Zweck | ✅ fertig | — |
| Kapitel 2 – Architekturübersicht | ✅ fertig | — |
| Kapitel 3 – Hauptkomponenten | 🔲 Platzhalter | Details zu QueryEngine, Tool-Interface, Command-System, Services schreiben. Dafür `Tool.ts` (793 Zeilen), `QueryEngine.ts` (1.296 Zeilen), `query.ts` (1.730 Zeilen), `toolExecution.ts` (1.746 Zeilen) tiefer analysieren. |
| Kapitel 4 – Typische Abläufe | 🔲 geplant | Sequenzdiagramme für konkreten Query-Turn, BashTool-Aufruf mit Permission, Agent-Spawning |
| Kapitel 5 – Erweiterungen | 🔲 geplant | Plugin-/Skill-/MCP-System im Detail |
| Kapitel 6 – API-Referenz | 🔲 geplant | Zentrale Typen (`Tool`, `ToolUseContext`, `AppState`, `Message`) |
| README.md | ✅ fertig | Dokumentationstabelle aktualisieren, wenn Kapitel fertig werden |
| .gitignore | ✅ fertig | — |
| Making-of | ✅ dieses Dokument | Fortlaufend ergänzen |

### Hinweise an den nächsten Claude-Thread

- **Quellcode liegt in `src/`.** Er ist via `.gitignore` aus dem Repo ausgeschlossen, aber lokal vorhanden. Der Link zum Original steht in `.gitignore`: [ultraworkers/claw-code @ 5a774a2](https://github.com/ultraworkers/claw-code/tree/5a774a2b62d7949c1d94e0b726281554d7893cfd).
- **Stil beibehalten:** Deutsche Sprache, technisch präzise, ASCII-Diagramme, Tabellen mit `| --- | --- |`-Syntax (Leerzeilen um `---` für Markdown-Lint), Leerzeile nach jeder `###`-Überschrift.
- **Kapitel 3 ist das wichtigste nächste Kapitel.** Es sollte die Schnittstellen der Kernmodule so beschreiben, dass Kapitel 4 (Abläufe) darauf aufbauen kann.
- **Die 6-stufige Komprimierungs-Pipeline** (Kapitel 2.4) verdient in Kapitel 3 oder 4 eine Detailanalyse mit Token-Schwellwerten und Entscheidungslogik – die Konstanten stehen in `autoCompact.ts`.

---

> Bitte mache vor dem Commit & Push noch ein kurzes Review.

## Claude Opus 4.6 (5) – Review

### ⚠️ Kleinigkeiten zum Beheben

1. **z_Entstehungsprotokoll.md fehlt in der README-Projektstruktur** — Das Making-of wird weder in der Dokumentationstabelle noch im Strukturbaum der README.md erwähnt. Da es ein Meta-Dokument ist, sollte es zumindest im Strukturbaum auftauchen:

   ````markdown
   // ...existing code...
   ├── docs/
   │   ├── 1_Einleitung_&_Zweck.md        ← Was und warum
   │   ├── 2_Architekturübersicht.md       ← Wie es zusammenhängt
   │   ├── 3_Hauptkomponenten.md           ← Details (in Arbeit)
   │   └── z_Entstehungsprotokoll.md       ← Entstehungsprotokoll "MAKING OF"
   └── src/                               ← NICHT im Repo (siehe .gitignore)
   // ...existing code...
   ````

2. **Obsidian-Bildverweise in z_Entstehungsprotokoll.md** — Zwei Stellen enthalten `![[Pasted image ...]]`, die auf GitHub nicht rendern.

---

> Beginne mit dem Schreiben von Kapitel `# 3. Hauptkomponenten`.

## Claude Opus 4.6 (6) – Kapitel 3

Ich beginne mit der Analyse der Quelldateien für Kapitel 3. Zuerst lese ich die Kerndateien, die die Hauptkomponenten definieren. 

- Created 4 todos
- Starting (2/4) *Quelldateien analysieren (Runde 2)*
- Now let me look at specific tool implementations and the coordinator module: 
- Now let me read the existing docs to maintain consistency: 
- Now let me read the remaining critical sections and check agent loading: 
- I now have sufficient context to write a comprehensive Kapitel 3. Let me also quickly check the services for compact/context management: 
- Starting (3/4) *Kapitel 3 schreiben*
- Using "Replace String in File"
- Jetzt prüfe ich auf Fehler: 

Checked [3_Hauptkomponenten.md](./3_Hauptkomponenten.md), no problems found

Keine Fehler. Jetzt die README aktualisieren:

- Starting (4/4) *README aktualisieren*
- Using "Replace String in File"
- Using "Replace String in File"
- Checked [README.md](../README.md), no problems found
- Completed (4/4) *README aktualisieren*

Kapitel 3 ist fertig — **keine Lint-Fehler** in beiden geänderten Dateien. Hier die Zusammenfassung:

## Was wurde erstellt / geändert

### 3_Hauptkomponenten.md — Neues Kapitel

| Abschnitt | Inhalt |
| --- | --- |
| **3.1 QueryEngine** | Konversations-Manager: Config, Turn-Ablauf in `submitMessage()`, interner State (mutableMessages, readFileState, usage, denials) |
| **3.2 Query-Schleife** | `while(true)`-Loop mit 5 Phasen: Kontext-Komprimierung (6-stufige Pipeline), API-Streaming, Fehler-Recovery, Tool-Ausführung, Post-Processing. 10 Return-Reasons. State-Objekt. |
| **3.3 Tool-System** | Drei Schichten: **Interface** (`Tool`, `buildTool()`, `ToolUseContext`), **Registry** (`getAllBaseTools → getTools → assembleToolPool`), **Ausführung** (`partitionToolCalls`, `StreamingToolExecutor`, `runToolUse`) |
| **3.4 Command-System** | 3 Command-Typen (prompt/local/local-jsx), 6 Quellen, Availability-Filter, Skill-Invocation |
| **3.5 Konkrete Tools** | BashTool (semantische Befehlsanalyse, Sandbox, Security), AgentTool (Sub-Agenten, Isolation, Background), Coordinator-Modus |
| **3.6 Tool-Referenz** | Tabelle aller ~40 Built-in Tools mit Kategorie, Parallel-Fähigkeit, Feature Gate |
| **3.7 Skills & Plugins** | Lade-Hierarchie, Frontmatter-Felder, Plugin-System |
| **3.8 Zusammenfassung** | End-to-End-Datenfluss-Diagramm durch alle Komponenten |

### README.md — Zwei Änderungen

1. Kapitel 3 Status: `🔲` → `✅` mit vollständiger Inhaltsbeschreibung
2. `z_Entstehungsprotokoll.md` in die Projektstruktur aufgenommen

### Quellcode-Basis

Analysierte Dateien für Kapitel 3: QueryEngine.ts, query.ts, `Tool.ts`, tools.ts, commands.ts, command.ts, toolOrchestration.ts, StreamingToolExecutor.ts, toolExecution.ts, AgentTool.tsx, runAgent.ts, loadAgentsDir.ts, constants.ts, BashTool.tsx, coordinatorMode.ts, loadSkillsDir.ts.

- Made changes.

---

> ...
