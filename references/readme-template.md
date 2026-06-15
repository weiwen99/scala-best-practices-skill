# README 模板

> 生成项目时必须输出 `README.md`，包含以下全部章节。占位符 `${...}` 按实际项目替换。

```markdown
# ${ProjectName}

${一句话项目简介}

## 项目介绍

${2-5 句描述：项目做什么、技术栈、核心特性}

- **语言**：Scala ${scalaVersion}
- **构建**：Mill ${millVersion}
- **HTTP 框架**：${http4s / pekko-http / ...}
- **数据库**：${PostgreSQL / MySQL / 无}

## 快速开始

### 前置依赖

- JDK 21+
- Mill（项目自带 `./mill` wrapper，无需单独安装）

### 启动

```bash
# 编译
./mill __.compile

# 运行（开发模式）
./mill app.run
```

服务默认监听 `http://localhost:${port}`。

## 本地开发

```bash
# 格式化 + lint 修复
./fix.sh

# CI 自检（格式化 + lint + 测试）
./check.sh

# 仅运行测试
./mill __.test

# 持续编译（文件变更自动重编译）
./mill -w __.compile
```

## 打包

### fat JAR（单文件）

```bash
./mill app.assembly
# 产物：out/app/assembly.dest/out.jar
```

`java -jar out.jar` 直接启动。

### universalStage（散 JAR + 配置文件）

```bash
./mill app.universalStage
# 产物：out/app/universalStage.dest/
#   ├── lib/          # 所有依赖 JAR（散放）
#   ├── conf/         # 可编辑配置文件
#   └── run.sh        # 启动脚本
```

选 universalStage 的场景：Docker 分层部署、需要热改配置文件、排查依赖版本。

## 配置

编辑 `app/resources/application.conf`（HOCON 格式）：

```hocon
server {
  host = "0.0.0.0"
  port = ${port}
}
```

## API 文档

启动后访问 Swagger UI：`http://localhost:${port}/docs`
```

## 规则

- 占位符 `${...}` 用实际值替换，不要原样输出
- 如果项目无数据库，数据库行写"无"
- 如果项目无 HTTP 服务，去掉 API 文档、HTTP 框架和端口相关内容
- `scalaVersion` 和 `millVersion` 与 `build.mill` 保持一致
- 端口与 `application.conf` 保持一致
