---
name: my-frontend-skills
description: 个人前端开发笔记。包含 pnpm/npm 问题、环境配置、常用命令等踩坑记录。
metadata:
  version: "2026.5.15"
---

# 个人前端开发笔记

## 环境与工具

### pnpm 全局安装命令找不到

**症状**：通过 `pnpm -g` 安装了包，但命令提示 `not recognized`。

**原因**：
- pnpm 的 `global-bin-dir`（即命令链接存放的位置）默认不在系统 PATH 中，或在升级包时未能重建命令链接
- 命令链接文件（`.cmd` / `.ps1`）存放在 `global-bin-dir` 下，pnpm 通过它们找到实际的可执行文件

**排查步骤**：
1. 检查包是否已安装：`pnpm ls -g`
2. 确认 global-bin-dir：`pnpm bin -g`
3. 查看该目录下是否有对应命令文件：`ls <global-bin-dir>/*.cmd`
4. 如果包存在但命令链接缺失 → 重新安装即可

**修复命令**：
```bash
pnpm add -g <包名>
```
或用 `--force` 强制重建：
```bash
pnpm install -g <包名> --force
```

**相关命令对比**：
| 命令 | 说明 |
|------|------|
| `pnpm add -g <pkg>` | 全局安装并创建命令链接 |
| `pnpm i -g <pkg>` | 等价于 `add -g`，但包已存在时可能跳过重建链接 |
| `pnpm bin -g` | 查看全局命令链接目录 |

### pnpm 全局配置查看

```bash
pnpm config list                    # 查看所有配置
pnpm root -g                        # 全局包安装目录
pnpm bin -g                         # 全局命令链接目录
pnpm ls -g                          # 已安装的全局包
```

### 常用 pnpm 命令速查

```bash
pnpm add <pkg>                      # 安装到 dependencies
pnpm add -D <pkg>                   # 安装到 devDependencies
pnpm remove <pkg>                   # 卸载
pnpm up <pkg>                       # 更新
pnpm audit                          # 安全审计
pnpm commit                         # commitizen 提交（本仓库约定）
pnpm store path                     # 查看 store 目录
```

### Windows 多 Java 版本切换

项目可能需要不同 Java 版本（如 Java 8 用于旧项目，Java 17 用于新项目）。

**切换方法（PowerShell）**：
```powershell
# 切换到 Java 17
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.8.101-hotspot"
$env:Path = "$env:JAVA_HOME\bin;$env:Path"

# 切换到 Java 8
$env:JAVA_HOME = "C:\Program Files\Java\jdk1.8.0_151"
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
```

**验证**：
```powershell
java -version
javac -version
$env:JAVA_HOME
```

**永久设置**（管理员权限）：
```powershell
[System.Environment]::SetEnvironmentVariable("JAVA_HOME", "你的JDK路径", "Machine")
```

> 注意：JRE 只有 `java` 没有 `javac`，编译需要 JDK。报错 `No compiler is provided in this environment` 说明当前用的是 JRE 而非 JDK。

### MongoDB 本地环境快速搭建

项目依赖 MongoDB 但远程连不上时，用 Docker 一行搞定：

```bash
docker run -d -p 27017:27017 --name mongodb-local mongo:latest
```

验证：
```bash
docker ps | grep mongodb-local
mongosh    # 连接测试，看到 shell 提示符即可
```

配置文件改为本地连接：
```yaml
spring:
  data:
    mongodb:
      host: localhost
      port: 27017
      database: 你的数据库名
```
