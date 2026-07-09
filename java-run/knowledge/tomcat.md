# Tomcat — WAR 项目部署

## When

`pom.xml` 中 `<packaging>war</packaging>`，项目为纯 Servlet/JSP（非 Spring Boot 内嵌）。

## Goal

Tomcat 启动成功，`curl http://localhost:8080/<context-path>` 返回 200。

---

## Deployment

### 1. Locate or Download Tomcat

优先级：`{{root_path}}/tomcat` > `C:\local_tomcat` > 全局安装 > 下载

```powershell
# Download Tomcat 10.1.x
mkdir -p {{root_path}}/tomcat
curl -L -o {{root_path}}/tomcat/tomcat.zip "https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.56/bin/apache-tomcat-10.1.56.zip"
unzip -q {{root_path}}/tomcat/tomcat.zip -d {{root_path}}/tomcat/
mv {{root_path}}/tomcat/apache-tomcat-*/* {{root_path}}/tomcat/
rm -rf {{root_path}}/tomcat/apache-tomcat-* {{root_path}}/tomcat/tomcat.zip
```

### 2. Fix Console Encoding（Windows GBK 环境）

创建 `{{root_path}}/tomcat/bin/setenv.bat`：

```bat
@echo off
set "CATALINA_OPTS=-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8"
```

同时确认 `conf/logging.properties` 中 `ConsoleHandler.encoding = UTF-8`。

### 3. Build & Deploy

```powershell
mise exec -- mvn clean package -DskipTests -q
copy /Y target\*.war {{root_path}}\tomcat\webapps\
```

### 4. Start

```powershell
$env:JAVA_HOME = "[mise managed JDK path]"
Start-Process {{root_path}}\tomcat\bin\startup.bat -WindowStyle Hidden
Start-Sleep 8
curl http://localhost:8080/[context-path]
```

### Stop

```powershell
{{root_path}}\tomcat\bin\shutdown.bat
```

---

## Repair

| 现象 | Repair |
|------|--------|
| 启动闪退 | 确认 JAVA_HOME 指向 JDK（非 JRE） |
| 控制台中文乱码 | 创建 setenv.bat + 检查 logging.properties |
| 404 | 确认 WAR 包名 = URL context-path，检查 web.xml welcome-file-list |
