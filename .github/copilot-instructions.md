# Copilot-Instructions für agent-harness-anatomy

## Sprache & Stil

- **Dokumentationssprache:** Deutsch (technische Fachbegriffe dürfen englisch bleiben)
- **Commit-Messages:** Deutsch, Conventional Commits (`docs:`, `fix:`, `chore:`)
- **ASCII-Diagramme** bevorzugt (kein Mermaid, keine Bilder für Architekturdiagramme)
- **Tabellen** mit `| --- | --- |`-Syntax

## Markdown Lint-Regeln

Diese Regeln gelten für alle `.md`-Dateien im Projekt:

| Regel | Beschreibung | Hinweis |
| --- | --- | --- |
| MD009 | Keine Trailing Spaces | `Expected: 0 or 2; Actual: 1` — genau 0 oder 2 Leerzeichen am Zeilenende, nie 1 |
| MD024 | Keine doppelten Überschriften | Jede `###`-Überschrift muss innerhalb einer Datei einzigartig sein |
| MD033 | Kein Inline-HTML | Ausnahme: `y_Quellenverzeichnis.md` verwendet bewusst `<table>`/`<a>`-Tags (GitHub Pages rendert Markdown-Links in HTML-Tabellen nicht) |

## Formatierung

- **Leerzeile nach `###`-Überschriften** (vor dem ersten Absatz)
- **Leerzeile vor und nach `---`-Trennlinien**
- **Leerzeile vor und nach Code-Blöcken** (vor ` ``` ` und nach ` ``` `)
- **Keine Leerzeile** zwischen Tabellen-Header und erster Datenzeile

## Quellenverweise

- Quelldateien werden in `docs/y_Quellenverzeichnis.md` zentral verlinkt
- In den Kapiteln wird per Anker verwiesen: `[Quelle](y_Quellenverzeichnis.md#src-queryengine)`
- Links zu den zwei öffentlichen Mirrors (ultraworkers/claw-code, Exhen/claude-code-2.1.88)

## Projektstruktur

- `docs/` — Dokumentation (wird auf GitHub Pages veröffentlicht)
- `src/` — Quellcode (gitignored, nur lokal für Analyse vorhanden)
- Pre-commit Hook kopiert `README.md` → `docs/index.md` automatisch
