# Scala Best Practices Skill

[中文版](README.zh.md)

An **AI coding agent skill** that enforces Scala development conventions — covering both **project scaffolding** and **ongoing coding standards** for Scala 3, 2.13, and 2.12. Primarily designed for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), but compatible with any agent that loads skill definitions from `~/.claude/skills/` or `~/.agents/skills/`.

## What This Skill Does

### Project Creation

When you ask your AI agent to create a new Scala project, this skill ensures:

- Asks you to choose **Mill 1.x or sbt** before creating any files
- Asks for the **package name** and **HTTP framework** (pekko-http / http4s / tapir)
- Generates correct `build.mill` (Mill 1.x syntax only — no `Agg`/`ivy`/`T{}`)
- Creates all **foundation files**: `.scalafmt.conf`, `.scalafix.conf`, `fix.sh`, `check.sh`, `application.conf`, `run.sh`, `README.md`
- Auto-generates **OpenAPI + Swagger UI** for any Tapir-based project
- Enforces **configurable host/port** (no hardcoded values)

### Development Conventions

When you ask your AI agent to write or review Scala code, this skill enforces:

| ✅ Do | ❌ Avoid |
|---|---|
| `val` / case class / `copy()` | `var` / mutable fields |
| `Either[Error, A]` | throwing exceptions for control flow |
| `match` + sealed trait/enum | `isInstanceOf` / `asInstanceOf` |
| `opt.getOrElse` / `opt.fold` | `opt.get` |
| explicit return types (public) | omit return types |

Full details: [ecosystem conventions](references/ecosystem.md), [naming & structure](references/naming-and-structure.md), [formatting & lint](references/formatting.md).

## Installation

### Manual Installation

**For Claude Code:**

```bash
git clone git@github.com:weiwen99/scala-best-practices-skill.git ~/.claude/skills/scala-best-practices
```

Claude Code automatically loads skills from `~/.claude/skills/` — no extra configuration needed.

**For other AI agents (that read from `~/.agents/skills/`):**

```bash
git clone git@github.com:weiwen99/scala-best-practices-skill.git ~/.agents/skills/scala-best-practices
```

## Example Usage

### Create a New Scala Project

Copy this prompt into your AI coding agent:

> Please help me create a new Scala 3 project with an HTTP Service that provides an API: /v1/ping, which returns client-side information in as much detail as possible. Follow the scala-best-practices-skill SKILL conventions.

The agent will then:
1. Ask you to choose Mill or sbt
2. Ask for the package name
3. Ask you to choose pekko-http or http4s (and tapir vs native)
4. Generate the complete project with all foundation files
5. Auto-generate OpenAPI docs at `/docs`

### Check Your Code Against Conventions

> Review my Scala code for best practices compliance — check for mutable state, exception-based control flow, missing return types, and `.get` usage.

### Add a New HTTP Endpoint

> Add a `POST /v1/users` endpoint to my Tapir-based project, following skill conventions.

## Version Coverage

| Scala Version | Project Scaffolding | Dev Conventions |
|---|---|---|
| 3.x | ✅ Full support | ✅ Full support |
| 2.13 | ❌ (use sbt manually) | ✅ Full support |
| 2.12 | ❌ (use sbt manually) | ✅ Full support |

See [SKILL.md](SKILL.md) for the Scala 2 syntax compatibility table.

## Project Structure

```
.
├── SKILL.md                          # Skill definition (the AI agent reads this)
├── references/
│   ├── ecosystem.md                  # PureConfig / Circe / Tapir / Slick / Testing / Logging
│   ├── formatting.md                 # .scalafmt.conf + .scalafix.conf (Scala 3 & 2)
│   ├── mill-build-reference.md       # Mill 1.x build templates
│   ├── naming-and-structure.md       # Naming conventions, package layout
│   └── readme-template.md            # README template for generated projects
├── README.md                         # This file (English)
├── README.zh.md                      # Chinese version
└── LICENSE                           # Apache 2.0
```

## License

Apache 2.0 — see [LICENSE](LICENSE).
