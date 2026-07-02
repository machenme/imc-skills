# imc-skill

个人 Claude Code 技能集，覆盖产品规划、技术方案设计、Java 项目一键启动与数据库 ER 图自动生成四大场景。

## 安装

将目标 skill 文件夹复制到 Claude Code 的 skills 目录即可：

```bash
# 用户级安装（所有项目可用）
cp -r <skill-name> ~/.claude/skills/

# 项目级安装（仅当前项目可用）
cp -r <skill-name> <project-root>/.claude/skills/
```

装完重启 Claude Code 会话，skill 自动注册。

---

## 技能列表

### 1. prd-maker — PRD 生成器 + 领域 DSL 制品

**一句话**：把模糊的产品想法变成结构化 PRD，同时生成术语表、状态机、规则、能力、验收五组小 DSL——在产品和代码之间架起 AI 可精确读取的桥梁。

**触发方式**：对着 Claude Code 说"帮我写个 PRD"、"帮我规整一下这个产品思路"、"这个功能怎么做需求分析"。

**做的事**：
- 把零散想法补全为逻辑闭环的产品方案
- 以资深产品经理视角审视商业漏洞，写入 PM 备注
- 输出包含目标用户画像、P0/P1/P2 功能矩阵、用户故事 Given-When-Then、非功能需求的结构化 Markdown PRD
- **（新增）** 提炼五组小 DSL：术语表 (glossary) → 状态机 (lifecycle) → 规则 (rules) → 能力 (capabilities) → 验收 (acceptance)
- **（新增）** 生成 `IMPLEMENTATION_STATUS.md` 差距文档，显式标记「产品定义 vs 当前实现」的差距
- **（新增）** 输出 AI 治理规则片段（冲突优先级），可直接写入 `AGENTS.md`

**核心理念**：PRD 管「用户要什么」，DSL 管「代码该怎么写、改到哪算完、哪里还不能假装做完」。每个 DSL 条目自带 `id` / `status` / `gap` 三元数据——没有 status/gap，AI 会把规划当实现。

**输出**：一份 Markdown PRD（可直接开需求评审）+ 五组 YAML DSL 片段（可直接放入 `domain/` 目录驱动 AI 编码）。

---

### 2. spec-maker — 技术规范生成器

**一句话**：拿到 PRD（含 DSL 制品更佳）后，自动输出生产级技术规范文档。

**触发方式**：喂一份 PRD 后说"帮我生成技术方案"、"帮我写 SPEC"、"设计技术架构"。

**前置依赖**：需要一份 PRD 作为输入（建议先用 `prd-maker` 生成）。若 PRD 附带 DSL 制品（glossary/lifecycle/rules/capabilities/acceptance），会自动提取状态枚举、业务规则和验收标准作为结构化输入，提升数据模型和 API 设计精度。

**做的事**：
- 结合 PRD 的规模与复杂度做技术选型，每项附具体理由
- 输出完整数据模型（逐字段列出 + ER 关系），优先从 DSL 状态枚举提取字段设计
- 输出 RESTful API 设计（Request/Response JSON 示例 + 错误码），业务校验层对齐 DSL 规则
- 给出生产级项目目录结构
- CTO 评审备注指出 PRD 中的技术风险

**输出**：一份可以直接丢给开发团队开干的技术 SPEC，技术栈版本号、数据表字段、API JSON 全部写死，不含任何"…"省略符。

---

### 3. java-run — Java 项目一键启动

**一句话**：全自动扫描、配环境、导数据库、启动 Java 后端 + 前端。

**触发方式**：给一个项目根目录路径，说"帮我启动这个项目"。

**做的事**（ReAct 自愈架构，遇错不抛给用户而是智能重试）：
- **Phase 0**：安装 mise 统一管理 JDK/Node.js/Maven 版本，扫描 pom.xml/package.json/SQL 文件，智能推断所需版本
- **Phase 1**：对比本地环境，缺失组件自动下载安装（JDK/Node/MySQL/Maven），配置阿里云 Maven 镜像加速
- **Phase 2**：自动建库 → 导入 SQL → 验证中文无乱码（MySQL PowerShell 管道编码坑全部规避）
- **Phase 3**：覆写 `application-dev.yml` 对齐本地数据库凭证
- **Phase 4**：拉起后端 → 拉起前端 → 每 5s 探活 → 崩溃自动抓日志诊断 → 重试（端口冲突/凭证错误/依赖缺失/拦截器配置等全自动修复）
- **Phase 5**：生成 `RUNBOOK.md`，含启动命令、访问地址、登录账号

**输出**：一个跑起来的项目 + 一份拿给同事就能用的运行手册。

**环境要求**：Windows 11，全程自动适配，零人工干预。

---

### 4. er-maker — 数据库 ER 图自动生成器

**一句话**：连接项目数据库，一键生成标准 DBML ER 图 + SQL 建表语句 + 精简版 ER 图。

**触发方式**：对着 Claude Code 说"生成 ER 图"、"导出 DBML"、"帮我逆向这个数据库"、"生成数据库文档"。

**做的事**：
- 自动扫描项目配置文件（`.env` / `application.yml`）获取数据库连接信息
- 智能识别 MySQL / PostgreSQL / SQLite 并使用对应方言查询 Schema 元数据
- 生成 `e-r/建表语句.sql` — 完整 DDL（只读，禁止任何写操作）
- 生成 `e-r/E-R图.dbml` — 标准版 ER 图（含全部字段细节 + 外键关系）
- 生成 `e-r/E-R图(精简).dbml` — 精简版 ER 图（仅表名 + 关系，快速掌握宏观架构）
- 无外键约束的遗留数据库自动按命名约定推断关系，标注 `inferred`

**输出**：`e-r/` 目录下的三个文件，可直接用 dbdocs.io / dbdiagram.io 渲染。

---

## 推荐工作流

```
用户的想法 → prd-maker → PRD 文档 + 领域 DSL 制品
                             │  (glossary / lifecycle / rules
                             │   capabilities / acceptance
                             │   IMPLEMENTATION_STATUS.md)
                             ↓
                        spec-maker → 技术 SPEC
                             │
                             ├─→ 开发团队编码
                             │        ↓
                             │   java-run → 一键启动 & RUNBOOK.md
                             │
                             └─→ er-maker → ER 图 + DDL
                                   (随时可用，不依赖前序步骤)
```

**DSL 层的角色**：PRD 管「用户要什么」，DSL 管「代码该怎么写、改到哪算完」。五组小 DSL 用同一组 `id` 把产品故事、业务规则、权限、测试和实现差距钉在同一张网上——让后续的 spec-maker 和 AI 编码工具知道改哪、改到什么算做完、哪里还不能假装做完。

四个 skill 可独立使用，但串起来就是一条完整的产品→技术→交付+文档流水线。
