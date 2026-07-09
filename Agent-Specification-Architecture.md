# ASA — Agent Specification Architecture

## 核心思想

> **Skill 不是 Prompt，而是一份可编译的 Agent Specification。**

以前大家认为 Skill 就是一段 Prompt：

```
User → Prompt → LLM
```

ASA 把它变成了：

```
User → Agent Specification → Platform Adapter → LLM Runtime
```

**Prompt 不再是最终产物。Prompt 是编译结果。** Specification 是源码——它描述 Agent 应该怎样工作，而不是 Claude/Codex/Gemini 怎么工作。同一个 Spec，换个 Adapter 就能编译到不同平台，业务逻辑完全一致。

ASA 的核心理念浓缩成一句话：

> **用 Specification 描述 Agent 的能力，用 Adapter 适配不同模型，用 Runtime 执行任务。**

---

## 为什么需要 ASA

传统写 Skill 的方式是一段自然语言 Prompt，角色、步骤、约束、示例、故障排查全部塞在一起。三个致命问题：

| 问题 | 后果 |
|------|------|
| **注意力衰减** | Prompt 超过 500 行后，LLM 开始遗忘前面的指令——Transformer 的 Attention 会随距离衰减 |
| **Policy 和 Implementation 混杂** | 改一条 PowerShell 约束要翻遍全文。策略和脚本耦合在一起，维护成本高 |
| **不能跨平台** | Claude Code 的 SKILL.md 放到 OpenAI Codex / Gemini Gem 基本没法用——每个平台有自己的格式 |

ASA 的解法：**分层。** 把 Policy（做什么）、State Machine（在哪一步）、Knowledge（怎么做）、Tool（用什么做）拆到不同层。主文件只放策略，具体实现按需加载。

---

## 七层模型

```
Agent Specification（平台无关，唯一真源）
│
├── Identity        — 你是谁，唯一成功标准是什么
├── Policy          — 行为原则（抽象，不含平台命令）
├── State Machine   — 当前在哪，往哪走，怎么验证，失败了怎么修
├── Planner         — 执行前规划，执行中更新。Planner 不执行，执行器不规划
├── Verify Rules    — 完成前逐条检查（MUST / MUST NOT）
├── Knowledge       — 按需读取的决策支持，不是教程。回答"如果发生 XX 怎么办"
└── Tool Interface  — 能力声明（install_mysql()），不是脚本。Tool 不负责策略，Policy 决定何时调用
        │
        ▼
Platform Adapter    — 翻译层。Spec → Claude Markdown / Codex Instructions / Gemini Gem
        │
        ▼
LLM Runtime         — Claude / OpenAI / Google。不断变化，但 Spec 稳定
```

### 每一层的职责

**Specification（第一层）** — 唯一真源。描述 Agent 应该怎样工作，禁止出现 Claude、Codex、Gemini、Tool Calling Format、XML、Markdown——这些都是 Runtime 的事。

**Policy（第二层）** — 行为原则。回答"Agent 应该遵循什么原则？"禁止出现 PowerShell、bash、mysql——这些是 Implementation。Policy 永远是抽象的。

**State Machine（第三层）** — Agent 的生命周期。回答"Agent 当前在哪？"每个状态定义 Goal / Verify / Repair / Exit Condition。Agent 围绕状态变化工作，不是固定步骤。

**Planner（第四层）** — 职责分离。Planner 不负责执行，只输出 Execution Plan。执行器永远不要规划，Planner 永远不要执行。

**Knowledge（第五层）** — Decision Support。回答"如果发生 XX 怎么办？"而不是"一步一步安装"。格式：Problem → Diagnosis → Strategy → Fallback。不是 Tutorial。

**Tool（第六层）** — 能力声明。只写 Capability（Scan、Read、Run、Verify），不写 PowerShell Script。Tool 不负责策略，Policy 决定什么时候调用。

**Runtime Adapter（第七层）** — 翻译层。Specification → Claude Markdown / Codex Instructions / Gemini Gem。业务逻辑完全一致，只换壳。

---

## 三层设计原则

### Principle 1: Policy ≠ Implementation

Policy 层禁止出现具体工具名、平台命令、文件路径。

```
❌ "PowerShell Expand-Archive 解压 MySQL ZIP"
✅ "Install MySQL（实现方案见 knowledge/mysql.md）"
```

Policy 说**做什么**，Knowledge/Tool 说**怎么做**。改实现不碰 Policy，改 Policy 不受实现干扰。

### Principle 2: State > Workflow

Agent 围绕**状态变化**工作，不是固定步骤。

```
❌ Step1 → Step2 → Step3 → Step4   （写死的流程，任何意外都卡住）
✅ Current State → Goal → Gap → Next Action   （Agent 自主决定下一步）
```

每个状态定义：**Goal**（目标）、**Verify**（怎么验证达成了）、**On Fail**（失败了去哪）、**Knowledge**（需要时读哪个文件）。

### Principle 3: Knowledge is Lazy

Knowledge 永远按需读取，不一次塞给 Prompt。

```
SKILL.md          197 行 ← 每次加载（从 809 行瘦身 76%）
knowledge/        按需读 ← Agent 需要时才进上下文，注意力不衰减
```

---

## 两类 Skill

| | Generative | Orchestration |
|------|------------|---------------|
| **做什么** | 接收输入 → 输出文档/设计 | 接收输入 → 操作系统/服务 → 探活验证 |
| **ASA 层** | Identity + Policy + SM + Planner + Verify + Output | 以上全部 + Repair / Rollback Strategy + Tool Trust |
| **State 数量** | 4-7 个 | 6-12 个 |
| **SKILL.md 上限** | ≤ 200 行 | ≤ 250 行 |
| **推荐模型** | 推理型（推演/DSL 生成）→ Sonnet；提取型（结构清晰）→ Haiku/Sonnet | Sonnet / Opus |
| **示例** | prd-maker, spec-maker, er-maker, skill-maker | java-run |

> **为什么 prd-maker 不能用 Haiku？** 推演业务闭环、抽象出状态机和规则 DSL 需要深度推理。Haiku 做简单翻译或文本分类还行，面对结构化 DSL 生成大概率输出残疾。prd-maker 和 spec-maker 的底线推荐是 Sonnet，Haiku 只留给 er-maker 这种结构边界极其清晰的逆向提取任务。

---

## 文件结构

```
skill-name/
├── SKILL.md              # 主 Spec（唯一入口，Agent 每次加载）
└── knowledge/            # 按需加载（Policy 格式，非教程）
    ├── index.md           # 场景→文件映射（超过 3 个文件时）
    └── *.md
```

### SKILL.md 模板

```markdown
## 1. Identity         — Role + 唯一成功标准（不因"执行完脚本"而终止，只因"目标达成"而结束）
## 2. Policy            — 行为原则表（抽象，无平台命令）
## 3. State Machine     — 状态图 + 每状态 Goal/Verify/OnFail/Knowledge 引用
## 4. Planner           — 执行前规划，Priority 分级（Critical > Blocking > Optional），Replan 条件
## 5. Verify Rules      — MUST/MUST NOT 验收检查项，每条可被执行验证
## 6. Repair / Rollback  — Retry → Alternative → Fallback → Human（仅 orchestration）
## 7. Tool Calling      — Tool Trust 优先级 + Shell 约束（仅 orchestration）
## 8. Output Format     — 🟢静默 🟡里程碑 🔴致命 + Snapshot 进度
## 9. Knowledge Index   — 场景→文件速查表
```

### Knowledge 文件格式（Policy 格式，非教程）

```markdown
## When       — 触发条件（什么时候读这个文件）
## Goal       — 目标状态（读完要达成什么）
## Success    — 验证标准（怎么知道达到了）
## Repair     — 修复策略（如果没达到怎么办，可选）
## Reference  — 关键命令/模板/数据（具体怎么做）
```

---

## 验证清单（发布前逐条检查）

```
1. SKILL.md MUST ≤ 200 行（generative）/ ≤ 250 行（orchestration）
2. Policy 表中 MUST NOT 出现平台/工具名（PowerShell/bash/mysql 等）
3. State Machine 每个状态 MUST 有 Goal + Verify
4. Planner 段 MUST 存在且有依赖/优先级声明
5. Verify Rules 每项 MUST 可被执行验证
6. knowledge/ 文件 MUST 用 Policy 格式（When/Goal/Success/Reference）
7. MUST NOT 出现 THINK/ACT/OBSERVE 伪代码（LLM 本来就会，不用教）
8. MUST NOT 出现重复约束（同一规则只在一处出现）
9. 模板/脚本 MUST 在 knowledge/ 中，不在 SKILL.md 中
10. Output Format MUST 有 Snapshot 示例
```

---

## 禁止项

- SKILL.md 中嵌入大段模板/脚本 → 放 knowledge/
- Policy 中出现平台命令 → Policy 是抽象的
- 固定 Step1→Step2→Step3 流程 → 用 State Machine 替代
- 同一约束重复出现在多处 → 一层只做一件事
- THINK/ACT/OBSERVE 伪代码 → LLM 本来就懂 ReAct
- "此处省略"、`...` 等占位符 → 每条信息必须真实可用
- 生成的 Skill 中 Policy 和 Implementation 混杂 → 用 skill-maker 的验证清单把关

---

## 与其他 Agent 框架的关系

ASA 不是封闭标准，是从实践中抽象出的设计模式。它做的事情是把分散在各家框架里的概念统一成一套**平台无关的 Spec 格式**：

| 概念 | ASA | Dev-Crew Agent Spec | Anthropic Skill | OpenAI Codex |
|------|-----|---------------------|-----------------|--------------|
| 身份定义 | Identity | Part I: Core Identity | Role 描述 | Instructions |
| 行为约束 | Policy | Guiding_Principles | Constraints | Rules |
| 状态管理 | State Machine | Execution Flows | Workflow | Steps |
| 质量验证 | Verify Rules | Enforceable_Standards | — | — |
| 故障恢复 | Repair/Rollback | Resilience_Patterns | — | — |
| 知识拆分 | knowledge/ | Protocols | — | Files |
| 模型推荐 | Resource Profile | Resource_Consumption_Profile | — | — |
| 反模式防御 | Forbidden_Patterns | Forbidden_Patterns | — | — |

---

## Agent Compiler：长远愿景

```
                 Agent Spec (.spec)
                      │
        ┌─────────────┴─────────────┐
        │                           │
   Static Analyzer            Knowledge Index
        │                           │
        └─────────────┬─────────────┘
                      │
                 Agent Compiler
                      │
      ┌───────────────┼────────────────┐
      │               │                │
 Claude Skill     Codex AGENTS     Gemini Gem
      │               │                │
 Anthropic        OpenAI          Google
 Runtime          Runtime         Runtime
```

核心思想：

- **Specification 是稳定的。** 不管底层模型怎么换，Spec 不动。
- **Platform Adapter 是可替换的。** 换平台只需换 Adapter，业务逻辑不变。
- **Runtime 是不断变化的。** Claude 5、GPT-5、Gemini 3——Adapter 吸收差异。

未来真正需要维护的不是 Prompt，而是 **Specification**。这也是为什么从优化一个 `java-run` Skill 出发，最终讨论到了类似 LLVM、OpenAPI、IR 的概念——本质上，已经把 Skill 从"提示词"提升成了"软件架构"。

---

## 元技能：skill-maker

skill-maker 是 ASA 范式的自举实现——一个按 ASA 规范生成 ASA Skill 的 Skill。

```
输入：新 Skill 的需求描述
  ↓ INTERVIEW（4 维度面试：用途→触发词→输入源→成功标准）
  ↓ CLASSIFY（判定 generative / orchestration）
  ↓ PLAN（生成结构大纲）
  ↓ GENERATE（输出 SKILL.md + knowledge/）
  ↓ VERIFY（10 项 ASA 验证清单逐条检查）
输出：可直接部署到 ~/.claude/skills/ 的完整技能目录
```

所有技能遵循同一架构标准，无需每次从零设计。skill-maker 自身也是 ASA 范式的产物——它的 SKILL.md 同样遵循 Identity → Policy → State Machine → Planner → Verify → Output → Knowledge Index 结构。

---

## 局限与代价（什么时候 ASA 帮不上忙）

ASA 不是银弹。以下三个问题是架构层面的固有约束，不是实现层面的 bug。

### 1. Verify Rules 的套娃困境

规范要求"Verify Rules 每项 MUST 可被执行验证"，但实际落地时分三档：

| 层级 | 示例 | 验证方式 | 成本 |
|------|------|----------|------|
| **Hard** | YAML 缩进一致、括号匹配、文件数 = 3 | grep / wc / linter 直接验 | 0 Token |
| **Soft** | 数据模型与 API 字段命名风格一致、covers 引用的 id 存在 | 同一次 LLM 调用的 VERIFY 状态顺带验 | +1 轮 |
| **Deep** | PRD 商业逻辑是否闭环、技术选型理由是否硬核 | **只能 LLM-as-judge** | 逃不掉 |

Hard 和 Soft 两道防线拦住了 80% 的问题。Deep 层的 LLM 套娃是行业瓶颈——ASA 不解决它，但通过前两层压缩了套娃的触发频率。

### 2. Runtime Adapter 的编译器陷阱

ASA 的愿景是"同一份 Spec，编译到不同平台"。但 ASA Compiler 目前不存在，也不该现在写。

务实的做法是：**把 ASA 当设计规范用，不是工具链用。**

```
ASA Spec（设计规范）
  ├─→ 手写 Claude SKILL.md    150 行，照着翻，很快
  ├─→ 手写 Gemini Gem prompt  150 行，同上
  └─→ 手写 Codex AGENTS.md    150 行，同上
```

150 行的文件手动翻译成本远低于开发和维护一个 Compiler。当你有 20+ Skill × 5 平台时，Compiler 才值得投入。提前写 Compiler 是过早抽象。

### 3. 模型幻觉会击穿架构

ASA 依赖 LLM 的自觉性——Knowledge Index 告诉模型"发生 XX 时去读 knowledge/yy.md"，但如果模型脑子一抽没去读，而是盲猜了一个命令，整套防线一秒崩溃。

ASA 能做的不是**消除**幻觉，而是**缩小幻觉的破坏半径**：

| | 800 行混合 Prompt | ASA 200 行 + knowledge/ |
|------|---------------------|--------------------------|
| 盲猜概率 | 高（信息密度大，LLM 倾向"自己编"） | 低（主文件只告诉它"去哪查"） |
| 盲猜后果 | 编一个完整的安装脚本 | 编一个命令（因为具体步骤不在上下文里） |
| 恢复路径 | 需要人工排查整个输出 | 回到 knowledge/ 重新读正确文件 |

底线不变：**推理型 Skill 用 Sonnet 起步，用 Haiku 测试 ASA 是测试 ASA 的极限，不是测试 ASA 的价值。**

---

## 参考实现

本仓库包含 5 个按 ASA 范式构建的 Skill：

| Skill | 类型 | SKILL.md | knowledge/ | 说明 |
|-------|------|----------|------------|------|
| [java-run](java-run/SKILL.md) | orchestration | 217 行 | 6 文件 + tools | 最完整的 ASA 实现，含 Repair/Rollback + Tool Trust |
| [prd-maker](prd-maker/SKILL.md) | generative | 120 行 | 3 文件 | DSL 生成型，含反模式防御 + Litmus Test |
| [spec-maker](spec-maker/SKILL.md) | generative | 118 行 | 2 文件 | 8 道 Quality Gate + Trade-off Alternative |
| [er-maker](er-maker/SKILL.md) | generative | 113 行 | 2 文件 | 提取型，结构清晰 |
| [skill-maker](skill-maker/SKILL.md) | generative | 128 行 | 1 文件 | 元技能，ASA 自举 |

每个 Skill 的 SKILL.md 都可以作为新 Skill 的结构参考。推荐从同类型的 skill 复制模板开始。
