# maestro-testgen Usage Guide

**English** | [中文](./USAGE.zh-CN.md)

> A skill that derives UI-behavior regression tests from a **git diff** or a
> **natural-language description**: it first produces a durable **manual test case
> document**, and only when automation is feasible generates a runnable
> **Maestro YAML flow**. Targets UI-behavior regression for Android / iOS / Web.
>
> Note: this file is project usage documentation and is **not part of the standard
> skill structure**. You can exclude it when packaging the standard skill.

---

## 1. What it does / doesn't do

| Does | Doesn't |
|---|---|
| Judge whether a change is user-visible UI behavior | Generate YAML straight from a diff |
| Write a durable, human-runnable manual test case first | Generate Maestro for pure logic (Mapper/Service/sort/format…) |
| Generate Maestro YAML when feasible (stable selectors, no coordinates) | Delete core assertions just to make a test pass |
| Auto-run YAML via CLI when a device is online & triage failures; inspect the live hierarchy via MCP to calibrate selectors | Hardcode YAML when evidence is insufficient |
| Degrade when no device is online: print the run command + prompt to start a device | Hard-run `maestro test` with no device |

Core idea: **decide first → keep a manual case → then automate**.

---

## 2. Install / deploy

This directory is itself a standard skill source directory:

```
maestro-testgen/
├── SKILL.md            # required, contains name + description
├── references/         # 5 on-demand reference files
└── agents/openai.yaml  # Codex UI metadata
```

Pick the deploy location for your agent (just copy the whole directory):

```bash
# Claude Code (project scope)
cp -R maestro-testgen <your-project>/.claude/skills/

# Claude Code (user scope, available globally)
cp -R maestro-testgen ~/.claude/skills/

# Codex
cp -R maestro-testgen <your-project>/.codex/skills/
```

After deploying, the skill is auto-invoked in that project by the trigger phrases below.

Install the Maestro CLI (required to auto-run YAML):
```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash
maestro --version
```

Optional: register the Maestro MCP server so the agent can inspect the live UI
hierarchy and calibrate selectors:
```bash
# project scope (writes .mcp.json, shared with the team)
claude mcp add -s project maestro -- maestro mcp
# or user scope
claude mcp add maestro -- maestro mcp
```

---

## 3. How to trigger

### Manual trigger command flow (from zero to "test result is out")

Follow in order. Step 0 is one-time; afterwards you only need steps 1–2.

**0. One-time install (once per project/machine)**
```bash
cp -R maestro-testgen <your-project>/.claude/skills/     # ① copy the skill
curl -fsSL "https://get.maestro.mobile.dev" | bash       # ② install Maestro CLI
claude mcp add -s project maestro -- maestro mcp          # ③ register Maestro MCP
claude mcp get maestro                                    #    verify it's connected
```

**1. Start a device (before every test run)**
```bash
maestro start-device     # start an emulator/Simulator; or connect a real device / open a browser
maestro list-devices     # confirm a device is online (the gate for auto-run)
```

**2. Trigger manually in the agent chat** (not the terminal)
```text
/maestro-testgen generate a Maestro test from the current diff
```
Once triggered, the agent runs through it **automatically**: routing decision →
write the manual case → feasibility judgment → generate YAML (inspecting the
hierarchy via MCP to pick selectors first) → if a device is detected, run
`maestro test` end-to-end → report pass/fail + logs. **No manual run needed.**

**3. (Optional) Re-run the same flow yourself / wire into CI**
```bash
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml
```

What's manual vs automatic:

| Step | Who does it |
|---|---|
| Install, register MCP, start a device | **You, manually** (terminal) |
| `/maestro-testgen ...` trigger | **You, manually** (one line in chat) |
| Generate case + YAML + run `maestro test` + triage failures | **Agent, automatically** |
| Re-run / wire into CI | You, manually (optional) |

> When no device is online: the agent won't hard-run. It just gives you the
> `maestro test ...` command + a "run `maestro start-device` first" prompt; start a
> device and trigger again.

### Trigger phrase examples

Just state your intent in natural language, e.g.:

- "Generate a Maestro test from the current diff"
- "Generate test cases from this change"
- "Generate a Maestro case for the login flow"
- "Test editing the profile in the user center"
- "Generate a test report and YAML for the video-room gifting feature"
- "Analyze whether this change needs UI automation testing"

The skill supports two input modes and detects them automatically:

- **Mode A — git diff**: based on the current change. Reads `git diff --name-only` /
  `git diff -U0`, then reads related pages/routes/ViewModel/Compose/XML/SwiftUI/React
  as needed.
- **Mode B — natural language**: based on a business description. First locates the
  relevant business code and existing tests from the description, then generates the
  case and YAML.

---

## 4. Output order (fixed)

On each invocation the agent outputs in this order:

1. **Test Routing Decision**
2. **Manual test case** content + save path
3. **Automation feasibility** judgment
4. **Maestro YAML** content + save path (only when feasibility is `ready`)
5. **Auto-run result**: when a device is online, runs `maestro test` end-to-end and
   gives pass/fail + logs; when none, degrades to printing the run command and a
   "start a device first" prompt
6. If no YAML: the `blocked` reason and what info is still needed

### A Test Routing Decision looks like this

```yaml
test_routing_decision:
  input_mode: git_diff | natural_language
  platforms: [android | ios | web | unknown]
  change_type: ui_behavior | pure_logic | mixed | unknown
  recommended_test_level: maestro | unit_test | integration_test | manual_review
  automation_runner: maestro | none
  reason: "<why>"
  evidence:
    - "<file path or description clue>"
```

---

## 5. Deliverables & save paths

| Deliverable | Default path |
|---|---|
| Manual test case | `qa/manual-cases/<case_id>.md` |
| Maestro flow | `qa/manual-cases/maestro-flows/<case_id>.yaml` (or the project's existing Maestro dir, e.g. `maestro/flows/`) |

If the project already has a test-case / Maestro directory, prefer the existing one.

### Manual case template (excerpt)

```markdown
# Manual test case: <title>
## Basic info       # case ID / platform / linked change / test type / priority
## Test goal
## Preconditions    # test account / login state / environment / server-side data / permissions / other deps
## Test steps
## Expected results # ← these ARE the assertions; never weaken or delete them for automation
## Risk points
## Automation feasibility   # ready / blocked / needs_selector / needs_test_data + reason
```

---

## 6. Decision rules cheat sheet

**Suits Maestro (UI behavior)**
- Page display changes; tap / input / tab / dialog / permission-dialog changes
- Navigation changes; user paths like login / home / profile / room / gift / follow / chat
- ViewModel / State / Store / Controller state that ultimately affects UI display or interaction

**Doesn't suit Maestro (pure logic) → route to unit / integration tests**
- Pure-function computation; DTO / Entity / Mapper conversion
- Repository / Service internal logic; sort / filter / format
- SDK wrappers; network/cache strategy and other logic with no direct UI entry point

For mixed changes mark `change_type: mixed`: the UI part goes to a manual case +
Maestro, and the pure-logic part is recommended for unit / integration tests in `reason`.

---

## 7. Full examples

### Example A: based on a diff (new OTP input field on an Android login page)

You say:
> Generate a Maestro test from the current diff

Agent output (excerpt):

```yaml
test_routing_decision:
  input_mode: git_diff
  platforms: [android]
  change_type: mixed
  recommended_test_level: maestro     # the UI part; the pure-logic part goes to unit_test
  automation_runner: maestro
  reason: "LoginScreen adds an OTP field (testTag login_otp) and an error hint (login_error) — UI behavior; PhoneFormatter.mask only changes string masking — pure logic."
  evidence:
    - "app/.../LoginScreen.kt — adds TextField(login_otp) + Text(login_error)"
    - "app/.../PhoneFormatter.kt — pure logic, recommend unit test"
```

Manual case: `qa/manual-cases/login-otp-error-001.md`
Feasibility: happy path `ready`; error-hint path `needs_test_data` (needs a phone +
OTP that triggers failure)

Maestro YAML: `qa/manual-cases/maestro-flows/login-otp-error-001.yaml`

```yaml
appId: com.example.app
---
- launchApp
- assertVisible: { id: "login_phone" }
- assertVisible: { id: "login_otp" }
- assertVisible: { id: "login_submit" }
- tapOn: { id: "login_phone" }
- inputText: "13800000000"
- tapOn: { id: "login_otp" }
- inputText: "123456"
- assertVisible: { id: "login_otp", text: "123456" }
# --- error path (needs_test_data; uncomment once the failing case is supplied) ---
# - tapOn: { id: "login_submit" }
# - extendedWaitUntil: { visible: { id: "login_error" }, timeout: 10000 }
# - assertVisible: { id: "login_error" }
```

> Note: Jetpack Compose's `testTag` is matched by the `id` selector in Maestro.

Auto-run (when a device is detected):
```bash
maestro list-devices                                                   # detect first
maestro test qa/manual-cases/maestro-flows/login-otp-error-001.yaml     # run if a device is online
```

### Example B: pure-logic change (YAML generation is refused)

You say:
> I changed PriceFormatter's amount-formatting logic, generate a Maestro test

Agent output:
```yaml
test_routing_decision:
  change_type: pure_logic
  recommended_test_level: unit_test
  automation_runner: none
  reason: "PriceFormatter is a pure formatting function with no UI entry point; not suited to Maestro."
```
→ **No YAML generated**; recommends a unit test instead (e.g. `format(1234.5) == "¥1,234.50"`).

---

## 8. Auto-run & failure triage

When the YAML is `ready` and a device is online, the agent **auto-runs** — no manual
click in Maestro Studio:
```bash
maestro list-devices                                          # detect device (the gate for running)
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml     # device online → run end-to-end
maestro start-device                                          # suggested when none is online
```

When no device is online it degrades: only prints the run command and prompts you to
start a device/emulator or connect a real device — it won't hard-run.
After MCP is registered, the agent also inspects the live UI hierarchy before
generating YAML to calibrate selectors (see `references/run-and-mcp.md`).

On failure the agent reads logs / screenshots and classifies the cause:

| Failure cause | Handling |
|---|---|
| Real App / Web bug | Report it; **don't** paper over with YAML changes |
| Selector issue | Switch to a more stable selector |
| Timing issue | Add / adjust `waitForAnimationToEnd`, `extendedWaitUntil` |
| Test-data issue | Fix the precondition data, **don't** change the assertion |
| Environment issue | Note the environment requirement |

Iron law: **only edit the YAML when the evidence is clear; never delete core
assertions to force a pass.**

---

## 9. The four automation-feasibility states

| State | Meaning | Result |
|---|---|---|
| `ready` | Account, data, and stable selectors all present | Generate YAML |
| `blocked` | A hard dependency is missing | Don't generate; explain what's missing |
| `needs_selector` | UI lacks a stable selector (id/testTag/data-testid…) | Don't generate; recommend adding a selector first |
| `needs_test_data` | Missing test account / server-side data | Don't generate; explain the data to supply |

---

## 10. Selector priority (avoid coordinate taps)

Prefer stable anchors, priority decreasing top to bottom:

```
text / id / accessibilityLabel / contentDescription / testTag / data-testid
```

- Android Compose: `Modifier.testTag("xxx")` → use `id: "xxx"` in Maestro
- iOS: `accessibilityIdentifier` / `accessibilityLabel`
- Web: `data-testid` / stable `text`
- ❌ Avoid `point:` coordinate taps (they break the moment screen size changes)

---

## 11. FAQ

**Q: Why not generate YAML straight from the diff?**
A: Many changes aren't UI behavior at all (pure logic); force-generated YAML has no
value and is misleading. Deciding first avoids waste and ensures a human-runnable case
is left behind.

**Q: Can I use it without Maestro installed in the project?**
A: Yes. The skill still produces the Test Routing Decision + manual case + (when
feasible) YAML text — it just won't actually execute.

**Q: Why are there commented-out steps in the YAML?**
A: When a path is `needs_test_data` / `needs_selector`, the assertions are kept as
comments (rather than deleted); uncomment them once you've supplied the dependency.

**Q: Does it conflict with an existing test directory?**
A: No. If the project already has a case / Maestro directory, the skill prefers the
existing one.

---

## 12. Reference file index

| File | Content |
|---|---|
| `SKILL.md` | Core workflow, hard rules, output order |
| `references/workflow.md` | Input handling and evidence gathering for modes A / B |
| `references/decision-rules.md` | What suits / doesn't suit Maestro |
| `references/manual-case-template.md` | Manual case template and how to fill it |
| `references/maestro-yaml-rules.md` | Maestro YAML authoring rules and failure triage |
| `references/run-and-mcp.md` | MCP registration, device detection, auto-run, failure triage |
