# Decision Rules: Maestro vs not

Use these to set `change_type` and `recommended_test_level` in the Test Routing
Decision. When in doubt, classify by whether there is a **user-visible UI entry
point** that exercises the change.

## Suitable for Maestro (UI behavior)

- Page / screen display changes
- Tap, input, tab switch, dialog, permission prompt changes
- Page navigation / routing changes
- User-path changes: login, home, profile, room, gifts, follow, chat, etc.
- ViewModel / State / Store / Controller state that ultimately affects what the
  user sees or can interact with

→ `change_type: ui_behavior` (or `mixed`), `recommended_test_level: maestro`,
`automation_runner: maestro`.

## NOT suitable for Maestro (pure logic)

- Pure function computation
- DTO / Entity / Mapper transformations
- Repository / Service internal logic
- Sorting, filtering, formatting
- SDK wrappers / encapsulation
- Network layer, caching strategy, and other logic with no direct UI entry point

→ `change_type: pure_logic`, `recommended_test_level: unit_test` or
`integration_test`, `automation_runner: none`. **Stop—do not generate Maestro YAML.**
Recommend unit/integration tests instead.

## Mixed changes

If a change has both UI and pure-logic parts, set `change_type: mixed`: cover the
user-visible behavior with a manual case + Maestro (if `ready`), and route the
pure-logic part to unit/integration tests in the `reason`.
