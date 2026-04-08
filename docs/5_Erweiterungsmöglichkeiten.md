# 5. Erweiterungsmöglichkeiten

> Kapitel 3 beschreibt *was* Skills, Plugins und MCP-Server sind.
> Dieses Kapitel zeigt, *wie* man sie praktisch nutzt und wo die Architektur
> Erweiterungspunkte vorsieht.

---

## 5.1 Überblick: Vier Erweiterungsebenen

```text
┌────────────────────────────────────────────────────┐
│  4. Plugins     Git-Repos mit Skills + Hooks + MCP │
├────────────────────────────────────────────────────┤
│  3. MCP-Server  Externe Prozesse, JSON-RPC 2.0    │
├────────────────────────────────────────────────────┤
│  2. Skills      Markdown + Frontmatter → Prompt    │
├────────────────────────────────────────────────────┤
│  1. CLAUDE.md   Statische Projekt-Instruktionen    │
└────────────────────────────────────────────────────┘
```

Jede Ebene baut auf der darunter liegenden auf.
CLAUDE.md ist der einfachste Einstieg, Plugins das mächtigste Werkzeug.

---

## 5.2 Skills — Prompts als Erweiterungen

Skills sind Markdown-Dateien, die das Modell bei Bedarf als Kontext laden kann.
Die Architektur dahinter ist in [Kapitel 3.7](3_Hauptkomponenten.md#37-skills-und-plugins) dokumentiert.

### Lade-Hierarchie und Priorität

Skills werden aus bis zu sechs Quellen geladen — in dieser Reihenfolge
(siehe `loadSkillsDir.ts`, ~1087 Zeilen):

| Quelle | Pfad | `loadedFrom` |
| --- | --- | --- |
| Policy-verwaltet | `managed/.claude/skills/` | `managed` |
| User-global | `~/.claude/skills/` | `skills` |
| Projekt-lokal | `.claude/skills/` | `skills` |
| Plugin-bereitgestellt | `plugins/*/skills/` | `plugin` |
| Bundled | Eingebaut im Binary | `bundled` |
| MCP-Server | Per Feature-Flag | `mcp` |

Die Deduplizierung erfolgt per `realpath()` — Symlinks auf dieselbe
Datei werden nur einmal geladen.

### Frontmatter-Referenz

Jeder Skill kann im YAML-Frontmatter sein Verhalten konfigurieren
(vollständige Tabelle in [Kapitel 3.7](3_Hauptkomponenten.md#37-skills-und-plugins)).
Die drei wichtigsten Felder für eigene Skills:

- **`when_to_use`** — Wann das Modell den Skill *selbstständig* auswählen soll
- **`context: fork`** — Skill läuft als Sub-Agent statt inline
- **`allowed-tools`** — Whitelist der Tools, die der Skill verwenden darf

### Praktisches Beispiel: Eigener Projekt-Skill

```yaml
---
description: Führt die Projektspezifischen Lint-Checks aus
when_to_use: Wenn der Benutzer Code geändert hat und Qualität prüfen will
allowed-tools:
  - Bash
  - FileRead
context: inline
---

# Lint-Check

1. Führe `npm run lint` aus
2. Fasse Fehler zusammen und schlage Fixes vor
```

Speicherort: `.claude/skills/lint-check.md`

### Bundled Skills

Claude Code liefert ~12 eingebaute Skills aus (Stand Version 2.1.88).
Registrierung erfolgt in `skills/bundled/index.ts`:

| Skill | Zweck |
| --- | --- |
| `verify` | Code-Änderungen verifizieren |
| `remember` | Auto-Memory überprüfen und bereinigen |
| `debug` | Debugging-Workflow |
| `simplify` | Code vereinfachen |
| `skillify` | Aus Konversation neuen Skill generieren |
| `batch` | Mehrere Dateien parallel verarbeiten |
| `stuck` | Hilfe bei blockierten Aufgaben |
| `keybindings` | Tastenkürzel konfigurieren |

Einige weitere Skills (Dream, Hunter, Loop) sind hinter Feature-Flags
versteckt und nur für Anthropic-interne Nutzung aktiviert.

---

## 5.3 MCP-Server — Externe Tools per Protokoll

MCP (Model Context Protocol) ist das wichtigste offene Erweiterungsprotokoll.
Die interne Implementierung ist in [Kapitel 3.3](3_Hauptkomponenten.md#33-tool-system)
und [Kapitel 4.4](4_Typische_Abläufe.md#44-tool-pipeline-im-detail) beschrieben.

### Architektur der MCP-Integration

```text
Claude Code Harness                         Externer Prozess
┌──────────────────────┐                   ┌──────────────┐
│  MCPConnectionManager│──stdio/WebSocket──│  MCP-Server  │
│       (React Context)│   JSON-RPC 2.0    │              │
├──────────────────────┤                   │  tools/list  │
│  useManageMCP-       │◄──────────────────│  tools/call  │
│  Connections()       │                   │  resources/* │
├──────────────────────┤                   └──────────────┘
│  mcpClients[]        │
│  in QueryEngine      │
└──────────────────────┘
```

**Schlüssel-Dateien:**

| Datei | Verantwortung |
| --- | --- |
| `services/mcp/MCPConnectionManager.tsx` | React-Context für Verbindungen |
| `services/mcp/useManageMCPConnections.ts` | Connect, Reconnect, Toggle |
| `tools/MCPTool/MCPTool.ts` | Tool-Wrapper (wird in `mcpClient.ts` überschrieben) |
| `tools/ListMcpResourcesTool/` | MCP-Ressourcen auflisten |
| `tools/ReadMcpResourceTool/` | MCP-Ressourcen lesen |
| `tools/McpAuthTool/` | OAuth-Authentifizierung |
| `entrypoints/mcp.ts` | Claude Code *als* MCP-Server exponieren |

### Zwei Richtungen

Claude Code kann MCP in **zwei Richtungen** nutzen:

1. **MCP-Client** — verbindet sich mit externen MCP-Servern und stellt deren
   Tools dem Modell zur Verfügung (Standard-Anwendungsfall)
2. **MCP-Server** — exponiert die eigenen Built-in-Tools über das
   MCP-Protokoll (`entrypoints/mcp.ts`), sodass andere Clients darauf
   zugreifen können

### Tool-Pool-Integration

MCP-Tools werden bei der Zusammenstellung des Tool-Pools
*nach* den Built-in-Tools eingefügt und alphabetisch sortiert.
Das ist kein kosmetisches Detail — es dient der **Prompt-Cache-Stabilität**
(Details in [Kapitel 3.3](3_Hauptkomponenten.md#33-tool-system)).

---

## 5.4 Plugins — Das vollständige Paket

Plugins sind das strukturierteste Erweiterungssystem.
Sie werden aus Git-Repositories geladen und können kombinieren:

- **Skills** — eigene Prompt-Erweiterungen
- **Hooks** — Pre/Post-Processing bei Tool-Aufrufen
- **MCP-Server** — eigene externe Tools
- **Agenten** — eigene Agent-Konfigurationen

### Built-in vs. Marketplace

| Merkmal | Built-in Plugins | Marketplace Plugins |
| --- | --- | --- |
| ID-Format | `name@builtin` | `name@marketplace` |
| Quelle | Im Binary enthalten | Git-Repository |
| Aktivierung | Per `/plugin`-UI | Per `/plugin`-UI |
| Persistierung | User-Settings | User-Settings |
| Implementierung | `plugins/builtinPlugins.ts` | `utils/plugins/` |

### Plugin-Identifikation

Jedes Plugin wird durch einen eindeutigen Identifier im Format
`{name}@{marketplace}` identifiziert. Built-in Plugins verwenden
`@builtin` als Marketplace-Suffix. Die Parsing-Logik liegt in
`utils/plugins/pluginIdentifier.ts`.

---

## 5.5 Custom Agents

Das AgentTool (siehe [Kapitel 3.5](3_Hauptkomponenten.md#35-konkrete-tool-implementierungen))
kann mit benutzerdefinierten Agent-Konfigurationen erweitert werden.
Die Konfiguration erfolgt über Markdown-Dateien in `.claude/agents/`:

```yaml
---
description: Spezialisierter Code-Reviewer
model: claude-sonnet-4-20250514
allowed-tools:
  - FileRead
  - Grep
  - Glob
mcpServers:
  - name: github
    command: npx @modelcontextprotocol/server-github
---

# Code Review Agent

Du bist ein spezialisierter Code-Reviewer.
Prüfe den Code auf Sicherheit, Performance und Best Practices.
```

Der Agent-Name leitet sich vom Dateinamen ab.
Das Frontmatter-Feld `mcpServers` erlaubt Agent-spezifische MCP-Server,
die nur innerhalb dieses Agenten verfügbar sind.

---

## 5.6 Zusammenfassung: Entscheidungshilfe

| Anforderung | Empfohlene Ebene |
| --- | --- |
| Projekt-Konventionen dokumentieren | CLAUDE.md |
| Wiederkehrende Prompts automatisieren | Skill (`.claude/skills/`) |
| Externes Tool anbinden (API, DB, ...) | MCP-Server |
| Spezialisierten Agenten erstellen | Custom Agent (`.claude/agents/`) |
| Skills + Hooks + MCP bündeln | Plugin |
| Organisation-weite Policies | Managed Settings (`managed/.claude/`) |

Alle Erweiterungen durchlaufen dieselbe Permission-Pipeline
wie Built-in-Tools (beschrieben in [Kapitel 4.4](4_Typische_Abläufe.md#44-tool-pipeline-im-detail)).
