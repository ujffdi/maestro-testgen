# maestro-testgen

[English](./README.md) | **中文**

一个根据 **git diff** 或 **自然语言描述** 推导 UI 行为回归测试的 skill：先产出「人工测试用例留存文档」，仅在自动化确实可行时再生成可运行的 **Maestro YAML flow**。面向 Android / iOS / Web 的 UI 行为回归测试。

核心理念：**先决策 → 先留存人工用例 → 再谈自动化**。

## 能做 / 不做

| 能做 | 不做 |
|---|---|
| 判断一次改动是否属于「用户可见 UI 行为」 | 不直接从 diff 一把梭生成 YAML |
| 先写一份人工测试用例（可留存、可人工执行） | 不为纯逻辑（Mapper/Service/排序/格式化…）生成 Maestro |
| 自动化可行时生成 Maestro YAML（稳定 selector，不用坐标） | 不会为了让测试通过而删除核心断言 |
| 有设备在线时用 CLI 自动跑 YAML 并分析失败；用 MCP 探查实时层级校准 selector | 不会在证据不足时硬编 YAML |
| 无设备在线时降级：只给运行命令并提示先启动设备 | 不会在没有设备时硬跑 `maestro test` |

## 目录结构

```
maestro-testgen/
├── .claude-plugin/     # plugin.json + marketplace.json（本仓自托管 marketplace）
├── .mcp.json           # 内置 Maestro MCP server（安装时自动注册）
├── SKILL.md            # 必需，含 name + description
├── references/         # 5 个参考文件（按需加载）
│   ├── workflow.md             # 输入模式 A(git diff) / B(自然语言)，该读什么
│   ├── decision-rules.md       # 什么适合 Maestro，什么不适合
│   ├── manual-case-template.md # 人工测试用例格式
│   ├── maestro-yaml-rules.md   # Maestro YAML 编写规则
│   └── run-and-mcp.md          # MCP 注册、设备探测、自动运行、失败归因
├── agents/openai.yaml  # Codex UI 元数据
└── USAGE.md            # 项目使用文档（非标准 skill 结构，分发时可排除）
```

## 安装

### 作为插件安装（推荐）

```text
/plugin marketplace add ujffdi/maestro-testgen
/plugin install maestro-testgen@maestro-testgen
```

插件内置 `.mcp.json`，安装时会**自动注册** Maestro MCP server，无需手动 `claude mcp add`。

### 手动安装（拷贝 skill）

```bash
# Claude Code（项目级）
cp -R maestro-testgen <你的项目>/.claude/skills/
# 然后自行注册 MCP server：
claude mcp add -s project maestro -- maestro mcp
```

两种方式都需安装 Maestro CLI（自动运行 YAML 必需）：

```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash
```

详细用法见 [USAGE.zh-CN.md](./USAGE.zh-CN.md)。

## 工作流（始终按此顺序）

1. 检测输入模式（git diff vs 自然语言）
2. 收集证据（读改动/相关代码）
3. 输出 Test Routing Decision（区分 UI 行为 vs 纯逻辑）
4. 写人工测试用例文档（**主交付物**）
5. 判断自动化可行性
6. 仅当 `ready` 时生成 Maestro YAML
7. 自动运行：`maestro list-devices` 探测设备 → 有设备则 `maestro test` 跑全程并归因失败；无设备则降级（给命令 + 提示先 `maestro start-device`）。或说明 blocker 及需补充的信息
