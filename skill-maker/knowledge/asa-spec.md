# ASA Spec Reference — 各层定义与约束

## 两层技能分类

| 类型 | 特征 | 需要哪些 ASA 层 | 示例 |
|------|------|----------------|------|
| **generative** | 接收输入 → 输出文档/设计 | Identity + Policy + State Machine + Planner + Verify + Output | prd-maker, spec-maker, er-maker |
| **orchestration** | 接收输入 → 操作系统/服务 → 探活验证 | generative 全部 + Repair Strategy + Tool Trust | java-run |

---

## SKILL.md 各节生成规范

### 1. Identity（~15 行）

- Role 一句话（你是谁）
- 唯一成功标准一句话
- 核心理念引用（如有）

### 2. Policy（~15-25 行）

用表格列出行为原则，每条：原则名 + 约束内容。
**禁止出现**：具体工具名、平台命令、文件路径。
**允许出现**：抽象行为描述（如"先验证再执行"而不是"先跑 curl"）。

### 3. State Machine（按需行数）

两种写法：

**generative 型**（文档生成流）：
```
IDLE → ANALYZE_INPUT → PLAN → GENERATE → VERIFY → DONE
```
每状态：Goal / Verify / On Fail 即可。不需要 Repair Strategy 和 Tool Trust。

**orchestration 型**（系统操作流）：
```
IDLE → SCAN → ANALYZE → CHECK → FIX → INIT → START → PROBE → DONE
                                                        ↓
                                                    FAILED
```
每状态：Goal / Verify / On Fail → Repair / Knowledge 引用。

状态数量：generative 4-7 个，orchestration 6-12 个。

### 4. Planner（~15-25 行）

- 执行前生成输出计划
- 若为 orchestration 型：Priority 三级（Critical > Blocking > Optional）
- 若为 generative 型：标注依赖关系（如"术语 DSL 必须先出"）

### 5. Verify Rules（~10-20 行）

按优先级列出验收检查项。每条可被执行验证。

### 6. Repair Strategy（仅 orchestration 型）

```
Retry → Alternative → Fallback → Human
```

### 7. Tool Calling Rules（仅 orchestration 型）

Tool Trust 优先级 + Shell 约束。

### 8. Output Format

- 🟢🟡🔴 分层汇报
- Snapshot 格式
- 汇报示例

### 9. Knowledge Index

表格：场景 → 对应 knowledge 文件。

---

## knowledge/ 文件规范

### 格式：Policy 格式（非文档格式）

```
# 标题 — 一句话定位

## When（触发条件）

## Goal（目标状态）

## Success（验证标准）

## Repair（修复策略，可选）

## Reference（关键命令/模板/数据）
```

### 索引文件（index.md，可选）

若 knowledge/ 超过 3 个文件，需要 index.md 做场景映射。

---

## Model 推荐

生成新 skill 时，根据技能类型和推理复杂度建议模型档位：

| 技能类型 | 推荐模型 | 理由 |
|----------|---------|------|
| **generative 推理型**（需推演/抽象/DSL 生成，如 prd-maker, spec-maker） | Sonnet | 需要业务逻辑推演、DSL 结构化输出、多方案权衡，Haiku 不够 |
| **generative 提取型**（结构边界清晰，如 er-maker） | Haiku / Sonnet | 逆向提取为主，推理负担低，Haiku 可胜任 |
| **orchestration**（系统编排，如 java-run） | Sonnet / Opus | 需要多轮自愈推理、故障诊断、环境适配 |

> 推荐写入生成的 SKILL.md 的 Identity 段或 Planner 段，帮助用户正确配置。

---

## 禁止项

- SKILL.md 中禁止嵌入大段模板/代码（放 knowledge/）
- Policy 中禁止出现平台命令（PowerShell/bash/mysql）
- 禁止固定 Step1→Step2→Step3 流程（用 State Machine 替代）
- 禁止重复写同一个约束（统一到一层）
- 禁止 THINK/ACT/OBSERVE 伪代码
- 禁止出现"此处省略"、`...` 等占位符
- 禁止中间插入闲聊或反问

---

## 验证清单（生成后逐条检查）

1. SKILL.md MUST ≤ 200 行（generative）/ ≤ 250 行（orchestration）
2. Policy 表中 MUST NOT 出现平台/工具名
3. State Machine 每个状态 MUST 有 Goal + Verify
4. Planner 段 MUST 存在且有依赖/优先级
5. Verify Rules MUST 至少含 1 条 Hard 规则（非 LLM 验证：行数/文件数/YAML 语法/字段存在性）
6. Verify Rules 每项 MUST 可执行
7. knowledge/ 文件 MUST 用 Policy 格式（When/Goal/Success）
8. MUST NOT 出现 THINK/ACT/OBSERVE 伪代码
9. MUST NOT 出现重复约束（同一规则只在一处出现）
10. SKILL.md MUST NOT 包含多平台适配逻辑（当前仅 Claude Adapter，这是 Skill 的事，不是 Compiler 的事）
11. 实现细节 MUST 全部在 knowledge/ 中（幻觉发生时：猜错一条命令，不是整套方案）
12. 模板/脚本 MUST 在 knowledge/ 中，不在 SKILL.md 中
13. Output Format MUST 有 Snapshot 示例
