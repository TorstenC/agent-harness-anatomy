# 3. Hauptkomponenten

Dieses Kapitel beschreibt die zentralen Bausteine von Claude Code im Detail.
Kapitel 2 zeigte *wie* sie zusammenhängen — hier geht es um *was* jede Komponente intern tut.

## 3.1 QueryEngine — Der Konversations-Manager

Die `QueryEngine` (Datei: `src/QueryEngine.ts`, ~1.296 Zeilen) besitzt den gesamten Lebenszyklus einer Konversation.
Pro Konversation existiert genau eine Instanz; jeder `submitMessage()`-Aufruf startet einen neuen *Turn* innerhalb derselben Session.

### Verantwortlichkeiten

```text
┌──────────────────────────────────────────────────────────┐
│                      QueryEngine                         │
│                                                          │
│  ┌────────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  submitMessage │─►│  Vorverarbei-│─►│   query()    │   │
│  │  (Eingang)     │  │  tung       │  │   (Schleife) │   │
│  └────────────────┘  └─────────────┘  └──────┬───────┘   │
│                                              │           │
│  ┌────────────────┐  ┌─────────────┐         │           │
│  │  Ergebnis      │◄─│  Persistenz │◄────────┘           │
│  │  (SDKMessage)  │  │  & Tracking │                     │
│  └────────────────┘  └─────────────┘                     │
└──────────────────────────────────────────────────────────┘
```

### Konfiguration (`QueryEngineConfig`)

Die `QueryEngine` wird über ein umfangreiches Config-Objekt initialisiert:

| Feld | Typ | Beschreibung |
| --- | --- | --- |
| `cwd` | `string` | Arbeitsverzeichnis für Dateisystem-Operationen |
| `tools` | `Tools` | Verfügbare Werkzeuge (Built-in + MCP) |
| `commands` | `Command[]` | Registrierte Slash-Commands |
| `mcpClients` | `MCPServerConnection[]` | Verbundene MCP-Server |
| `agents` | `AgentDefinition[]` | Verfügbare Agenten-Definitionen |
| `canUseTool` | `CanUseToolFn` | Permission-Callback für Tool-Aufrufe |
| `customSystemPrompt` | `string?` | Ersetzt den Default-System-Prompt |
| `appendSystemPrompt` | `string?` | Wird nach dem Haupt-System-Prompt angehängt |
| `maxTurns` | `number?` | Maximale Agentic-Turns pro Nachricht |
| `maxBudgetUsd` | `number?` | Kosten-Limit in USD |
| `taskBudget` | `{total}?` | API-seitiges Token-Budget für den gesamten Turn |
| `jsonSchema` | `Record?` | Erzwingt Structured Output über SyntheticOutputTool |
| `snipReplay` | `fn?` | Callback für Snip-Komprimierung (feature-gated) |

### Turn-Ablauf in `submitMessage()`

Jeder Aufruf von `submitMessage()` durchläuft folgende Phasen:

```text
submitMessage(prompt)
│
├─ 1. Input-Verarbeitung
│     processUserInput() → Slash-Commands erkennen und verarbeiten
│     Ergebnis: messages[], shouldQuery, allowedTools, model
│
├─ 2. Persistenz (vor API-Aufruf)
│     recordTranscript() → Transkript schreiben, damit --resume funktioniert
│
├─ 3. System-Init
│     fetchSystemPromptParts() → System-Prompt + User-Context + System-Context
│     yield buildSystemInitMessage() → SDK erhält Tool-Liste, Modell, Mode
│
├─ 4. Query-Schleife delegieren
│     yield* query({messages, systemPrompt, ...})
│     → siehe Abschnitt 3.2
│
├─ 5. Ergebnis-Nachrichten yieldden
│     Jede assistant/user/compact_boundary-Nachricht → SDK-Format konvertieren
│     Permission-Denials tracken
│     Usage akkumulieren
│
└─ 6. Ergebnis
      yield {type: 'result', subtype: 'success', usage, cost, ...}
```

### Interner State

Die `QueryEngine` verwaltet über ihre Lebensdauer hinweg:

- **`mutableMessages`** — das vollständige Konversations-Array (wächst mit jedem Turn)
- **`readFileState`** — LRU-Cache gelesener Datei-Inhalte (vermeidet Re-Reads)
- **`totalUsage`** — kumulierte API-Token-Nutzung
- **`permissionDenials`** — Liste aller abgelehnten Tool-Aufrufe (für SDK-Reporting)
- **`discoveredSkillNames`** — pro Turn zurückgesetzt (für Telemetrie)
- **`loadedNestedMemoryPaths`** — Dedup für CLAUDE.md-Injektionen (über Turns hinweg)

---

## 3.2 Die Query-Schleife — Das Herzstück

Die Funktion `query()` (Datei: `src/query.ts`, ~1.730 Zeilen) ist die **Agentic Loop**: eine `while(true)`-Schleife, die erst terminiert, wenn das Modell keine weiteren Tool-Aufrufe mehr erzeugt — oder ein Abbruchgrund eintritt.

### Gesamtarchitektur der Schleife

```text
query(params) → queryLoop()
│
│  while (true) {
│  ┌─────────────────────────────────────────────────┐
│  │ Phase 1: Kontext-Komprimierung                  │
│  │   applyToolResultBudget()   → Ergebnis-Budgets  │
│  │   snipCompactIfNeeded()     → Snip-Komprimierung │
│  │   microcompact()            → Microcompact       │
│  │   applyCollapsesIfNeeded()  → Context Collapse   │
│  │   autocompact()             → Auto-Compact       │
│  ├─────────────────────────────────────────────────┤
│  │ Phase 2: API-Aufruf (Streaming)                 │
│  │   callModel({messages, systemPrompt, tools})    │
│  │   for await (message of stream) {               │
│  │     yield message  → an UI/SDK durchreichen     │
│  │     Tool-Blöcke erkennen → StreamingToolExecutor│
│  │   }                                             │
│  ├─────────────────────────────────────────────────┤
│  │ Phase 3: Fehler-Recovery                        │
│  │   prompt-too-long → Context Collapse / Reactive │
│  │   max_output_tokens → Escalate / Recovery-Loop  │
│  │   Modell-Fallback → anderes Modell, Retry       │
│  ├─────────────────────────────────────────────────┤
│  │ Phase 4: Tool-Ausführung                        │
│  │   StreamingToolExecutor.getRemainingResults()   │
│  │   oder runTools() (Legacy-Pfad)                 │
│  │   → yield tool_result Nachrichten               │
│  ├─────────────────────────────────────────────────┤
│  │ Phase 5: Post-Processing                        │
│  │   Stop-Hooks ausführen                          │
│  │   Token-Budget prüfen                           │
│  │   Attachments laden (Memory, Skills, Queue)     │
│  │   state = {messages + assistantMessages +       │
│  │            toolResults, ...}                    │
│  │   continue  ← nächste Iteration                 │
│  └─────────────────────────────────────────────────┘
│  } // while (true)
```

### Schleifen-State

Der gesamte veränderliche Zustand einer Iteration steckt in einem einzigen `State`-Objekt:

```typescript
type State = {
  messages: Message[]              // Kumulierte Konversation
  toolUseContext: ToolUseContext    // Kontext für Tool-Aufrufe
  autoCompactTracking              // Tracking für Auto-Compact-Zyklen
  maxOutputTokensRecoveryCount     // Wie oft max_output_tokens Recovery lief
  hasAttemptedReactiveCompact      // Guard gegen Endlos-Compact-Schleifen
  maxOutputTokensOverride          // Einmaliger Override (Escalate: 8k → 64k)
  pendingToolUseSummary            // Haiku-Summary des vorherigen Turns (Promise)
  stopHookActive                   // Ob gerade ein Stop-Hook läuft
  turnCount                        // Aktuelle Turn-Nummer
  transition                       // Warum die vorherige Iteration fortfuhr
}
```

### Kontext-Komprimierungspipeline (Phase 1)

Vor jedem API-Aufruf durchlaufen die Nachrichten eine **6-stufige Pipeline**. Jede Stufe operiert auf dem Ergebnis der vorherigen:

| Stufe | Modul | Feature Flag | Wann aktiv | Was es tut |
| --- | --- | --- | --- | --- |
| 1 | `applyToolResultBudget()` | — | Immer | Kürzt überlange Tool-Ergebnisse per Token-Budget |
| 2 | `snipCompactIfNeeded()` | `HISTORY_SNIP` | Wenn aktiviert | Entfernt alte Turns, behält einen „Tail" |
| 3 | `microcompact()` | (immer) | Nach Snip | Entfernt redundante Tool-Ergebnisse aus dem Cache |
| 4 | `applyCollapsesIfNeeded()` | `CONTEXT_COLLAPSE` | Wenn aktiviert | Fasst Gruppen von Nachrichten zu Zusammenfassungen zusammen |
| 5 | `autocompact()` | (konfigurierbar) | Wenn Token-Threshold erreicht | Sendet gesamten Kontext an ein Modell zur Zusammenfassung |
| 6 | Reactive Compact | `REACTIVE_COMPACT` | Bei prompt-too-long Fehlern | Emergency-Compact nach 413-Antwort |

Die Stufen 4 und 5 können sich gegenseitig ersetzen: Context Collapse läuft *vor* Auto-Compact — wenn Collapse den Kontext genug reduziert, wird Auto-Compact zum No-Op.

### API-Streaming und Tool-Erkennung (Phase 2)

```text
for await (message of callModel(...)) {
  │
  ├─ message.type === 'assistant'?
  │    ├─ Tool-Use-Blöcke extrahieren
  │    ├─ StreamingToolExecutor.addTool(block)  ← sofort starten!
  │    └─ assistantMessages.push(message)
  │
  ├─ Withheld?  (prompt-too-long / max_output_tokens / media-error)
  │    └─ Nicht yielden, sondern Recovery abwarten
  │
  └─ yield message  → an UI/SDK
}
```

**Schlüssel-Insight:** Der `StreamingToolExecutor` startet concurrent-safe Tools (Grep, Glob, FileRead) bereits *während der API noch streamt*. Das spart Latenz, weil read-only Tools oft schneller fertig sind als die vollständige API-Antwort.

### Terminierung

Die Schleife endet durch einen von 10 `return`-Pfaden:

| Return-Reason | Auslöser |
| --- | --- |
| `completed` | Modell hat keine Tool-Aufrufe mehr → natürliches Ende |
| `blocking_limit` | Token-Limit erreicht (Auto-Compact aus) |
| `prompt_too_long` | 413-Fehler, Recovery fehlgeschlagen |
| `image_error` | Bild zu groß, Recovery fehlgeschlagen |
| `model_error` | API-Fehler (nicht-retryable) |
| `aborted_streaming` | User-Interrupt während Streaming |
| `aborted_tools` | User-Interrupt während Tool-Ausführung |
| `hook_stopped` | Stop-Hook hat Fortsetzung verhindert |
| `stop_hook_prevented` | Stop-Hook mit `preventContinuation` |
| `max_turns` | `maxTurns`-Limit erreicht |

---

## 3.3 Tool-System

Das Tool-System besteht aus drei Schichten: **Definition** (was ein Tool *ist*), **Registry** (welche Tools *verfügbar* sind) und **Ausführung** (wie Tools *laufen*).

### 3.3.1 Tool-Interface (`Tool.ts`)

Die Datei `src/Tool.ts` (~793 Zeilen) definiert das zentrale `Tool`-Interface. Jedes Tool — ob Built-in, MCP oder Plugin — muss dieses Interface implementieren.

**Die wichtigsten Methoden:**

| Methode | Signatur | Beschreibung |
| --- | --- | --- |
| `call()` | `(args, context, canUseTool, parentMsg, onProgress?) → ToolResult` | Hauptlogik — führt das Tool aus |
| `prompt()` | `(options) → string` | Erzeugt den Prompt-Text, den das Modell sieht |
| `description()` | `(input, options) → string` | Dynamische Beschreibung (kontextabhängig) |
| `checkPermissions()` | `(input, context) → PermissionResult` | Tool-eigene Berechtigungsprüfung |
| `validateInput()` | `(input, context) → ValidationResult` | Input-Validierung vor Ausführung |
| `inputSchema` | `z.ZodType` | Zod-v4-Schema für Input-Validierung |

**Concurrency-relevante Properties:**

| Property | Typ | Beschreibung |
| --- | --- | --- |
| `isConcurrencySafe(input)` | `boolean` | Darf parallel laufen? (Default: `false`) |
| `isReadOnly(input)` | `boolean` | Verändert nichts? (Default: `false`) |
| `isDestructive(input)` | `boolean` | Kann irreversiblen Schaden anrichten? |
| `interruptBehavior()` | `'cancel' \| 'block'` | Was bei User-Interrupt passiert |

**Rendering-Methoden** (für Terminal-UI):

Das `Tool`-Interface enthält über 10 `render*`-Methoden, die steuern, wie ein Tool-Aufruf in der Ink-UI dargestellt wird: `renderToolUseMessage`, `renderToolResultMessage`, `renderToolUseProgressMessage`, `renderToolUseRejectedMessage`, `renderToolUseErrorMessage`, `renderGroupedToolUse` u.v.m.

### `buildTool()` — Die Factory-Funktion

Statt das vollständige Interface zu implementieren, verwenden Tools die `buildTool()`-Funktion. Sie füllt sichere Defaults ein:

```text
buildTool(def) → { ...TOOL_DEFAULTS, ...def }

TOOL_DEFAULTS:
  isEnabled       → () => true
  isConcurrencySafe → () => false     (konservativ: nicht parallel)
  isReadOnly      → () => false       (konservativ: nimmt Schreibzugriff an)
  isDestructive   → () => false
  checkPermissions → allow (delegiert an Permission-System)
  toAutoClassifierInput → '' (Security-relevante Tools müssen überschreiben)
  userFacingName  → name
```

### `ToolUseContext` — Der Kontext für jeden Tool-Aufruf

`ToolUseContext` ist der zentrale Kontext, der bei *jedem* Tool-Aufruf mitgegeben wird. Er enthält:

```text
ToolUseContext
├── options
│   ├── commands[]         Verfügbare Slash-Commands
│   ├── tools[]            Alle registrierten Tools
│   ├── mainLoopModel      Aktuelles Modell (z.B. "claude-sonnet-4")
│   ├── mcpClients[]       MCP-Server-Verbindungen
│   ├── agentDefinitions   Verfügbare Agenten
│   └── thinkingConfig     Thinking-Modus (adaptive/disabled)
│
├── State-Zugriff
│   ├── getAppState()      Globalen App-State lesen
│   ├── setAppState()      Globalen App-State ändern
│   └── readFileState      LRU-Cache gelesener Dateien
│
├── Steuerung
│   ├── abortController    Für User-Interrupts
│   ├── messages[]         Aktuelle Konversation
│   └── agentId?           Falls Sub-Agent: seine ID
│
├── UI-Callbacks
│   ├── setToolJSX()       Tool-UI im Terminal anzeigen
│   ├── addNotification()  Benachrichtigungen anzeigen
│   └── sendOSNotification()  OS-Benachrichtigungen
│
└── Tracking
    ├── queryTracking      Chain-ID + Tiefe
    ├── updateFileHistoryState()  Datei-Änderungen tracken
    └── updateAttributionState()  Attribution für Commits
```

### 3.3.2 Tool-Registry (`tools.ts`)

Die Datei `src/tools.ts` (~390 Zeilen) ist die **zentrale Quelle der Wahrheit** für alle verfügbaren Tools. Drei Funktionen bilden die Tool-Assemblierung:

```text
getAllBaseTools()                    // Alle Built-in Tools (ungefiltert)
    │
    ▼
getTools(permissionContext)          // Gefiltert nach Deny-Rules + isEnabled()
    │
    ▼
assembleToolPool(permCtx, mcpTools)  // + MCP-Tools, dedupliziert, sortiert
```

#### `getAllBaseTools()` — Alle Built-in Tools

Diese Funktion listet *alle* Tools, die in der aktuellen Build-Konfiguration verfügbar sind:

| Kategorie | Tools | Concurrency |
| --- | --- | --- |
| **Dateisystem** | `FileRead`, `FileEdit`, `FileWrite`, `NotebookEdit` | Read: parallel, Write: seriell |
| **Shell** | `Bash`, `PowerShell` (optional) | Seriell |
| **Suche** | `Glob`, `Grep` (wenn kein eingebettetes bfs/ugrep) | Parallel |
| **Web** | `WebFetch`, `WebSearch`, `WebBrowser` (feature-gated) | Parallel |
| **Agenten** | `Agent`, `TaskStop`, `TaskOutput`, `SendMessage` | Seriell |
| **Tasks (v2)** | `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList` | — |
| **Planung** | `EnterPlanMode`, `ExitPlanModeV2` | — |
| **MCP** | `ListMcpResources`, `ReadMcpResource` | Parallel |
| **Sonstige** | `TodoWrite`, `AskUserQuestion`, `SkillTool`, `Brief`, `Config`, `ToolSearch`, ... | Variabel |

Viele Tools werden **Feature-gated** importiert — sie existieren nur, wenn das entsprechende Flag zur Build-Zeit aktiv ist:

```text
feature('COORDINATOR_MODE')   → Coordinator-spezifische Tools
feature('WEB_BROWSER_TOOL')   → WebBrowserTool
feature('HISTORY_SNIP')       → SnipTool
feature('WORKFLOW_SCRIPTS')   → WorkflowTool
feature('TERMINAL_PANEL')     → TerminalCaptureTool
feature('UDS_INBOX')          → ListPeersTool
```

#### `getTools()` — Gefilterter Tool-Pool

`getTools()` wendet drei Filter an:

1. **Deny-Rules:** Tools mit blanket-deny (`alwaysDenyRules` ohne `ruleContent`) werden entfernt, *bevor* das Modell sie sieht
2. **REPL-Mode:** Wenn `REPLTool` aktiv ist, werden die darin gekapselten Primitiv-Tools (`Bash`, `FileRead`, etc.) ausgeblendet
3. **`isEnabled()`:** Jedes Tool kann sich dynamisch deaktivieren

#### `assembleToolPool()` — Die finale Tool-Liste

```text
assembleToolPool(permissionContext, mcpTools):
  builtInTools = getTools(permCtx)     // sortiert nach Name
  allowedMcp = filterByDenyRules(mcp)  // sortiert nach Name
  return uniqBy([...builtIn, ...mcp], 'name')
         ↑ Built-in gewinnen bei Namenskonflikten
```

Die Sortierung ist **nicht kosmetisch** — sie dient der **Prompt-Cache-Stabilität**: Die API legt Cache-Breakpoints nach dem letzten Built-in-Tool. Wenn MCP-Tools zwischen Built-ins sortiert würden, würde jede MCP-Änderung alle Cache-Keys invalidieren.

### 3.3.3 Tool-Ausführung

Die Tool-Ausführung verteilt sich auf drei Dateien:

```text
toolOrchestration.ts        Partitionierung + Ablaufsteuerung
StreamingToolExecutor.ts    Streaming-Ausführung während API-Response
toolExecution.ts            Einzelnes Tool: Validierung → Permission → Call
```

#### Partitionierung (`partitionToolCalls`)

Wenn das Modell mehrere Tools in einer Antwort aufruft, partitioniert `partitionToolCalls()` sie in Batches:

```text
Eingabe: [Grep, Grep, FileWrite, FileRead, FileRead]
                                  ↓
Batch 1: [Grep, Grep]        isConcurrencySafe: true   → parallel (max 10)
Batch 2: [FileWrite]          isConcurrencySafe: false  → seriell
Batch 3: [FileRead, FileRead] isConcurrencySafe: true   → parallel
```

Die Regel: **Aufeinanderfolgende concurrent-safe Tools** werden gebatcht; sobald ein nicht-concurrent-safe Tool kommt, wird es allein ausgeführt; danach kann wieder gebatcht werden.

#### `StreamingToolExecutor`

Der `StreamingToolExecutor` (`src/services/tools/StreamingToolExecutor.ts`, ~531 Zeilen) startet Tools *während die API noch streamt*:

```text
API-Stream:
  ──[text]──[tool_use:Grep1]──[text]──[tool_use:Grep2]──[tool_use:FileEdit]──►
                  │                          │                    │
  addTool(Grep1)──┘     addTool(Grep2)───────┘    addTool(Edit)──┘
       │                      │                        │
       ▼                      ▼                        ▼
  ┌─────────┐          ┌─────────┐              ┌──────────┐
  │ execute │          │ execute │              │  queued   │
  │ (sofort)│          │ (sofort)│              │(wartet auf│
  └─────────┘          └─────────┘              │Grep 1+2) │
                                                └──────────┘
```

Kernlogik der Concurrency-Prüfung:

```text
canExecuteTool(isConcurrencySafe):
  executing = tools.filter(status === 'executing')
  return executing.length === 0
      || (isConcurrencySafe && ALL executing are concurrencySafe)
```

Besonderes Verhalten bei Fehlern:

- **Bash-Fehler:** Erzeugt einen Child-AbortController, der Geschwister-Prozesse sofort stoppt (aber nicht den Parent-Query)
- **User-Interrupt:** Erzeugt synthetische `REJECT_MESSAGE` für queued Tools
- **Streaming-Fallback:** `discard()` verwirft alle Ergebnisse des fehlgeschlagenen Versuchs

#### `runToolUse()` — Einzelner Tool-Aufruf

Die Funktion `runToolUse()` in `toolExecution.ts` (~1.746 Zeilen) orchestriert einen einzelnen Tool-Aufruf:

```text
runToolUse(toolUse, assistantMessage, canUseTool, context)
│
├─ 1. Tool finden
│     findToolByName() → auch Aliases prüfen (z.B. "KillShell" → "TaskStop")
│
├─ 2. Input validieren
│     tool.inputSchema.safeParse(input)
│     tool.validateInput?()  → tool-spezifische Validierung
│
├─ 3. Permission prüfen
│     tool.checkPermissions(input)  → tool-eigene Prüfung
│     canUseTool(tool, input)       → Permission-Pipeline (5 Stufen)
│     ├─ allow    → weiter
│     ├─ deny     → tool_result mit Fehlermeldung
│     └─ ask      → User-Prompt (interaktiv) / auto-deny (headless)
│
├─ 4. Pre-Tool-Use Hooks
│     runPreToolUseHooks() → benutzerdefinierte Hooks
│
├─ 5. Tool ausführen
│     tool.call(parsedInput, context, canUseTool, assistantMessage)
│     ├─ Timeout/Abort → AbortError
│     ├─ Fehler → formatiertes error tool_result
│     └─ Erfolg → ToolResult
│
├─ 6. Post-Tool-Use Hooks
│     runPostToolUseHooks() → benutzerdefinierte Hooks
│     Stop-Hooks prüfen → können Fortsetzung verhindern
│
└─ 7. Ergebnis aufbereiten
      mapToolResultToToolResultBlockParam()
      processToolResultBlock() → Große Ergebnisse auf Disk auslagern
      yield {message, contextModifier?}
```

---

## 3.4 Command-System (Slash-Commands)

Die Datei `src/commands.ts` (~755 Zeilen) verwaltet die Slash-Commands, die Benutzer mit `/` aufrufen können (z.B. `/compact`, `/help`, `/review`).

### Command-Typen

Es gibt drei grundlegende Typen (definiert in `src/types/command.ts`):

| Typ | Ausführung | Beispiele |
| --- | --- | --- |
| `prompt` | Wird in Text expandiert und an das Modell gesendet | `/review`, `/commit`, Skills, Workflows |
| `local` | Führt lokale Logik aus, gibt Text zurück | `/cost`, `/compact`, `/clear` |
| `local-jsx` | Rendert interaktive Ink-UI-Komponenten | `/config`, `/mcp`, `/install` |

### Command-Zusammensetzung

Commands kommen aus **sechs Quellen**, die beim Aufruf von `getCommands()` zusammengeführt werden:

```text
getCommands(cwd)
│
├─ loadAllCommands(cwd)  ← memoized nach cwd
│  ├─ getBundledSkills()              Eingebaute Skills (synchron)
│  ├─ getBuiltinPluginSkillCommands() Skills aus Built-in-Plugins
│  ├─ getSkillDirCommands(cwd)        .claude/skills/ Verzeichnisse
│  ├─ getWorkflowCommands(cwd)        Workflow-Scripts (feature-gated)
│  ├─ getPluginCommands()             Plugin-Commands
│  ├─ getPluginSkills()               Plugin-Skills
│  └─ COMMANDS()                      ~80 Built-in Commands (memoized)
│
├─ getDynamicSkills()                 Skills, die während Dateioperationen entdeckt wurden
│
└─ Filter: meetsAvailabilityRequirement() && isCommandEnabled()
```

### Command-Verfügbarkeit

Commands können ihre Sichtbarkeit über zwei unabhängige Mechanismen steuern:

| Mechanismus | Prüfzeitpunkt | Beispiel |
| --- | --- | --- |
| `availability: ['claude-ai', 'console']` | Statisch (Auth/Provider) | `/extra-usage` nur für Claude.ai-Subscriber |
| `isEnabled()` | Dynamisch (Feature Flags, Env) | `/voice` nur wenn `VOICE_MODE` aktiv |

### Besondere Command-Sets

| Set | Zweck |
| --- | --- |
| `INTERNAL_ONLY_COMMANDS` | Nur für Anthropic-interne Builds (`USER_TYPE === 'ant'`) |
| `REMOTE_SAFE_COMMANDS` | Sicher in `--remote`-Mode (keine lokale Filesystem-Abhängigkeit) |
| `BRIDGE_SAFE_COMMANDS` | Dürfen über die Remote-Control-Bridge (Mobile/Web) ausgeführt werden |

### Skill-Invocation durch das Modell

Nicht alle Commands sind direkt vom Modell aufrufbar. Die Funktion `getSkillToolCommands()` filtert:

```text
getSkillToolCommands(cwd):
  allCommands.filter(cmd =>
    cmd.type === 'prompt' &&
    !cmd.disableModelInvocation &&
    cmd.source !== 'builtin' &&
    (cmd.loadedFrom in ['bundled', 'skills', 'commands_DEPRECATED']
     || cmd.hasUserSpecifiedDescription
     || cmd.whenToUse)
  )
```

Das Modell sieht diese Commands über das `SkillTool`, das sie als auswählbare Skills präsentiert.

---

## 3.5 Konkrete Tool-Implementierungen

### 3.5.1 BashTool

Das mächtigste Tool — es gibt dem Agenten eine Shell. Datei: `src/tools/BashTool/BashTool.tsx` (~1.144 Zeilen).

**Input-Schema:**

| Feld | Typ | Beschreibung |
| --- | --- | --- |
| `command` | `string` | Der auszuführende Shell-Befehl |
| `timeout` | `number?` | Timeout in Millisekunden (Default: 120s, Max: 600s) |
| `description` | `string?` | Beschreibung für Permission-Dialog |

**Concurrency-Klassifikation:**

Das BashTool führt eine **semantische Analyse** des Befehls durch, um zu entscheiden, ob er parallel laufen darf:

```text
isSearchOrReadBashCommand(command):
  splitCommandWithOperators(command)
  für jedes Teil:
    ├─ BASH_SEARCH_COMMANDS: grep, rg, find, ag, ...  → isSearch
    ├─ BASH_READ_COMMANDS: cat, head, wc, jq, ...     → isRead
    ├─ BASH_LIST_COMMANDS: ls, tree, du               → isList
    ├─ BASH_SEMANTIC_NEUTRAL: echo, printf, true, :   → neutral (ignoriert)
    └─ alles andere                                    → NICHT read-only
  Pipeline: ALLE Teile müssen search/read/list sein
```

**Sicherheits-Features:**

- AST-Parsing (`parseForSecurity()`) für Sicherheitsanalyse
- Sandbox-Ausführung (`SandboxManager`) für gefährliche Befehle
- Pfad-Validierung (`pathValidation.ts`) — Zugriff auf Dateien außerhalb des Projekts blockieren
- Destruktive-Command-Warnung (`destructiveCommandWarning.ts`)
- Read-Only-Validierung (`readOnlyValidation.ts`) im Plan-Modus
- `sed`-Edit-Erkennung (`sedEditParser.ts`) — `sed -i` wird wie ein File-Edit behandelt

### 3.5.2 AgentTool — Sub-Agenten

Das `AgentTool` (`src/tools/AgentTool/AgentTool.tsx`, ~1.398 Zeilen) spawnt Sub-Agenten — vollständige Claude-Instanzen mit eigenem Kontext, eigenen Tools und eigenem Token-Budget.

**Agent-Definition (`AgentDefinition`):**

Agenten werden aus Markdown-Dateien in `.claude/agents/` geladen (ähnlich wie Skills). Jede Definition enthält:

| Feld | Beschreibung |
| --- | --- |
| `name` | Eindeutiger Name (z.B. `"Explore"`, `"Plan"`) |
| `description` | Kurzbeschreibung für das Modell |
| `prompt` | System-Prompt des Agenten |
| `tools` | Erlaubte Tools (Whitelist) |
| `disallowedTools` | Verbotene Tools (Blacklist) |
| `model` | Modell-Override (`"sonnet"`, `"opus"`, `"haiku"`, `"inherit"`) |
| `permissionMode` | Berechtigungsmodus (`"plan"`, `"auto"`, ...) |
| `maxTurns` | Maximale Turns |
| `mcpServers` | Agenten-spezifische MCP-Server |
| `hooks` | Frontmatter-definierte Hooks |
| `isolation` | `"worktree"` (Git-Worktree) oder `"remote"` (Remote-Umgebung) |
| `memory` | Memory-Scope (`"user"`, `"project"`, `"local"`) |

**Built-in Agents:**

- `Explore` — One-Shot-Agent für Code-Exploration
- `Plan` — One-Shot-Agent für Planung
- `GENERAL_PURPOSE_AGENT` — Default-Agent ohne spezielle Einschränkungen

**Ausführungsmodi:**

```text
AgentTool.call()
│
├─ Foreground (default)
│   runAgent() → query() mit eigenem Kontext
│   ├─ Eigener AbortController
│   ├─ Eigene readFileState (geklont)
│   ├─ Eingeschränktes Tool-Set (resolveAgentTools)
│   └─ Eigener Transcript (Sidechain)
│
├─ Background (run_in_background: true)
│   Wie Foreground, aber:
│   ├─ Läuft in eigenem Promise (nicht blockierend)
│   └─ Benachrichtigung bei Completion
│
├─ Worktree-Isolation
│   createAgentWorktree() → Git-Worktree erstellen
│   Agent arbeitet auf Kopie des Repos
│
└─ Remote (Anthropic-intern)
    teleportToRemote() → Agent in Cloud-Umgebung
```

### 3.5.3 Coordinator-Modus

Der Coordinator-Modus (`src/coordinator/coordinatorMode.ts`, ~370 Zeilen) verwandelt die Haupt-Instanz in einen **Orchestrator**, der *nur* Sub-Agenten spawnt und koordiniert:

```text
Coordinator-Modus:
┌─────────────────────────────────────────────┐
│  Coordinator (Hauptprozess)                 │
│  Tools: Agent, TaskStop, SendMessage        │
│  Aufgabe: Aufgaben zerlegen & delegieren    │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │   │
│  │ Bash,    │  │ Bash,    │  │ Bash,    │   │
│  │ FileRead,│  │ FileRead,│  │ FileRead,│   │
│  │ FileEdit │  │ FileEdit │  │ FileEdit │   │
│  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────┘
```

Aktivierung: `CLAUDE_CODE_COORDINATOR_MODE=1`

Die Worker erhalten ein eingeschränktes Tool-Set (`ASYNC_AGENT_ALLOWED_TOOLS`), während der Coordinator selbst nur `Agent`, `TaskStop` und `SendMessage` verwendet.

---

## 3.6 Referenz: Alle ~40 Built-in Tools

Die folgende Tabelle listet alle Tools, die `getAllBaseTools()` registriert:

| Tool | Kategorie | Parallel | Feature Gate | Beschreibung |
| --- | --- | --- | --- | --- |
| `Agent` | Multi-Agent | ✗ | — | Sub-Agent spawnen |
| `TaskOutput` | Multi-Agent | ✗ | — | Ergebnis eines Task abrufen |
| `TaskStop` | Multi-Agent | ✗ | — | Laufenden Task/Shell beenden |
| `Bash` | Shell | ✗ | — | Shell-Befehl ausführen |
| `Glob` | Suche | ✓ | ¬embedded | Datei-Suche per Glob-Pattern |
| `Grep` | Suche | ✓ | ¬embedded | Text-Suche per Regex |
| `FileRead` | Dateisystem | ✓ | — | Datei lesen |
| `FileEdit` | Dateisystem | ✗ | — | Datei bearbeiten (Search & Replace) |
| `FileWrite` | Dateisystem | ✗ | — | Datei schreiben/erstellen |
| `NotebookEdit` | Dateisystem | ✗ | — | Jupyter-Notebook bearbeiten |
| `WebFetch` | Web | ✓ | — | URL abrufen |
| `WebSearch` | Web | ✓ | — | Web-Suche |
| `WebBrowser` | Web | ✓ | `WEB_BROWSER_TOOL` | Headless Browser |
| `TodoWrite` | Planung | ✗ | — | Todo-Liste verwalten |
| `EnterPlanMode` | Planung | ✗ | — | In Plan-Modus wechseln |
| `ExitPlanModeV2` | Planung | ✗ | — | Plan-Modus verlassen |
| `SkillTool` | Skills | ✗ | — | Skill/Command aufrufen |
| `AskUserQuestion` | Interaktion | ✗ | — | Benutzer eine Frage stellen |
| `Brief` | Kommunikation | ✗ | — | Kurzzusammenfassung |
| `Config` | Konfiguration | ✗ | `ant` | Claude-Code-Config bearbeiten |
| `Tungsten` | Intern | ✗ | `ant` | Anthropic-internes Tool |
| `SendMessage` | Multi-Agent | ✗ | — | Nachricht an anderen Agenten |
| `ListPeers` | Multi-Agent | ✓ | `UDS_INBOX` | Peer-Agenten auflisten |
| `TeamCreate` | Multi-Agent | ✗ | Agent Swarms | Team erstellen |
| `TeamDelete` | Multi-Agent | ✗ | Agent Swarms | Team löschen |
| `EnterWorktree` | Isolation | ✗ | Worktree-Mode | Git-Worktree erstellen |
| `ExitWorktree` | Isolation | ✗ | Worktree-Mode | Git-Worktree verlassen |
| `TaskCreate` | Tasks v2 | ✗ | Todo v2 | Task erstellen |
| `TaskGet` | Tasks v2 | ✓ | Todo v2 | Task abrufen |
| `TaskUpdate` | Tasks v2 | ✗ | Todo v2 | Task aktualisieren |
| `TaskList` | Tasks v2 | ✓ | Todo v2 | Tasks auflisten |
| `LSP` | IDE | ✓ | `ENABLE_LSP_TOOL` | LSP-Server abfragen |
| `ListMcpResources` | MCP | ✓ | — | MCP-Ressourcen auflisten |
| `ReadMcpResource` | MCP | ✓ | — | MCP-Ressource lesen |
| `ToolSearch` | Meta | ✓ | Tool-Search | Verfügbare Tools durchsuchen |
| `SnipTool` | Kontext | ✗ | `HISTORY_SNIP` | Manuelles Snipping |
| `TerminalCapture` | IDE | ✓ | `TERMINAL_PANEL` | Terminal-Output erfassen |
| `Sleep` | Async | ✗ | `PROACTIVE`/`KAIROS` | Warten auf Events |
| `CronCreate/Delete/List` | Scheduling | ✗ | `AGENT_TRIGGERS` | Cron-Aufgaben verwalten |
| `Workflow` | Workflows | ✗ | `WORKFLOW_SCRIPTS` | Workflow-Scripts ausführen |
| `CtxInspect` | Debug | ✓ | `CONTEXT_COLLAPSE` | Kontext inspizieren |
| `Monitor` | Monitoring | ✗ | `MONITOR_TOOL` | File/Process-Monitoring |

---

## 3.7 Skills und Plugins

### Skills

Skills sind **vom Benutzer definierte Prompt-Erweiterungen** — Markdown-Dateien mit optionalem Frontmatter, die das Modell bei Bedarf als Kontext laden kann.

**Lade-Hierarchie:**

```text
~/.claude/skills/           ← User-global (userSettings)
.claude/skills/             ← Projekt-lokal (projectSettings)
managed/.claude/skills/     ← Policy-verwaltet (policySettings)
plugins/*/skills/           ← Plugin-bereitgestellt
built-in                    ← Eingebaute Skills (bundled)
MCP-Server                  ← MCP-bereitgestellte Skills (feature-gated)
```

**Frontmatter-Felder:**

| Feld | Beschreibung |
| --- | --- |
| `description` | Kurzbeschreibung (sichtbar für Modell und Benutzer) |
| `whenToUse` | Wann das Modell diesen Skill verwenden soll |
| `tools` | Erlaubte Tools für diesen Skill |
| `model` | Modell-Override |
| `context` | `'inline'` (expandiert in Konversation) oder `'fork'` (als Sub-Agent) |
| `agent` | Agent-Typ für `fork`-Kontext |
| `effort` | Effort-Level |
| `paths` | Glob-Patterns — Skill wird nur nach Berühren passender Dateien sichtbar |
| `hooks` | Hooks, die beim Aufruf des Skills registriert werden |

### Plugins

Plugins sind ein **strukturierteres Erweiterungssystem** als Skills. Sie werden aus Git-Repositories geladen und können eigene Commands, Skills, Agenten und Hooks bereitstellen. Die Plugin-Architektur ist in `src/plugins/` und `src/utils/plugins/` implementiert.

---

## 3.8 Zusammenfassung: Datenfluss durch die Hauptkomponenten

```text
User-Eingabe
    │
    ▼
QueryEngine.submitMessage()
    │
    ├─ processUserInput()  → Slash-Commands verarbeiten
    │
    ├─ fetchSystemPromptParts()  → System-Prompt aufbauen
    │
    ▼
query() / queryLoop()
    │
    ├─ Kontext-Komprimierung (6 Stufen)
    │
    ├─ callModel()  → API-Streaming
    │   │
    │   └─ StreamingToolExecutor.addTool()  → Parallel-Start
    │
    ├─ runTools() / getRemainingResults()
    │   │
    │   ├─ partitionToolCalls()  → Batching
    │   │
    │   └─ runToolUse()
    │       │
    │       ├─ validateInput()
    │       ├─ checkPermissions()
    │       ├─ canUseTool()  → 5-stufige Pipeline
    │       ├─ tool.call()   → BashTool / FileEditTool / AgentTool / ...
    │       └─ post-Hooks
    │
    ├─ Attachments (Memory, Skills, Queue-Commands)
    │
    └─ state = {...} → continue (nächste Iteration)
         oder
    return {reason: 'completed'}
    │
    ▼
QueryEngine → yield {type: 'result', ...}
```
