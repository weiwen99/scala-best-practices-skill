# Scalafmt + Scalafix Formatting & Lint Rules

`.scalafmt.conf` and `.scalafix.conf` are the foundation config for code formatting and static analysis. **These configs are build-tool agnostic** (Mill / sbt behave identically); the only difference is how you invoke them. This document provides versions for both Scala 3 and Scala 2.13/2.12.

## .scalafmt.conf

### Scala 3

```hocon
version = "3.10.6"
maxColumn = 200
align.preset = most
continuationIndent.defnSite = 2
runner.dialect = scala3

project {
  excludePaths = [
    "glob:**/target/**",
    "glob:**/out/**",
    "glob:**/.metals/**"
  ]
}

rewrite.imports.groups = [
  ["java[x]*\\..*"],
  ["scala\\..*"],
  ["^(?!(com.example.))|(?!(com.example.)).*"]
]
newlines.topLevelStatementBlankLines = [{
    blanks { before = 1 }
  }]

rewrite.rules = [ AvoidInfix, SortImports, RedundantBraces, RedundantParens, SortModifiers, PreferCurlyFors ]
```

### Scala 2.13 / 2.12

Only change `runner.dialect`. All other settings are identical.

```hocon
version = "3.10.6"
maxColumn = 200
align.preset = most
continuationIndent.defnSite = 2
runner.dialect = scala213    # ← only difference
```

(Use `scala212` for Scala 2.12 projects.)

### Key Settings Explained

- `maxColumn = 200` — Modern displays + Scala generic/long-chain calls make 200 more practical than 120. This is an **upper limit**, not a target; break lines where it improves readability.
- `align.preset = most` — Align similar declarations (assignment `=`, arrows `=>`, colons, etc.) for faster visual scanning.
- `continuationIndent.defnSite = 2` — 2-space indent at definition sites, consistent with normal code indent (avoiding the visual gap of 4-space indent in parameter lists).
- `runner.dialect = scala3` — Force Scala 3 syntax recognition (`given` / `using` / `enum` / `extension`, etc.). For Scala 2, use `scala213` or `scala212`.
- `excludePaths` — Exclude build output directories to avoid formatting generated code.

### Rewrite Rules

| Rule | Effect |
|---|---|
| `AvoidInfix` | Convert unsuitable infix method calls back to dot notation (`a method b` → `a.method(b)`) |
| `SortImports` | Sort imports alphabetically within each group |
| `RedundantBraces` | Remove redundant braces (single-line lambda `{ x => ... }` → `x => ...`) |
| `RedundantParens` | Remove redundant parentheses |
| `SortModifiers` | Enforce modifier order (`private final override` → `override final private`) |
| `PreferCurlyFors` | `for (x <- ..) yield ..` → `for { x <- .. } yield ..` |

### Import Group Order

scalafmt sorts imports into three groups, separated by blank lines:

```scala
import java.time.LocalDateTime
import java.util.UUID
import javax.sql.DataSource

import scala.concurrent.Future
import scala.util.Try

import com.example.base.models.ApiResponse
import com.example.internal.xxx.YyyClient
import io.circe.generic.auto.*    // Scala 3: import foo.*
// Scala 2: io.circe.generic.auto._  (wildcard uses `_` not `*`)
import org.apache.pekko.http.scaladsl.server.Route
```

**Order**:
1. `java` / `javax.*`
2. `scala.*`
3. Everything else (project packages + third-party libs, alphabetically within group)

> The regex `^(?!(com.example.))|(?!(com.example.)).*` puts everything except java/scala into group 3. No need to manually separate project and third-party imports.

**Scala 2 note**: wildcard imports use `_` instead of `*` (e.g., `import io.circe.generic.auto._`), but scalafmt handles this automatically based on `runner.dialect`.

### Top-level Blank Lines

`newlines.topLevelStatementBlankLines = [{ blanks { before = 1 } }]` — Automatically inserts one blank line before each top-level `def` / `val` / `class` / `object` for visual grouping.

## .scalafix.conf

### Scala 3

```hocon
rules = [
  ExplicitResultTypes,
  DisableSyntax,
  RemoveUnused
]

RemoveUnused.imports = true
RemoveUnused.privates = true
RemoveUnused.locals = true
RemoveUnused.patternvars = true
RemoveUnused.params = true
OrganizeImports.targetDialect = Scala3

ExplicitResultTypes.memberKind = [Def, Val, Var]
ExplicitResultTypes.memberVisibility = [Public, Protected]
ExplicitResultTypes.fetchScala3CompilerArtifactsOnVersionMismatch = true

DisableSyntax.noVars = false
DisableSyntax.noThrows = false
DisableSyntax.noNulls = true
DisableSyntax.noReturns = false
DisableSyntax.noWhileLoops = false
DisableSyntax.noAsInstanceOf = true
DisableSyntax.noIsInstanceOf = true
DisableSyntax.noXml = true
DisableSyntax.noDefaultArgs = false
DisableSyntax.noFinalVal = true
DisableSyntax.noFinalize = true
DisableSyntax.noValPatterns = true
DisableSyntax.noGet = true
DisableSyntax.regex = []
```

### Scala 2.13 / 2.12

Three differences from the Scala 3 config:

1. `OrganizeImports.targetDialect` → `Scala2`
2. Remove `ExplicitResultTypes.fetchScala3CompilerArtifactsOnVersionMismatch` (Scala 2 doesn't download Scala 3 compiler)
3. `noFinalVal` → `false` (Scala 2 has no `inline val`; `final val` is the only constant declaration)

```hocon
rules = [
  ExplicitResultTypes,
  DisableSyntax,
  RemoveUnused
]

RemoveUnused.imports = true
RemoveUnused.privates = true
RemoveUnused.locals = true
RemoveUnused.patternvars = true
RemoveUnused.params = true
OrganizeImports.targetDialect = Scala2              # ← Scala 2 dialect

ExplicitResultTypes.memberKind = [Def, Val, Var]
ExplicitResultTypes.memberVisibility = [Public, Protected]
# fetchScala3CompilerArtifactsOnVersionMismatch removed — not needed for Scala 2

DisableSyntax.noVars = false
DisableSyntax.noThrows = false
DisableSyntax.noNulls = true
DisableSyntax.noReturns = false
DisableSyntax.noWhileLoops = false
DisableSyntax.noAsInstanceOf = true
DisableSyntax.noIsInstanceOf = true
DisableSyntax.noXml = true
DisableSyntax.noDefaultArgs = false
DisableSyntax.noFinalVal = false                   # ← Scala 2: final val is OK (no inline alternative)
DisableSyntax.noFinalize = true
DisableSyntax.noValPatterns = true
DisableSyntax.noGet = true
DisableSyntax.regex = []
```

### ExplicitResultTypes

- Targets: `Def` / `Val` / `Var`
- Visibility: `Public` / `Protected`
- **Private / local members are exempt**, but if type inference produces a complex chain (e.g., long `.map.flatMap.sequence`), consider hand-writing the signature to speed up IDE resolution.

#### Why Public Members Must Have Explicit Types

1. **API stability** — downstream consumers won't be broken by implementation changes that alter inferred types
2. **Documentation as signature** — the type is the most reliable form of documentation
3. **Faster IDE navigation and search** — no need for full compiler inference

**Scala 2 note**: `ExplicitResultTypes.fetchScala3CompilerArtifactsOnVersionMismatch` is a Scala-3-only setting. Remove it for Scala 2 projects. The rule itself works identically on both versions.

### RemoveUnused

Enable everything (imports / privates / locals / patternvars / params). **Discard unused params explicitly with `_`**:

```scala
// ❌ RemoveUnused will flag this
def handle(req: Request, ctx: Context): Response = Response.ok

// ✅
def handle(req: Request, _ctx: Context): Response = Response.ok
// or (if ctx is truly irrelevant)
def handle(req: Request): Response = Response.ok
```

### DisableSyntax Rules

| Rule | Setting | Meaning |
|---|---|---|
| `noVars` | `false` | **Allowed** (legacy compat); new code should avoid, see SKILL.md |
| `noThrows` | `false` | **Allowed** (framework boundaries); business logic uses Either |
| `noNulls` | `true` | Disallow `null` |
| `noReturns` | `false` | **Allowed** (legacy compat); new code avoids `return` |
| `noWhileLoops` | `false` | **Allowed** (performance-critical paths); prefer higher-order functions |
| `noAsInstanceOf` | `true` | Disallow `x.asInstanceOf[T]` |
| `noIsInstanceOf` | `true` | Disallow `x.isInstanceOf[T]` |
| `noXml` | `true` | Disallow XML literals |
| `noDefaultArgs` | `false` | Allow default arguments |
| `noFinalVal` | `true` (Scala 3) / `false` (Scala 2) | Scala 3: prefer `inline val` over `final val`. Scala 2: `final val` is the only constant option |
| `noFinalize` | `true` | Disallow `override def finalize()` |
| `noValPatterns` | `true` | Disallow top-level tuple destructuring `val (a, b) = tup` |
| `noGet` | `true` | Disallow `.get` on Option/Try/etc. |

### OrganizeImports.targetDialect

- **Scala 3**: `Scala3` — supports `import foo.*` (wildcard), `import foo.{given, *}`, etc.
- **Scala 2**: `Scala2` — uses traditional `import foo._` wildcard syntax.

## fix.sh / check.sh

The project root `fix.sh` and `check.sh` are unified entry points. Internally they differ by build tool. Works identically for Scala 3 and Scala 2.

### Mill 1.x (Recommended)

```bash
# fix.sh
./mill __.fix       # scalafix auto-fix
./mill __.reformat  # scalafmt auto-format

# check.sh (CI equivalent)
./mill __.checkFormat   # scalafmt check (no file changes)
./mill _.fix --check    # scalafix check (no file changes)
./mill __.test          # run all tests
```

`__` matches all modules recursively; `_` matches only one level. `check.sh` uses `_.fix --check` (not `__.fix --check`) because some sub-modules (e.g., build plugins) don't run scalafix.

Mill plugins needed: built-in support for `scalafmt` and `scalafix` (`mill.scalalib.scalafmt.ScalafmtModule` and `mill.scalalib.scalafix.ScalafixModule`).

### sbt 1.10+ (Compatible)

```bash
# fix.sh
sbt "scalafixAll ; scalafmtAll ; scalafmtSbt"
# or individually:
sbt scalafixAll      # scalafix auto-fix all configs
sbt scalafmtAll      # scalafmt format sources
sbt scalafmtSbt      # format *.sbt build files

# check.sh (CI equivalent)
sbt scalafmtCheckAll ; scalafmtSbtCheck ; "scalafixAll --check" ; test
```

sbt plugins needed (in `project/plugins.sbt`):

```scala
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.5.2")
addSbtPlugin("ch.epfl.scala" % "sbt-scalafix" % "0.12.1")
```

Enable scalafix in `build.sbt`:

```scala
ThisBuild / scalafixDependencies += "com.github.liancheng" %% "organize-imports" % "0.6.0"
ThisBuild / semanticdbEnabled := true
ThisBuild / semanticdbVersion := scalafixSemanticdb.revision
```

> **Note**: sbt projects need `semanticdbEnabled := true` to run scalafix's `ExplicitResultTypes` rule (it requires semantic information). Mill handles this automatically in `ScalafixModule`. This applies to **both Scala 3 and Scala 2**.

### Build Tool Recommendation

For **new** projects, Mill 1.x is strongly recommended (cleaner config, faster builds). For **existing** sbt projects, migration is optional — this skill's code rules are build-tool agnostic and apply to both Scala 3 and Scala 2.

## Common Pitfalls

### Build-Tool Agnostic

- **First scalafix run is slow** — Downloads compiler artifacts. For Scala 3, this is triggered by `fetchScala3CompilerArtifactsOnVersionMismatch = true`. For Scala 2, ExplicitResultTypes still needs a one-time download but is generally faster.
- **scalafmt version mismatch** — The IDE scalafmt plugin version must match `version` in `.scalafmt.conf`, otherwise formatting results will drift.
- **checkFormat fails but reformat changes nothing** — Usually a syntax error in `.scalafmt.conf`. Run checkFormat standalone to see the detailed error.

### Mill-Specific

- **Formatting command must be `./mill __.reformat`** (with underscores), not `./mill reformat`. The latter will report "task not found."
- **Out of memory**: `MILL_OPTS="-Xmx8g" ./mill ...`

### sbt-Specific

- **scalafix reports "missing semanticdb"** — `semanticdbEnabled := true` or `semanticdbVersion` not set. See plugin config above.
- **scalafmtAll doesn't process `*.sbt` files** — Run `scalafmtSbt` / `scalafmtSbtCheck` separately.
- **Out of memory**: `SBT_OPTS="-Xmx8g" sbt ...` or set in `.sbtopts` / `.jvmopts`.
