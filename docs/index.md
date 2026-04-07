# Agent Harness – Anatomie eines KI-Software-Engineering-Agenten

> Wie macht man aus einem reinen LLM-Textgenerator einen handlungsfähigen Software-Engineering-Agenten?

Dieses Repository dokumentiert und analysiert die interne Architektur von **Claude Code** — Anthropics offiziellem CLI-Tool, das das KI-Modell Claude über eine Laufzeitumgebung (Agent Harness) mit realen Werkzeugen verbindet: Shell, Dateisystem, Web, IDE-Integration und mehr.

## Hintergrund

Am 31. März 2026 wurde der Quellcode von Claude Code durch eine versehentlich im npm-Registry veröffentlichte `.map`-Datei öffentlich. Der Code umfasst **~512.000 Zeilen TypeScript** und bietet einen seltenen, ungeschminkten Einblick in die Architektur eines produktionsreifen Agent Harness.

Der Originalcode ist in diesem Repository **nicht enthalten** (`src/` ist in `.gitignore` ausgeschlossen). Der Quellcode kann über den referenzierten Commit eingesehen werden:

> **Quelle:** [ultraworkers/claw-code @ `5a774a2`](https://github.com/ultraworkers/claw-code/tree/5a774a2b62d7949c1d94e0b726281554d7893cfd)

## Was ist ein Agent Harness?

```text
┌─────────────────────────────────────────────────────┐
│                  Agent Harness                      │
│                                                     │
│  ┌───────────┐   ┌──────────────┐   ┌───────────┐   │
│  │  Benutzer │◄─►│  Query-Loop  │◄─►│    LLM    │   │
│  │  (CLI/UI) │   │  (Steuerung) │   │ (API)     │   │
│  └───────────┘   └──────┬───────┘   └───────────┘   │
│                         │                           │
│                         ▼                           │
│              ┌─────────────────────┐                │
│              │    Tool-System      │                │
│              │  Shell · Dateien ·  │                │
│              │  Web · MCP · IDE ·  │                │
│              │  Sub-Agenten · ...  │                │
│              └─────────────────────┘                │
└─────────────────────────────────────────────────────┘
```

Ein Agent Harness ist die Infrastruktur, die ein großes Sprachmodell (LLM) zu einem autonomen Agenten macht. Es verbindet drei Dinge:

1. **Benutzer-Schnittstelle** — Eingabe und Ausgabe (hier: Terminal-UI mit React/Ink)
2. **Query-Loop** — die Kern-Schleife, die Nachrichten an die API sendet, Streaming-Antworten verarbeitet, Tool-Aufrufe erkennt und die Ergebnisse zurückspeist
3. **Tool-System** — ~40 konkrete Werkzeuge, mit denen der Agent auf die reale Welt zugreift

Die Dokumentation in diesem Repository erklärt, wie diese drei Teile zusammenspielen.

## Dokumentation

| Kapitel | Status | Inhalt |
| --- | --- | --- |
| [1.‌‌‌ Einleitung & Zweck](https://torstenc.github.io/agent-harness-anatomy/1_Einleitung_&_Zweck) | ✅ | Was ist Claude Code? Was ist ein Agent Harness? Technologiestack |
| [2. Architekturübersicht](https://torstenc.github.io/agent-harness-anatomy/2_Architekturübersicht) | ✅ | 9-Schichten-Modell, Startup-Ablauf, Query-Lifecycle, State-Management, Berechtigungen, Feature Flags |
| [3. Hauptkomponenten](https://torstenc.github.io/agent-harness-anatomy/3_Hauptkomponenten) | ✅ | QueryEngine, Query-Schleife, Tool-System (Interface, Registry, Ausführung), Command-System, AgentTool, BashTool, Coordinator, Skills/Plugins |
| 4. Typische Abläufe | 🔲 | Sequenzdiagramme: Startup, Query, Tool-Ausführung, Agent-Spawning (geplant) |
| 5. Erweiterungsmöglichkeiten | 🔲 | Plugin-System, Skill-System, MCP-Integration, Custom Agents (geplant) |
| 6. API-Referenz | 🔲 | Wichtige Typen, Interfaces und Funktionen (geplant) |
| Anhang: [Quellenverzeichnis](https://torstenc.github.io/agent-harness-anatomy/y_Quellenverzeichnis) | ✅ | Analysierte Quelldateien mit Links zu zwei öffentlichen Mirrors |
| Anhang: [Entstehungsprotokoll](https://torstenc.github.io/agent-harness-anatomy/z_Entstehungsprotokoll) | ✅ | Making-of, Analyseprozess, Herausforderungen, Learnings |

## Kernerkenntnisse

Die Analyse zeigt, wie ein modernes Agent Harness die folgenden Probleme löst:

### Kontext-Management

Ein LLM hat ein begrenztes Kontextfenster. Claude Code löst das durch eine **6-stufige Kompressions-Pipeline** (Tool-Result-Budget → Snip → Microcompact → Context Collapse → Auto-Compact → Reactive Compact), die den Kontext intelligent komprimiert, ohne relevante Informationen zu verlieren.

### Tool-Orchestrierung

Nicht alle Werkzeuge sind gleich: Read-Only-Tools (Grep, Glob, FileRead) laufen **parallel** (bis zu 10 gleichzeitig), mutierende Tools (Bash, FileWrite) **seriell**. Der `StreamingToolExecutor` startet Tools sogar **während des API-Streamings**, noch bevor die vollständige Antwort vorliegt.

### Sicherheit & Berechtigungen

Jeder Tool-Aufruf durchläuft eine **5-stufige Permission-Pipeline**: Config-Rules → Permission-Mode → Auto-Classifier → Hooks → User-Prompt. Vier Modi (`default`, `plan`, `auto`, `bypass`) erlauben feingranulare Kontrolle.

### Multi-Agent-Architektur

Über `AgentTool` können Sub-Agenten mit eingeschränkten Tool-Sets gestartet werden. Der `Coordinator-Mode` orchestriert mehrere Worker-Agenten für parallele Aufgaben.

### Build-Time Feature Gates

~30 Feature Flags via `bun:bundle`'s `feature()` ermöglichen **Dead Code Elimination** beim Build — deaktivierte Features werden physisch aus dem Binary entfernt.

## Technologiestack (des analysierten Codes)

| Kategorie | Technologie |
| --- | --- |
| Runtime | Bun |
| Sprache | TypeScript (strict) |
| Terminal-UI | React + Ink |
| CLI-Parsing | Commander.js |
| Validierung | Zod v4 |
| Code-Suche | ripgrep |
| Protokolle | MCP SDK, LSP |
| API | Anthropic SDK |
| Telemetrie | OpenTelemetry |
| Feature Flags | GrowthBook |

## Projektstruktur

```text
.
├── README.md                          ← dieses Dokument
├── .gitignore                          ← schließt src/ aus (Link zum Originalcode)
├── docs/
│   ├── 1_Einleitung_&_Zweck.md        ← Was und warum
│   ├── 2_Architekturübersicht.md       ← Wie es zusammenhängt
│   ├── 3_Hauptkomponenten.md           ← Detailanalyse der Kernmodule
│   ├── y_Quellenverzeichnis.md         ← Analysierte Quelldateien mit Links
│   └── z_Entstehungsprotokoll.md       ← Making-of & Entstehungsprotokoll
└── src/                               ← NICHT im Repo (siehe .gitignore)
    └── (Originalcode via Link oben)
```

## Lizenz & Disclaimer

Dieses Repository enthält **keinen Quellcode von Anthropic**. Es enthält ausschließlich eigenständig verfasste Dokumentation und Analyse. Der Originalcode kann über den in `.gitignore` referenzierten Link eingesehen werden.

Die Dokumentation dient ausschließlich Bildungs- und Forschungszwecken.

---

*Erstellt mit Unterstützung von Claude (Anthropic) als Analyse- und Dokumentationswerkzeug.*
