# Running Maestro: CLI + MCP

Once the YAML is `ready`, the skill **runs the flow automatically**—no manual click
in Maestro Studio. CLI runs the generated YAML end-to-end; the Maestro MCP server is
used to inspect the live UI and calibrate stable selectors.

## One-time setup: register the Maestro MCP server

Maestro ships an MCP server (`maestro mcp`) that exposes device + automation tools to
the agent. Register it once per machine (or per project so the repo carries it):

```bash
# project scope — writes .mcp.json into the repo so teammates pick it up
claude mcp add -s project maestro -- maestro mcp

# or user scope — available across all your projects
claude mcp add maestro -- maestro mcp
```

Verify: `claude mcp get maestro`. Requires the Maestro CLI on `PATH`
(`~/.maestro/bin/maestro`, install via `curl -fsSL https://get.maestro.dev | bash`).

If the MCP server is not registered, fall back to the CLI-only path below
(`maestro hierarchy` covers selector inspection).

## Step 1 — detect a device (gate)

Never auto-run without a connected device/emulator/browser. Detect first:

```bash
maestro list-devices
```

- **A device is listed** → proceed to run.
- **None listed** → **degrade** (see below). Do not run.

## Step 2 — calibrate selectors with MCP (when a device is online)

**Read selectors from layout XML / resource files FIRST**, fall back to live inspect
only for what static reads can't answer:

- Static first: layout XML (`android:id`, `onClick` bindings), string resources
  (`R.string`, per-locale), Compose `testTag`s. This is faster and cheaper than
  dumping a hierarchy and usually gives you the id/text directly.
- Live inspect only resolves: (a) no-id / dynamic elements, (b) position-dependent
  elements (RTL/LTR), (c) which screen you are actually on, (d) overlap surprises.
- When you do inspect, **ignore platform-chrome noise** — `com.android.systemui:*`
  (status bar) and IME packages (`com.google.android.inputmethod*`) are never your
  selectors. Only `<appId>:id/*` nodes matter. Match each abbreviated `txt`/`rid`
  off the **real node**, never off a screenshot.

Apply the selector ladder in `maestro-yaml-rules.md`; a documented point tap for a
truly id-less element is the bottom rung, not a blocker.

### Calibrate once, validate once (avoid traversing the flow 3×)

The expensive mistake is walking the whole flow via MCP **and then** re-running it
all via CLI — two full traversals plus any reset is 3-4× the work.

- Walk the screens via MCP **once**, only far enough to read each screen's
  selectors. Do **not** complete the flow MCP-by-MCP as your validation.
- Assemble the YAML, then let the **CLI run (Step 3) be the single validation
  traversal**.
- For state-mutating flows (settings toggles, profile edits, language switch):
  snapshot the precondition once, and if the MCP calibration already changed state,
  **reset once** to the precondition before the CLI run. Budget exactly two
  traversals (calibrate + validate), not four.

## Step 3 — run the flow (CLI, full YAML) and emit a report

Maestro's report format defaults to `NOOP` — **no report file is written**, only debug
logs under `~/.maestro/tests/<timestamp>/`. **Always pass `--format`** so a durable,
archivable report is produced:

```bash
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml \
  --format HTML-DETAILED --output qa/manual-cases/reports/<case_id>.html \
  --test-output-dir qa/manual-cases/evidence/<case_id>
# CI: use --format JUNIT --output qa/manual-cases/reports/<case_id>.xml
```

Formats: `HTML-DETAILED` / `HTML` (human review), `JUNIT` (CI), `NOOP` (default, none).

**Screenshots go in the qa dir, not Maestro's temp.** `--test-output-dir
qa/manual-cases/evidence/<case_id>` routes screenshots and test artifacts there; combine
with in-flow `takeScreenshot: qa/manual-cases/evidence/<case_id>/<step>` for durable PASS
evidence. Without this, shots land under `~/.maestro/tests/<timestamp>/` and are purged
after 14 days.

Report pass/fail with the relevant logs, the saved report path **and the evidence dir**.
On pass, you're done. On failure, triage.

## Degrade path (no device online)

Do not run. Instead:

1. Print the exact run command above.
2. Tell the user to start a device first:
   ```bash
   maestro start-device          # creates/starts an emulator or simulator
   ```
   (or connect a physical device / open the target browser), then re-run.

## Failure triage

On failure, read logs + screenshots and classify the root cause:

- **App/Web bug** — report it; do not paper over with YAML changes.
- **Selector issue** — re-inspect the hierarchy (MCP / `maestro hierarchy`) and
  switch to a more stable selector.
- **Timing issue** — add/adjust waits (`waitForAnimationToEnd`, `extendedWaitUntil`).
- **Test data issue** — fix the precondition data, not the assertion.
- **Environment issue** — note the env requirement (device, account, network).

Only edit the YAML when the evidence is clear, and **never by removing core
assertions** just to force a pass.
