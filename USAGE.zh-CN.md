# maestro-testgen 使用教程

[English](./USAGE.md) | **中文**

> 一个根据 **git diff** 或 **自然语言描述** 推导 UI 行为回归测试的 skill：
> 先产出「人工测试用例留存文档」，仅在自动化可行时再生成可运行的 **Maestro YAML flow**。
> 面向 Android / iOS / Web 的 UI 行为回归测试。
>
> 注：本文件是项目使用文档，**不属于标准 skill 结构**。打包分发标准 skill 时可以排除它。

---

## 1. 它能做什么 / 不做什么

| 能做 | 不做 |
|---|---|
| 判断一次改动是否属于「用户可见 UI 行为」 | 不直接从 diff 一把梭生成 YAML |
| 先写一份人工测试用例（可留存、可人工执行） | 不为纯逻辑（Mapper/Service/排序/格式化…）生成 Maestro |
| 自动化可行时生成 Maestro YAML（稳定 selector，不用坐标） | 不会为了让测试通过而删除核心断言 |
| 有设备在线时用 CLI 自动跑 YAML 并归因失败；用 MCP 探查实时层级校准 selector | 不会在证据不足时硬编 YAML |
| 无设备在线时降级：只给运行命令并提示先启动设备 | 不会在没有设备时硬跑 `maestro test` |

核心理念：**先决策 → 先留存人工用例 → 再谈自动化**。

---

## 2. 安装 / 部署

这个目录本身就是标准 skill 源目录：

```
maestro-testgen/
├── SKILL.md            # 必需，含 name + description
├── references/         # 5 个参考文件（按需加载）
└── agents/openai.yaml  # Codex UI 元数据
```

按你的 agent 选择部署位置（把整个目录拷过去即可）：

```bash
# Claude Code（项目级）
cp -R maestro-testgen <你的项目>/.claude/skills/

# Claude Code（用户级，全局可用）
cp -R maestro-testgen ~/.claude/skills/

# Codex
cp -R maestro-testgen <你的项目>/.codex/skills/
```

部署后，在对应项目里直接用下面的「触发语」即可被自动调用。

安装 Maestro CLI（自动运行 YAML 必需）
```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash
maestro --version
```

可选：注册 Maestro MCP，让 agent 能探查实时 UI 层级、校准 selector
```bash
# 项目级（写入 .mcp.json，团队共享）
claude mcp add -s project maestro -- maestro mcp
# 或用户级
claude mcp add maestro -- maestro mcp
```

---

## 3. 触发方式

### 手动触发指令流程（从零到「测试已跑出结果」）

按顺序操作即可。第 0 步装一次，之后每次只需第 1～2 步。

**0. 一次性安装（每项目/机器一次）**
```bash
cp -R maestro-testgen <你的项目>/.claude/skills/        # ① 拷 skill
curl -fsSL "https://get.maestro.mobile.dev" | bash       # ② 装 Maestro CLI
claude mcp add -s project maestro -- maestro mcp          # ③ 注册 Maestro MCP
claude mcp get maestro                                    #    验证已连上
```

**1. 启动一个设备（每次跑测试前）**
```bash
maestro start-device     # 起模拟器/Simulator；或自己连真机/开浏览器
maestro list-devices     # 确认有设备在线（自动运行的前置门槛）
```

**2. 在 agent 对话里手动触发**（不是终端）
```text
/maestro-testgen 根据当前 diff 生成 Maestro 测试
```
触发后 agent 会**自动**走完：路由判断 → 人工用例落盘 → 可行性判断 → 生成 YAML（生成前用 MCP 探层级挑 selector）→ 探到设备就 `maestro test` 跑全程 → 回报 pass/fail + 日志。**无需再手动跑。**

**3.（可选）自己手动复跑同一条 flow / 接 CI**
```bash
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml
```

哪些手动、哪些自动：

| 步骤 | 谁来做 |
|---|---|
| 安装、注册 MCP、起设备 | **你手动**（终端） |
| `/maestro-testgen ...` 触发 | **你手动**（对话框一句话） |
| 生成用例 + YAML + 跑 `maestro test` + 归因失败 | **agent 自动** |
| 复跑 / 接 CI | 你手动（可选） |

> 没设备在线时：agent 不会硬跑，只把 `maestro test ...` 命令 + 「先 `maestro start-device`」提示给你，起完设备再触发一次即可。

### 触发语示例

直接用自然语言说出意图即可，例如：

- 「根据当前 diff 生成 Maestro 测试」
- 「根据这次改动生成测试用例」
- 「给登录流程生成 Maestro 用例」
- 「测试个人中心编辑资料」
- 「给视频房送礼业务生成测试报告和 YAML」
- 「分析这次改动是否需要 UI 自动化测试」

skill 支持两种输入模式，会自动识别：

- **模式 A — git diff**：基于当前改动。会读 `git diff --name-only` / `git diff -U0`，再按需读相关页面/路由/ViewModel/Compose/XML/SwiftUI/React 等。
- **模式 B — 自然语言**：基于业务描述。会先根据描述定位相关业务代码和已有测试，再生成用例与 YAML。

---

## 4. 输出顺序（固定）

每次调用，agent 会按这个顺序输出：

1. **Test Routing Decision**（测试路由判断）
2. **人工测试用例**内容 + 保存路径
3. **自动化可行性**判断
4. **Maestro YAML** 内容 + 保存路径（仅当可行性为 `ready`）
5. **自动运行结果**：有设备在线时自动 `maestro test` 跑全程并给出 pass/fail + 日志；无设备时降级，只给运行命令并提示先启动设备
6. 若未生成 YAML：说明 `blocked` 原因和需要补充的信息

### Test Routing Decision 长这样

```yaml
test_routing_decision:
  input_mode: git_diff | natural_language
  platforms: [android | ios | web | unknown]
  change_type: ui_behavior | pure_logic | mixed | unknown
  recommended_test_level: maestro | unit_test | integration_test | manual_review
  automation_runner: maestro | none
  reason: "<判断原因>"
  evidence:
    - "<文件路径或描述线索>"
```

---

## 5. 产物与保存路径

| 产物 | 默认路径 |
|---|---|
| 人工测试用例 | `qa/manual-cases/<case_id>.md` |
| Maestro flow | `qa/manual-cases/maestro-flows/<case_id>.yaml`（或项目已有 Maestro 目录，如 `maestro/flows/`） |

如果项目已有测试用例 / Maestro 目录，优先使用现有目录。

### 人工用例模板（节选）

```markdown
# 人工测试用例：<标题>
## 基本信息       # 用例 ID / 平台 / 关联变更 / 测试类型 / 优先级
## 测试目标
## 前置条件       # 测试账号 / 登录状态 / 环境 / 服务端数据 / 权限 / 其他依赖
## 测试步骤
## 预期结果       # ← 这些就是断言，禁止为自动化而弱化或删除
## 风险点
## 自动化可行性    # ready / blocked / needs_selector / needs_test_data + 原因
```

---

## 6. 决策规则速查

**适合 Maestro（UI 行为）**
- 页面展示变化；点击 / 输入 / Tab / 弹窗 / 权限弹窗变化
- 页面跳转变化；登录 / 首页 / 个人中心 / 房间 / 礼物 / 关注 / 聊天等用户路径
- ViewModel / State / Store / Controller 状态最终影响 UI 展示或交互

**不适合 Maestro（纯逻辑）→ 转单元 / 集成测试**
- 纯函数计算；DTO / Entity / Mapper 转换
- Repository / Service 内部逻辑；排序 / 过滤 / 格式化
- SDK 封装；网络层 / 缓存策略等无直接 UI 入口的逻辑

混合改动标 `change_type: mixed`：UI 部分走人工用例 + Maestro，纯逻辑部分在 `reason` 里建议走单测 / 集成测试。

---

## 7. 完整示例

### 示例 A：基于 diff（Android 登录页新增验证码输入框）

你说：
> 根据当前 diff 生成 Maestro 测试

agent 输出（节选）：

```yaml
test_routing_decision:
  input_mode: git_diff
  platforms: [android]
  change_type: mixed
  recommended_test_level: maestro     # UI 部分；纯逻辑部分走 unit_test
  automation_runner: maestro
  reason: "LoginScreen 新增 OTP 输入框(testTag login_otp)与错误提示(login_error)，属 UI 行为；PhoneFormatter.mask 仅改字符串掩码，纯逻辑。"
  evidence:
    - "app/.../LoginScreen.kt — 新增 TextField(login_otp) + Text(login_error)"
    - "app/.../PhoneFormatter.kt — 纯逻辑，建议单测"
```

人工用例：`qa/manual-cases/login-otp-error-001.md`
可行性：正向路径 `ready`；错误提示路径 `needs_test_data`（需要会触发失败的手机号+验证码）

Maestro YAML：`qa/manual-cases/maestro-flows/login-otp-error-001.yaml`

```yaml
appId: com.example.app
---
- launchApp
- assertVisible: { id: "login_phone" }
- assertVisible: { id: "login_otp" }
- assertVisible: { id: "login_submit" }
- tapOn: { id: "login_phone" }
- inputText: "13800000000"
- tapOn: { id: "login_otp" }
- inputText: "123456"
- assertVisible: { id: "login_otp", text: "123456" }
# --- 错误路径（needs_test_data，补齐失败用例后取消注释） ---
# - tapOn: { id: "login_submit" }
# - extendedWaitUntil: { visible: { id: "login_error" }, timeout: 10000 }
# - assertVisible: { id: "login_error" }
```

> 注：Jetpack Compose 的 `testTag` 在 Maestro 里用 `id` selector 匹配。

自动运行（检测到设备时）：
```bash
maestro list-devices                                                   # 先探测
maestro test qa/manual-cases/maestro-flows/login-otp-error-001.yaml     # 有设备则自动跑
```

### 示例 B：纯逻辑改动（会被拒绝生成 YAML）

你说：
> 我改了 PriceFormatter 的金额格式化逻辑，生成 Maestro 测试

agent 输出：
```yaml
test_routing_decision:
  change_type: pure_logic
  recommended_test_level: unit_test
  automation_runner: none
  reason: "PriceFormatter 是纯格式化函数，无 UI 入口，不适合 Maestro。"
```
→ **不生成 YAML**，改为建议单元测试（例如 `format(1234.5) == "¥1,234.50"`）。

---

## 8. 自动运行与失败排查

YAML 为 `ready` 且有设备在线时，agent 会**自动运行**——无需在 Maestro Studio 手动点击：
```bash
maestro list-devices                                          # 探测设备（运行前置门槛）
maestro test qa/manual-cases/maestro-flows/<case_id>.yaml     # 有设备 → 自动跑全程
maestro start-device                                          # 无设备时提示先执行
```

无设备在线时降级：只给出运行命令并提示先启动设备/模拟器或连接真机，不会硬跑。
MCP 注册后，agent 还会在生成 YAML 前探查实时 UI 层级来校准 selector（详见 `references/run-and-mcp.md`）。

失败时 agent 会读日志 / 截图，并把原因归类：

| 失败原因 | 处理方式 |
|---|---|
| App / Web 真 bug | 上报，**不**用改 YAML 掩盖 |
| selector 问题 | 换更稳定的 selector |
| 等待时机问题 | 加 / 调 `waitForAnimationToEnd`、`extendedWaitUntil` |
| 测试数据问题 | 修前置数据，**不**改断言 |
| 环境问题 | 标注环境要求 |

铁律：**只在证据明确时改 YAML；绝不删核心断言来制造通过。**

---

## 9. 自动化可行性四态

| 状态 | 含义 | 结果 |
|---|---|---|
| `ready` | 账号、数据、稳定 selector 齐全 | 生成 YAML |
| `blocked` | 硬依赖缺失 | 不生成，说明缺什么 |
| `needs_selector` | UI 缺稳定 selector（id/testTag/data-testid…） | 不生成，建议先加 selector |
| `needs_test_data` | 缺测试账号 / 服务端数据 | 不生成，说明需补的数据 |

---

## 10. selector 优先级（避免坐标点击）

优先用稳定锚点，从上到下优先级递减：

```
text / id / accessibilityLabel / contentDescription / testTag / data-testid
```

- Android Compose：`Modifier.testTag("xxx")` → Maestro 用 `id: "xxx"`
- iOS：`accessibilityIdentifier` / `accessibilityLabel`
- Web：`data-testid` / 稳定 `text`
- ❌ 避免 `point:` 坐标点击（屏幕尺寸一变就挂）

---

## 11. FAQ

**Q：为什么不直接从 diff 生成 YAML？**
A：很多改动根本不是 UI 行为（纯逻辑），硬生成的 YAML 没有价值且会误导。先决策能避免浪费，也保证留下可人工执行的用例。

**Q：项目没装 Maestro 还能用吗？**
A：能。skill 仍会产出 Test Routing Decision + 人工用例 + （可行时）YAML 文本，只是不会真正执行。

**Q：YAML 里为什么有注释掉的步骤？**
A：当某条路径是 `needs_test_data` / `needs_selector` 时，断言会以注释保留（而非删除），等你补齐依赖后取消注释即可启用。

**Q：和现有测试目录冲突吗？**
A：不会。如果项目已有用例 / Maestro 目录，skill 会优先用现有目录。

---

## 12. 参考文件索引

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 核心流程、硬规则、输出顺序 |
| `references/workflow.md` | 模式 A / B 的输入处理与取证 |
| `references/decision-rules.md` | 适合 / 不适合 Maestro 的判定 |
| `references/manual-case-template.md` | 人工用例模板与填写指引 |
| `references/maestro-yaml-rules.md` | Maestro YAML 编写规则与失败排查 |
| `references/run-and-mcp.md` | MCP 注册、设备探测、自动运行、失败归因 |
