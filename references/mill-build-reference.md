# Mill 1.x 构建文件参考

> **动机**：Agent 常常自行生成 Mill 0.x 风格代码（`Agg()`、`ivy"..."`、`T {}` 等），导致反复编译失败。本文档提供 Mill 1.x 的正确写法，**禁止使用任何 0.x 遗留 API**。
>
> ⚠️ **版本声明**：以下模板中的版本号（Scala、http4s、circe、logback、munit 等）**仅为示例占位符**。生成项目时必须使用**最新的稳定版本**，通过官网 / Maven Central / GitHub Releases 查询。

---

## ⛔ 禁止使用的 Mill 0.x 写法

| 0.x（禁止） | 1.x（正确） | 说明 |
|---|---|---|
| `import mill._` | `import mill.*` | Scala 3 通配导入 |
| `Agg(ivy"...")` | `Seq(mvn"...")` | `Agg` 已移除，`ivy` 已废弃 |
| `def ivyDeps` | `def mvnDeps` | 1.x 重命名 |
| `ivy"org::lib:ver"` | `mvn"org::lib:ver"` | `mvn` 字符串插值器 |
| `T { ... }` 包简单返回值 | 直接返回值 | 简单 `def` 不需要 `T` 包装 |
| `T.command { ... }` | `Task.Command { ... }` | 1.x 改名 |
| `Task.Source(path)` | `Task.Source(path)` | 不变，但必须 import `mill.*` |
| `mill.define.Target` | 直接用 `Task` | 1.x 简化了 task API |
| `override def` | `def`（通常不需要 override） | Mill 1.x 大部分方法非 final |

---

## 标准单模块模板（含打包）

> **生成项目必须用这个模板，不能更简。** `mill app.universalStage` 和 `mill app.assembly` 必须可用。

```scala
import mill.*
import mill.api.BuildCtx
import scalalib.*

object app extends ScalaModule:

  def scalaVersion = "3.8.3"

  def scalacOptions = Seq(
    "-deprecation",
    "-feature",
    "-Wunused:all",
    "-source:future",
  )

  def mvnDeps = Seq(
    mvn"org.http4s::http4s-ember-server:0.23.30",
    mvn"io.circe::circe-core:0.14.10",
    mvn"ch.qos.logback:logback-classic:1.5.15",
  )

  def mainClass = Some("com.example.Main")

  // ── 打包：fat JAR ──
  // ServiceLoader 按行追加语义，META-INF/services/* 必须 append 不能覆盖（Flyway / JDBC 等 SPI 注册）。
  // AppendPattern 默认 separator = ""，会吞掉边界 entry，必须显式给 "\n"。
  def assemblyRules =
    super.assemblyRules ++ Seq(
      mill.scalalib.Assembly.Rule.AppendPattern("META-INF/services/.*", "\n")
    )

  // ── 打包：散 JAR + 配置文件 + run.sh（universalStage） ──
  // 优势：无 META-INF/services 合并坑、Docker 分层友好、排查依赖方便。

  /** 部署侧可编辑配置（logback.xml）→ conf/。reference.conf / application.conf 仍走 JAR 内 classpath。 */
  def logbackResource: T[PathRef] = Task.Source(moduleDir / "resources" / "logback.xml")
  def configMappings: T[Seq[(PathRef, os.SubPath)]] = Task {
    val ref = logbackResource()
    if os.exists(ref.path) then Seq(ref -> (os.sub / "conf" / "logback.xml"))
    else Seq.empty
  }

  /** classpath：本 module jar + 所有 transitive module jars + 所有 mvnDep jars → lib/ */
  def classpathMappings: T[Seq[(PathRef, os.SubPath)]] = Task {
    val selfMapping = Seq(jar() -> (os.sub / "lib" / (artifactName() + ".jar")))
    val moduleMappings =
      Task.traverse(transitiveModuleDeps.distinct) { module =>
        Task.Anon { module.jar() -> (os.sub / "lib" / (module.artifactName() + ".jar")) }
      }()
    val mvnMappings =
      resolvedRunMvnDeps().toSeq.map(dep => dep -> (os.sub / "lib" / dep.path.last))
    selfMapping ++ moduleMappings ++ mvnMappings
  }

  /** 项目根的 run.sh（universalStage 原样带过去并赋可执行位） */
  def runScriptSource: T[PathRef] = Task.Source(BuildCtx.workspaceRoot / "run.sh")

  /** universalStage 产物：lib/（散 jar）+ conf/ + run.sh */
  def universalStage: T[PathRef] = Task {
    val dest = Task.dest
    os.remove.all(dest)
    os.makeDir.all(dest)
    for ((ref, sub) <- classpathMappings() ++ configMappings()) do
      val target = dest / sub
      os.makeDir.all(target / os.up)
      os.copy(from = ref.path, to = target, replaceExisting = true)
    val runSh = runScriptSource().path
    if os.exists(runSh) then
      os.copy(from = runSh, to = dest / "run.sh", replaceExisting = true)
      os.perms.set(dest / "run.sh", "rwxr-xr-x")
    PathRef(dest)
  }
```

## 多模块模板

```scala
package build

import mill.*
import mill.api.BuildCtx
import scalalib.*

// ── 版本集中管理 ──
object Versions:
  val scalaV = "3.8.3"
  val munitV = "1.0.0"

// ── 公共 trait（共享 scalaVersion / scalacOptions） ──
trait CommonModule extends ScalaModule:
  def scalaVersion = Versions.scalaV
  def scalacOptions = Seq(
    "-deprecation",
    "-feature",
    "-Wunused:all",
    "-source:future",
  )

// ── 业务模块 ──
object core extends CommonModule:
  def mvnDeps = Seq(
    mvn"io.circe::circe-core:0.14.10",
  )
  object test extends ScalaTests with TestModule.Munit:
    def mvnDeps = Seq(mvn"org.scalameta::munit:${Versions.munitV}")

// ── 主模块（含打包） ──
object server extends CommonModule:
  def moduleDeps = Seq(core)

  def mainClass = Some("com.example.Main")

  // assembly：单 fat JAR
  // ServiceLoader 按行追加语义，META-INF/services/* 必须 append 不能覆盖。
  // AppendPattern 默认 separator = ""，必须显式给 "\n"。
  def assemblyRules =
    super.assemblyRules ++ Seq(
      mill.scalalib.Assembly.Rule.AppendPattern("META-INF/services/.*", "\n")
    )

  // universalStage：散 JAR + 配置文件 + run.sh
  // 优势：无 META-INF/services 合并坑、Docker 分层友好、排查依赖方便。

  /** 部署侧可编辑配置（logback.xml）→ conf/ */
  def logbackResource: T[PathRef] = Task.Source(moduleDir / "resources" / "logback.xml")
  def configMappings: T[Seq[(PathRef, os.SubPath)]] = Task {
    val ref = logbackResource()
    if os.exists(ref.path) then Seq(ref -> (os.sub / "conf" / "logback.xml"))
    else Seq.empty
  }

  /** classpath：本 module jar + transitive module jars + mvnDep jars → lib/ */
  def classpathMappings: T[Seq[(PathRef, os.SubPath)]] = Task {
    val selfMapping = Seq(jar() -> (os.sub / "lib" / (artifactName() + ".jar")))
    val moduleMappings =
      Task.traverse(transitiveModuleDeps.distinct) { module =>
        Task.Anon { module.jar() -> (os.sub / "lib" / (module.artifactName() + ".jar")) }
      }()
    val mvnMappings =
      resolvedRunMvnDeps().toSeq.map(dep => dep -> (os.sub / "lib" / dep.path.last))
    selfMapping ++ moduleMappings ++ mvnMappings
  }

  /** 项目根的 run.sh（universalStage 原样带过去并赋可执行位） */
  def runScriptSource: T[PathRef] = Task.Source(BuildCtx.workspaceRoot / "run.sh")

  /** universalStage 产物：lib/（散 jar）+ conf/ + run.sh */
  def universalStage: T[PathRef] = Task {
    val dest = Task.dest
    os.remove.all(dest)
    os.makeDir.all(dest)
    for ((ref, sub) <- classpathMappings() ++ configMappings()) do
      val target = dest / sub
      os.makeDir.all(target / os.up)
      os.copy(from = ref.path, to = target, replaceExisting = true)
    val runSh = runScriptSource().path
    if os.exists(runSh) then
      os.copy(from = runSh, to = dest / "run.sh", replaceExisting = true)
      os.perms.set(dest / "run.sh", "rwxr-xr-x")
    PathRef(dest)
  }

  def mvnDeps = Seq(
    mvn"org.http4s::http4s-ember-server:0.23.30",
  )
  object test extends ScalaTests with TestModule.Munit:
    def mvnDeps = Seq(mvn"org.scalameta::munit:${Versions.munitV}")
```

---

## Task API 速查

| 用途 | 写法 | 说明 |
|---|---|---|
| 简单常量 | `def foo = "bar"` | 不需要任何包装 |
| 依赖序列 | `def mvnDeps = Seq(mvn"...")` | 直接返回 `Seq` |
| 计算任务 | `def foo: T[PathRef] = Task { ... }` | 需要读取其他 task 或访问文件时用 `Task {}` |
| 文件输入 | `def src: T[PathRef] = Task.Source(path)` | 声明文件依赖，Mill 感知变化 |
| 用户命令 | `def deploy(): Command[Unit] = Task.Command { ... }` | CLI 可见的 `mill app.deploy` |
| 匿名任务 | `Task.Anon { ... }` | 在另一个 Task 内部临时构造子任务 |
| 多任务遍历 | `Task.traverse(seq)(f)` | 并行执行多个 task，收集结果 |

---

## Mill 1.x vs sbt 常用命令对照

| 操作 | Mill | sbt |
|---|---|---|
| 编译 | `./mill app.compile` | `sbt compile` |
| 运行 | `./mill app.run` | `sbt run` |
| 测试 | `./mill app.test` | `sbt test` |
| 格式化 | `./mill __.reformat` | `sbt scalafmtAll` |
| Lint 修复 | `./mill __.fix` | `sbt scalafixAll` |
| 打包 fat JAR | `./mill app.assembly` | `sbt assembly` |
| 打包散 JAR | `./mill app.universalStage` | `sbt universal:stage` |
| REPL | `./mill app.repl` | `sbt console` |
| 显示依赖树 | `./mill app.ivyDepsTree` | `sbt dependencyTree` |

---

## Mill 项目根目录文件清单

```
project-root/
├── build.mill          # 构建定义（Scala 3 语法）
├── .mill-version       # 锁定 Mill 版本（使用官网最新稳定版）
├── mill                # 可执行 wrapper 脚本
├── .scalafmt.conf      # 代码格式规范
├── .scalafix.conf      # 静态检查规则
├── fix.sh              # scalafix + scalafmt
├── check.sh            # CI 自检
├── run.sh              # 生产启动脚本
└── app/
    ├── src/            # 源码
    └── resources/      # 资源文件（含 application.conf）
```

> **`./mill` wrapper 下载方式**：参考官方文档
> - https://mill-build.org/mill/cli/installation-ide.html
> - https://mill-build.org/mill
>
> 也可以通过 `curl` / `wget` 直接下载 wrapper 脚本到项目根目录并 `chmod +x mill`。`.mill-version` 内容为 Mill 版本号，**使用官网最新的稳定版本**（非写死的旧版本号）。
> - https://mill-build.org/mill/cli/installation-ide.html
> - https://mill-build.org/mill
