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

Before/while authoring the YAML, inspect the live view hierarchy and prefer the most
stable selector you can see:

- Via MCP: launch the app and query the hierarchy through the Maestro MCP tools,
  then read `id` / `text` / `accessibilityLabel` / `contentDescription` / `testTag`
  / `data-testid` off real nodes.
- Via CLI: `maestro hierarchy` prints the connected device's hierarchy.

Use what you observe to replace guessed selectors. **Never fall back to coordinate
taps.** If no stable selector exists, that is a `needs_selector` blocker—report it.

## Step 3 — run the flow (CLI, full YAML)

```bash
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml
```

Report pass/fail with the relevant logs. On pass, you're done. On failure, triage.

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
