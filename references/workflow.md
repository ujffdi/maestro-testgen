# Workflow & Input Modes

Two supported input modes. Detect which one applies, then gather evidence before
emitting the Test Routing Decision.

## Mode A — based on git diff

The user says things like "根据当前 diff 生成 Maestro 测试" / "根据这次改动生成测试用例".

Read the change first:

```bash
git diff --name-only      # which files changed
git diff -U0              # the actual change hunks (minimal context)
```

Then, as needed, read the related sources to understand the UI impact:

- Routes / navigation graphs
- Android: Activity, Fragment, ViewModel, Jetpack Compose, layout XML
- iOS: SwiftUI views, UIKit view controllers
- Web: React / Vue pages and components
- Existing Maestro flows (to reuse selectors and conventions)

Map each changed file to: does it produce or alter a user-visible behavior?
Feed that into `decision-rules.md`.

## Mode B — based on natural-language description

The user describes a business flow, e.g.:

- "给视频房送礼业务生成测试报告和 YAML"
- "给登录流程生成 Maestro 测试"
- "测试个人中心编辑资料"

Steps:

1. Locate the relevant business code from the description (search for the screen,
   route, component, or feature keywords).
2. Read existing tests / Maestro flows for that area.
3. Then proceed to the Test Routing Decision, manual case, and (if feasible) YAML.

## After evidence gathering (both modes)

Proceed through the SKILL.md workflow:
Test Routing Decision → manual case → feasibility → Maestro YAML → run command.
