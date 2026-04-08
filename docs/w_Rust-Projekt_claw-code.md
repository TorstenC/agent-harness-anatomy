# Rust-Projekt (`claw-code`) — Einordnung und Abgrenzung

Die vorliegende Dokumentation untersucht primär das am 31. März 2026 geleakte originale Claude-Code-System von Anthropic: ein TypeScript-/Bun-basiertes CLI-Agent-Harness mit React/Ink-Terminaloberfläche, umfangreicher Tool-Landschaft und einer sehr großen internen Codebasis. Die Kapitel 1 bis 5 beschreiben das ursprüngliche System, nicht dessen spätere öffentliche Nachbauten oder Weiterentwicklungen. ([GitHub][5])

Seit dem Leak hat sich unter `ultraworkers/claw-code` ein eigenständiges öffentliches Projekt etabliert. Es beschreibt sich heute selbst als „public Rust implementation of the `claw` CLI agent harness“. ([GitHub][1])

## 1. Was das heutige Rust-Projekt ist

Das heutige `ultraworkers/claw-code` versteht sich als öffentliche Rust-Implementierung eines CLI-Agent-Harness namens `claw`. Es ist kein bloßes Archiv des geleakten Materials, sondern ein lauffähiges Projekt mit eigener CLI, eigener REPL und eigener Entwicklungsdynamik. ([GitHub][1], [GitHub][2])

## 2. Architektur des Rust-Projekts

Das heutige Rust-Projekt ist architektonisch eigenständig. Das geleakte Original wurde als TypeScript-/Bun-/React-Ink-System dokumentiert. Das heutige `claw-code` ist dagegen als Rust-Workspace mit getrennten Crates für `api`, `commands`, `runtime`, `tools`, `plugins`, `telemetry`, `rusty-claude-cli`, `mock-anthropic-service` und `compat-harness` organisiert. ([GitHub][2], [GitHub][5])

Die kanonische Implementierung liegt in `rust/`. Die Verzeichnisse `src/` und `tests/` gelten nur noch als begleitender Python-/Referenz-Arbeitsbereich und nicht als primäre Runtime-Oberfläche. Diese Struktur spricht gegen einen bloßen 1:1-Port. Sie zeigt einen neuen technischen Zuschnitt. ([GitHub][1], [GitHub][2])

Hinzu kommt eine eigene Test- und Verifikationsinfrastruktur. Das Projekt dokumentiert einen deterministischen Anthropic-kompatiblen Mock-Service, reproduzierbare CLI-Harness-Tests und eine eigene Parity-Prüfkette mit fest benannten Szenarien. Das ist mehr als eine bloße Sprachportierung. Es ist eine neue Runtime mit eigener Testarchitektur. ([GitHub][2], [GitHub][3])

Vollständig vom Original gelöst ist das Projekt jedoch nicht. Die Parity-Dokumentation beschreibt den Rust-Port ausdrücklich als Parity-Vorhaben. Das Crate `compat-harness` extrahiert zudem Tool- und Prompt-Manifeste aus der Upstream-TypeScript-Quelle. Das Rust-Projekt ist daher kein isolierter Neuentwurf. Es ist eine eigenständige Reimplementierung mit bewusstem Verhaltensbezug zum geleakten Original. ([GitHub][2], [GitHub][3])

## 3. Abgrenzung zum geleakten Original

Im Unterschied zum internen Anthropic-System ist das heutige Rust-Projekt ein öffentlich entwickeltes Reimplementierungsprojekt. Das aktuelle README grenzt sich ausdrücklich von Anthropic ab. Es erklärt, dass weder Eigentum am ursprünglichen Claude-Code-Material beansprucht noch eine formelle Zugehörigkeit zu Anthropic behauptet wird. ([GitHub][1])

Hinzu kommt eine konzeptionelle Ausweitung. In `PHILOSOPHY.md` beschreibt sich das Projekt nicht nur als CLI-Tool, sondern als Demonstration eines größeren agentenkoordinierten Entwicklungssystems. Genannt werden dabei insbesondere `oh-my-codex`, `clawhip` und `oh-my-openagent`. Das heutige `claw-code` ist damit nicht nur ein Rust-Rewrite, sondern zugleich ein Showcase für ein breiteres agentisches Arbeitsmodell. ([GitHub][4])

## 4. Zusammenfassung in einem Satz

Das geleakte Claude Code ist der Analysegegenstand dieser Dokumentation; das heutige `ultraworkers/claw-code` ist ein öffentliches Rust-basiertes Reimplementierungs- und Demonstrationsprojekt, das an viele Fähigkeiten des Originals anschließt, dieses aber weder organisatorisch noch architektonisch unverändert fortsetzt. ([GitHub][1], [GitHub][2], [GitHub][3], [GitHub][4])

[1]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/README.md
[2]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/rust/README.md
[3]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/PARITY.md
[4]: https://raw.githubusercontent.com/ultraworkers/claw-code/main/PHILOSOPHY.md
[5]: https://github.com/TorstenC/agent-harness-anatomy/blob/main/docs/1_Einleitung_%26_Zweck.md
