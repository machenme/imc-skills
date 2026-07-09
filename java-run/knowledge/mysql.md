# MySQL — 部署与数据注入策略

## When

- 本地无 MySQL CLI（`where mysql` 失败，且 `C:\Program Files\MySQL\` 搜索无结果）
- MySQL 服务未启动或无法连接
- 数据库不存在 / 为空 / 中文乱码

## Goal

MySQL 服务可连接，所有 SQL 正确导入，SELECT 中文验证通过。

## Default Credentials

- 用户：`root`
- 密码：`123456`
- 端口：3306 起（被占用自动递增）

---

## Deployment：免安装 ZIP 静默部署

### Port Detection

从 3306 起轮询 `Get-NetTCPConnection`，被占用则递增：

```powershell
$port = 3306
while ((Get-NetTCPConnection -LocalPort $port -ErrorAction SilentlyContinue).Count -gt 0) {
    $port++
}
```

### Version Selection

| 推断版本 | 下载链接 |
|----------|----------|
| MySQL 8.x | `https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.45-winx64.zip` |
| MySQL 5.x | `https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.44-winx64.zip` |

### Install & Initialize

```powershell
# 1. Download
New-Item -ItemType Directory -Force -Path "C:\local_mysql"
Set-Location "C:\local_mysql"
Invoke-WebRequest -Uri $url -OutFile "mysql.zip"

# 2. Extract
Expand-Archive -Path "mysql.zip" -DestinationPath "C:\local_mysql\temp" -Force
$extracted = Get-ChildItem "C:\local_mysql\temp" -Directory | Select-Object -First 1
Move-Item -Path $extracted.FullName -Destination "C:\local_mysql\server" -Force
Remove-Item -Recurse -Force "C:\local_mysql\temp", "mysql.zip"

# 3. my.ini (dynamic port)
$my_ini = @"
[mysqld]
port=$port
basedir=C:\local_mysql\server
datadir=C:\local_mysql\server\data
character-set-server=utf8mb4
default-authentication-plugin=mysql_native_password
"@
$my_ini | Out-File -FilePath "C:\local_mysql\server\my.ini" -Encoding utf8

# 4. Initialize
cd "C:\local_mysql\server\bin"
.\mysqld --initialize-insecure --user=mysql

# 5. Start service (fallback to background process)
try {
    .\mysqld --install MySQL_Auto --defaults-file="C:\local_mysql\server\my.ini" -ErrorAction Stop
    Start-Service -Name "MySQL_Auto" -ErrorAction Stop
} catch {
    Start-Process -FilePath ".\mysqld.exe" -ArgumentList "--defaults-file=C:\local_mysql\server\my.ini" -WindowStyle Hidden
}

# 6. Set password (default: 123456)
Start-Sleep -Seconds 3
$default_pwd = "123456"
$result = .\mysql -u root -P $port --skip-password -e "SELECT 1;" 2>&1
if ($LASTEXITCODE -eq 0) {
    $final_pwd = if ($project_pwd) { $project_pwd } else { $default_pwd }
    .\mysql -u root -P $port --skip-password -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '$final_pwd'; FLUSH PRIVILEGES;"
} else {
    # Try default → project password
    $result2 = .\mysql -u root -P $port -p"$default_pwd" -e "SELECT 1;" 2>&1
    if ($LASTEXITCODE -ne 0) {
        $result3 = .\mysql -u root -P $port -p"$project_pwd" -e "SELECT 1;" 2>&1
    }
}
```

### Verify

```powershell
C:\local_mysql\server\bin\mysql.exe -u root -p[password] -P [port] -e "SELECT 1;"
```

---

## SQL Import：编码安全的数据库注入

### Import Order

1. 有数字序号按序号执行
2. 无序号：`CREATE TABLE` 优先 → `INSERT INTO` 次之 → 权限/安全脚本最后
3. MySQL CLI `source` 命令优先（路径用单引号包裹）

### Pre-Import：禁用外键检查

```powershell
& "$mysql_path" -u root -p[密码] -P [端口] --default-character-set=utf8mb4 [库名] -e "SET FOREIGN_KEY_CHECKS = 0;"
```

### Import

```powershell
& "$mysql_path" -u root -p[密码] -P [端口] --default-character-set=utf8mb4 [库名] -e "source 'C:\path\to\file.sql'"
```

### Post-Import：恢复外键

```powershell
& "$mysql_path" -u root -p[密码] -P [端口] --default-character-set=utf8mb4 [库名] -e "SET FOREIGN_KEY_CHECKS = 1;"
```

### Verify

通过 Python dbapi（charset='utf8mb4'）执行 `SELECT` 验证中文内容，避免终端 GBK 误判。

---

## Repair Strategy

| 现象 | Repair |
|------|--------|
| 端口占用 | 杀旧进程后重试；仍失败则递增端口 |
| 密码不匹配 | 保留当前密码，覆写 application-dev.yml 对齐 |
| 中文 `????` | 切换到 Python subprocess + UTF-8 input 流 |
| 表已存在/外键冲突 | `DROP DATABASE` 重建，按 DDL→DML 重排后重导入 |
| `DELIMITER //` 报错 | 跳过存储过程，不影响核心功能 |
