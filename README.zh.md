# Scala 最佳实践 Skill

[English](README.md)

一个 **AI 编码 Agent skill**，强制执行 Scala 开发规范——覆盖 **项目脚手架** 和 **日常编码标准**，支持 Scala 3、2.13、2.12。主要为 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 设计，但也兼容任何从 `~/.claude/skills/` 或 `~/.agents/skills/` 加载 skill 定义的 Agent。

## 功能概述

### 创建项目

当你让 AI Agent 新建 Scala 项目时，本 skill 会确保：

- **先问构建工具**："选 Mill 1.x 还是 sbt？推荐 Mill"
- **询问包名**和 **HTTP 框架**（pekko-http / http4s / tapir）
- 生成正确的 `build.mill`（仅限 Mill 1.x 语法——禁用 `Agg`/`ivy`/`T{}`）
- 创建全部**根基文件**：`.scalafmt.conf`、`.scalafix.conf`、`fix.sh`、`check.sh`、`application.conf`、`run.sh`、`README.md`
- 自动生成 **OpenAPI + Swagger UI**（任何使用 Tapir 的项目）
- **host/port 必须可配置**（禁止硬编码）

### 开发规范

当你让 AI Agent 编写或审查 Scala 代码时，本 skill 强制执行：

| ✅ 推荐 | ❌ 禁止 |
|---|---|
| `val` / case class / `copy()` | `var` / 可变字段 |
| `Either[Error, A]` | 抛异常做控制流 |
| `match` + sealed trait/enum | `isInstanceOf` / `asInstanceOf` |
| `opt.getOrElse` / `opt.fold` | `opt.get` |
| 显式返回类型（public） | 省略返回类型 |

详见：[生态库惯例](references/ecosystem.md)、[命名与结构](references/naming-and-structure.md)、[格式化与 Lint](references/formatting.md)。

## 安装

### 手动安装

**Claude Code：**

```bash
git clone git@github.com:weiwen99/scala-best-practices-skill.git ~/.claude/skills/scala-best-practices
```

Claude Code 会自动加载 `~/.claude/skills/` 下的 skill，无需额外配置。

**其他 AI Agent（从 `~/.agents/skills/` 读取）：**

```bash
git clone git@github.com:weiwen99/scala-best-practices-skill.git ~/.agents/skills/scala-best-practices
```

## 使用示例

### 创建新的 Scala 项目

将以下 prompt 复制到你的 AI 编码 Agent：

> 请帮我开一个新的Scala3项目，里面实现一个Http Service，提供一个API：/v1/ping，返回用户端的信息，尽可能详尽。注意遵循 scala-best-practices-skill SKILL 开发规范。

Agent 会按以下流程执行：
1. 询问你选择 Mill 还是 sbt
2. 询问包名
3. 询问选择 pekko-http 还是 http4s（以及 tapir 还是原生）
4. 生成完整项目及所有根基文件
5. 自动在 `/docs` 挂载 OpenAPI 文档

### 检查代码是否符合规范

> 帮我审查这段 Scala 代码是否符合最佳实践——检查可变状态、异常控制流、缺少返回类型、.get 使用等问题。

### 添加新的 HTTP 端点

> 在我的 Tapir 项目中添加 `POST /v1/users` 端点，遵循 skill 规范。

## 版本覆盖

| Scala 版本 | 项目脚手架 | 开发规范 |
|---|---|---|
| 3.x | ✅ 完整支持 | ✅ 完整支持 |
| 2.13 | ❌（手动用 sbt） | ✅ 完整支持 |
| 2.12 | ❌（手动用 sbt） | ✅ 完整支持 |

详见 [SKILL.md](SKILL.md) 中的 Scala 2 语法兼容对照表。

## 项目结构

```
.
├── SKILL.md                          # Skill 定义（AI Agent 读取此文件）
├── references/
│   ├── ecosystem.md                  # PureConfig / Circe / Tapir / Slick / 测试 / 日志
│   ├── formatting.md                 # .scalafmt.conf + .scalafix.conf（Scala 3 和 2）
│   ├── mill-build-reference.md       # Mill 1.x 构建模板
│   ├── naming-and-structure.md       # 命名规范、包结构
│   └── readme-template.md            # 生成项目的 README 模板
├── README.md                         # 英文版
├── README.zh.md                      # 本文件（中文）
└── LICENSE                           # Apache 2.0
```

## 许可证

Apache 2.0 — 详见 [LICENSE](LICENSE)。
