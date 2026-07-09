# SPEC Template — 各节结构与格式

## 1. CTO 评审备注 (Critical)

引用块形式，置于 SPEC 最顶部：

- PRD 中可能存在的技术风险、性能瓶颈或安全漏洞
- 对 PRD 不合理预期的纠正建议（如"要求 100 万并发但预算仅单机"）
- **标注「人类保留地」**：性能关键路径、安全敏感代码、核心支付逻辑——Agent 出方案，最终签字权在人
- 如果输入包含 DSL 制品：检查 `IMPLEMENTATION_STATUS.md` 差距项对技术方案的影响；`status: partial` 的 DSL 条目对应的数据模型/API 标注「待补全」
- 如果 PRD 技术上合理，注明"无重大风险点"

## 2. 技术选型与理由

### 前端
- **框架**：[选择及版本号]
- **选型理由**：结合 PRD 场景给硬核理由

### 后端
- **语言 & 框架**：[选择及版本号]
- **选型理由**：...
- **替代方案**：[备选框架及版本号] — 放弃原因：[具体场景化理由，如"Go 并发更优但团队 Java 技术栈成熟，迁移成本过高"]

### 数据库
- **主库**：[选择及版本号]
- **缓存 / 搜索引擎**（如有）：[选择及版本号]
- **选型理由**：结合数据结构特点和读写比例
- **替代方案**：[备选数据库及版本号] — 放弃原因：[具体理由]

### 部署 & DevOps
- **容器化**：Docker + 编排方案
- **CI/CD**：推荐方案
- **可观测性**（不可省略）：日志聚合 + 链路追踪 + 指标监控
- **选型理由**：...

> 所有技术组件必须标注大版本号（React 18.x、PostgreSQL 16.x、Redis 7.x），严禁"最新版""稳定版"。选型理由必须结合 PRD 具体场景，禁止"因为流行所以选择"。

## 3. 数据模型设计

每张核心表用 Markdown 表格描述，包含：字段名、数据类型、键(PK/FK)、约束条件、备注。

```
#### 表名：users

| 字段名 | 数据类型 | 键 | 约束 | 备注 |
|--------|----------|-----|------|------|
| id | BIGINT | PK | NOT NULL, AUTO_INCREMENT | 用户唯一标识 |
| email | VARCHAR(255) | - | NOT NULL, UNIQUE | 登录邮箱 |
| created_at | TIMESTAMP | - | NOT NULL, DEFAULT NOW() | 创建时间 |
```

**约束**：
- 每张表逐字段完整列出，禁止"..."或"此处省略"
- 表间外键关联附 ER 关系简述（如 `users 1:N bookmarks`、`bookmarks N:N tags`）

## 4. API 接口设计

RESTful 风格，每个接口包含：请求方法、URL 路径、Request Body (JSON)、Response Body (JSON)。

```
#### POST /api/v1/auth/login

**描述**：用户登录

**Request Body**：
{
  "email": "user@example.com",
  "password": "securePassword123"
}

**Response Body (200)**：
{
  "code": 0,
  "data": { "token": "...", "user": { "id": 1, "email": "...", "name": "..." } }
}

**Response Body (401)**：
{
  "code": 401,
  "message": "邮箱或密码错误"
}
```

**约束**：
- 接口设计与数据模型完全对应，操作的每个资源必须在数据模型中有定义
- 覆盖 PRD 中所有 P0 功能接口，P1 至少覆盖核心接口
- 每个接口至少给成功(2xx)和错误(4xx/5xx)两种 Response
- 首个接口给完整错误 JSON 结构，后续同类错误可简写 `同 /auth/login 错误结构`
- 字段命名风格在数据模型与 API 中严格保持一致
- 核心业务字段保留，去冗余，但结构必须完整可用

## 5. 项目目录结构

生产级目录树 + 核心目录职责说明：

```
project-root/
├── client/                 # 前端应用
│   ├── src/
│   │   ├── components/     # 通用 UI 组件
│   │   ├── pages/          # 页面级组件
│   │   ├── hooks/          # 自定义 Hooks
│   │   ├── services/       # API 请求封装
│   │   ├── stores/         # 状态管理
│   │   └── utils/          # 工具函数
│   └── package.json
├── server/                 # 后端应用
│   ├── src/
│   │   ├── controllers/    # 路由处理器
│   │   ├── services/       # 业务逻辑层
│   │   ├── models/         # 数据模型 (ORM)
│   │   ├── middlewares/    # 中间件
│   │   ├── utils/          # 工具函数
│   │   └── config/         # 配置文件
│   └── package.json
├── docker-compose.yml      # 本地开发环境编排
└── README.md
```

> 目录结构必须真实可用，禁止"..."或"此处省略"。
