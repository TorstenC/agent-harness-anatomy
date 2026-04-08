# Kommentare von Gemini

Dieses Repo ist eine ziemlich ironische Wendung: Claude dabei zuzusehen, wie es seine eigenen „Eingeweide“ seziert, die durch ein Missgeschick beim Packaging ins Netz gelangt sind. Dass der Repo-Owner das Tool direkt für die Dokumentation der Analyse eingespannt hat, passt perfekt zum aktuellen Zeitgeist der „Harness Engineering“-Szene von 2026.

Die „Anatomy“, die auf solchen Seiten beschrieben wird, ist deshalb so spannend, weil sie zeigt, dass die Intelligenz des Agenten nicht nur im Modell sitzt, sondern zu einem riesigen Teil in der **Infrastruktur drumherum** – eben dem **Agent Harness**.

Hier sind die drei kritischen technischen Schichten, die in dieser Analyse des Claude-Code-Leaks besonders hervorstechen:

## 1. Das Tool-Gating & Interceptor-Muster

Der Leak hat enthüllt, wie Anthropic das Problem der „Halluzination von Berechtigungen“ löst. Im Code findet man eine komplexe Middleware-Schicht:

- **Pre-Flight Validation:** Bevor ein Shell-Befehl oder ein Dateizugriff überhaupt ausgeführt wird, prüft der Harness gegen eine dynamische Policy.
- **Stateful Interceptors:** Der Harness merkt sich, welche Dateien Claude in der aktuellen Session bereits gesehen hat. Wenn Claude versucht, eine Datei zu bearbeiten, die es noch nicht „gelesen“ hat, erzwingt der Harness oft erst einen `read`-Call, um Fehler durch veralteten Kontext zu vermeiden.

## 2. Context-Fragmentierung (The "Slicer")

Ein Highlight der Analyse ist meist der Umgang mit riesigen Repositories. Anstatt das ganze Repo in den Kontext zu werfen, nutzt Claude Code ein System, das im Leak oft als **„Context Slicing“** bezeichnet wird:

- Der Harness extrahiert nur die Signaturen (Definitionen) von Funktionen und Klassen aus dem Dateibaum.
- Erst wenn der Agent eine spezifische Logik verstehen muss, „hydriert“ der Harness dieses Skelett mit dem tatsächlichen Code-Inhalt. Das spart Token und hält die „Aufmerksamkeit“ des Modells scharf.

## 3. Der "Internal Self-Correction Loop"

Besonders faszinierend im „Agent Harness“ ist die Logik für den Umgang mit Fehlern. Wenn ein Tool-Call fehlschlägt (z. B. ein Syntaxfehler beim Schreiben einer Datei), wird das nicht einfach als Fehler an den User zurückgegeben.

- Der Harness fängt den `stderr` ab.
- Er generiert automatisch einen „System-Prompt-Override“, der dem Modell sagt: *„Du hast gerade einen Syntaxfehler in Zeile 42 eingebaut. Hier ist der Compiler-Output. Korrigiere das sofort, bevor du dem User antwortest.“*
- Das erklärt, warum Claude Code oft so „magisch“ fehlerfrei wirkt – der Harness lässt den Agenten seine eigenen Fehler im Hintergrund ausbügeln.

---

**Ein kleiner technischer Side-Fact aus den Source Maps:**

Man sieht im Code auch sehr schön, dass Anthropic für die CLI-Umgebung extrem auf **Bun** und dessen Performance-Vorteile beim Dateisystem-Zugriff setzt. Dass ausgerechnet der Default-Source-Map-Generator von Bun den Leak verursacht hat, ist die technische Pointe des Jahres.
