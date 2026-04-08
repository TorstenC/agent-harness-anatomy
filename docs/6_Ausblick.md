# 6. Ausblick — Wie die Story weitergeht

> Die Analyse eines 512.000-Zeilen-Leaks ist eine Momentaufnahme.
> Dieses Kapitel ordnet ein, was wir daraus gelernt haben — und wohin
> sich Agent Harnesses als Kategorie entwickeln.

---

## 6.1 Was wir aus dem Leak gelernt haben

Der geleakte Quellcode von Claude Code (Version 2.1.88, Mai 2025) ist die
bisher detaillierteste öffentliche Quelle darüber, wie ein kommerzielles
AI-Agent-System intern aufgebaut ist.

**Zentrale Erkenntnisse:**

1. **Architektur schlägt Modell-Größe.**
   Die eigentliche Komplexität liegt nicht im LLM-Aufruf, sondern in den
   ~40 Tools, dem Permission-System, der Kontext-Komprimierung und dem
   Session-Management. Das Modell ist austauschbar — das Harness nicht.

2. **Kontext-Management ist das Kernproblem.**
   Die sechs Stufen der Kontext-Komprimierung (beschrieben in
   [Kapitel 4.3](4_Typische_Abläufe.md#43-die-query-schleife-im-detail))
   zeigen, dass der begrenzende Faktor nicht die Modell-Intelligenz ist,
   sondern das Fenster, durch das es die Welt sieht.

3. **Permissions als First-Class-Konzept.**
   Die 5-stufige Permission-Pipeline
   ([Kapitel 4.4](4_Typische_Abläufe.md#44-tool-pipeline-im-detail))
   ist kein nachträgliches Feature, sondern durchzieht die gesamte
   Architektur. Jedes Tool, jeder MCP-Server, jeder Skill durchläuft
   denselben Gating-Mechanismus.

4. **MCP als strategischer Erweiterungspunkt.**
   Die umfangreiche MCP-Integration
   ([Kapitel 5.3](5_Erweiterungsmöglichkeiten.md#53-mcp-server--externe-tools-per-protokoll))
   mit ~30 Quelldateien zeigt, dass Anthropic auf ein offenes
   Ökosystem setzt — nicht auf ein geschlossenes Plugin-System.

5. **React/Ink für Terminal-UIs.**
   Die Wahl von React als UI-Framework für ein Terminal-Tool ist
   ungewöhnlich, aber ermöglicht ein komponentenbasiertes
   Permission-Dialog-System, das mit klassischen CLI-Frameworks
   kaum realisierbar wäre.

---

## 6.2 Offene Fragen

Einige Aspekte der Architektur konnten wir in dieser Analyse nicht
vollständig klären:

| Frage | Kontext |
| --- | --- |
| Wie funktioniert der Coordinator im Detail? | `src/coordinator/` enthält die Multi-Agent-Orchestrierung, ist aber stark Feature-gated und im Leak nur teilweise dokumentiert |
| Wie skaliert das Permission-System mit MCP? | Bei vielen MCP-Servern mit dutzenden Tools entsteht ein kombinatorisches Problem für die Permission-Regeln |
| Was passiert bei Prompt-Cache-Invalidierung? | Die Cache-Stabilität durch Tool-Sortierung ist ein cleveres Detail — aber wie reagiert das System auf häufige MCP-Server-Änderungen? |
| Wie arbeitet das Bridge-System genau? | `src/bridge/` (~25 Dateien) implementiert Remote-Sessions, ist aber in der öffentlichen Version weitgehend deaktiviert |
| Wie werden Kosten tatsächlich kontrolliert? | `cost-tracker.ts` und `costHook.ts` existieren, aber die Limits und Eskalationsmechanismen sind nur angedeutet |

---

## 6.3 Die nächste Generation

### Rust-Reimplementierung (claw-code)

Das Open-Source-Projekt **claw-code** (`ultraworkers/claw-code`) versucht,
die Kernfunktionalität von Claude Code in Rust nachzubauen.
Eine ausführliche Einordnung findet sich im
[Anhang: Rust-Projekt claw-code](w_Rust-Projekt_claw-code.md).

Die Existenz eines Rust-Reimplementierungsprojekts zeigt, dass die
Community die Architektur des Harness als wertvoll genug erachtet,
um sie in einer performanteren Sprache nachzubauen.

### Coordinator und Multi-Agent-Systeme

Der im Quellcode vorhandene `src/coordinator/` deutet auf eine Zukunft
hin, in der Claude Code nicht einen Agenten steuert, sondern ein
*Team* von Agenten. Die `TeamCreateTool`- und `TeamDeleteTool`-Dateien
im Tool-Verzeichnis bestätigen diese Richtung.

### Agents-as-Platform

Claude Code entwickelt sich von einem Entwickler-Tool zu einer
Plattform. Die Kombination aus:

- **MCP** für externe Tool-Integration
- **Skills** für Prompt-Erweiterungen
- **Plugins** für gebündelte Erweiterungen
- **Custom Agents** für spezialisierte Rollen
- **Hooks** für Event-basierte Automatisierung

...bildet ein vollständiges Erweiterungs-Ökosystem, das über
reine Code-Assistenz hinausgeht.

---

## 6.4 Vergleichbare Systeme

Claude Code ist nicht das einzige Agent Harness auf dem Markt.
Ein Vergleich der Architekturen zeigt, dass sich bestimmte
Muster als Industriestandard herauskristallisieren:

| System | Hersteller | Sprache | Tool-Modell | Erweiterungen |
| --- | --- | --- | --- | --- |
| **Claude Code** | Anthropic | TypeScript | ~40 Built-in + MCP | Skills, Plugins, MCP |
| **Cursor** | Anysphere | TypeScript | VS Code Fork + Agent | Extensions, MCP |
| **Windsurf** | Codeium | TypeScript | VS Code Fork + Agent | Flows, MCP |
| **Cline** | Open Source | TypeScript | VS Code Extension | MCP |
| **Aider** | Open Source | Python | Git-zentriert | Konfigurationsdateien |
| **Codex CLI** | OpenAI | TypeScript/Rust | Terminal-basiert | MCP (geplant) |

**Gemeinsame Muster:**

- **Tool-Loop:** Alle Systeme implementieren eine Variante der
  Query-Schleife aus [Kapitel 4.3](4_Typische_Abläufe.md#43-die-query-schleife-im-detail)
- **Permission-Gating:** Alle benötigen ein Berechtigungskonzept
  für Dateisystem- und Shell-Zugriff
- **MCP als Lingua Franca:** MCP etabliert sich als gemeinsames
  Protokoll für Tool-Integration über Hersteller-Grenzen hinweg
- **Kontext-Management:** Alle kämpfen mit dem gleichen
  Grundproblem der begrenzten Kontextfenster

**Was Claude Code auszeichnet:**

- Die umfangreichste und am besten strukturierte Tool-Bibliothek
- Ein ausgereiftes Permission-System mit Policy-Unterstützung
- React/Ink als UI-Framework (einzigartig im Feld)
- Sub-Agent-Spawning mit eigenem Kontext

---

## 6.5 Fazit

Agent Harnesses sind eine neue Software-Kategorie, die sich gerade
erst herausbildet. Der Leak von Claude Code gibt uns einen seltenen
Einblick in den Stand der Technik bei einem der führenden Anbieter.

Die Kernarchitektur — eine agentic Loop mit Tool-Aufrufen, Permission-Gating
und Kontext-Management — wird sich als Pattern etablieren, ähnlich wie
MVC in Web-Frameworks oder Event Loops in Server-Architekturen.

Was sich ändern wird:

- **Sprache:** Der Trend geht von TypeScript zu Rust (Performance, Sicherheit)
- **Koordination:** Von Single-Agent zu Multi-Agent (Coordinator-Pattern)
- **Integration:** Von proprietären Plugins zu offenen Protokollen (MCP)
- **Autonomie:** Von Tool-Aufrufen zu längerfristigen Tasks (Cron, Remote Agents)

Diese Dokumentation ist eine Momentaufnahme von Mai 2025.
Die nächste Version von Claude Code wird anders aussehen —
aber die Grundmuster, die wir hier beschrieben haben, werden
in irgendeiner Form bestehen bleiben.
