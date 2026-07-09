# Troubleshooting — 已知问题速查

## When

服务启动失败、探活不通过、运行时异常。

## 日志诊断流程

按优先级读取：
1. `backend/target/*.log` / `logs/spring.log`
2. Maven 控制台输出最后 50 行
3. Windows 事件查看器应用日志

---

## 已知问题速查表

| 现象 | 根因 | Repair |
|------|------|--------|
| 中文 `????` | PowerShell 管道破坏 UTF-8 | 用 mysql `-e "source '...'"` 或 Python subprocess |
| Tomcat 控制台 `淇℃伅` | Windows GBK → Java 默认编码 | 创建 `setenv.bat` 设 `-Dfile.encoding=UTF-8` |
| `Unknown lifecycle phase ".run.profiles=dev"` | PowerShell 拆分 `-D` 参数 | 双引号包裹：`"-Dspring-boot.run.profiles=dev"` |
| `cmd /c` 被拦截 | 沙箱安全策略 | 全量切换为 PowerShell 语法 |
| `mysql` 不是内部命令 | MySQL 未加入 PATH | 扫描 `C:\Program Files\MySQL\` 取绝对路径 |
| `DELIMITER //` 存储过程报错 | mysql CLI 非交互模式不支持 | 跳过，不影响功能 |
| JDK 下载缓慢 | GitHub Releases 受限 | 重试 `mise use java@temurin-<version>` |
| `Could not find artifact` | 依赖无效或不可达 | 移除不需要的依赖（如项目用 MySQL 却有 sqljdbc4）；或配置阿里云镜像 |
| 前端 401 / 页面空白 / 静态资源挂掉 | 安全框架（Security/Shiro/Interceptor）误拦截主页或静态资源 | 1. Grep 定位 SecurityConfig / ShiroConfig / WebMvcConfigurer，报告拦截器类名与可疑行号<br>2. 检查 Shiro 过滤链顺序，确保 `anon` 声明在 `authc` 之前<br>3. **禁止** Agent 擅自修改权限认证源码，触发熔断并请求用户介入 |
| 前端静态资源 404 | `addResourceLocations` 与实际路径不匹配 | Glob 扫描 `resources/` 后修正映射 |
| `OutOfMemoryError` | JVM 堆不足 | 添加 `-Xmx512m` |
| `Access denied for user` | 数据库凭证错误 | 回 Phase 3 覆写 application-dev.yml |
| `Port already in use` | 端口冲突 | 杀旧进程后重试 |
| `Connection refused` | MySQL 未启动 | `net start MySQL80` 或重新部署 |
| `ClassNotFoundException` | 依赖缺失 | `mvn clean compile` 后重试 |
| `mise` 安装失败 | Gitee 不可达 | 重试 1 次；仍失败回退系统命令 |
| `mise install` 下载慢 | GitHub 代理失效 | `config.toml` 中 `url_replacements` 降级重试 |
| Maven 构建挂起 > 300s | 国内网络不畅 | 强制终止进程，释放端口，下一轮重试 |
