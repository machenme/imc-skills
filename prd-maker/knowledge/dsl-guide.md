# DSL Guide — 五组小 DSL 格式与治理规则

> **设计原则**：不要仿自然语言造一门大 DSL。小 DSL 组合——每种只解决一类差距。覆盖：术语 → 行为 → 权限 → 契约 → 验收。

**每个条目必须包含 `id`、`status`、`gap` 三元数据**。`status: partial` 且 `gap` 非空时，AI 才知道「这里还没做完」。
`status: planned` 允许 `gap: ""`。

**必选**：术语 DSL (7.1) + 验收 DSL (7.5)。**按需**：行为/能力/契约 DSL 根据领域复杂度决定。

---

## 7.1 术语 DSL (Glossary)

最先做，收益最大。id 是全局稳定引用键，后续 lifecycle、rules、acceptance 都回溯到这里。

```yaml
terms:
  - id: {feature}_{action}       # 全体系唯一稳定标识，如 order_cancel
    product: {中文名}
    aliases:
      - {别名1}
      - {别名2}
    code:
      - {模块/服务}/{文件}#{函数}
    notes: {补充说明}
    status: planned               # done | partial | planned
    gap: ""
```

## 7.2 行为 DSL (Lifecycle + Rules)

**状态机 (Lifecycle)** — 管「能发生哪些转移」：

```yaml
transitions:
  - id: t_{verb}_{from}          # 如 t_cancel_paid
    from: {源状态}
    to: {目标状态}
    on: {触发事件}                # 如 user_cancel / admin_force_cancel
    guards:
      - {条件1}
      - within_minutes: {N}
    effects:
      - {效果1}
    status: planned
    gap: ""
    mapping:
      code:
        - {预估代码路径}
      tests:
        - {预估测试路径}
      ui:
        - {预估 UI 路径}
```

**决策表 (Rules)** — 管「转移时业务判定与错误码」：

```yaml
rules:
  - id: {rule_name}               # 如 refund_full_within_window
    when:
      status: {状态}
      minutes_since_pay: { lte: 30 }
      shipped: false
    then: {动作}                   # 如 refund_full / reject
    reason: {错误码}               # then=reject 时必填
    status: planned
    gap: ""
```

> lifecycle 给架构师和 AI 看全局；rules 给 QA 和产品看细则——同一规则刻意分层，不是重复劳动。

## 7.3 能力 DSL (Capabilities)

「谁在什么时候能做什么」：

```yaml
capabilities:
  - id: cap_{actor}_{action}      # 如 cap_buyer_cancel
    actor: {角色}                  # buyer / admin / system
    action: {动作}
    allowed_states: [{状态1}, {状态2}]
    ui_visible: true
    status: planned
    gap: ""
```

## 7.4 契约 DSL (Contract)

优先用现成标准（OpenAPI / JSON Schema / Zod），在 schema 上挂产品语义：

```yaml
schemas:
  {Entity}Status:
    enum: [{值1}, {值2}, {值3}]
    x-product:
      {值1}: {产品侧含义}
      {值2}: {产品侧含义，含关键约束如"30 分钟内可取消"}
```

## 7.5 验收 DSL (Acceptance)

把 PRD 第 5 节的用户故事转成可追溯、可执行的验收条目：

```yaml
stories:
  - id: AC-{NNN}
    label: {验收标题}
    covers:                        # 回溯到 lifecycle / rules / capabilities
      - t_{xxx}
      - {rule_id}
    given: {前置条件}
    when: {触发动作}
    then:
      - {结果1}
      - {结果2}
    verify:
      - {可执行验证命令}
    status: planned
    gap: ""
```

> `covers` 建立 验收 ↔ 规则 ↔ 状态机 的追溯链。`verify` 是 AI 与 CI 的完成判定。

---

## 8. 实现差距总览 (IMPLEMENTATION_STATUS)

```markdown
## {功能名称}

**产品定义**：{一句话目标}
**当前实现**：
- {已实现的点}
**差距**：
- {未实现的点}
**差距原因**：{技术债 / 依赖未就绪 / 排期在后}
```

---

## 9. AI 治理规则（写入 AGENTS.md 片段）

### 冲突优先级（从上到下递减）

1. 用户明确指令
2. acceptance.verify 通过
3. 契约 Schema（OpenAPI / JSON Schema）
4. 行为 DSL（lifecycle / rules）
5. PRD 叙述
6. 代码现状（补全任务除外）

### 禁止

- PRD 或口头说"已完成"，而 DSL 仍为 partial——这会让 AI 误判
- 任务描述应带产品意图 + 实现锚点：
  - 术语：glossary.yaml#{term_id}
  - 状态机：lifecycle.yaml#{transition_id}
  - 规则：rules.yaml#{rule_id}
  - 验收：acceptance.yaml#{AC_id}
  - 差距：IMPLEMENTATION_STATUS.md §{功能名称}

### DSL 推荐文件结构

```
domain/
  glossary.yaml
  {feature}/
    lifecycle.yaml
    rules.yaml
    capabilities.yaml
    acceptance.yaml
  IMPLEMENTATION_STATUS.md
packages/shared/schemas/     # OpenAPI / JSON Schema
AGENTS.md                    # 文档索引 + 冲突优先级
tools/
  domain-lint.ts             # 校验 id 互引、status 与测试一致性
  domain-check.ts            # DSL ↔ 代码 ↔ 测试对比
```
