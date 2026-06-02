# Manual Test Case Template

Always produce this document first. Save to `qa/manual-cases/<case_id>.md` (or the
project's existing test-case directory if one exists). Use a clear, stable
`case_id` (e.g. `login-phone-otp-001`).

```markdown
# 人工测试用例：<测试标题>

## 基本信息
- 用例 ID：
- 平台：
- 关联变更或业务：
- 测试类型：
- 优先级：

## 测试目标

## 前置条件
- 测试账号：
- 登录状态：
- 环境：
- 服务端数据：
- 权限状态：
- 起始状态 / 幂等性：     # 如「当前语言=阿拉伯语（非目标语言）」——开关类用例常非幂等
- 其他依赖：

## 清理 / 复位 (Teardown)
- 运行后残留状态：       # 如「语言已变英文、昵称已改为 Test Test」
- 复位步骤：             # 重跑前如何恢复前置条件（无则写「无需复位」）

## 测试步骤
1. 
2. 
3. 

## 预期结果
1. 
2. 

## 风险点

## 自动化可行性
- 状态：ready / blocked / needs_selector / needs_test_data
- 原因：
```

## Filling guidance

- **测试步骤** must be concrete and reproducible by a human (no ambiguous "verify it
  works").
- **起始状态 / 幂等性** — toggles (settings switches, language pickers, follow/block)
  are often **not idempotent**: e.g. a language picker's Confirm button only appears
  when the chosen value differs from the current one, so the test must start from a
  non-target value. State the required start value, and whether re-running needs a reset.
- **清理 / 复位** — if the flow mutates account/server state, record what it leaves
  behind and how to restore it. The run report should warn the user about residual
  state and offer to reset.
- **预期结果** are the assertions—these map directly to Maestro `assertVisible` etc.
  Never weaken or drop them to ease automation.
- **自动化可行性** drives whether YAML is generated:
  - `ready` — accounts, data, and stable selectors all exist → generate YAML.
  - `blocked` — a hard dependency is missing → no YAML; state what's needed.
  - `needs_selector` — UI lacks stable selectors (id/testTag/data-testid/etc.).
  - `needs_test_data` — required server-side data / test account missing.
