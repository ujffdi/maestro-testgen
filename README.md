# maestro-testgen

A skill that derives UI-behavior regression tests from a **git diff** or a
**natural-language description**: it first produces a durable **manual test case
document**, and only when automation is actually feasible generates a runnable
**Maestro YAML flow**. Targets UI-behavior regression for Android / iOS / Web.

Core idea: **decide first → keep a manual case → then automate**.

> 一个根据 **git diff** 或 **自然语言描述** 推导 UI 行为回归测试的 skill：先产出「人工测试用例留存文档」，仅在自动化确实可行时再生成可运行的 **Maestro YAML flow**。面向 Android / iOS / Web 的 UI 行为回归测试。
>
> 核心理念：**先决策 → 先留存人工用例 → 再谈自动化**。

## Does / Doesn't · 能做 / 不做

| Does · 能做 | Doesn't · 不做 |
|---|---|
| Judge whether a change is user-visible UI behavior<br>判断一次改动是否属于「用户可见 UI 行为」 | Generate YAML straight from a diff<br>不直接从 diff 一把梭生成 YAML |
| Write a durable, human-runnable manual test case first<br>先写一份人工测试用例（可留存、可人工执行） | Generate Maestro for pure logic (Mapper/Service/sort/format…)<br>不为纯逻辑（Mapper/Service/排序/格式化…）生成 Maestro |
| Generate Maestro YAML when feasible (stable selectors, no coordinates)<br>自动化可行时生成 Maestro YAML（稳定 selector，不用坐标） | Delete core assertions just to make a test pass<br>不会为了让测试通过而删除核心断言 |
| Auto-run YAML via CLI when a device is online & triage failures; inspect the live hierarchy via MCP to calibrate selectors<br>有设备在线时用 CLI 自动跑 YAML 并分析失败；用 MCP 探查实时层级校准 selector | Hardcode YAML when evidence is insufficient<br>不会在证据不足时硬编 YAML |
| Degrade when no device is online: print the run command + prompt to start a device<br>无设备在线时降级：只给运行命令并提示先启动设备 | Hard-run `maestro test` with no device<br>不会在没有设备时硬跑 `maestro test` |

## Directory layout · 目录结构

```
maestro-testgen/
├── SKILL.md            # required, contains name + description · 必需，含 name + description
├── references/         # 5 on-demand reference files · 5 个参考文件（按需加载）
│   ├── workflow.md             # input modes A(git diff) / B(natural language) · 输入模式与该读什么
│   ├── decision-rules.md       # what suits Maestro vs not · 什么适合 Maestro，什么不适合
│   ├── manual-case-template.md # manual test case format · 人工测试用例格式
│   ├── maestro-yaml-rules.md   # Maestro YAML authoring rules · Maestro YAML 编写规则
│   └── run-and-mcp.md          # MCP setup, device detect, auto-run, triage · MCP 注册、设备探测、自动运行、失败归因
├── agents/openai.yaml  # Codex UI metadata · Codex UI 元数据
└── USAGE.md            # usage guide (non-standard skill file, excludable when distributing) · 项目使用文档（非标准 skill 结构，分发时可排除）
```

## Install · 安装

Copy the whole directory into the agent's skills location.
把整个目录拷到对应 agent 的 skills 位置即可：

```bash
# Claude Code (project scope) · Claude Code（项目级）
cp -R maestro-testgen <your-project>/.claude/skills/
```

For auto-run, also install the Maestro CLI and register the MCP server (one-time):
自动运行还需装 Maestro CLI 并注册 MCP（一次性）：

```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash   # Maestro CLI
claude mcp add -s project maestro -- maestro mcp      # register Maestro MCP · 注册 Maestro MCP
```

Full usage → [USAGE.md](./USAGE.md). · 详细用法见 [USAGE.md](./USAGE.md)。

## Workflow (always in this order) · 工作流（始终按此顺序）

1. Detect input mode (git diff vs natural language) · 检测输入模式（git diff vs 自然语言）
2. Gather evidence (read the change / related code) · 收集证据（读改动/相关代码）
3. Emit the Test Routing Decision (UI behavior vs pure logic) · 输出 Test Routing Decision（区分 UI 行为 vs 纯逻辑）
4. Write the manual test case document (**primary deliverable**) · 写人工测试用例文档（**主交付物**）
5. Judge automation feasibility · 判断自动化可行性
6. Generate Maestro YAML only when `ready` · 仅当 `ready` 时生成 Maestro YAML
7. Auto-run: `maestro list-devices` → if a device is online, `maestro test` end-to-end and triage failures; if none, degrade (print command + prompt `maestro start-device`). Otherwise explain the blocker and what to supply. · 自动运行：`maestro list-devices` 探测设备 → 有设备则 `maestro test` 跑全程并归因失败；无设备则降级（给命令 + 提示先 `maestro start-device`）。或说明 blocker 及需补充的信息
