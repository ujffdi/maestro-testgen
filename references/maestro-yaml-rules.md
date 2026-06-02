# Maestro YAML Rules

Generate a flow **only when automation feasibility is `ready`**. Android, iOS, and
Web are all supported as long as the project and target platform support Maestro.

## Authoring rules

- Each flow covers **one clear UI behavior**. Don't bundle unrelated paths.
- Include a correct launch target: `appId` for Android/iOS, or the Web URL/launch
  config for web flows.
- **Prefer stable selectors**: `text`, `id`, `accessibilityLabel`,
  `contentDescription`, `testTag`, `data-testid`. **Avoid coordinate taps.**
- Add necessary waits: `waitForAnimationToEnd`, `extendedWaitUntil`, and
  `assertVisible` before interacting / asserting.
- **Do not hardcode YAML** when test accounts, server-side data, or stable selectors
  are missing—report the blocker instead (see feasibility states).
- Keep the assertions from the manual case's 预期结果. Never delete core assertions
  to force a pass.
- **Save screenshots into the qa dir, not the default Maestro temp.** After the key
  assertion(s), capture a durable evidence shot with
  `takeScreenshot: qa/manual-cases/evidence/<case_id>/<step>` (path is relative to the
  project root; `.png` is appended automatically). This keeps PASS evidence next to the
  case—`~/.maestro/tests/` only holds Maestro's auto failure shots and is purged after
  14 days.

## Save path

`qa/manual-cases/maestro-flows/<case_id>.yaml` (or the project's existing Maestro
directory, e.g. `maestro/flows/<case_id>.yaml`).

## Example (Android login)

```yaml
appId: com.example.app
---
- launchApp
- assertVisible:
    id: "btn_login"
- tapOn:
    id: "input_phone"
- inputText: "13800000000"
- tapOn:
    id: "input_otp"
- inputText: "123456"
- tapOn:
    text: "登录"
- extendedWaitUntil:
    visible:
      id: "home_tab"
    timeout: 10000
- assertVisible:
    id: "home_tab"
- takeScreenshot: qa/manual-cases/evidence/<case_id>/home-after-login   # PASS evidence into qa dir
```

## Web note

For web, launch via the configured URL/browser target instead of `appId`, and
prefer `data-testid` / `text` selectors.

## Running & failure triage

Running is automatic once the flow is `ready` and a device is online: detect with
`maestro list-devices`, then run **with a report format** —
`maestro test <path> --format HTML-DETAILED --output qa/manual-cases/reports/<case_id>.html
--test-output-dir qa/manual-cases/evidence/<case_id>` (Maestro defaults to `NOOP`, i.e.
no report, so always pass `--format`; use `JUNIT` for CI; `--test-output-dir` routes
screenshots/artifacts into the qa dir instead of `~/.maestro/tests/`). If no device is
online, degrade to printing the command. The full detect → run →
degrade flow, Maestro MCP setup, selector calibration, and failure-triage rules live in
`references/run-and-mcp.md`.

Always: only edit the YAML when the evidence is clear, and never by removing core
assertions just to force a pass.
