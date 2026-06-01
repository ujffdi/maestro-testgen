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
```

## Web note

For web, launch via the configured URL/browser target instead of `appId`, and
prefer `data-testid` / `text` selectors.

## Running & failure triage

```bash
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml
```

On failure, read logs + screenshots and classify the root cause:
- **App/Web bug** — report it; do not paper over with YAML changes.
- **Selector issue** — switch to a more stable selector.
- **Timing issue** — add/adjust waits.
- **Test data issue** — fix the precondition data, not the assertion.
- **Environment issue** — note the env requirement.

Only edit the YAML when the evidence is clear, and never by removing core assertions.
