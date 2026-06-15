---
name: scala-best-practices
description: ⚠️ ALWAYS ask "Mill 1.x or sbt? Recommend Mill" before creating any files. Mill strictly forbids Agg/ivy/T{}, must use Seq/mvn/mvnDeps. Foundation files: .scalafmt.conf .scalafix.conf fix.sh check.sh application.conf logback.xml run.sh. Functional immutable Either explicit return types. This skill covers BOTH project creation AND ongoing development conventions. 触发词 / Triggers: scala-best-practices scala3-best-practices create Scala project create scala3 project new Scala 3 project scala3 development conventions scala best practices scala3 best practices scaffold start a new project scala3 project scala3 coding style scala3 code style scala3 style guide scala3 code review scala conventions check how to write scala code scala3 patterns Either error handling functional programming scala immutable scala given using scala3 explicit return type scala3 dos and donts build.mill mvnDeps 创建 Scala 项目 新建 Scala3 项目 新建 Scala 3 项目 scala3 开发 规范 scala 规范 scala 开发规范 scala3 开发规范 scala 最佳实践 Scala3 最佳实践 开一个新项目 scala3 项目 scala3 代码风格 scala3 编码规范 函数式编程 Either 错误处理 不可变 显式返回类型 代码审查
tags: [scala, scala3, best-practices, conventions]
author: weiwen99
user-invocable: true
---

# Scala Best Practices

## ⛔ New Project: Always Ask Build Tool First

1. **ALWAYS ask first**: "Mill 1.x or sbt? Recommend Mill." **Do NOT create any files** before the user confirms.
2. **ALWAYS ask for the package name** (e.g. `com.example` / `org.myapp`). Default to `simple` if not specified.
3. **If the project uses HTTP Service, ALWAYS ask**: "pekko-http or http4s? Default http4s." If http4s is chosen, ask: "http4s native or tapir for the API layer? Default tapir."
4. **Any project using Tapir (regardless of pekko-http or http4s backend) MUST auto-generate OpenAPI docs and serve Swagger UI**:
   - Dependency: `com.softwaremill.sttp.tapir::tapir-swagger-ui-bundle`
   - Mount paths: `/docs` (Swagger UI), `/docs/openapi.json` (raw spec)
   - Collect all endpoint definitions into `List[AnyEndpoint]` and feed them in one shot when generating OpenAPI
   - Complex / opaque types must explicitly provide `given Schema[T]` to ensure doc quality
   - See `${SKILL_DIR}/references/ecosystem.md` section 9
5. **Mill build.mill strictly forbids 0.x APIs**: `Seq(mvn"...")` ✅, `Agg(ivy"...")` / `ivyDeps` / `T{}` ❌
6. **Foundation files are mandatory**: `README.md` `.scalafmt.conf` `.scalafix.conf` `fix.sh` `check.sh` `application.conf` `logback.xml` `run.sh`
7. **README.md must include**: project intro, getting started (dependency install), local development (`./mill __.compile` / `./mill __.test`), packaging (`./mill app.assembly` + `./mill app.universalStage`), how to run. Template at `${SKILL_DIR}/references/readme-template.md`
8. **`build.mill` must include `universalStage`**: `mill app.universalStage` must work (exploded JARs + config files + `run.sh`). `assembly` (fat JAR) must also work. Template at `${SKILL_DIR}/references/mill-build-reference.md` section 1
9. **HTTP host/port must be configurable** via `application.conf` (HOCON), never hardcoded
10. Mill projects must have `./mill` wrapper + `.mill-version`. Download the wrapper script and find the latest stable version from official docs:
    - https://mill-build.org/mill/cli/installation-ide.html
    - https://mill-build.org/mill
11. **All library versions (Scala, Mill, http4s, pekko-http, tapir, circe, logback, munit, etc.) must be the latest stable release** — do NOT hardcode old version numbers. Use the official website / Maven Central / GitHub releases to find the current stable version at project creation time.

## Code Style Quick Reference

| ✅ Do (Scala 3) | ✅ Do (Scala 2) | ❌ Avoid |
|---|---|---|
| `val` / case class / `copy()` | same | `var` / mutable fields |
| `Either[Error, A]` | same | throwing exceptions for control flow |
| `match` + sealed enum | `match` + sealed trait | `isInstanceOf` / `asInstanceOf` |
| `opt.getOrElse` / `opt.fold` | same | `opt.get` |
| `given` / `using` | `implicit` | — |
| `for { a <- f1; b <- f2(a) }` | same | nested `flatMap` |
| explicit return types (public) | same | omit return types |
| `inline val` | `final val` | — |

## Reference Files

| When You Need | Read This |
|---|---|
| Mill build templates, Task API | `${SKILL_DIR}/references/mill-build-reference.md` |
| README template | `${SKILL_DIR}/references/readme-template.md` |
| scalafmt / scalafix config (Scala 3 + Scala 2) | `${SKILL_DIR}/references/formatting.md` |
| Naming, package structure (Scala 3 + Scala 2) | `${SKILL_DIR}/references/naming-and-structure.md` |
| PureConfig / Circe / Tapir / Slick / Testing / Logging / OpenAPI+Swagger (Scala 3 + Scala 2) | `${SKILL_DIR}/references/ecosystem.md` |
| logback.xml template (console + rolling file to ./logs) | `${SKILL_DIR}/references/logback-template.xml` |

## Scala 2 Compatibility

The design principles in this skill (immutability, Either for errors, explicit types, no `.get`) apply to **both Scala 3 and Scala 2.13/2.12**. The `references/` files include Scala 2 syntax equivalents alongside Scala 3 examples. Key syntax mappings:

| Concept | Scala 3 | Scala 2 |
|---|---|---|
| Implicit params | `(using ec: ExecutionContext)` | `(implicit ec: ExecutionContext)` |
| Typeclass instances | `given Foo[T] = ...` | `implicit val foo: Foo[T] = ...` |
| Enum | `enum X { case A, B }` | `sealed trait X` + `case object A extends X` |
| Extension methods | `extension (x: T) def f = ...` | `implicit class TOps(val x: T) { def f = ... }` |
| Deriving typeclasses | `derives ConfigReader` | manual `implicit val` or `import auto._` |
| Wildcard import | `import foo.*` | `import foo._` |
| Opaque types | `opaque type Id = String` | `class Id(val value: String) extends AnyVal` |
| Inline constants | `inline val X = 1` | `final val X = 1` |
