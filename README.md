# maestro-testgen

一个根据 **git diff** 或 **自然语言描述** 推导 UI 行为回归测试的 skill：先产出「人工测试用例留存文档」，仅在自动化确实可行时再生成可运行的 **Maestro YAML flow**。面向 Android / iOS / Web 的 UI 行为回归测试。

核心理念：**先决策 → 先留存人工用例 → 再谈自动化**。

## 能做 / 不做

| 能做 | 不做 |
|---|---|
| 判断一次改动是否属于「用户可见 UI 行为」 | 不直接从 diff 一把梭生成 YAML |
| 先写一份人工测试用例（可留存、可人工执行） | 不为纯逻辑（Mapper/Service/排序/格式化…）生成 Maestro |
| 自动化可行时生成 Maestro YAML（稳定 selector，不用坐标） | 不会为了让测试通过而删除核心断言 |
| 给出运行命令；CLI 可用时还能跑并分析失败原因 | 不会在证据不足时硬编 YAML |

## 目录结构

```
maestro-testgen/
├── SKILL.md            # 必需，含 name + description
├── references/         # 4 个参考文件（按需加载）
│   ├── workflow.md             # 输入模式 A(git diff) / B(自然语言)，该读什么
│   ├── decision-rules.md       # 什么适合 Maestro，什么不适合
│   ├── manual-case-template.md # 人工测试用例格式
│   └── maestro-yaml-rules.md   # Maestro YAML 编写规则
├── agents/openai.yaml  # Codex UI 元数据
└── 使用教程.md          # 项目使用文档（非标准 skill 结构，分发时可排除）
```

## 安装

把整个目录拷到对应 agent 的 skills 位置即可：

```bash
# Claude Code（项目级）
cp -R maestro-testgen <你的项目>/.claude/skills/
```

详细用法见 [使用教程.md](./使用教程.md)。

## 工作流（始终按此顺序）

1. 检测输入模式（git diff vs 自然语言）
2. 收集证据（读改动/相关代码）
3. 输出 Test Routing Decision（区分 UI 行为 vs 纯逻辑）
4. 写人工测试用例文档（**主交付物**）
5. 判断自动化可行性
6. 仅当 `ready` 时生成 Maestro YAML
7. 给出运行命令，或说明 blocker 及需补充的信息
