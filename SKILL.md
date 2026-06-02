---
name: maestro-testgen
description: Use when generating UI behavior regression tests from git diff or natural-language QA requests, producing manual test cases and Maestro YAML flows for Android, iOS, or Web.
---

# Maestro Testgen

Derive the UI behaviors that need regression coverage from a **git diff** or a
**natural-language description**, write a durable **manual test case document**,
and—only when automation is actually feasible—generate a runnable **Maestro YAML
flow** for Android / iOS / Web.

## Hard rules (never violate)

1. **Never generate Maestro YAML straight from a git diff.** Always reason first.
2. **First decide whether the change is user-visible UI behavior.** If not, stop.
3. **Always produce the manual test case document first.** It is the primary deliverable.
4. **Generate Maestro YAML only when automation feasibility is `ready`.**
5. If the change is pure logic (pure functions, Mapper, Repository, Service, DTO,
   sorting/filtering, formatting, SDK wrappers, network/cache layers with no UI
   entry point), **stop—do not generate Maestro YAML**; recommend unit or
   integration tests instead.
6. **Never delete core assertions just to make a Maestro test pass.**
7. **Never default to coordinate taps.** Prefer stable selectors.
8. **Auto-run the flow only when a device is online.** Never run `maestro test`
   without a connected device/emulator/browser—detect first, otherwise degrade.

## Workflow (always in this order)

1. **Detect input mode** — git diff vs natural language. See `references/workflow.md`.
2. **Gather evidence** — read the changed/relevant code (pages, routes, Activity/
   Fragment, ViewModel, Compose/XML, SwiftUI/UIKit, React/Vue, existing Maestro
   flows). See `references/workflow.md`. When a device is online, you may use the
   Maestro MCP tools (or `maestro hierarchy`) to inspect the live view hierarchy
   and pick stable selectors. See `references/run-and-mcp.md`.
3. **Emit the Test Routing Decision** (see schema below). Apply
   `references/decision-rules.md` to classify UI behavior vs pure logic.
4. **Write the manual test case** using `references/manual-case-template.md`.
   Save to `qa/manual-cases/<case_id>.md` (or the project's existing case dir).
5. **Judge automation feasibility** (`ready` / `blocked` / `needs_selector` /
   `needs_test_data`).
6. **If `ready`, generate the Maestro YAML** per `references/maestro-yaml-rules.md`.
   Save to `qa/manual-cases/maestro-flows/<case_id>.yaml` (or the project's existing
   Maestro dir, e.g. `maestro/flows/`).
7. **Auto-run the flow and emit a report.** Detect a device with `maestro list-devices`.
   If one is online, run the flow **with a report format** (Maestro defaults to `NOOP`,
   i.e. no report file, so always pass `--format`):
   `maestro test <path> --format HTML-DETAILED --output qa/manual-cases/reports/<case_id>.html --test-output-dir qa/manual-cases/evidence/<case_id>`
   (use `--format JUNIT` for CI; `--test-output-dir` puts screenshots/artifacts in the qa
   dir, not `~/.maestro/tests/`). Report pass/fail with logs, the saved report path and
   the evidence dir;
   on failure, triage per `references/run-and-mcp.md`. If no device is online, **degrade**:
   print the run command and tell the user to start a device (`maestro start-device`)
   first—do not run. If not `ready`, explain the blocker and what to supply.

## Test Routing Decision (emit this first, always)

```yaml
test_routing_decision:
  input_mode: git_diff | natural_language
  platforms:
    - android | ios | web | unknown
  change_type: ui_behavior | pure_logic | mixed | unknown
  recommended_test_level: maestro | unit_test | integration_test | manual_review
  automation_runner: maestro | none
  reason: "<why>"
  evidence:
    - "<file path or description clue>"
```

## Required output order

1. Test Routing Decision
2. Manual test case content + save path
3. Automation feasibility judgment
4. Maestro YAML content + save path
5. Auto-run result (pass/fail + logs) when a device is online, or the run command
   plus a "start a device first" note when none is online
6. Test report path (`qa/manual-cases/reports/`) and evidence/screenshot dir
   (`qa/manual-cases/evidence/<case_id>/`) when a device is online
7. If no YAML: the `blocked` reason and what info is still needed

## Running Maestro (CLI + MCP)

Running is automatic once the YAML is `ready` and a device is online. The detect →
run → degrade flow, the one-time Maestro MCP registration, MCP-driven selector
calibration, and failure triage all live in `references/run-and-mcp.md`.

```bash
maestro list-devices                                          # detect first
# run if a device is online — always pass --format (default is NOOP = no report):
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml \
  --format HTML-DETAILED --output qa/manual-cases/reports/<case_id>.html \
  --test-output-dir qa/manual-cases/evidence/<case_id>        # screenshots/artifacts → qa dir
maestro start-device                                          # suggest this when none online
```

On failure: read logs/screenshots, classify the cause (App/Web bug vs selector vs
timing vs test data vs environment), and only edit the YAML when the evidence is
clear. Never strip core assertions to force a pass.

## Reference files (load on demand)

- `references/workflow.md` — input modes A (git diff) / B (natural language), what to read.
- `references/decision-rules.md` — what suits Maestro vs what doesn't.
- `references/manual-case-template.md` — the manual test case format.
- `references/maestro-yaml-rules.md` — Maestro YAML authoring rules.
- `references/run-and-mcp.md` — MCP setup, device detection, auto-run, failure triage.
