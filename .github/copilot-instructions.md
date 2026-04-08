# Copilot-Instructions für agent-harness-anatomy

## Sprache & Stil

- **Dokumentationssprache:** Deutsch (technische Fachbegriffe dürfen englisch bleiben)
- **Commit-Messages:** Deutsch, Conventional Commits (`docs:`, `fix:`, `chore:`)
- **ASCII-Diagramme** bevorzugt (kein Mermaid, keine Bilder für Architekturdiagramme)
- **Tabellen** mit `| --- | --- |`-Syntax

## Markdown Lint-Regeln

Diese Regeln gelten für Antworten im Chat sowie für alle `.md`-Dateien im Projekt:

| Regel | Beschreibung | Hinweis |
| --- | --- | --- |
| MD009 | Keine Trailing Spaces | Genau 0 oder 2 Leerzeichen am Zeilenende, nie 1 |
| MD024 | Keine doppelten Überschriften | Jede Überschrift muss innerhalb einer Datei einzigartig sein |
| MD033 | Kein Inline-HTML | Ausnahme: `y_Quellenverzeichnis.md` verwendet bewusst HTML-Tags (GitHub Pages rendert Markdown-Links in HTML-Tabellen nicht) |

## Formatierung

- **Leerzeile nach Überschriften** (vor dem ersten Absatz)
- **Leerzeile vor und nach `---`-Trennlinien**
- **Leerzeile vor und nach Code-Blöcken** (vor ` ``` ` und nach ` ``` `)
- **Keine Leerzeile** zwischen Tabellen-Header und erster Datenzeile

## README-Dokumentationstabelle

In der Spalte „Kapitel" unter `## Dokumentation` in der `README.md` wird zwischen der Kapitelnummer und dem Titel ein **Narrow No-Break Space** (U+202F, UTF-8 `e2 80 af`) verwendet, damit der Eintrag nicht mitten im Wort umbricht, z. B. `5.␣Erweiterungsmöglichkeiten` (wobei `␣` = U+202F).

- Kapiteleinträge: `1.` + U+202F + Titel
- Anhang-Einträge: `Anhang:` + normales Leerzeichen (U+0020) — hier darf umbrochen werden

## Quellenverweise

- Quelldateien werden in `docs/y_Quellenverzeichnis.md` zentral verlinkt
- In den Kapiteln wird per Anker verwiesen, z.B. Quelle als Link auf `y_Quellenverzeichnis.md`
- Links zu den zwei öffentlichen Mirrors (ultraworkers/claw-code, Exhen/claude-code-2.1.88)

## Projektstruktur

- `docs/` — Dokumentation (wird auf GitHub Pages veröffentlicht)
- `src/` — Quellcode (gitignored, nur lokal für Analyse vorhanden)
- Pre-commit Hook kopiert `README.md` → `docs/index.md` automatisch

## Entstehungsprotokoll (z_Entstehungsprotokoll.md)

Stil-Konventionen für das Making-of:

- Kapitelüberschrift: `## Claude Opus 4.6 – Thema`
- Tool-Aufrufe als kompakte Stichpunkte: `- Read datei.ts, lines X to Y`
- Keine rohen file-URIs oder leere Markdown-Links
- Keine leeren Code-Blöcke
- Keine Todo-Nummern wie `Completed (1/4)` — stattdessen Fließtext oder Stichpunkte
- Zusammenfassungstabelle am Ende jedes Abschnitts
- Abschluss mit `- Made changes.` und `---`
