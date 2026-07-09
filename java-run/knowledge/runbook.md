# RUNBOOK — 项目运行手册模板

## When

所有服务探活通过，需要固化运行知识。

## Template

```markdown
# 项目运行手册

## 如何启动

### 前提条件
- 已安装 `mise`（统一管理 Java/Node.js/uv 运行时，安装命令：`irm https://gitee.com/osmc/dev-bootstrap/raw/main/mise-install.ps1 | iex`）
- Java [版本]（由 mise 管理）
- Node.js [版本]（由 mise 管理，如有前端）
- uv — Python 包管理器（由 mise 管理）
- 已安装 MySQL [版本]
- Maven 已配置阿里云镜像加速（国内环境自动启用）
- **WAR 项目额外**：Tomcat 10.1.x（已自动下载至 `{{root_path}}/tomcat`）

### 启动步骤

**JAR 项目 — 启动后端**：
```powershell
cd "[真实后端绝对路径]"
mise exec -- mvn spring-boot:run "-Dspring-boot.run.profiles=dev"
```

**JAR 项目 — 启动前端**（如有）：
```powershell
cd "[真实前端绝对路径]"
if (Test-Path node_modules) { mise exec -- npm run dev } else { mise exec -- npm install; mise exec -- npm run dev }
```

**WAR 项目 — 构建+部署+启动**：
```powershell
cd "{{root_path}}"
mise exec -- mvn clean package -DskipTests -q
copy /Y target\*.war {{root_path}}\tomcat\webapps\
$env:JAVA_HOME = "[Java Home 路径]"
Start-Process {{root_path}}\tomcat\bin\startup.bat -WindowStyle Hidden
Start-Sleep 8
curl http://localhost:8080/[context-path]
```

**WAR 项目 — 停止**：
```powershell
{{root_path}}\tomcat\bin\shutdown.bat
```

## 访问页面

| 页面 | 地址 |
|------|------|
| 主页 | http://localhost:[端口]/[context-path] |
| 登录页 | http://localhost:[端口]/[context-path]/login |
| 管理后台 | http://localhost:[端口]/[context-path]/admin/login |
| API | http://localhost:[端口]/[context-path]/api/... |

## 登录账号

| 角色 | 用户名 | 密码 | 说明 |
|------|--------|------|------|
| 管理员 | [从 SQL 提取] | [明文或"已加密"] | 后台管理 |
| 测试用户 | [从 SQL 提取] | [明文或"已加密"] | 普通用户 |

> 若 SQL 中无账户数据（Flyway/CommandLineRunner 注入），注明"请参考项目文档或启动后自行注册"。

---

## 项目概述
- **项目名称**：[pom.xml `<name>` 或 package.json `name`]
- **项目路径**：[{{root_path}}]

## 技术栈
| 层级 | 技术 | 版本 |
|------|------|------|
| 后端框架 | [WAR: Jakarta EE Servlet / JAR: Spring Boot] | [版本号] |
| JDK | Java | [推断版本] |
| 持久层 | [MyBatis-Plus / JDBC / JPA] | [版本号] |
| 前端框架 | [JSP / Vue / React] | [版本号] |
| 构建工具 | [Maven / Gradle] | [版本号] |
| 服务器 (WAR) | Tomcat | 10.1.x |
| 数据库 | MySQL | [推断版本] |

## 数据库信息
| 配置项 | 值 |
|--------|-----|
| 数据库名 | [application.yml 提取] |
| 主机 | [application.yml 提取] |
| 端口 | [application.yml 提取] |
| 用户名 | [application.yml 提取] |
| 密码 | [application.yml 提取] |

## 数据库初始化
```sql
-- 按顺序执行
source [SQL文件路径1];
source [SQL文件路径2];
```

## 常见问题
- **WAR 404**：确认 WAR 包名 = URL context-path
- **Tomcat 闪退**：确认 JAVA_HOME 指向 JDK
- **数据库连接错误**：确认 MySQL 已启动，凭证正确
- **前端 401**：检查 Interceptor excludePathPatterns
- **Maven 依赖失败**：检查无用依赖 + 阿里云镜像
- **端口占用**：修改端口或杀进程
```

## Data Extraction Rules

- **项目名称**：`pom.xml` `<name>` 或 `package.json` `name`
- **后端版本**：`pom.xml` `<parent>` 的 `<version>`
- **JDK 版本**：从推断结果取
- **前端版本**：`package.json` 中 `vue`/`react` 依赖
- **构建工具**：`vite.config.js` → Vite，`webpack.config.js` → Webpack
- **UI 组件库**：`package.json` 识别（element-plus / ant-design-vue / naive-ui）
- **数据库配置**：`application.yml` `datasource` 段
- **管理员账号**（尽力而为）：搜索 SQL 中 `INSERT INTO` + `user`/`admin`/`account` 表
- **服务端口**：后端 `server.port` + `context-path`；前端 `vite.config.js` 的 `server.port` + `.env.development` 的 `VITE_PORT=`/`PORT=`

> 文件编码：UTF-8 无 BOM。
