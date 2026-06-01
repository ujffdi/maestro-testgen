# maestro-testgen

**English** | [中文](./README.zh-CN.md)

A skill that derives UI-behavior regression tests from a **git diff** or a
**natural-language description**: it first produces a durable **manual test case
document**, and only when automation is actually feasible generates a runnable
**Maestro YAML flow**. Targets UI-behavior regression for Android / iOS / Web.

Core idea: **decide first → keep a manual case → then automate**.

## Does / Doesn't

| Does | Doesn't |
|---|---|
| Judge whether a change is user-visible UI behavior | Generate YAML straight from a diff |
| Write a durable, human-runnable manual test case first | Generate Maestro for pure logic (Mapper/Service/sort/format…) |
| Generate Maestro YAML when feasible (stable selectors, no coordinates) | Delete core assertions just to make a test pass |
| Auto-run YAML via CLI when a device is online & triage failures; inspect the live hierarchy via MCP to calibrate selectors | Hardcode YAML when evidence is insufficient |
| Degrade when no device is online: print the run command + prompt to start a device | Hard-run `maestro test` with no device |

## Directory layout

```
maestro-testgen/
├── SKILL.md            # required, contains name + description
├── references/         # 5 on-demand reference files
│   ├── workflow.md             # input modes A(git diff) / B(natural language), what to read
│   ├── decision-rules.md       # what suits Maestro vs not
│   ├── manual-case-template.md # manual test case format
│   ├── maestro-yaml-rules.md   # Maestro YAML authoring rules
│   └── run-and-mcp.md          # MCP setup, device detect, auto-run, failure triage
├── agents/openai.yaml  # Codex UI metadata
└── USAGE.md            # usage guide (non-standard skill file, excludable when distributing)
```

## Install

Copy the whole directory into the agent's skills location:

```bash
# Claude Code (project scope)
cp -R maestro-testgen <your-project>/.claude/skills/
```

For auto-run, also install the Maestro CLI and register the MCP server (one-time):

```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash   # Maestro CLI
claude mcp add -s project maestro -- maestro mcp      # register Maestro MCP
```

Full usage → [USAGE.md](./USAGE.md).

## Workflow (always in this order)

1. Detect input mode (git diff vs natural language)
2. Gather evidence (read the change / related code)
3. Emit the Test Routing Decision (UI behavior vs pure logic)
4. Write the manual test case document (**primary deliverable**)
5. Judge automation feasibility
6. Generate Maestro YAML only when `ready`
7. Auto-run: `maestro list-devices` → if a device is online, `maestro test`
   end-to-end and triage failures; if none, degrade (print command + prompt
   `maestro start-device`). Otherwise explain the blocker and what to supply.
