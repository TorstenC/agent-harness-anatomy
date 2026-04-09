# 2. Architekturübersicht

## 2.1 Schichtenmodell

Claude Code ist in klar abgegrenzte Schichten organisiert. Jede Schicht hat genau definierte Verantwortlichkeiten und kommuniziert nur mit den direkt angrenzenden Schichten:

```text
┌─────────────────────────────────────────────────────────────┐
│                    1. CLI & Entrypoint                      │
│  main.tsx · Commander.js · Argument-Parsing · Migrations    │
├─────────────────────────────────────────────────────────────┤
│                    2. Initialisierung                       │
│  entrypoints/init.ts · setup.ts · bootstrap/state.ts        │
│  Config · Auth · Telemetrie · Proxy · mTLS · GrowthBook     │
├─────────────────────────────────────────────────────────────┤
│                    3. Präsentation (UI)                     │
│  React/Ink REPL · screens/REPL.ts · components/App.ts       │
│  Hooks · Keybindings · Vim-Mode · Voice-Input               │
├─────────────────────────────────────────────────────────────┤
│                    4. Query-Steuerung                       │
│  QueryEngine.ts · query.ts · context.ts                     │
│  Auto-Compact · Snip · Microcompact · Context Collapse      │
├─────────────────────────────────────────────────────────────┤
│                    5. API-Integration                       │
│  services/api/claude.ts · Anthropic SDK · Streaming         │
│  Retry · Fallback · Token-Budget · Task-Budget              │
├─────────────────────────────────────────────────────────────┤
│                    6. Tool-System                           │
│  Tool.ts · tools.ts · services/tools/toolExecution.ts       │
│  toolOrchestration.ts · StreamingToolExecutor.ts            │
├─────────────────────────────────────────────────────────────┤
│                    7. Berechtigungen                        │
│  hooks/toolPermission/ · utils/permissions/                 │
│  Classifier · Hooks · User-Prompts · Auto-Mode              │
├─────────────────────────────────────────────────────────────┤
│                    8. Externe Dienste                       │
│  MCP-Server · LSP · Bridge (IDE) · Plugins · Skills         │
│  OAuth · Analytics · Remote Sessions · Cron-Triggers        │
├─────────────────────────────────────────────────────────────┤
│                    9. Infrastruktur                         │
│  bootstrap/state.ts · state/store.ts · utils/               │
│  Logging · Telemetrie · File-I/O · Git · Shell              │
└─────────────────────────────────────────────────────────────┘
```

## 2.2 Modulübersicht

Die folgende Tabelle zeigt die Top-Level-Verzeichnisse und ihre Verantwortlichkeiten:

| Verzeichnis | Schicht | Verantwortlichkeit |
| --- | --- | --- |
| `main.tsx` | Entrypoint | CLI-Parsing, Commander.js-Programm, `run()` → `setup()` → `launchRepl()` |
| `entrypoints/` | Initialisierung | `init()` — Config laden, Proxy/mTLS, Telemetrie, Scratchpad, Cleanup |
| `setup.ts` | Initialisierung | Session-Setup, CWD, Worktree, Hooks-Snapshot, UDS-Messaging |
| `bootstrap/state.ts` | State | Globaler Singleton-State: Session-ID, Kosten, Modell, Telemetrie-Counter |
| `state/` | State | `AppStateStore` (React-ähnlicher Store) + `createStore()` (Pub/Sub) |
| `screens/` | UI | REPL-Bildschirm, Doctor, Resume-Picker |
| `components/` | UI | ~140 Ink/React-Komponenten (Permissions-Dialog, Spinner, Markdown-Renderer) |
| `hooks/` | UI | ~80 React-Hooks (`useCanUseTool`, `useTextInput`, `useVimInput`, …) |
| `QueryEngine.ts` | Query | Konversations-Lifecycle, Turn-Management, System-Prompt-Aufbau |
| `query.ts` | Query | Kern-Schleife: Streaming → Tool-Erkennung → Ausführung → Retry |
| `context.ts` | Query | System-/User-Kontext (Git-Status, CLAUDE.md, Datum) |
| `services/api/` | API | `claude.ts` — Anthropic SDK-Wrapper, Streaming, Betas, Caching |
| `services/compact/` | Query | Auto-Compact, Snip-Compact, Reactive-Compact, Microcompact |
| `services/tools/` | Tools | `toolExecution.ts`, `toolOrchestration.ts`, `StreamingToolExecutor.ts` |
| `tools/` | Tools | ~40 Tool-Implementierungen (je eigenes Verzeichnis) |
| `Tool.ts` | Tools | Typdefinitionen: `Tool`, `ToolUseContext`, `ToolPermissionContext` |
| `tools.ts` | Tools | Tool-Registry: `getAllBaseTools()`, `getTools()` |
| `commands.ts` | Commands | Command-Registry: `getCommands()`, ~50 Slash-Commands |
| `commands/` | Commands | Command-Implementierungen (je eigenes Verzeichnis) |
| `services/mcp/` | Extern | MCP-Client, Server-Verbindungen, Tool/Resource-Discovery |
| `services/lsp/` | Extern | LSP-Server-Manager |
| `bridge/` | Extern | IDE-Integration (VS Code, JetBrains): Bidirektionale Kommunikation |
| `plugins/` | Extern | Plugin-System (Bundled + Third-Party) |
| `skills/` | Extern | Skill-System (Bundled + Custom-Skills aus `.claude/skills/`) |
| `coordinator/` | Multi-Agent | Coordinator-Mode: Orchestrierung mehrerer Worker-Agenten |
| `utils/permissions/` | Berechtigungen | Permission-Logik: Rules, Modes, Classifier, Filesystem-Checks |
| `utils/` | Infrastruktur | ~200 Utility-Module (Git, Shell, Auth, Crypto, Format, …) |
| `types/` | Infrastruktur | Zentrale TypeScript-Typen (Message, Permission, Plugin, …) |
| `schemas/` | Infrastruktur | Zod-Schemas für Config-Validierung |
| `migrations/` | Infrastruktur | Config-Migrationen (Modell-Renames, Setting-Umzüge) |

### Paketstruktur und Bundling

Das veröffentlichte npm-Paket (`@anthropic-ai/claude-code`) enthält **keine**
Ordnerstruktur wie oben beschrieben. Bun bündelt den gesamten TypeScript-Quellcode
in eine einzige Datei `cli.js` (~16.700 Zeilen minifizierter Code, Shebang
`#!/usr/bin/env node`). Die `package.json` deklariert:

```json
"bin": { "claude": "cli.js" },
"dependencies": {},
"optionalDependencies": {
  "@img/sharp-darwin-arm64": "^0.34.2",
  "@img/sharp-linux-x64":   "^0.34.2"
}
```

Bemerkenswert: Es gibt **keine** Runtime-Dependencies — alle Bibliotheken
(Anthropic SDK, React, Ink, Commander.js, Zod, lodash, …) sind inline gebundelt.
Nur die plattformspezifischen Sharp-Native-Binaries werden als
`optionalDependencies` nachgeladen (9 Plattformen).

Zusätzlich enthält das Paket gebundelte Binärdateien im `vendor/`-Verzeichnis:

| Binary | Zweck |
| --- | --- |
| `vendor/ripgrep/` | Plattform-Binaries von ripgrep (`rg`) für `GrepTool` |
| `vendor/audio-capture/` | NAPI-Bindings für Mikrofon-Aufnahme (Voice Mode) |

Die `source/vendor/`-Quellen zeigen vier weitere NAPI-Module (Image-Processor,
Keyboard-Modifier-Detection, URL-Handler), die alle dasselbe Muster verwenden:
Lazy-Loading mit gecachtem Modul-Handle und plattformspezifischen Guards.

## 2.3 Startup-Ablauf

Der Startup-Prozess folgt einem strikten, mehrstufigen Ablauf. Jede Phase baut auf der vorherigen auf:

```text
Process Start
│
├─ 1. Side-Effects (vor allen Imports)
│     ├─ profileCheckpoint('main_tsx_entry')
│     ├─ startMdmRawRead()          ← MDM-Subprozesse (parallel)
│     └─ startKeychainPrefetch()    ← macOS Keychain lesen (parallel)
│
├─ 2. Import-Phase (~135ms)
│     └─ ~200 statische Imports + Feature-Flag-bedingte require()
│
├─ 3. main() → Sicherheitschecks
│     ├─ NoDefaultCurrentDirectoryInExePath (Windows PATH-Schutz)
│     ├─ initializeWarningHandler()
│     ├─ Deep-Link / SSH / Assistant URL-Rewriting
│     ├─ isInteractive bestimmen (TTY / -p / --init-only / --sdk-url)
│     ├─ clientType bestimmen (cli / sdk-* / remote / github-action)
│     └─ eagerLoadSettings()
│
├─ 4. run() → Commander.js Setup
│     ├─ program.hook('preAction')
│     │     ├─ ensureMdmSettingsLoaded()       ← MDM await
│     │     ├─ ensureKeychainPrefetchCompleted() ← Keychain await
│     │     ├─ init()                           ← Kern-Initialisierung
│     │     │     ├─ enableConfigs()
│     │     │     ├─ applySafeConfigEnvironmentVariables()
│     │     │     ├─ applyExtraCACertsFromConfig()
│     │     │     ├─ setupGracefulShutdown()
│     │     │     ├─ configureGlobalMTLS()
│     │     │     ├─ configureGlobalAgents() (Proxy)
│     │     │     ├─ preconnectAnthropicApi()
│     │     │     └─ ensureScratchpadDir()
│     │     ├─ initSinks() (Telemetrie)
│     │     ├─ runMigrations()
│     │     └─ loadRemoteManagedSettings()
│     │
│     └─ program.action() → Hauptaktion
│           ├─ setup(cwd, permissionMode, ...)
│           │     ├─ setCwd()
│           │     ├─ captureHooksConfigSnapshot()
│           │     ├─ initializeFileChangedWatcher()
│           │     └─ Worktree / Tmux (optional)
│           │
│           ├─ showSetupScreens() (Trust-Dialog, Auth, Onboarding)
│           ├─ Tools + Commands laden
│           ├─ MCP-Server verbinden
│           ├─ Plugins + Skills initialisieren
│           │
│           └─ launchRepl(root, appProps, replProps, renderAndRun)
│                 ├─ App (React/Ink-Wrapper)
│                 └─ REPL (Interaktive Hauptschleife)
```

### Parallele Initialisierung

Ein zentrales Entwurfsmuster ist die **parallele Prefetch-Strategie**: Langsame I/O-Operationen (MDM-Subprozesse, Keychain-Reads, API-Preconnect, GrowthBook-Fetch) werden als Fire-and-Forget-Promises gestartet und erst im `preAction`-Hook awaited. So überlappen sich ~200ms Netzwerk-Latenz mit den ~135ms Import-Phase.

## 2.4 Query-Lifecycle (Agentic Loop)

Die Agentic Loop ist das Herzstück des Agent Harness. Sie implementiert den Zyklus *Nachricht senden → Antwort streamen → Tools erkennen → Tools ausführen → Ergebnisse zurückspeisen*:

```text
                    ┌──────────────────┐
                    │  User-Nachricht  │
                    └────────┬─────────┘
                             │
                             ▼
              ┌──────────────────────────┐
              │  Kontext-Aufbereitung    │
              │  ├─ Snip-Compact         │
              │  ├─ Microcompact         │
              │  ├─ Context Collapse     │
              │  ├─ Auto-Compact         │
              │  └─ Token-Budget-Check   │
              └──────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │  API-Aufruf (Streaming)  │
              │  services/api/claude.ts  │
              │  ├─ System-Prompt        │
              │  ├─ User-Context         │
              │  ├─ Tool-Definitionen    │
              │  └─ Thinking-Config      │
              └──────────┬───────────────┘
                         │
                    ┌────┴────┐
                    │ Stream  │
                    │ Events  │
                    └────┬────┘
                         │
              ┌──────────┴───────────────┐
              │  Tool-Use erkannt?       │
              │                          │
              │  NEIN ──► Antwort an UI  │──► FERTIG
              │                          │
              │  JA ──┐                  │
              └───────┼──────────────────┘
                      │
                      ▼
              ┌──────────────────────────┐
              │  Tool-Orchestrierung     │
              │  ┌────────────────────┐  │
              │  │ Read-Only Tools    │  │──► Parallel (bis 10)
              │  │ (Grep, Glob, Read) │  │
              │  └────────────────────┘  │
              │  ┌────────────────────┐  │
              │  │ Mutable Tools      │  │──► Seriell
              │  │ (Bash, Write, Edit)│  │
              │  └────────────────────┘  │
              └──────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │  Berechtigungsprüfung    │
              │  ├─ Config-Rules         │
              │  ├─ Classifier (Auto)    │
              │  ├─ Hooks                │
              │  └─ User-Prompt (UI)     │
              └──────────┬───────────────┘
                         │
                    ┌────┴────┐
                    │ Erlaubt │
                    │   ?     │
                    └────┬────┘
                    JA   │   NEIN
                    │    │    │
                    ▼    │    ▼
              Ausführen  │  Reject-Message
                    │    │    │
                    └────┴────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │  Tool-Ergebnis als       │
              │  tool_result anhängen    │
              └──────────┬───────────────┘
                         │
                         └──────► zurück zum API-Aufruf
                                  (nächste Iteration)
```

### Kontext-Management-Kaskade

Bevor jede API-Anfrage gesendet wird, durchlaufen die Nachrichten eine **mehrstufige Kompressions-Pipeline**. Die Reihenfolge ist entscheidend, da jede Stufe die Eingabe für die nächste reduziert:

| Stufe | Modul | Strategie |
| --- | --- | --- |
| 1. Tool-Result-Budget | `utils/toolResultStorage.ts` | Große Tool-Ergebnisse auf Festplatte auslagern, Referenz einsetzen |
| 2. Snip-Compact | `services/compact/snipCompact.ts` | Alte Nachrichten abschneiden (FIFO-Truncation) |
| 3. Microcompact | `services/compact/apiMicrocompact.ts` | Cache-Edit-Optimierung für API-Prompt-Caching |
| 4. Context Collapse | `services/contextCollapse/` | Zusammenklappen vergangener Tool-Interaktionen zu Summaries |
| 5. Auto-Compact | `services/compact/autoCompact.ts` | LLM-basierte Zusammenfassung bei >80% Kontextfenster-Auslastung |
| 6. Reactive Compact | `services/compact/reactiveCompact.ts` | Notfall-Komprimierung bei `prompt_too_long`-Fehler |

### Tool-Ausführungsmodell

Die Tool-Orchestrierung (`services/tools/toolOrchestration.ts`) partitioniert Tool-Aufrufe automatisch:

- **Concurrency-safe Tools** (Read-Only: `FileReadTool`, `GrepTool`, `GlobTool`) → parallele Ausführung (max. 10 gleichzeitig, konfigurierbar via `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`)
- **Mutable Tools** (`BashTool`, `FileWriteTool`, `FileEditTool`) → serielle Ausführung

Der `StreamingToolExecutor` kann Tools bereits **während des Streamings** starten, sobald der JSON-Input eines `tool_use`-Blocks komplett ist — noch bevor die gesamte Assistenten-Antwort fertig gestreamt wurde.

## 2.5 State-Management

Claude Code verwendet ein **zweigeteiltes State-Modell**:

### Globaler Singleton-State (`bootstrap/state.ts`)

Ein prozessweites Modul mit veränderlichen Variablen. Es enthält:

- Session-ID, CWD, Projekt-Root
- Kosten-Tracking (Token, USD, API-Duration)
- Modell-Konfiguration (Override, Initial, Strings)
- Telemetrie-Counter (OpenTelemetry Meter)
- Feature-Flags und Session-spezifische Einstellungen
- Agent-Farben, Cron-Tasks, Plugin-State

Zugriff erfolgt über exportierte Getter/Setter-Funktionen (z.B. `getSessionId()`, `getTotalCostUSD()`).

### Reaktiver App-State (`state/store.ts` + `state/AppStateStore.ts`)

Ein leichtgewichtiger Pub/Sub-Store nach dem Muster:

```text
createStore(initialState, onChange?)
  ├─ getState()     → aktueller Snapshot (immutable)
  ├─ setState(fn)   → Updater-Funktion mit Prev-State
  └─ subscribe(fn)  → Listener für Änderungen
```

Der `AppState` enthält UI-relevanten Zustand:

- Tool-Permission-Context (aktueller Modus, Allow/Deny-Regeln)
- MCP-Verbindungen und -Tools
- Plugin-/Skill-Zustand
- Thinking-Config, Effort-Level, Fast-Mode
- Spekulations-State (Idle-Prediction)
- Nachrichten-Queue, Task-Panels, Companion-Sprite

React-Komponenten abonnieren den Store über Hooks, die bei State-Änderungen Re-Renders auslösen.

## 2.6 Berechtigungsarchitektur

Das Berechtigungssystem ist **mehrstufig** aufgebaut und entscheidet für jeden Tool-Aufruf, ob er erlaubt, verweigert oder dem Nutzer vorgelegt wird:

```text
Tool-Aufruf eingehend
│
├─ 1. Config-Rules prüfen (alwaysAllow / alwaysDeny / alwaysAsk)
│     Quellen: User-Settings, Projekt-Settings, CLI-Flags
│     → allow / deny / weiter
│
├─ 2. Permission-Mode prüfen
│     ├─ "bypass"  → sofort erlauben (--dangerously-skip-permissions)
│     ├─ "plan"    → nur Read-Only erlaubt, sonst deny
│     ├─ "auto"    → Classifier entscheidet (Schritt 3)
│     └─ "default" → User-Prompt (Schritt 4)
│
├─ 3. Auto-Mode Classifier (wenn aktiviert)
│     ├─ Transcript-basierte Risikoanalyse
│     ├─ Bash-Command-Classifier (Whitelist/Blacklist)
│     └─ → approve / weiter zu User-Prompt
│
├─ 4. Pre-Tool-Use Hooks
│     ├─ Plugin-Hooks (hookEvents)
│     └─ → approve / deny / weiter
│
└─ 5. User-Prompt (interaktiv)
      ├─ PermissionRequest-Komponente
      ├─ "Allow once" / "Allow always" / "Deny"
      └─ Bridge-Permission-Callbacks (IDE-Modus)
```

### Permission-Modes

| Modus | Beschreibung |
| --- | --- |
| `default` | Standard: Alle mutierenden Tools erfordern User-Bestätigung |
| `plan` | Nur lesende Tools erlaubt; schreibende werden abgelehnt |
| `auto` | Classifier-basierte automatische Genehmigung mit Fallback auf User-Prompt |
| `bypass` | Alle Berechtigungsprüfungen umgangen (nur für Sandboxes) |

## 2.7 Externe Integrationen

### MCP (Model Context Protocol)

```text
services/mcp/client.ts
│
├─ MCP-Server-Discovery
│     ├─ .claude/mcp.json (Projekt)
│     ├─ ~/.claude/mcp.json (User)
│     ├─ --mcp-config (CLI)
│     └─ Claude.ai MCP-Configs (Enterprise)
│
├─ Transport-Layer
│     ├─ StdioClientTransport  (lokale Prozesse)
│     ├─ SSEClientTransport    (HTTP SSE)
│     ├─ StreamableHTTPClientTransport
│     └─ WebSocketTransport    (WS/WSS)
│
├─ Tool-/Command-/Resource-Discovery
│     ├─ MCP-Tools → als MCPTool in Tool-Registry registriert
│     ├─ MCP-Prompts → als Slash-Commands verfügbar
│     └─ MCP-Resources → über ListMcpResourcesTool / ReadMcpResourceTool
│
└─ Auth: OAuth 2.0 (PKCE) + XAA IdP Login
```

### IDE-Bridge (`bridge/`)

Die Bridge ermöglicht bidirektionale Kommunikation zwischen Claude Code und IDEs:

```text
┌─────────────┐           ┌──────────────┐
│  VS Code /  │◄── JWT ──►│  bridgeMain  │
│  JetBrains  │ WebSocket │     .ts      │
└─────────────┘           ├──────────────┤
                          │ sessionRunner│──► Spawnt Claude-Code-Prozesse
                          │ bridgeApi    │──► REST-API zum Bridge-Server
                          │ bridgeUI     │──► Status-Logging
                          │ workSecret   │──► Session-Verschlüsselung
                          │ jwtUtils     │──► Token-Refresh
                          │ pollConfig   │──► Long-Polling für Tasks
                          └──────────────┘
```

### Plugin-System (`plugins/`)

Plugins erweitern Claude Code um zusätzliche Tools, Commands und Hooks. Sie werden aus konfigurierbaren Verzeichnissen geladen und können sowohl bundled (im Binary) als auch extern (NPM/lokaler Pfad) sein.

### Skill-System (`skills/`)

Skills sind wiederverwendbare Prompt-Templates, die über den `SkillTool` aufgerufen werden. Sie können aus `.claude/skills/`-Verzeichnissen geladen oder als Bundled Skills mitgeliefert werden.

## 2.8 Feature-Flag-Architektur

Claude Code nutzt **Build-Time Feature Flags** via `bun:bundle`'s `feature()`:

```typescript
// Dead Code Elimination: unreferenzierte Branches werden beim Build entfernt
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js')
  : null;
```

Dieses Muster wird konsistent für ~30 Feature-Gates verwendet:

| Flag | Modul | Funktion |
| --- | --- | --- |
| `KAIROS` | `assistant/` | Proaktiver Assistent-Modus |
| `COORDINATOR_MODE` | `coordinator/` | Multi-Agent-Orchestrierung |
| `BRIDGE_MODE` | `bridge/` | IDE-Integration |
| `VOICE_MODE` | `voice/` | Spracheingabe |
| `HISTORY_SNIP` | `services/compact/snipCompact` | Snip-basierte Kontext-Trunkierung |
| `CONTEXT_COLLAPSE` | `services/contextCollapse/` | Kontext-Zusammenklappung |
| `CACHED_MICROCOMPACT` | `services/compact/` | Cache-basierte Microcompaction |
| `TRANSCRIPT_CLASSIFIER` | `utils/permissions/` | Auto-Mode Transcript-Classifier |
| `AGENT_TRIGGERS` | `tools/ScheduleCronTool/` | Cron-basierte Agent-Trigger |
| `WEB_BROWSER_TOOL` | `tools/WebBrowserTool/` | Browser-Automatisierung |
| `WORKFLOW_SCRIPTS` | `tools/WorkflowTool/` | Workflow-Skripte |

Der Vorteil: Deaktivierte Features werden beim Bun-Build vollständig aus dem Bundle entfernt (*tree shaking*), sodass die Binärgröße und Import-Zeiten minimal bleiben.

## 2.9 Datenfluss-Gesamtbild

```text
┌─────────┐     ┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  User   │────►│  REPL    │────►│ QueryEngine  │────►│ Anthropic   │
│ (stdin) │     │ (React/  │     │  query.ts    │     │   API       │
│         │◄────│  Ink)    │◄────│              │◄────│ (Streaming) │
└─────────┘     └────┬─────┘     └──────┬───────┘     └─────────────┘
                     │                  │
                     │           ┌──────┴───────┐
                     │           │ Tool-System  │
                     │           │              │
                     │      ┌────┴────┐    ┌────┴────┐
                     │      │ Lokal   │    │ Extern  │
                     │      ├─────────┤    ├─────────┤
                     │      │Bash     │    │MCP-     │
                     │      │File R/W │    │Server   │
                     │      │Grep/Glob│    │LSP      │
                     │      │Agent    │    │Web      │
                     │      └─────────┘    └─────────┘
                     │
              ┌──────┴───────┐
              │   Bridge     │
              │  (optional)  │
              └──────┬───────┘
                     │
              ┌──────┴───────┐
              │  VS Code /   │
              │  JetBrains   │
              └──────────────┘
```

> **Weiter in Kapitel 3:** Die Hauptkomponenten (QueryEngine, Tool-System, Command-Registry, Services) werden im Detail beschrieben – mit Typdefinitionen, Schnittstellen und Codebeispielen.
