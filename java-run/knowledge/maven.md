# Maven — 阿里云镜像加速

## When

- 中国时区（UTC+8）
- `mvn` 首次执行下载速度 < 500KB/s 或超时
- 用户提及"国内"、"阿里云"、"镜像"

## Goal

Maven 依赖下载走阿里云镜像，构建在 300s 内完成。

## Success

`mvn` 命令正常完成，依赖解析不走 Maven Central 直连。

---

## Repair：注入阿里云镜像

```powershell
# 1. 定位 settings.xml
$m2_settings = "$env:USERPROFILE\.m2\settings.xml"
$maven_home_settings = "$env:MAVEN_HOME\conf\settings.xml"
$target = if (Test-Path $m2_settings) { $m2_settings } else { $maven_home_settings }

# 2. 备份
Copy-Item -Path $target -Destination "$target.bak" -Force

# 3. 注入阿里云镜像
[xml]$xml = Get-Content -Path $target -Encoding UTF8
if ($xml.settings.mirrors -eq $null) {
    $mirrors = $xml.CreateElement("mirrors")
    $xml.settings.AppendChild($mirrors)
}
$mirror = $xml.CreateElement("mirror")
$mirror.InnerXml = @"
<id>aliyunmaven</id>
<mirrorOf>*</mirrorOf>
<name>阿里云公共仓库</name>
<url>https://maven.aliyun.com/repository/public</url>
"@
$xml.settings.mirrors.PrependChild($mirror)
$xml.Save($target)
```

> **仅注入阿里云中央仓库镜像**。不额外添加 Spring 代理仓。项目 `pom.xml` 中已有 `<repositories>` 保留不动。

## Fallback

文件权限不足 / 非 XML 格式 → 跳过镜像配置，使用原始源（不阻塞流程）。
