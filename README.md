# imc-skill

个人 Claude Code 技能集，覆盖产品规划、技术方案设计与 Java 项目一键启动三大场景。

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

### 1. prd-maker — PRD 生成器

**一句话**：把模糊的产品想法变成结构化的 PRD 文档。

**触发方式**：对着 Claude Code 说"帮我写个 PRD"、"帮我规整一下这个产品思路"、"这个功能怎么做需求分析"。

**做的事**：
- 把零散想法补全为逻辑闭环的产品方案
- 以资深产品经理视角审视商业漏洞，写入 PM 备注
- 输出包含目标用户画像、P0/P1/P2 功能矩阵、用户故事 Given-When-Then、非功能需求的结构化 Markdown PRD

**输出**：一份可以直接拿去开需求评审的 Markdown PRD。

---

### 2. spec-maker — 技术规范生成器

**一句话**：拿到 PRD 后，自动输出生产级技术规范文档。

**触发方式**：喂一份 PRD 后说"帮我生成技术方案"、"帮我写 SPEC"、"设计技术架构"。

**前置依赖**：需要一份 PRD 作为输入（建议先用 `prd-maker` 生成）。

**做的事**：
- 结合 PRD 的规模与复杂度做技术选型，每项附具体理由
- 输出完整数据模型（逐字段列出 + ER 关系）
- 输出 RESTful API 设计（Request/Response JSON 示例 + 错误码）
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

## 推荐工作流

```
用户的想法 → prd-maker → PRD 文档
                ↓
           spec-maker → 技术 SPEC
                ↓
           开发团队编码
                ↓
           java-run → 一键启动 & RUNBOOK.md
```

三个 skill 可独立使用，但串起来就是一条完整的产品→技术→交付流水线。
