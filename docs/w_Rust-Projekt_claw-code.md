# Anhang: Rust-Projekt `claw-code` — Einordnung und Abgrenzung

Diese Dokumentation (Kapitel 1–6) analysiert das am 31. März 2026 geleakte
originale Claude-Code-System von Anthropic — ein TypeScript-/Bun-basiertes
CLI-Agent-Harness. Das öffentliche Projekt `ultraworkers/claw-code` ist
davon zu unterscheiden: eine eigenständige Rust-Reimplementierung, die an
die Fähigkeiten des Originals anschließt, es aber weder organisatorisch
noch architektonisch unverändert fortsetzt.
([README][1], [rust/README][2], [PARITY][3], [PHILOSOPHY][4])

---

## 1. Status des Rust-Projekts

`ultraworkers/claw-code` beschreibt sich als „public Rust implementation
of the `claw` CLI agent harness". Es ist kein Archiv des geleakten
Materials, sondern ein lauffähiges Projekt mit eigener CLI, eigener REPL
und eigener Entwicklungsdynamik. ([README][1])

Das aktuelle README grenzt sich ausdrücklich von Anthropic ab: Weder wird
Eigentum am ursprünglichen Claude-Code-Material beansprucht, noch eine
formelle Zugehörigkeit behauptet. ([README][1])

## 2. Architekturvergleich

| Aspekt | Geleaktes Original (Kap 1–6) | `claw-code` (Rust) |
| --- | --- | --- |
| Sprache | TypeScript (strict) | Rust |
| Runtime | Bun | Native Binary |
| Terminal-UI | React + Ink (geforkter TUI-Stack) | ratatui + crossterm (TUI) mit rustyline (REPL-Eingabe) |
| Modulstruktur | ~1.900 Dateien in `src/` | Rust-Workspace mit Crates |
| Crates / Module | — | `api`, `commands`, `runtime`, `tools`, `plugins`, `telemetry`, `rusty-claude-cli`, `mock-anthropic-service`, `compat-harness` |
| Testinfrastruktur | Intern (nicht veröffentlicht) | Deterministischer Mock-Service, Parity-Szenarien |
| Bezug zum Original | — | `compat-harness` extrahiert Tool-/Prompt-Manifeste aus der TypeScript-Quelle |

Die kanonische Implementierung liegt in `rust/`. Die Verzeichnisse `src/`
und `tests/` im Repository-Root dienen nur noch als begleitender
Python-/Referenz-Arbeitsbereich. ([rust/README][2])

## 3. Einordnung

Das Rust-Projekt ist weder ein isolierter Neuentwurf noch ein 1:1-Port:

- **Parity-Vorhaben:** Die `PARITY.md` beschreibt den Rust-Port
  ausdrücklich als Verhaltensabgleich zum geleakten Original,
  mit fest benannten Szenarien und reproduzierbaren Tests. ([PARITY][3])
- **Eigenständige Architektur:** Die Crate-Struktur, der Mock-Service
  und die Testinfrastruktur gehen deutlich über eine Sprachportierung
  hinaus.
- **Konzeptionelle Ausweitung:** `PHILOSOPHY.md` beschreibt das Projekt
  als Demonstration eines breiteren agentischen Arbeitsmodells — genannt
  werden `oh-my-codex`, `clawhip` und `oh-my-openagent`. ([PHILOSOPHY][4])

---

[1]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/README.md
[2]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/rust/README.md
[3]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/PARITY.md
[4]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/PHILOSOPHY.md
