# Quellenverzeichnis

Dieses Verzeichnis listet die analysierten Quelldateien mit Verweisen auf öffentlich
zugängliche Kopien. Der Originalcode ist geistiges Eigentum von Anthropic, PBC.
Er wurde nicht unter einer Open-Source-Lizenz veröffentlicht.

Die Links dienen ausschließlich der wissenschaftlichen Nachvollziehbarkeit
(vergleichbar mit Quellenangaben in einer Analyse oder einem Review).

> **Hinweis:**  
> Sollte Anthropic die Entfernung der verlinkten Repositories veranlassen, zeigen die Links in diesem Dokument ins Leere.

## Primärquellen

| Kürzel | Repository | Commit |
| --- | --- | --- |
| **[UW]** | [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code?tab=readme-ov-file#readme) | [`5a774a2`](https://github.com/ultraworkers/claw-code/tree/5a774a2b62d7949c1d94e0b726281554d7893cfd) |
| **[EX]** | [Exhen/claude-code-2.1.88](https://github.com/Exhen/claude-code-2.1.88?tab=readme-ov-file#readme) | [`2008cd9`](https://github.com/Exhen/claude-code-2.1.88/tree/2008cd913996f1fa830815626a6479295b1e0786) |

## Quelldateien

<!-- Anker-Format: id="src-dateiname" -->

<!-- ### Kern-Module -->

<table>
<tr><th>Anker</th><th>Datei</th><th>Kapitel</th><th>Links</th></tr>
<tr><th>Kern-Module</th><td colspan=3></td></tr>
<tr id="src-main">
<td><code>src-main</code></td>
<td><code>src/main.tsx</code></td>
<td>1, 2, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/main.tsx) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/main.tsx)

</td>
</tr>

<tr id="src-queryengine">
<td><code>src-queryengine</code></td>
<td><code>src/QueryEngine.ts</code></td>
<td>2, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/QueryEngine.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/QueryEngine.ts)

</td>
</tr>

<tr id="src-query">
<td><code>src-query</code></td>
<td><code>src/query.ts</code></td>
<td>2, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/query.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/query.ts)

</td>
</tr>

<tr id="src-tool">
<td><code>src-tool</code></td>
<td><code>src/Tool.ts</code></td>
<td>1, 2, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/Tool.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/Tool.ts)

</td>
</tr>

<tr id="src-tools">
<td><code>src-tools</code></td>
<td><code>src/tools.ts</code></td>
<td>1, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/tools.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/tools.ts)

</td>
</tr>

<tr id="src-commands">
<td><code>src-commands</code></td>
<td><code>src/commands.ts</code></td>
<td>1, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/commands.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/commands.ts)

</td>
</tr>

<!-- ### Startup & Infrastruktur -->
<tr><th>Startup & Infrastruktur</th><td colspan=3></td></tr>

<tr id="src-setup">
<td><code>src-setup</code></td>
<td><code>src/setup.ts</code></td>
<td>2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/setup.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/setup.ts)

</td>
</tr>

<tr id="src-init">
<td><code>src-init</code></td>
<td><code>src/entrypoints/init.ts</code></td>
<td>2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/entrypoints/init.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/entrypoints/init.ts)

</td>
</tr>

<tr id="src-repllauncher">
<td><code>src-repllauncher</code></td>
<td><code>src/replLauncher.tsx</code></td>
<td>2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/replLauncher.tsx) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/replLauncher.tsx)

</td>
</tr>

<tr id="src-context">
<td><code>src-context</code></td>
<td><code>src/context.ts</code></td>
<td>1, 2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/context.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/context.ts)

</td>
</tr>

<!-- ### State-Management -->
<tr><th>State-Management</th><td colspan=3></td></tr>

<tr id="src-bootstrap-state">
<td><code>src-bootstrap-state</code></td>
<td><code>src/bootstrap/state.ts</code></td>
<td>2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/bootstrap/state.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/bootstrap/state.ts)

</td>
</tr>

<!-- ### Tool-Orchestrierung -->
<tr><th>Tool-Orchestrierung</th><td colspan=3></td></tr>

<tr id="src-toolorchestration">
<td><code>src-toolorchestration</code></td>
<td><code>src/services/tools/toolOrchestration.ts</code></td>
<td>2, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/services/tools/toolOrchestration.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/services/tools/toolOrchestration.ts)

</td>
</tr>

<tr id="src-streamingtoolexecutor">
<td><code>src-streamingtoolexecutor</code></td>
<td><code>src/services/tools/StreamingToolExecutor.ts</code></td>
<td>2, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/services/tools/StreamingToolExecutor.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/services/tools/StreamingToolExecutor.ts)

</td>
</tr>

<tr id="src-toolexecution">
<td><code>src-toolexecution</code></td>
<td><code>src/services/tools/toolExecution.ts</code></td>
<td>3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/services/tools/toolExecution.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/services/tools/toolExecution.ts)

</td>
</tr>

<!-- ### Konkrete Tools -->
<tr><th>Konkrete Tools</th><td colspan=3></td></tr>

<tr id="src-agenttool">
<td><code>src-agenttool</code></td>
<td><code>src/tools/AgentTool/AgentTool.tsx</code></td>
<td>1, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/tools/AgentTool/AgentTool.tsx) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/tools/AgentTool/AgentTool.tsx)

</td>
</tr>

<tr id="src-runagent">
<td><code>src-runagent</code></td>
<td><code>src/tools/AgentTool/runAgent.ts</code></td>
<td>3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/tools/AgentTool/runAgent.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/tools/AgentTool/runAgent.ts)

</td>
</tr>

<tr id="src-loadagentsdir">
<td><code>src-loadagentsdir</code></td>
<td><code>src/tools/AgentTool/loadAgentsDir.ts</code></td>
<td>3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/tools/AgentTool/loadAgentsDir.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/tools/AgentTool/loadAgentsDir.ts)

</td>
</tr>

<tr id="src-bashtool">
<td><code>src-bashtool</code></td>
<td><code>src/tools/BashTool/BashTool.tsx</code></td>
<td>3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/tools/BashTool/BashTool.tsx) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/tools/BashTool/BashTool.tsx)

</td>
</tr>

<!-- ### Coordinator & Multi-Agent -->
<tr><th>Coordinator & Multi-Agent</th><td colspan=3></td></tr>

<tr id="src-coordinatormode">
<td><code>src-coordinatormode</code></td>
<td><code>src/coordinator/coordinatorMode.ts</code></td>
<td>1, 2, 3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/coordinator/coordinatorMode.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/coordinator/coordinatorMode.ts)

</td>
</tr>

<!-- ### Berechtigungen -->
<tr><th>Berechtigungen</th><td colspan=3></td></tr>

<tr id="src-permissioncontext">
<td><code>src-permissioncontext</code></td>
<td><code>src/hooks/PermissionContext.ts</code></td>
<td>2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/hooks/PermissionContext.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/hooks/PermissionContext.ts)

</td>
</tr>

<!-- ### Skills & Plugins -->
<tr><th>Skills & Plugins</th><td colspan=3></td></tr>

<tr id="src-loadskillsdir">
<td><code>src-loadskillsdir</code></td>
<td><code>src/skills/loadSkillsDir.ts</code></td>
<td>3</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/skills/loadSkillsDir.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/skills/loadSkillsDir.ts)

</td>
</tr>

<!-- ### Externe Integrationen -->
<tr><th>Externe Integrationen</th><td colspan=3></td></tr>

<tr id="src-mcp-client">
<td><code>src-mcp-client</code></td>
<td><code>src/services/mcp/client.ts</code></td>
<td>2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/services/mcp/client.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/services/mcp/client.ts)

</td>
</tr>

<tr id="src-bridgemain">
<td><code>src-bridgemain</code></td>
<td><code>src/bridge/bridgeMain.ts</code></td>
<td>2</td>
<td>

[UW](https://github.com/ultraworkers/claw-code/blob/5a774a2b62d7949c1d94e0b726281554d7893cfd/src/bridge/bridgeMain.ts) · [EX](https://github.com/Exhen/claude-code-2.1.88/blob/2008cd913996f1fa830815626a6479295b1e0786/source/src/bridge/bridgeMain.ts)

</td>
</tr>
</table>
