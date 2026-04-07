# 4. Typische Abläufe

Dieses Kapitel zeichnet die wichtigsten End-to-End-Abläufe in Claude Code nach.
Kapitel 3 beschrieb die Komponenten isoliert — hier wird gezeigt, wie sie in konkreten Szenarien zusammenspielen.

---

## 4.1 Startup — Vom Prozessstart bis zum REPL-Prompt

Der Start von Claude Code ist bewusst **mehrstufig** aufgebaut, damit die interaktive Oberfläche so schnell wie möglich erscheint.
Schwere Arbeit wird *nach* dem ersten Render angestoßen (Deferred Prefetches).

### Sequenz

```text
┌───────────────────────────────────────────────────────────────────────┐
│  cli.tsx  →  main.tsx  →  init.ts  →  setup.ts  →  replLauncher.tsx │
└───────────────────────────────────────────────────────────────────────┘

  Prozessstart (cli.tsx)
       │
       │  ◄── Side-Effects vor allen Imports:
       │      profileCheckpoint('main_tsx_top')
       │      startMdmRawRead()           // MDM/Policy-Settings (async)
       │      startKeychainPrefetch()     // Keychain-Credentials (async)
       │
  ┌────▼──────────────────────────────────────────────────┐
  │  ~200 Imports laden (~135 ms)                         │
  │  Commander.js, React/Ink, Feature-Gates, Tools, ...   │
  └────┬──────────────────────────────────────────────────┘
       │
  main()                                     [main.tsx:587]
       │
       ├─ initializeWarningHandler()
       ├─ Argv-Rewriting (cc://, ssh, assistant)
       ├─ isNonInteractive?  (--print / --sdk-url / !tty)
       ├─ setClientType()    (cli / sdk-cli / remote / ...)
       ├─ eagerLoadSettings()  (--settings / --setting-sources)
       │
       ▼
  run()                                      [main.tsx:866]
       │
       ├─ Commander initialisieren + Optionen registrieren
       │
       ├─ program.hook('preAction')          [main.tsx:917]
       │   │
       │   ├─ await ensureMdmSettingsLoaded()
       │   ├─ await ensureKeychainPrefetchCompleted()
       │   ├─ await init()                   ◄── [entrypoints/init.ts]
       │   │     │
       │   │     ├─ enableConfigs()
       │   │     ├─ applySafeConfigEnvironmentVariables()
       │   │     ├─ applyExtraCACertsFromConfig()
       │   │     ├─ setupGracefulShutdown()
       │   │     ├─ 1P-Event-Logging + GrowthBook
       │   │     ├─ OAuth-Account-Info
       │   │     ├─ configureGlobalMTLS()
       │   │     ├─ configureGlobalAgents()  (Proxy)
       │   │     ├─ preconnectAnthropicApi() (TCP+TLS-Handshake)
       │   │     ├─ initUpstreamProxy()      (nur CCR)
       │   │     ├─ registerCleanup(...)
       │   │     └─ ensureScratchpadDir()
       │   │
       │   ├─ initSinks()  (Analytics + Error-Logging)
       │   ├─ runMigrations()  (CURRENT_MIGRATION_VERSION = 11)
       │   └─ loadRemoteManagedSettings()
       │
       ▼
  .action(prompt, options)                   [main.tsx:1028]
       │
       ├─ Optionen extrahieren & validieren
       │   (permissionMode, MCP-Config, systemPrompt, ...)
       │
       ├─ setup()                            [setup.ts]
       │   │
       │   ├─ setCwd() + setSessionId()
       │   ├─ captureHooksConfigSnapshot()
       │   ├─ initializeFileChangedWatcher()
       │   ├─ Worktree-Erstellung (falls --worktree)
       │   ├─ initSessionMemory()
       │   ├─ lockCurrentVersion()
       │   ├─ getCommands(), loadPluginHooks()
       │   ├─ initSinks()
       │   ├─ logEvent('tengu_started')
       │   └─ checkForReleaseNotes()
       │
       ├─ showSetupScreens()
       │   ├─ Trust-Dialog (bei erstem Start in Verzeichnis)
       │   ├─ initializeTelemetryAfterTrust()
       │   ├─ Login / API-Key-Check
       │   └─ Onboarding (falls neuer Nutzer)
       │
       ├─ new QueryEngine(config)
       │
       └─ launchRepl()                       [replLauncher.tsx]
             │
             ├─ React/Ink rendern (App + REPL)
             ├─ startDeferredPrefetches()     ◄── Erst NACH Render:
             │     initUser(), getUserContext(),
             │     prefetchSystemContextIfSafe(),
             │     countFilesRoundedRg(),
             │     initializeAnalyticsGates(),
             │     refreshModelCapabilities(),
             │     settingsChangeDetector.initialize(),
             │     skillChangeDetector.initialize()
             │
             └─ ► REPL-Prompt ist sichtbar ◄
```

### Zeitkritische Optimierungen

| Technik | Zweck |
| --- | --- |
| **Side-Effect-Imports** (Zeile 12–20 in main.tsx) | MDM + Keychain starten *vor* den ~135 ms Import-Durchlauf |
| **preconnectAnthropicApi()** | TCP+TLS-Handshake (~100–200 ms) überlappt mit Init-Arbeit |
| **startDeferredPrefetches()** | Alles was *nicht* für den ersten Render nötig ist: User-Info, Git-Kontext, Tips, Analytics |
| **`--bare` Mode** | Überspringt Hooks, LSP, Plugin-Sync, Attribution, CLAUDE.md — für Skripte/Pipelines |
| **`isEnvTruthy(EXIT_AFTER_FIRST_RENDER)`** | Benchmarking: misst nur Time-to-First-Render |

### Headless-Pfad (`--print`)

Im Non-Interactive-Modus weicht der Ablauf ab Zeile ~1028 ab:

```text
.action(prompt, options)
    │
    ├─ setup()  (wie oben, aber ohne Trust-Dialog)
    │
    └─ runHeadless()              statt launchRepl()
          │
          ├─ getInputPrompt()    (stdin lesen, 3s Timeout)
          ├─ new QueryEngine(config)
          ├─ queryEngine.submitMessage(prompt)
          ├─ Streaming → stdout  (text / json / stream-json)
          └─ process.exit()
```

---

## 4.2 Query-Lifecycle — Vom User-Prompt bis zur Antwort

### Überblick

```text
User tippt "Fix the bug in main.ts" + Enter
     │
     ▼
QueryEngine.submitMessage(prompt)
     │
     ├──────────────────────────────────────────────────────────┐
     │  Phase 1: Vorverarbeitung                                │
     │  ┌──────────────────────────────────────────────────┐    │
     │  │  processUserInput(prompt)                        │    │
     │  │    ├─ Slash-Command?  → command.execute()        │    │
     │  │    ├─ Attachments (Bilder, Dateien)              │    │
     │  │    └─ → messages[], shouldQuery, toolOverrides   │    │
     │  └──────────────────────────────────────────────────┘    │
     │                                                          │
     │  Phase 2: System-Prompt aufbauen                         │
     │  ┌──────────────────────────────────────────────────┐    │
     │  │  fetchSystemPromptParts()                        │    │
     │  │    ├─ Basis-Prompt (Identität, Regeln)           │    │
     │  │    ├─ Tool-Descriptions                          │    │
     │  │    ├─ CLAUDE.md-Inhalte                          │    │
     │  │    ├─ Custom/Append System-Prompt                │    │
     │  │    └─ Kontext (User, Projekt, Environment)       │    │
     │  └──────────────────────────────────────────────────┘    │
     │                                                          │
     │  Phase 3: API-Aufruf + Tool-Loop                         │
     │  ┌──────────────────────────────────────────────────┐    │
     │  │  query()  →  queryLoop()                         │    │
     │  │    (Details: Abschnitt 4.3)                      │    │
     │  └──────────────────────────────────────────────────┘    │
     │                                                          │
     │  Phase 4: Nachbereitung                                  │
     │  ┌──────────────────────────────────────────────────┐    │
     │  │  ├─ Session persistieren (JSONL)                 │    │
     │  │  ├─ Kosten-Tracking aktualisieren                │    │
     │  │  └─ yield {type: 'result', message, costUSD}     │    │
     │  └──────────────────────────────────────────────────┘    │
     └──────────────────────────────────────────────────────────┘
```

### User-Input-Verarbeitung im Detail

`processUserInput()` erkennt folgende Eingabe-Typen:

| Eingabe | Verhalten |
| --- | --- |
| `/compact` | Komprimiert den Kontext, setzt `shouldQuery = false` |
| `/clear` | Leert die Nachrichtenhistorie |
| `/model sonnet` | Wechselt das Modell, kein API-Aufruf |
| `/resume` | Lädt eine vorherige Session |
| `/skill-name args` | Lädt Skill-Content, baut in Prompt ein |
| Normaler Text | Erzeugt `UserMessage`, `shouldQuery = true` |
| Text + Bild (Paste) | `UserMessage` mit `image`-Content-Block |
| Leer-Eingabe | Wird ignoriert |

---

## 4.3 Die Query-Schleife — Kern des Agentic-Verhaltens

Die Funktion `queryLoop()` (`src/query.ts`, ~1.730 Zeilen) ist der **zentrale Ausführungskern** — eine `while(true)`-Schleife, die solange iteriert, bis keine weiteren Tool-Aufrufe nötig sind.

### Zustandsmodell

Jede Iteration operiert auf einem `State`-Objekt:

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined    // Warum die vorherige Iteration weitermachte
}
```

### Iterations-Sequenz

```text
while (true) {
     │
     ├─ 1. Kontext-Komprimierung (6 Stufen)
     │     ├─ applyToolResultBudget()     Einzelne Tool-Ergebnisse kürzen
     │     ├─ snipCompactIfNeeded()       Ältere Turns entfernen (feature-gated)
     │     ├─ microcompact()              Cache-aware Microcompaction
     │     ├─ applyCollapsesIfNeeded()    Context-Collapse (feature-gated)
     │     ├─ autocompact()               Proaktive Zusammenfassung bei >80% Fenster
     │     └─ reactiveCompact             Reaktiv bei API-413 (Prompt-Too-Long)
     │
     ├─ 2. Blocking-Limit prüfen
     │     calculateTokenWarningState() → ggf. Abbruch
     │
     ├─ 3. API-Streaming
     │     ┌──────────────────────────────────────────────────┐
     │     │  for await (message of callModel(...)) {         │
     │     │    ├─ yield message           → UI/SDK           │
     │     │    ├─ assistantMessages.push() (tool_use blocks) │
     │     │    └─ streamingToolExecutor.addTool()            │
     │     │        → Tool startet PARALLEL zum Stream        │
     │     │  }                                               │
     │     └──────────────────────────────────────────────────┘
     │
     ├─ 4. Fehlerbehandlung (withheld messages)
     │     ├─ Prompt-Too-Long → Collapse-Drain → Reactive Compact
     │     ├─ Media-Size-Error → Reactive Compact (Strip+Retry)
     │     └─ Max-Output-Tokens → Escalation (8k→64k) → Recovery-Loop (3×)
     │
     ├─ 5. needsFollowUp?
     │     │
     │     ├─ NEIN → Stop-Hooks → Token-Budget-Check → return 'completed'
     │     │
     │     └─ JA ──────────────────────────────────┐
     │                                              │
     ├─ 6. Tool-Ausführung                          │
     │     ├─ streamingToolExecutor.getRemainingResults()
     │     │   oder
     │     ├─ runTools(toolUseBlocks, ...)
     │     │     ├─ partitionToolCalls() → Batching
     │     │     │   ├─ Read-only → concurrent (max 10)
     │     │     │   └─ Mutierend  → seriell
     │     │     └─ runToolUse() pro Block  (→ Abschnitt 4.4)
     │     │
     │     └─ shouldPreventContinuation?  (Hook-Stopped)
     │
     ├─ 7. Attachments injizieren
     │     ├─ getAttachmentMessages()     CLAUDE.md-Änderungen, File-Diffs
     │     ├─ Memory-Prefetch konsumieren (falls settled)
     │     ├─ Skill-Discovery-Prefetch
     │     └─ Queued Commands (Notifications, User-Prompts)
     │
     ├─ 8. maxTurns prüfen
     │     Überschritten → return 'max_turns'
     │
     └─ 9. state = { messages: [...messages, ...assistant, ...toolResults],
     │               turnCount: turnCount + 1,
     │               transition: {reason: 'next_turn'} }
     │     continue → nächste Iteration
}
```

### Continue-Gründe (Transitions)

Die `transition`-Eigenschaft protokolliert, *warum* die Schleife eine weitere Iteration startet:

| Transition | Bedeutung |
| --- | --- |
| `next_turn` | Normaler Fall: Tool-Ergebnisse liegen vor, Modell soll fortfahren |
| `collapse_drain_retry` | Context-Collapse-Drain nach Prompt-Too-Long |
| `reactive_compact_retry` | Reaktive Komprimierung nach API-413 |
| `max_output_tokens_escalate` | Retry mit 64k statt 8k max_tokens |
| `max_output_tokens_recovery` | Fortsetzung nach Token-Limit (bis 3×) |
| `stop_hook_blocking` | Stop-Hook hat Fehler → Korrektur-Versuch |
| `token_budget_continuation` | Token-Budget erlaubt weitere Iteration |

### Terminal-Gründe (Exits)

| Reason | Bedeutung |
| --- | --- |
| `completed` | Modell hat Antwort ohne Tool-Use beendet |
| `aborted_streaming` | Ctrl+C während API-Streaming |
| `aborted_tools` | Ctrl+C während Tool-Ausführung |
| `hook_stopped` | Hook hat Fortsetzung verhindert |
| `max_turns` | `--max-turns` erreicht |
| `blocking_limit` | Token-Fenster voll (Auto-Compact deaktiviert) |
| `prompt_too_long` | Recovery fehlgeschlagen |
| `image_error` | Bild zu groß / nicht konvertierbar |
| `model_error` | API-Fehler (Auth, Rate-Limit, ...) |
| `stop_hook_prevented` | Stop-Hook hat explizit abgebrochen |

---

## 4.4 Tool-Ausführung — Die 5-Phasen-Pipeline

Jeder einzelne Tool-Aufruf durchläuft in `toolExecution.ts` (~1.746 Zeilen) eine strikte Pipeline.

### Phasen eines Tool-Aufrufs

```text
runToolUse(toolUseBlock, assistantMessage, canUseTool, context)
     │
     ├─ 1. Tool-Lookup
     │     findToolByName(tools, name)
     │     Falls nicht gefunden: Alias-Fallback (deprecated Names)
     │     Falls immer noch nicht: → "No such tool" Error
     │
     ├─ 2. Abbruch-Prüfung
     │     abortController.signal.aborted? → Cancel-Message
     │
     └─ streamedCheckPermissionsAndCallTool()
          │
          ├─ 3. Input-Validierung
          │     ├─ Zod-Schema: tool.inputSchema.safeParse(input)
          │     │   Fehlschlag → InputValidationError
          │     │   (+ Schema-Not-Sent-Hint bei deferred Tools)
          │     │
          │     └─ tool.validateInput(parsedInput)
          │         Semantische Prüfung (z.B. Pfad existiert?)
          │
          ├─ 4. Permission-Pipeline
          │     │
          │     ├─ a) PreToolUse-Hooks
          │     │     runPreToolUseHooks() → kann:
          │     │       ├─ Input modifizieren (hookUpdatedInput)
          │     │       ├─ Permission-Entscheidung treffen (hookPermissionResult)
          │     │       ├─ Fortsetzung verhindern (preventContinuation)
          │     │       └─ Ausführung stoppen (stop)
          │     │
          │     ├─ b) Bash-Spekulative Klassifizierung
          │     │     startSpeculativeClassifierCheck()
          │     │     (nur Bash-Tool, läuft parallel zu Hooks)
          │     │
          │     ├─ c) canUseTool(tool, input)
          │     │     ┌────────────────────────────────────────────┐
          │     │     │  5 Stufen (absteigend nach Priorität):    │
          │     │     │                                            │
          │     │     │  1. Hook-Entscheidung (aus PreToolUse)     │
          │     │     │  2. Policy-Settings (Unternehmens-Regeln)  │
          │     │     │  3. Permission-Rules (User-Regeln)         │
          │     │     │     ├─ --allowedTools / --disallowedTools  │
          │     │     │     ├─ settings.json allow/deny            │
          │     │     │     └─ Session-scoped Entscheidungen       │
          │     │     │  4. Permission-Mode                        │
          │     │     │     ├─ bypassPermissions → allow all       │
          │     │     │     ├─ plan  → read: allow, write: ask     │
          │     │     │     ├─ auto  → Classifier entscheidet      │
          │     │     │     └─ default → ask                       │
          │     │     │  5. User-Prompt (interaktiv)               │
          │     │     │     oder permissionPromptTool (SDK)        │
          │     │     └────────────────────────────────────────────┘
          │     │     Ergebnis: allow / deny / (updatedInput)
          │     │
          │     └─ deny? → executePermissionDeniedHooks()
          │                → tool_result mit Fehlermeldung
          │
          ├─ 5. Ausführung
          │     ├─ startToolSpan() (OTel Tracing)
          │     ├─ startSessionActivity()
          │     ├─ tool.call(parsedInput, context)
          │     │     → Generator yield: ToolProgress-Events
          │     │     → return: ToolResult (string | ContentBlock[])
          │     ├─ stopSessionActivity()
          │     └─ endToolSpan()
          │
          └─ 6. Nachbereitung
                ├─ PostToolUse-Hooks
                │     runPostToolUseHooks() → kann:
                │       ├─ Ausführungsdaten loggen
                │       ├─ Zusätzliche Attachments liefern
                │       └─ Fortsetzung verhindern (shouldPreventContinuation)
                │
                ├─ Context-Modifier anwenden
                │     toolResult.contextModifier?.(context)
                │     (z.B. readFileState aktualisieren)
                │
                └─ tool_result-Message erzeugen
                      createUserMessage({content: [{type: 'tool_result', ...}]})
```

### Parallele vs. Serielle Ausführung

`toolOrchestration.ts` partitioniert die Tool-Aufrufe einer Antwort:

```text
Modell-Antwort enthält:  [FileRead, FileRead, Grep, FileEdit, Bash]
                          ──────────────────   ─────────────────────
                          isConcurrencySafe    nicht concurrencySafe

Batch 1 (parallel):   FileRead ─┐
                      FileRead ─┤── max 10 gleichzeitig
                      Grep     ─┘

Batch 2 (seriell):    FileEdit
Batch 3 (seriell):    Bash
```

Die Entscheidung basiert auf `tool.isConcurrencySafe(input)`:

| Tool | Parallel? | Begründung |
| --- | --- | --- |
| `FileRead`, `Grep`, `Glob` | ✓ | Reine Lese-Operationen |
| `WebFetch`, `WebSearch` | ✓ | Kein lokaler Zustand betroffen |
| `FileEdit`, `FileWrite` | ✗ | Mutieren Dateisystem |
| `Bash` | ✗ | Seiteneffekte unbekannt |
| `Agent` | ✗ | Eigener Kontext, potentielle Mutation |
| MCP-Tools | ✗ (default) | Server-Zustand unbekannt |

### StreamingToolExecutor — Parallelität mit dem API-Stream

Der `StreamingToolExecutor` startet Tools **noch während das Modell streamt**:

```text
Zeit ────────────────────────────────────────────►

API-Stream:  ████ text ████ tool_use(FileRead) ████ tool_use(Bash) ████ end_turn
                              │                        │
StreamingTE:                  ▼                        ▼
                         FileRead startet          Bash startet
                         FileRead fertig ──►       Bash läuft ...
                                                   Bash fertig ──►
                                                                     ▼
                                                            getRemainingResults()
```

Vorteile:

- Read-only Tools (FileRead, Grep) liefern Ergebnisse oft schon, bevor das Streaming endet
- Bei Abbruch (`Ctrl+C`) erzeugt der Executor synthetische `tool_result`-Blöcke für laufende Tools

---

## 4.5 Model-Fallback und Recovery

Die Query-Schleife implementiert mehrere **Resilienz-Mechanismen**:

### Modell-Fallback

```text
callModel(model: "sonnet")
     │
     ├─ 200 OK → normaler Ablauf
     │
     └─ 529 Overloaded → FallbackTriggeredError
          │
          ├─ Tombstones für bisherige Assistant-Messages
          ├─ currentModel = fallbackModel
          ├─ Thinking-Signaturen entfernen (modell-gebunden)
          ├─ yield SystemMessage("Switched to opus due to high demand...")
          └─ attemptWithFallback = true → retry
```

### Max-Output-Tokens Recovery

```text
Modell-Antwort: stop_reason = 'max_output_tokens'
     │
     ├─ Schritt 1: Escalation (8k → 64k)
     │   maxOutputTokensOverride = ESCALATED_MAX_TOKENS
     │   → Retry DESSELBEN Requests (kein meta-message)
     │
     ├─ Schritt 2: Multi-Turn Recovery (bis 3×)
     │   Inject: "Output token limit hit. Resume directly..."
     │   → Modell setzt fort, ohne Zusammenfassung oder Entschuldigung
     │
     └─ Schritt 3: Recovery erschöpft
         → Fehler an User/SDK weiterreichen
```

### Prompt-Too-Long Recovery (Kaskade)

```text
API → 413 Prompt Too Long
     │
     ├─ 1. Context-Collapse-Drain
     │     recoverFromOverflow() → staged Collapses committen
     │     Erfolgreich? → continue (collapse_drain_retry)
     │
     ├─ 2. Reactive Compact
     │     tryReactiveCompact() → Konversation zusammenfassen
     │     Erfolgreich? → continue (reactive_compact_retry)
     │     Nur 1× pro Turn (hasAttemptedReactiveCompact-Guard)
     │
     └─ 3. Aufgeben
           → Fehler an User/SDK weiterreichen
           → executeStopFailureHooks()
```

---

## 4.6 Sub-Agent-Spawning (AgentTool)

Das `AgentTool` startet eigenständige **Sub-Agenten**, die einen vollständigen `query()`-Zyklus in einem isolierten Kontext durchlaufen.

### Foreground-Agent-Sequenz

```text
Hauptagent (query-loop, Turn N)
     │
     ├─ Modell: tool_use(Agent, {name:"Explore", prompt:"Analysiere..."})
     │
     └─ AgentTool.call()
          │
          ├─ resolveAgentTools()
          │     Ausgangs-Tools filtern nach Agent-Definition
          │     (allowedTools, disallowedTools)
          │
          ├─ Neuen Kontext erstellen:
          │     ├─ Eigener AbortController
          │     ├─ Geklonte readFileState
          │     ├─ Eigene agentId (UUID)
          │     ├─ Eingeschränktes Tool-Set
          │     └─ Eigener System-Prompt (Agent-Beschreibung)
          │
          ├─ runAgent() → query(subParams)
          │     │
          │     │  ┌─ Eigene query-loop ──────────────────────┐
          │     │  │  callModel() → Tools → callModel() → ... │
          │     │  │  (gleiche Logik wie Hauptagent)           │
          │     │  └──────────────────────────────────────────┘
          │     │
          │     └─ Ergebnis: Terminal {reason, ...}
          │
          ├─ Sidechain-Transcript persistieren
          │     (eigene JSONL-Datei pro Agent-Run)
          │
          └─ tool_result: Zusammenfassung des Agent-Ergebnisses
               → zurück an Hauptagent → nächste Iteration
```

### Background-Agent

```text
AgentTool.call(run_in_background: true)
     │
     ├─ Promise starten (nicht blockierend)
     │     └─ runAgent() → query() im Hintergrund
     │
     ├─ tool_result: "Task gestartet (ID: xxx)"
     │     → Hauptagent arbeitet sofort weiter
     │
     └─ Bei Completion:
           addNotification() → Message-Queue
           → nächster Sleep/Turn-Grenze: Attachment injiziert
```

### Coordinator-Modus (Multi-Agent-Orchestrierung)

```text
User: "Implementiere Feature X"
     │
     ▼
Coordinator (eingeschränkte Tools: Agent, TaskStop, SendMessage)
     │
     ├─ Analysiert Aufgabe, zerlegt in Teilaufgaben
     │
     ├─ Agent("Worker-A", "Implementiere Backend-API")  ──►  Worker A
     │                                                         ├─ Bash, FileRead,
     │                                                         │  FileEdit, FileWrite
     │                                                         └─ Ergebnis
     │
     ├─ Agent("Worker-B", "Schreibe Tests")             ──►  Worker B
     │                                                         ├─ Bash, FileRead,
     │                                                         │  FileEdit, FileWrite
     │                                                         └─ Ergebnis
     │
     ├─ Ergebnisse zusammenführen
     │
     └─ Antwort an User
```

---

## 4.7 Session-Resume und Compact

### Session-Resume (`--resume` / `--continue`)

```text
claude --resume <session-id>
     │
     ├─ Session-Datei laden
     │     ~/.claude/projects/<hash>/sessions/<id>.jsonl
     │
     ├─ Messages deserialisieren
     │     ├─ Thinking-Blöcke entfernen (modell-gebunden)
     │     ├─ Content-Replacements anwenden (Tool-Result-Budget)
     │     └─ Snip-Boundaries respektieren
     │
     ├─ Optional: --fork-session
     │     Neue Session-ID, aber gleiche Nachrichtenhistorie
     │
     ├─ Optional: --resume-session-at <message-id>
     │     Nur Nachrichten bis einschließlich des angegebenen Turns
     │
     └─ QueryEngine initialisieren mit restored Messages
         → REPL-Prompt, User kann weiter interagieren
```

### Auto-Compact (Proaktiv)

```text
Token-Nutzung nähert sich Context-Window-Grenze
     │
     ├─ calculateTokenWarningState()
     │     >80% → Warning-Level erreicht
     │
     ├─ autocompact() aufgerufen (innerhalb queryLoop)
     │     │
     │     ├─ Fork: Komprimierungs-Agent spawnen
     │     │     query(querySource='compact', ...)
     │     │     System-Prompt: "Fasse zusammen, behalte alle Details..."
     │     │
     │     ├─ summaryMessages erzeugen (1-3 Messages)
     │     │
     │     └─ Ergebnis:
     │           ├─ preCompactTokenCount: 180.000
     │           ├─ postCompactTokenCount: 45.000
     │           └─ summaryMessages + Attachments + hookResults
     │
     └─ buildPostCompactMessages()
           messagesForQuery = postCompactMessages
           → continue (nächste Iteration mit komprimiertem Kontext)
```

---

## 4.8 Hooks im Ablauf

Hooks sind **benutzerdefinierte Shell-Befehle**, die an verschiedenen Stellen im Ablauf ausgeführt werden.

### Hook-Trigger-Punkte

```text
Session-Start
     │
     ├─ SessionStart:startup    (einmalig pro Session)
     ├─ Setup-Hooks             (--init / --maintenance)
     │
     ▼
queryLoop() Iteration N
     │
     ├─────── PreToolUse        (vor jedem Tool-Aufruf)
     │        │
     │        └─ Kann: Input ändern, Permission entscheiden, Abbruch
     │
     ├─────── PostToolUse       (nach jedem Tool-Aufruf)
     │        │
     │        └─ Kann: Loggen, Attachments, Fortsetzung verhindern
     │
     ├─────── PostSampling      (nach jeder Modell-Antwort)
     │
     ├─────── Stop              (wenn Modell fertig ist)
     │        │
     │        ├─ Erfolgreich → Turn endet
     │        └─ blockingErrors → Modell bekommt Fehler, soll korrigieren
     │
     └─────── StopFailure       (bei API-Fehler statt normalem Stop)

Session-Ende
     │
     └─ SessionEnd              (bei Beendigung)
```

### Hook-Datenfluss (PreToolUse-Beispiel)

```text
Hook-Definition in settings.json:
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": ["./my-linter.sh"]
    }]
  }

Ablauf:
  Bash(command: "npm test")
       │
       ├─ matcher passt? ("Bash" ✓)
       │
       ├─ ./my-linter.sh aufrufen
       │     stdin: {tool_name, tool_input, session_id, ...}
       │     stdout: JSON-Ergebnis
       │
       └─ Ergebnis interpretieren:
            ├─ {decision: "allow"} → weiter wie normal
            ├─ {decision: "deny", reason: "..."}
            │     → tool_result mit Fehler
            ├─ {updatedInput: {command: "npm test -- --ci"}}
            │     → Input wird überschrieben
            └─ {stopReason: "..."}
                  → Tool wird nicht ausgeführt
```

---

## 4.9 Zusammenfassung: Timing eines typischen Turns

```text
Zeit ─────────────────────────────────────────────────────────►
                                                               Total: ~5-30s

User-Eingabe
  │  submitMessage()
  │  ├── processUserInput()                         ~1-5 ms
  │  └── fetchSystemPromptParts()                   ~5-20 ms
  │
  │  queryLoop() Iteration
  │  ├── Komprimierung (snip/micro/auto)            ~0-500 ms
  │  ├── callModel() Streaming                      ~2-15 s
  │  │   ├── First token (Time-to-First-Token)      ~0.5-3 s
  │  │   └── StreamingToolExecutor parallel          overlapped
  │  │
  │  ├── Tool-Ausführung                            ~100 ms - 30 s
  │  │   ├── Permission-Check                       ~1-50 ms
  │  │   ├── PreToolUse-Hooks                       ~0-500 ms
  │  │   ├── tool.call()                            ~50 ms - 30 s
  │  │   └── PostToolUse-Hooks                      ~0-500 ms
  │  │
  │  ├── Attachments                                ~5-50 ms
  │  └── state = {...}; continue
  │
  │  ... (weitere Iterationen) ...
  │
  │  queryLoop() letzte Iteration
  │  ├── callModel() → end_turn (kein tool_use)
  │  └── Stop-Hooks                                 ~0-500 ms
  │
  ▼  return {reason: 'completed'}
Antwort sichtbar
```

---

*Nächstes Kapitel: [5. Erweiterungsmöglichkeiten](5_Erweiterungsmöglichkeiten.md)*
