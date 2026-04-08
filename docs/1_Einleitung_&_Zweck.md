# 1. Einleitung & Zweck

## 1.1 Was ist Claude Code?

**Claude Code** ist Anthropics offizielles CLI-Tool (Command-Line Interface), das eine direkte Interaktion mit dem KI-Modell Claude über das Terminal ermöglicht. Es handelt sich um ein vollständiges **Agent Harness** – eine Laufzeitumgebung, die ein großes Sprachmodell (LLM) mit konkreten Werkzeugen (Tools) verbindet und so aus einem reinen Textgenerator einen handlungsfähigen Software-Engineering-Agenten macht.

Der Quellcode wurde am **31. März 2026** über eine versehentlich im npm-Registry veröffentlichte `.map`-Datei geleakt und umfasst:

| Kennzahl | Wert |
| --- | --- |
| **Sprache** | TypeScript (strict) |
| **Runtime** | Bun |
| **Terminal-UI** | React + Ink (React für CLI) |
| **Dateien** | ~1.900 |
| **Codezeilen** | 512.000+ |

## 1.2 Zweck dieser Dokumentation

Diese Dokumentation analysiert die interne Architektur und die Entwurfsentscheidungen von Claude Code anhand des geleakten Quellcodes. Sie dient als Referenz, um zu verstehen:

1. **Wie ein modernes Agent Harness aufgebaut ist** – vom CLI-Einstiegspunkt über den Query-Lifecycle bis zur Tool-Ausführung.
2. **Welche Architekturmuster** bei der Kopplung eines LLM an reale Werkzeuge (Dateisystem, Shell, Web, MCP-Server, LSP) angewandt werden.
3. **Wie Skalierungsprobleme gelöst werden** – Kontext-Kompression, Token-Budgets, Multi-Agent-Orchestrierung, Feature Flags und Lazy Loading.
4. **Welche Sicherheits- und Berechtigungskonzepte** zum Schutz des Nutzersystems implementiert sind.

## 1.3 Was ist ein Agent Harness?

Ein Agent Harness (auch *Agentic Framework* oder *Tool-Use Runtime*) ist die Infrastruktur, die ein LLM zu einem autonomen Agenten macht. Es verbindet drei Kernkomponenten:

```text
┌─────────────────────────────────────────────────────┐
│                  Agent Harness                      │
│                                                     │
│  ┌───────────┐   ┌──────────────┐   ┌───────────┐   │
│  │  Benutzer │◄─►│  Query-Loop  │◄─►│    LLM    │   │
│  │  (CLI/UI) │   │  (Steuerung) │   │ (Anthropic│   │
│  └───────────┘   └──────┬───────┘   │    API)   │   │
│                         │           └───────────┘   │
│                         ▼                           │
│              ┌─────────────────────┐                │
│              │     Tool-System     │                │
│              ├─────────────────────┤                │
│              │ BashTool            │                │
│              │ FileReadTool        │                │
│              │ FileWriteTool       │                │
│              │ FileEditTool        │                │
│              │ GrepTool / GlobTool │                │
│              │ WebFetchTool        │                │
│              │ WebSearchTool       │                │
│              │ AgentTool (Sub-     │                │
│              │   agenten)          │                │
│              │ MCPTool / LSPTool   │                │
│              │ SkillTool           │                │
│              │ TaskCreate/Update   │                │
│              │ ... (~40 Tools)     │                │
│              └─────────────────────┘                │
└─────────────────────────────────────────────────────┘
```

Im Fall von Claude Code bedeutet das konkret:

- **Benutzer-Schnittstelle:** Ein React/Ink-basiertes Terminal-UI mit Commander.js als CLI-Parser (`main.tsx`).
- **Query-Loop:** Die Kern-Schleife in `query.ts` und `QueryEngine.ts`, die Nutzereingaben an die Anthropic API sendet, Streaming-Antworten verarbeitet, Tool-Aufrufe erkennt, Tools ausführt und die Ergebnisse in die nächste Anfrage einspeist.
- **Tools:** ~40 registrierte Werkzeuge in `src/tools/`, von Shell-Ausführung (`BashTool`) über Dateizugriff (`FileReadTool`, `FileWriteTool`, `FileEditTool`) bis hin zu Web-Suche, MCP-Protokoll-Integration und Sub-Agenten-Erzeugung (`AgentTool`).

## 1.4 Kernfähigkeiten im Überblick

### Dateisystem-Operationen

Claude Code kann Dateien lesen, erstellen, bearbeiten und durchsuchen. Der `FileEditTool` verwendet dabei ein diffing-basiertes Verfahren (String-Replacement), um präzise Teiländerungen an bestehenden Dateien vorzunehmen.

### Shell-Ausführung

Der `BashTool` erlaubt die Ausführung beliebiger Shell-Befehle – gesteuert durch ein mehrstufiges Berechtigungssystem, das bei kritischen Operationen eine explizite Nutzerfreigabe erfordert.

### Code-Suche

`GrepTool` (basierend auf ripgrep) und `GlobTool` ermöglichen schnelle Codebase-weite Suche nach Inhalten und Dateimustern.

### Web-Zugriff

`WebFetchTool` ruft Inhalte von URLs ab, `WebSearchTool` führt Web-Suchen durch – beides erlaubt dem Agenten, externe Informationen in den Kontext einzubeziehen.

### Multi-Agent-Orchestrierung

Über `AgentTool` können Sub-Agenten gestartet werden. Das `coordinator/`-Modul steuert die Multi-Agent-Orchestrierung, während `TeamCreateTool` und `SendMessageTool` die Kommunikation zwischen Agenten ermöglichen.

### IDE-Integration

Das Bridge-System (`src/bridge/`) stellt eine bidirektionale Kommunikationsschicht zu VS Code und JetBrains bereit, sodass Claude Code sowohl eigenständig im Terminal als auch eingebettet in eine IDE arbeiten kann.

### Plugin- und Skill-System

Wiederverwendbare Workflows (`src/skills/`) und ein Plugin-System (`src/plugins/`) erlauben die Erweiterung der Funktionalität durch eigene oder von Dritten bereitgestellte Module.

### Slash-Commands

Über 50 Slash-Commands (`src/commands/`) bieten direkte Nutzerinteraktionen wie `/commit`, `/review`, `/compact`, `/mcp`, `/doctor`, `/vim` und viele mehr.

## 1.5 Technologiestack

| Kategorie | Technologie |
| --- | --- |
| Runtime | [Bun](https://bun.sh) |
| Sprache | TypeScript (strict mode) |
| Terminal-UI | React + [Ink](https://github.com/vadimdemedes/ink) |
| CLI-Parsing | [Commander.js](https://github.com/tj/commander.js) (extra-typings) |
| Schema-Validierung | Zod v4 |
| Code-Suche | ripgrep (via `GrepTool`) |
| Protokolle | MCP SDK (Model Context Protocol), LSP |
| API | Anthropic SDK |
| Telemetrie | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| Authentifizierung | OAuth 2.0, JWT, macOS Keychain |

## 1.6 Lesehinweise

Diese Dokumentation ist wie folgt aufgebaut:

| Kapitel | Inhalt |
| --- | --- |
| **1. Einleitung & Zweck** | *(dieses Kapitel)* – Überblick, Motivation, Technologiestack |
| **2. Architekturübersicht** | Gesamtstruktur, Modulabhängigkeiten, Datenflussdiagramme |
| **3. Hauptkomponenten** | Detailbeschreibungen der Kern-Module (QueryEngine, Tool-System, Commands, Services) |
| **4. Typische Abläufe** | Sequenzdiagramme für Startup, Query-Lifecycle, Tool-Ausführung, Agent-Spawning |
| **5. Erweiterungsmöglichkeiten** | Plugin-System, Skill-System, MCP-Integration, Custom Agents |
| **6. API-Referenz** | Wichtige Typen, Interfaces und Funktionen |

> **Hinweis:** Alle Pfadangaben beziehen sich auf das Verzeichnis `src/` des geleakten Quellcodes. Dateireferenzen wie `main.tsx` meinen `src/main.tsx`.
