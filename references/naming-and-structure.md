# 命名、包、模块结构

## 根包

推荐项目统一根包格式：

```
com.example.<project>[.<module>][.<feature>]
```

`<project>` 是项目代号（如 `analysis`、`gateway`），`<module>` 对应构建工具的子模块（Mill 或 sbt 的 subproject，如 `core`、`base`）。

### 模块名到包名的转换

子模块目录常用 kebab-case（`query-module`、`base-models`、`service-clients`、`etl-feature`），对应包路径**转 snake_case**（Scala 标识符不允许连字符）：

| 模块目录 | 包路径 |
|---|---|
| `app/` | `com.example.analysis` |
| `base/` | `com.example.base` |
| `base-models/` | `com.example.base.models` |
| `config/` | `com.example.base.config` |
| `core/` | `com.example.analysis.core` |
| `query-module/` | `com.example.analysis.query_module` |
| `service-clients/` | `com.example.service_clients` |
| `etl-feature/` | `com.example.analysis.etl_feature` |

> `service-clients` 之所以直接挂在 `com.example` 下而不是 `com.example.analysis`，是因为它被多个上游项目复用，属于"基础设施"而非"analysis 专属"。

## 类 / trait / object 命名

| 类别 | 命名 | 例子 | 备注 |
|---|---|---|---|
| HTTP 入口 | `XxxApi` | `OpenApi` / `QueryModuleOpenApi` / `TestApi` | **不**叫 `Controller` / `Routes` |
| 业务服务 | `XxxService` | `QueryService` / `OperatorService` | 无状态或以依赖注入方式持有状态 |
| 生命周期 / 状态管理 | `XxxManager` | `SessionManager` / `InstanceRegistry` | 持有可变运行时状态（通过 actor / 流控） |
| 数据访问 | `XxxRepository` | `QueryModuleRepository` / `SchedulerLeaderRepository` | 必须有伴生 `object { def apply(...) }` |
| Slick 表组件 | `XxxComponent` | `SchedulerLeaderComponent` | trait，extends `BaseComponent` |
| 实体 / 领域模型 | `Xxx` 或 `XxxEntity` | `SchedulerLeader` / `BaseInfo` | DB 实体建议加 `Entity` 后缀避免与 case class 冲突 |
| 枚举 | `XxxStatus` / 业务词 | `QueryModuleStatus` / `FilterOperator` / `LeaderRole` | Scala 3 `enum`；Scala 2: `sealed trait` + `case object` |
| 公共 trait | `BaseXxx` / `HasXxx` | `BaseApi` / `BaseComponent` / `HasDatabaseConfig` | `Base` 表示公共父类，`Has` 表示能力 mixin |
| 配置 case class | `XxxConfig` | `LeaderElectionConfig` / `QueryModuleConfig` | Scala 3: `derives ConfigReader`；Scala 2: `import pureconfig.generic.auto._` |
| 测试 | `XxxSpec` | `QueryServiceSpec` / `SimpleJsonSpec` | extends `munit.FunSuite` |
| 错误 / 领域异常 | `XxxError` | `ParseError` / `ApiResponse.Error` | Scala 3 `enum`；Scala 2: `sealed trait` + `case class`；深层级时混合 trait |

### Extension Methods

**Scala 3**: Use `extension`. **Do NOT write `*Ops` implicit classes**.

```scala
// ✅ Scala 3
extension (s: String) {
  def reversed: String = s.reverse
}
```

**Scala 2**: Use `implicit class` with `AnyVal` for zero-overhead:

```scala
// ✅ Scala 2
implicit class StringOps(val s: String) extends AnyVal {
  def reversed: String = s.reverse
}
```

Group related extensions together in a trait or object (see `BaseApi.withCode`).

### 伴生 `object` 的 `apply` 工厂

所有 Repository 必须有伴生 `object` + `apply`：

```scala
class QueryModuleRepository(
  protected val dbConfig: DatabaseConfig[JdbcProfile]
)(using ec: ExecutionContext)        // Scala 2: (implicit ec: ExecutionContext)
    extends HasDatabaseConfig[JdbcProfile]
    with QueryModuleComponent
    with StrictLogging { /* ... */ }

object QueryModuleRepository {
  def apply(dbConfig: DatabaseConfig[JdbcProfile])(using ec: ExecutionContext): QueryModuleRepository =  // Scala 2: (implicit ec: ...)
    new QueryModuleRepository(dbConfig)
}
```

**理由**：统一构造入口；未来想改成私有构造器 + smart constructor 只需要改 object，不破坏调用方。

## 文件命名

- 每个 **公共 class / trait / enum (Scala 3) / sealed trait (Scala 2) / object** 独占一个文件，文件名与类型名一致（`QueryModuleRepository.scala`）
- 一个文件里可以有多个**私有辅助**类型，但对外暴露的只能有一个同名类型
- 伴生 object 与主类型同一文件

## 模块依赖顺序

```
config → base-models → base → service-clients → core → app
                                              → example
                                              → query-module / etl / etl-feature（上层功能子模块）
```

**规则**：
- 下层模块**不能**反向依赖上层（`base` 不能依赖 `core`）
- `base-models` 只放**纯数据模型**（case class / enum）和最小依赖
- 外部服务客户端集中在 `service-clients`
- 配置读取统一 `config` 模块

### 新项目初始化时推荐模块

| 模块 | 作用 | 最小依赖 |
|---|---|---|
| `config` | PureConfig case class | PureConfig only |
| `base-models` | 实体、枚举、错误类型 | Circe + Tapir schema |
| `base` | Tapir / Pekko / Slick / DB 基础 | Pekko HTTP + Tapir + Slick + StrictLogging |
| `core` | 业务逻辑 | `base` |
| `app` | HTTP server / Main / 路由装配 | `core` |

简单项目可以只要 `base` + `app`。

## 领域类型用 opaque type / value class

避免"stringly typed"代码。常见反例：

```scala
// ❌ 所有 ID 都是 String，容易混用
def getUser(userId: String): Future[User]
def getOrg(orgId: String): Future[Org]
// 调用方可能写 getUser(orgId) 不会被类型系统拦住
```

### Scala 3: opaque type

```scala
opaque type UserId = String
object UserId {
  def apply(s: String): UserId = s
  extension (id: UserId) def value: String = id
}
opaque type OrgId = String
object OrgId { /* ... */ }

def getUser(userId: UserId): Future[User]
def getOrg(orgId: OrgId): Future[Org]
```

### Scala 2: value class

```scala
class UserId(val value: String) extends AnyVal
object UserId {
  def apply(s: String): UserId = new UserId(s)
}

class OrgId(val value: String) extends AnyVal
object OrgId { /* ... */ }

def getUser(userId: UserId): Future[User]
def getOrg(orgId: OrgId): Future[Org]
```

> **Note**: Value classes have minor boxing overhead in some edge cases (generics, arrays). opaque types are truly zero-overhead. For most use cases, both are acceptable.

**什么时候用**：ID、枚举值、约束数值（`PositiveInt`）、语义边界清晰的原始类型。

## 包层次示意

```
com.example
├── analysis                # app 模块
│   ├── Main.scala
│   ├── Apis.scala
│   ├── OpenApi.scala
│   ├── ExampleApi.scala
│   ├── TestApi.scala
│   ├── core                # core 模块
│   │   ├── ExampleQueryService.scala
│   │   ├── ExampleSqlGen.scala
│   │   └── domain
│   │       ├── dimensions
│   │       └── metrics
│   ├── query_module             # query-module 模块（注意 snake_case）
│   │   ├── QueryModuleScheduler.scala
│   │   ├── QueryService.scala
│   │   └── api
│   │       └── QueryModuleOpenApi.scala
│   ├── etl                 # etl 模块
│   └── etl_feature         # etl-feature 模块
├── base                    # base 模块
│   ├── BaseApi.scala
│   ├── SchedulerLeaderElection.scala
│   ├── config
│   │   └── LeaderElectionConfig.scala
│   ├── models
│   │   ├── SchedulerLeader.scala
│   │   └── enums
│   │       ├── LeaderRole.scala
│   │       └── QueryModuleStatus.scala
│   └── repositories
│       ├── DAO.scala
│       ├── SchedulerLeaderRepository.scala
│       └── components
│           └── SchedulerLeaderComponent.scala
└── service_clients         # service-clients 模块
    ├── external-service-a
    ├── external-service-b
    └── metadata
```
