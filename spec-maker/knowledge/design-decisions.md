# Design Decisions — 技术选型决策框架

## 决策原则

选型理由必须结合 PRD 具体场景，禁止"因为流行所以选择"。

## 前端选型

| 场景 | 推荐 | 核心理由 |
|------|------|----------|
| 高交互/实时协作（如在线文档、Dashboard） | React 18.x | Concurrent Mode、Suspense 适合复杂状态 |
| 中小团队快速迭代/管理后台 | Vue 3.x | Composition API 降低协作成本，上手快 |
| SEO 敏感的内容型产品 | Next.js 14.x | SSR/SSG 开箱即用 |
| 跨平台（Web + 移动端） | React Native / Flutter | 一套代码多端复用 |

## 后端选型

| 场景 | 推荐 | 核心理由 |
|------|------|----------|
| 高并发 I/O 密集型（API Gateway、即时通讯） | Go + Gin | Goroutine 轻量并发 |
| 企业级/金融/复杂业务逻辑 | Java 17 + Spring Boot 3.x | 生态成熟，强类型安全 |
| 快速原型/全栈 JavaScript 团队 | Node.js 20 + Express/Fastify | 前后端统一语言 |
| AI/数据分析型产品 | Python 3.12 + FastAPI | ML 生态无缝衔接 |

## 数据库选型

| 数据特征 | 推荐 | 核心理由 |
|----------|------|----------|
| 强关系、事务、一致性（电商/ERP） | PostgreSQL 16.x | ACID、丰富索引、JSON 支持 |
| 读多写少、高并发简单 KV | Redis 7.x + MySQL 8.0 | 缓存加速，主库稳定 |
| 文档型、Schema 多变（CMS/日志） | MongoDB 7.x | 灵活 Schema |
| 全文搜索 | Elasticsearch 8.x | 倒排索引，分词搜索 |

## 可观测性（不可省略）

| 维度 | 推荐 |
|------|------|
| 日志聚合 | Loki (轻量) / ELK (重量) |
| 链路追踪 | Tempo (Loki 生态) / Jaeger |
| 指标监控 | Prometheus + Grafana |

## 架构原则

- **可测试性是第一原则**：难以测试的设计就是坏设计
- **Agent 粒度**：模块以「Agent 能否独立理解、修改、测试」为标准
- **人类保留地**：性能关键路径、安全敏感代码、核心支付逻辑——必须人审人签
- **最后一公里手写**：Agent 生成的代码可以跑，但关键路径必须有人签字
