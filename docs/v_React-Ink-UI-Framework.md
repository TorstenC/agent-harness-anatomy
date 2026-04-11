# Anhang: React/Ink als Terminal-UI-Framework

> **Ink** ist ein React-Renderer für die Kommandozeile — kein Web-Framework.
> Es nutzt Facebook's [Yoga](https://github.com/facebook/yoga) Layout-Engine,
> um Flexbox-Layouts direkt im Terminal zu berechnen, und gibt ANSI-Escape-
> Sequenzen auf `stdout` aus. Claude Code setzt auf einen stark erweiterten
> Ink-Fork als Grundlage seiner gesamten Benutzeroberfläche.

---

## Was ist Ink?

Ink ist ein Open-Source-Projekt von Vadim Demedes (MIT-Lizenz, npm: `ink`,
aktuell Version 7.0.0, ~2,9 Mio. Downloads/Woche). Die offizielle
Selbstbeschreibung lautet:

> *„React for CLIs. Build and test your CLI output using components."*

Ink verwendet einen eigenen **React-Reconciler** (`react-reconciler`), der
anstelle des Browser-DOM einen internen DOM-Baum aus Knoten wie `ink-box`,
`ink-text` und `ink-link` aufbaut. Dieser Baum wird über die Yoga-Layout-
Engine in Zeichen-Koordinaten umgerechnet und als ANSI-Ausgabe auf das
Terminal geschrieben.

```text
React-Komponenten
      │
      ▼
┌─────────────────┐
│  Ink Reconciler │  (react-reconciler)
│  DOM: ink-box,  │
│  ink-text, ...  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Yoga Layout    │  (Flexbox → Zeilen/Spalten)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ANSI-Renderer  │  (Escape-Sequenzen → stdout)
└─────────────────┘
```

**Ink ist ausschließlich für Terminal-UIs (TUIs) konzipiert.** Es hat nichts
mit Web-UIs, Browsern oder grafischen Oberflächen zu tun.

---

## Ink-Primitives

Ink stellt eine kleine Menge von Basis-Komponenten bereit, die alle
Terminal-Ausgabe abbilden:

| Komponente | Funktion |
| --- | --- |
| `<Box>` | Flexbox-Container (wie `<div style="display: flex">`) |
| `<Text>` | Text mit Farbe, Bold, Italic, Underline, Strikethrough |
| `<Newline>` | Zeilenumbruch innerhalb von `<Text>` |
| `<Spacer>` | Flexibler Leerraum (expandiert entlang der Hauptachse) |
| `<Static>` | Permanent gerenderte Ausgabe (Logs, abgeschlossene Tasks) |
| `<Transform>` | String-Transformation vor der Ausgabe |

Dazu kommen React-Hooks, die Terminal-spezifische Eingabe und
Zustandsverwaltung ermöglichen:

| Hook | Funktion |
| --- | --- |
| `useInput` | Tastatureingabe (Pfeiltasten, Ctrl, Shift, Tab, …) |
| `useApp` | App-Lifecycle (`exit()`, `waitUntilRenderFlush()`) |
| `useStdin` | Zugriff auf `stdin`, Raw Mode |
| `useFocus` / `useFocusManager` | Tab-basiertes Focus-Management |
| `useWindowSize` | Terminal-Größe mit Resize-Events |
| `useAnimation` | Frame-basierte Animationen (Spinner, Fortschritt) |

---

## Ink in Claude Code: ein tief integrierter Fork

Claude Code verwendet Ink nicht als externe Abhängigkeit, sondern hat
das Framework **geforkt und eingebettet**. Der gesamte Ink-Quellcode lebt
in `src/ink/` mit ~96 Dateien:

```text
src/ink/
├── ink.tsx              ← Haupt-Klasse (1.723 Zeilen)
├── reconciler.ts        ← React-Reconciler-Konfiguration (513 Zeilen)
├── dom.ts               ← Interner DOM-Baum (485 Zeilen)
├── screen.ts            ← Zeichen-Buffer mit Cell/Style/Hyperlink-Pools
├── selection.ts         ← Text-Selektion (Maus-basiert)
├── renderer.ts          ← Diff-basiertes Terminal-Rendering
├── optimizer.ts         ← Frame-Optimierung
├── components/          ← 18 Basis-Komponenten (Box, Text, Button, …)
├── hooks/               ← 12 Hooks (useInput, useFocus, useSelection, …)
├── events/              ← Event-System (Click, Keyboard, Focus, …)
├── layout/              ← Yoga-Integration (engine, geometry, node)
└── termio/              ← Terminal-Steuerung (CSI, DEC, OSC Sequenzen)
```

### Erweiterungen gegenüber Standard-Ink

Claude Codes Fork geht deutlich über das Open-Source-Ink hinaus:

| Feature | Standard-Ink | Claude Code Fork |
| --- | --- | --- |
| Mouse-Events | ❌ | ✅ Click, Hover, Drag mit Hit-Testing |
| Text-Selektion | ❌ | ✅ Wort-/Zeilen-/Block-Selektion |
| `<Button>` | ❌ | ✅ Interaktive Buttons mit States |
| `<ScrollBox>` | ❌ | ✅ Scroll-Container mit Smooth-Scrolling |
| `<Link>` | Nur Hyperlinks | ✅ Klickbare Hyperlinks mit Hit-Test |
| Alt-Screen | ❌ (erst Ink 7) | ✅ Alternate Screen Buffer |
| Kitty Keyboard | ❌ (erst Ink 7) | ✅ Erweiterte Tastatur-Erkennung |
| Bidi-Text | ❌ | ✅ Bidirektionale Textunterstützung |
| Yoga-Layout | Yoga (WASM) | Eigene Yoga-Bindings (`native-ts/yoga-layout`) |

---

## Die UI-Komponentenhierarchie

Auf dem Ink-Fork baut Claude Code eine umfangreiche Komponentenbibliothek
auf — insgesamt **~389 Dateien** in `src/components/`:

```text
src/components/
├── App.tsx                     ← Top-Level-Wrapper (FPS, Stats, AppState)
├── design-system/              ← 16 Dateien: ThemedBox, ThemedText, Dialog,
│                                  Divider, Tabs, ProgressBar, FuzzyPicker, …
├── permissions/                ← 51 Dateien: PermissionDialog, BashPermission-
│                                  Request, FileEditPermissionRequest, …
├── PromptInput/                ← Eingabefeld mit Vim-Modus
├── Markdown.tsx                ← Markdown-Rendering im Terminal
├── HighlightedCode.tsx         ← Syntax-Highlighting
├── StructuredDiff/             ← Datei-Diffs mit Farben
├── VirtualMessageList.tsx      ← Virtualisierte Nachrichtenliste
├── Spinner/                    ← Animierte Spinner
├── mcp/                        ← MCP-Server-Dialoge
├── teams/                      ← Multi-Agent-UI
└── ...                         ← ~150 weitere Komponenten
```

### Warum React im Terminal Sinn ergibt

Die Architektur zeigt, warum die Wahl von React für ein Terminal-Tool
keine Kuriosität ist, sondern eine strategische Entscheidung:

1. **Komponentenbaum = Dialog-Hierarchie.**
   Permission-Dialoge sind verschachtelte React-Komponenten. Ein
   `BashPermissionRequest` rendert einen `PermissionDialog`, der einen
   `PermissionRequestTitle` enthält. Dieses Compositing-Modell wäre mit
   klassischen CLI-Frameworks wie `inquirer` oder `prompts` nicht abbildbar.

2. **State-Management über React-Kontext.**
   `AppStateProvider`, `ThemeProvider`, `FpsMetricsProvider`, `StatsProvider`
   — der gesamte Zustand fließt über React Context durch den Baum.
   Kein globaler Singleton, sondern deklaratives State-Management.

3. **Hooks für Terminal-Interaktion.**
   `useInput` für Tastatureingabe, `useFocus` für Tab-Navigation,
   `useTerminalSize` für responsive Layouts — die Hook-API abstrahiert
   die rohe Terminal-I/O auf ein deklaratives Niveau.

4. **Concurrent-Rendering.**
   Der Fork verwendet React im `ConcurrentRoot`-Modus. Damit können
   langwierige Renders unterbrochen werden — etwa wenn während des
   Renderings einer langen Nachricht eine Permission-Anfrage eingeht.

5. **Design-System.**
   Über `ThemedBox`, `ThemedText` und den `ThemeProvider` kann die
   gesamte UI per Theme umgeschaltet werden — inklusive Farbschemata
   für verschiedene Terminal-Hintergründe.

---

## Der Render-Zyklus

Der Weg vom React-State-Update bis zur Terminal-Ausgabe durchläuft
mehrere Stufen:

```text
1. React-State-Update (setState / useReducer)
         │
2. React Reconciler (Fiber-Baum → Ink-DOM)
         │
3. Yoga-Layout (Flexbox → x,y,width,height)
         │
4. renderNodeToOutput (DOM → Screen-Buffer)
         │
5. Optimizer (Diff zum vorherigen Frame)
         │
6. writeDiffToTerminal (ANSI-Patches → stdout)
```

Die `ink.tsx`-Hauptklasse (1.723 Zeilen) orchestriert diesen Zyklus mit
einem FPS-Limiter (konfigurierbar, Standard: 30 fps), Synchronous-Output-
Support für flicker-freies Rendering und Alt-Screen-Management.

---

## Vergleich mit Alternativen

| Framework | Ansatz | Layout | Komponentenmodell |
| --- | --- | --- | --- |
| **Ink** | React-Reconciler + Yoga | Flexbox | JSX-Komponenten + Hooks |
| blessed / neo-blessed | Eigenes Widget-System | Box-Modell | Event-basierte Widgets |
| inquirer / prompts | Frage-Antwort-Ketten | Sequenziell | Prompt-Konfigurationen |
| Commander.js / yargs | Argument-Parsing | Keine UI | CLI-Definitionen |
| Bubbletea (Go) | Elm-Architektur (Model-View-Update) | Manuelles Layout | Funktionale Komponenten |

Claude Code ist nicht allein mit der Wahl von Ink — auch **Gemini CLI**
(Google), **Shopify CLI**, **GitHub Copilot CLI**, **Cloudflare Wrangler**
und **Prisma** setzen auf Ink als Terminal-UI-Framework.

---

## Korrektur: GPT-4o mini lag falsch

Eine frühere Version dieses Anhangs wurde von GPT-4o mini generiert und
enthielt die Behauptung:

> *„React/Ink ist speziell für die Erstellung von Benutzeroberflächen in*
> *React-Anwendungen entwickelt worden, was bedeutet, dass es hauptsächlich*
> *für Webanwendungen gedacht ist."*

**Das ist falsch.** Ink ist das genaue Gegenteil eines Web-Frameworks:

- Es rendert **ausschließlich** im Terminal über ANSI-Escape-Sequenzen
- Es verwendet **keinen** Browser, kein DOM, kein CSS
- Die Yoga-Layout-Engine berechnet Flexbox-Layouts **in Zeichen-Koordinaten**
  (Spalten × Zeilen), nicht in Pixeln
- Die npm-Beschreibung lautet explizit *„React for CLIs"*

Der Fehler zeigt ein bekanntes Problem kleiner Sprachmodelle: Die Begriffe
„React" und „UI-Framework" wurden mit Web-Entwicklung assoziiert, ohne den
tatsächlichen Kontext (Terminal-Rendering über `stdout`) zu berücksichtigen.
