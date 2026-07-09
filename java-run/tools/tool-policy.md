# Tool Policy — 调用优先级与原则

## Tool Trust（优先级从高到低）

| 优先级 | 类型 | 适用场景 |
|--------|------|----------|
| 1. Python Tool | `scan_directory` / `execute_sql_script` / `generate_dev_yaml` / `start_windows_service` / `patch_text_file` / `read_file_content` / `check_windows_env` | 所有结构化操作 |
| 2. IDE Tool | Glob / Read / Grep | 文件扫描与内容读取 |
| 3. Shell | Bash (PowerShell) | 命令执行（mise / mvn / npm / mysql CLI） |
| 4. LLM Inference | 版本推断、配置决策 | 仅当数据已通过 Tool 获取后 |

**原则**：能调用 Tool 绝不用 LLM Guess。所有文件、环境、进程状态必须通过 Tool 真实获取。

## Shell 约束

- 所有 Shell 命令使用 PowerShell 语法（非 cmd）
- **禁止** `cmd /c`
- **禁止** Linux 专属操作符（`&&`、`rm -rf`、`export`）
- `-D` JVM 参数必须双引号包裹：`"-Dspring-boot.run.profiles=dev"`
- 不拼接复合启动命令（`npm install && npm run dev` 拆为两步，避免孤儿进程）
- 运行环境：`mise exec -- <command>` 统一前缀

## 非破坏性原则

- 修改配置文件前先备份 `.bak`
- 动态配置写入 `application-dev.yml`，不污染原始配置
- 恢复 `.bak` 备份后，不再继续该阶段重试
- **【源码红线】** Agent 仅拥有配置文件（`.properties` / `.yml` / `.xml` / `.json`）的动态覆写权限。**严禁**使用 `patch_text_file` 修改任何 `*.java` 业务源码或安全配置类。涉及权限拦截（401/403）导致的探活失败，必须保留现场并输出 Diagnostic Report，将控制权交还用户

## 工具速查

| 工具 | 用途 |
|------|------|
| `scan_directory(root_path)` | 递归扫描，自动排除 `.git`/`node_modules`/`target`/`dist`/`build`/`.idea` |
| `read_file_content(path, max_lines)` | 读取文件前 N 行 |
| `check_windows_env(commands)` | 执行 `where` / `--version` 返回路径和版本 |
| `execute_sql_script(db_config, sql_path)` | 创建库 + 导入 SQL（UTF-8 安全） |
| `generate_dev_yaml(target_dir, config_data)` | 生成 `application-dev.yml` |
| `start_windows_service(cmd, cwd)` | `CREATE_NEW_CONSOLE` 异步启动，返回 PID |
| `patch_text_file(path, search, replace)` | 精确文本替换，自动备份 |
