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

## Gather selector evidence in one batched pass (don't drip greps)

The static facts a flow needs — appId, the `onClick`/`android:id` of each touched
view, and the per-locale `R.string` for any text selector — are all independent.
Pull them in **one batched command**, not N sequential greps. e.g. for Android:

```bash
# appId
grep -rn "applicationId" app/build.gradle*
# ids behind the onClick handlers you'll tap (repeat -e per handler)
grep -rn -A2 -e "openLanguage\|openEdit\|editName" <module>/src/main/res/layout/
# the dialog/input layout ids + per-locale strings in one shot
grep -rn "android:id\|EditText\|Button" <module>/src/main/res/layout/dialog_input.xml
for k in setting language confirm edit; do
  grep -rh "name=\"$k\">" lib/src/main/res/values/strings.xml lib/src/main/res/values-ar/strings.xml
done
```

Read the referenced source files together (one batch of Read calls), then move on.

## After evidence gathering (both modes)

Proceed through the SKILL.md workflow:
Test Routing Decision → manual case → feasibility → Maestro YAML → run command.
