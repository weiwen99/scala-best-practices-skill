# Ecosystem Library Conventions

以下写法从多个 Scala 3 生产项目中归纳。Scala 2 等价写法见各节末尾的 "Scala 2" 块。

## 1. 项目配置

### 配置格式

**所有新项目默认使用 TypeSafe Config（HOCON）格式**，配置文件路径：
- Mill：`app/resources/application.conf`
- sbt：`src/main/resources/application.conf`

```hocon
server {
  host = "0.0.0.0"
  port = 8080
}
```

### 配置加载方式

**优先使用 PureConfig**（类型安全，`derives ConfigReader` 自动映射 case class）。简单场景也可直接用 `com.typesafe.config.ConfigFactory` 手动解析。

关键原则：
- HTTP 服务 host / port / timeout 等参数**不可硬编码**，必须可配置
- 配置加载失败必须回退到合理默认值，不能 `loadOrThrow` 导致启动崩溃

## 2. PureConfig

### 基本写法

```scala
import pureconfig.ConfigReader

final case class LeaderElectionConfig(
  enabled: Boolean,
  lockName: String,
  heartbeatIntervalSeconds: Int,
  heartbeatTimeoutSeconds: Int,
  checkIntervalSeconds: Int
) derives ConfigReader
```

### 带默认值与可选字段

```scala
final case class QueryModuleConfig(
  defaultNamespace: String,
  defaultTaskAppName: String,
  owner: String,
  queue: Option[String],                                  // 真正可选，HOCON 可以不写
  minIdleSessionTasks: Int,
  resultRowLimit: Int = 10000,                            // 有默认值，HOCON 不写就用默认
  scheduler: QueryModuleConfig.SchedulerConfig
) derives ConfigReader
```

**要点**：
- PureConfig 0.17+ 已原生支持 Scala 3 case class 默认值，**不需要**额外写 `productHint` 或 `useDefaultsForMissingFields = true`
- 可选字段两种写法：`Option[T]`（"真正可选"，无默认）或 `T = default`（"有默认值"）
- 不要混用 `derives ConfigReader` 和手写 `given ConfigReader[Xxx]`，除非有字段需要自定义解码

**Scala 2**: `derives` 不存在。使用 `import pureconfig.generic.auto._` 自动派生，或手写 `implicit val reader: ConfigReader[Xxx] = ...`。

### 嵌套配置

```scala
final case class QueryModuleConfig(
  defaultNamespace: String,
  scheduler: QueryModuleConfig.SchedulerConfig
) derives ConfigReader

object QueryModuleConfig {
  final case class SchedulerConfig(
    enabled: Boolean,
    intervalSeconds: Int
  ) derives ConfigReader
}
```

嵌套 config 放在伴生 object 里，`derives ConfigReader` 各自独立。

### 加载配置

```scala
import pureconfig.*

val cfg = ConfigSource.default.loadOrThrow[QueryModuleConfig]
// 或
val cfg = ConfigSource.default.load[QueryModuleConfig]  // Either
```

## 3. Circe JSON

### 默认：auto derivation

```scala
import io.circe.generic.auto.*
import io.circe.syntax.*
import io.circe.parser.decode

final case class SubmitQueryRequest(sql: String, params: Map[String, String])

val json = request.asJson.noSpaces
val parsed = decode[SubmitQueryRequest](json)
```

**默认用 `io.circe.generic.auto.*`**，一行 import 管所有 case class。仅当有特殊需求（自定义字段名、判别器、默认值）才降级到 semi-auto 或 configured。

### 带配置的 derivation（discriminator + defaults）

```scala
import io.circe.derivation.{Configuration, ConfiguredCodec}

// 一次声明全局 given
given Configuration = Configuration.default.withDiscriminator("type").withDefaults

sealed trait Period derives ConfiguredCodec
final case class DatePeriod(startTime: LocalDate, endTime: LocalDate) extends Period
final case class LastPeriod(num: Int, todayIncluded: Boolean = true) extends Period

// 序列化结果：{"type":"DatePeriod","startTime":"...","endTime":"..."}
// 解析时字段缺失会用 case class 默认值填充
```

### Enum + Circe

```scala
import io.circe.derivation.ConfiguredEnumCodec
import com.example.base.json.JsonConfigurationWithDefaults.given

enum FilterOperator derives ConfiguredEnumCodec {
  case in, notIn, empty, notEmpty, isNull, isNotNull,
       `match`, `like`, `>`, `>=`, `<`, `<=`, `=`, `==`, `!=`
}
```

**要点**：
- enum 用 `derives ConfiguredEnumCodec`（而不是 auto），因为默认配置处理不了 backtick 名
- 必须 import 项目共享的 `JsonConfigurationWithDefaults.given`

**Scala 2**: 没有 `enum` 和 `derives`。用 `sealed trait` + `case object`，配合 `import io.circe.generic.auto._` 或手写 `implicit val encoder: Encoder[Xxx]` / `implicit val decoder: Decoder[Xxx]`。`given Configuration` 改为 `implicit val config: Configuration = ...`。

## 4. Tapir + Pekko HTTP

### BaseApi 模板

```scala
trait BaseApi(authConfig: AuthConfig)
    extends TapirJsonCirce
    with TapirBase
    with StrictLogging {

  protected type Endpoint = ServerEndpoint[Any, Future]
  protected lazy val auth: TapirAuth = TapirAuth(authConfig)

  // Scala 2: use implicit class instead of extension
  implicit class BodyWithCode[A, T](body: EndpointIO.Body[A, T]) {
    def withCode: EndpointIO[(T, Int)] =
      body.and(header[String](`X-Status`).map(_.toInt)(_.toString))
  }
}
```

### 定义 Endpoint

```scala
class QueryModuleOpenApi(queryService: QueryService)(using authConfig: AuthConfig)  // Scala 2: (implicit authConfig: AuthConfig)
    extends BaseApi(authConfig) {

  import io.circe.generic.auto.*
  import sttp.tapir.generic.auto.*
  import sttp.tapir.json.circe.*

  private val queryModuleTags = List("query-module")
  private val basePrefix = "v1" / "query-module"

  private val submitQueryEndpoint: Endpoint = auth.baseEndpoint
    .tags(queryModuleTags)
    .post
    .in(basePrefix / "query")
    .in(header[String]("X-User"))
    .in(jsonBody[SubmitQueryRequest])
    .out(jsonBody[ApiResponse[SubmitQueryResponse]].withCode)
    .serverLogic { implicit ctx =>
      { case (user, request) =>
        ctx.writeToMDC()
        logger.info(s"[QueryModule] submit: user = $user, query = ${request.query}")
        queryService.submitQuery(request).map(toApiResponse)
      }
    }

  val endpoints: List[Endpoint] = List(submitQueryEndpoint /* , ... */)
}
```

**要点**：
- 所有响应包 `ApiResponse[T]`（统一成功 / 错误结构）
- HTTP 状态码通过 `.withCode` 从 `X-Status` header 回传
- `serverLogic` 第一件事 `ctx.writeToMDC()`（把请求上下文写到 SLF4J MDC，日志链路用）
- 请求 / 响应 DTO 放在同包 `api/` 下
- `import io.circe.generic.auto.*` + `import sttp.tapir.generic.auto.*` 配对出现

### 错误建模

```scala
// ApiResponse 本身
enum ApiResponse[+A] derives ConfiguredCodec {
  case Success(data: A, code: Int = 0, msg: String = "ok")
  case Error(code: Int, msg: String)
}

// serverLogic 返回 Either[ApiResponse.Error, (ApiResponse.Success[A], Int)]
def submitQuery(req: SubmitQueryRequest): Future[Either[ApiResponse.Error, (ApiResponse[SubmitQueryResponse], Int)]] =
  queryService
    .submit(req)
    .map(r => Right((ApiResponse.Success(r), 200)))
    .recover { case e: DomainError => Left(ApiResponse.Error(e.code, e.msg)) }
```

### 路由装配（app/Apis.scala）

```scala
import sttp.tapir.server.pekkohttp.PekkoHttpServerInterpreter
import org.apache.pekko.http.scaladsl.server.Route

class Apis(
  openApi: OpenApi,
  querymoduleApi: QueryModuleOpenApi,
  exampleApi: ExampleApi,
  testApi: TestApi
)(using ec: ExecutionContext) {  // Scala 2: (implicit ec: ExecutionContext)

  val route: Route = PekkoHttpServerInterpreter().toRoute(
    openApi.endpoints ++
      querymoduleApi.endpoints ++
      exampleApi.endpoints ++
      testApi.endpoints
  )
}
```

## 5. Slick DB 六层（强推荐）

完整示例：`SchedulerLeader` 表。

### 1) DDL（`init-mysql/V1.30.*.sql`）

```sql
-- V1.30.1770111706421__DDL.query-module.sql
CREATE TABLE scheduler_leader (
  id            BIGINT      NOT NULL AUTO_INCREMENT COMMENT '主键',
  lock_name     VARCHAR(64) NOT NULL                COMMENT '锁名，一个业务一个',
  leader_id     VARCHAR(64) NOT NULL                COMMENT '当前 leader 实例 ID',
  info          TEXT        NOT NULL                COMMENT '基础信息 JSON',
  heartbeat_at  DATETIME    NOT NULL                COMMENT '最近心跳时间',
  created_at    DATETIME    NOT NULL                COMMENT '创建时间',
  updated_at    DATETIME    NOT NULL                COMMENT '更新时间',
  PRIMARY KEY (id),
  UNIQUE KEY uk_lock_name (lock_name)
) COMMENT '调度器 Leader 选举表';
```

**所有字段必须带 COMMENT**，主键 + 唯一键命名有规律（`pk_*` / `uk_*` / `idx_*`）。

### 2) Model（`base/models/SchedulerLeader.scala`）

```scala
package com.example.base.models

import java.time.LocalDateTime

final case class SchedulerLeader(
  id: Option[Long],       // 插入时为 None，查询回来有值
  lockName: String,
  leaderId: String,
  info: BaseInfo,
  heartbeatAt: LocalDateTime,
  createdAt: LocalDateTime,
  updatedAt: LocalDateTime
)
```

- `id: Option[Long]`（插入时为 `None`，依赖 DB 自增）
- DB 实体用 camelCase（与表的 snake_case 列名由 Slick 映射）
- 嵌套 JSON 字段（如 `info: BaseInfo`）用 custom column type 存 JSON

### 3) Enum（`base/models/enums/LeaderRole.scala`）

```scala
package com.example.base.models.enums

import io.circe.derivation.ConfiguredEnumCodec
import com.example.base.json.JsonConfigurationWithDefaults.given

enum LeaderRole derives ConfiguredEnumCodec {
  case Leader, Standby, Stopped
}
```

**Slick 列映射在 Component 里加**（见下）。

### 4) Component（`base/repositories/components/SchedulerLeaderComponent.scala`）

```scala
package com.example.base.repositories.components

import java.time.LocalDateTime
import slick.basic.DatabaseConfig
import slick.jdbc.JdbcProfile
import slick.lifted.ProvenShape

import com.example.base.models.{SchedulerLeader, BaseInfo}

trait SchedulerLeaderComponent extends BaseComponent {
  self: HasDatabaseConfig[JdbcProfile] =>
  import profile.api.*

  given baseInfoColumnType: BaseColumnType[BaseInfo] = jsonType[BaseInfo]    // Scala 2: implicit val baseInfoColumnType: BaseColumnType[BaseInfo] = jsonType[BaseInfo]

  protected class SchedulerLeaderTable(tag: Tag) extends Table[SchedulerLeader](tag, "scheduler_leader") {
    def id:           Rep[Long]          = column[Long]("id", O.PrimaryKey, O.AutoInc)
    def lockName:     Rep[String]        = column[String]("lock_name")
    def leaderId:     Rep[String]        = column[String]("leader_id")
    def info:         Rep[BaseInfo]      = column[BaseInfo]("info")
    def heartbeatAt:  Rep[LocalDateTime] = column[LocalDateTime]("heartbeat_at")(using localDateTimeType)
    def createdAt:    Rep[LocalDateTime] = column[LocalDateTime]("created_at")(using localDateTimeType)
    def updatedAt:    Rep[LocalDateTime] = column[LocalDateTime]("updated_at")(using localDateTimeType)

    override def * : ProvenShape[SchedulerLeader] =
      (id.?, lockName, leaderId, info, heartbeatAt, createdAt, updatedAt)
        .<>(SchedulerLeader.apply.tupled, SchedulerLeader.unapply)
  }

  protected lazy val schedulerLeaderQuery: TableQuery[SchedulerLeaderTable] = TableQuery[SchedulerLeaderTable]
}
```

**要点**：
- trait 而非 class；`self: HasDatabaseConfig[JdbcProfile] =>` 自类型绑定数据库配置
- `import profile.api.*` 拿 Slick DSL
- `LocalDateTime` 列显式 `using localDateTimeType`（Scala 2: `(implicit localDateTimeType)`）（`BaseComponent` 提供）
- JSON 列用 `jsonType[T]`（`BaseComponent` 提供）

### 5) Repository（`base/repositories/SchedulerLeaderRepository.scala`）

```scala
package com.example.base.repositories

import scala.concurrent.{ExecutionContext, Future}
import com.typesafe.scalalogging.StrictLogging
import slick.basic.DatabaseConfig
import slick.jdbc.JdbcProfile

import com.example.base.repositories.components.SchedulerLeaderComponent

class SchedulerLeaderRepository(
  protected val dbConfig: DatabaseConfig[JdbcProfile]
)(using ec: ExecutionContext)      // Scala 2: (implicit ec: ExecutionContext)
    extends HasDatabaseConfig[JdbcProfile]
    with SchedulerLeaderComponent
    with StrictLogging {

  import profile.api.*

  def tryInsert(lockName: String, leaderId: String, info: BaseInfo): Future[Boolean] = {
    val now = LocalDateTime.now()
    val action = sqlu"""
      INSERT IGNORE INTO scheduler_leader (lock_name, leader_id, info, heartbeat_at, created_at, updated_at)
      VALUES ($lockName, $leaderId, ${info.toJson}, $now, $now, $now)
    """
    db.run(action).map(_ > 0)
  }

  // 更多 CAS 操作：renewHeartbeat / tryTakeover / releaseLeadership ...
}

object SchedulerLeaderRepository {
  def apply(dbConfig: DatabaseConfig[JdbcProfile])(using ec: ExecutionContext): SchedulerLeaderRepository =  // Scala 2: (implicit ec: ExecutionContext)
    new SchedulerLeaderRepository(dbConfig)
}
```

**要点**：
- `protected val dbConfig` 构造参数注入
- 继承 `HasDatabaseConfig[JdbcProfile]` + 对应 `XxxComponent`
- `db.run(action): Future[T]`
- MySQL 特有函数（`CURRENT_TIMESTAMP` / `DATE_SUB` / `INSERT IGNORE`）用 Plain SQL (`sql` / `sqlu` 插值器)
- 伴生 object + apply 工厂

### 6) DAO（`base/repositories/DAO.scala`）

```scala
final case class DAO(
  queryModuleRepository: QueryModuleRepository,
  schedulerLeaderRepository: SchedulerLeaderRepository,
  sessionRepository: QuerySessionRepository
  // 新增 Repository 在这里追加字段
)

object DAO {
  def apply(dbConfig: DatabaseConfig[JdbcProfile])(using ec: ExecutionContext): DAO =  // Scala 2: (implicit ec: ExecutionContext)
    DAO(
      queryModuleRepository = QueryModuleRepository(dbConfig),
      schedulerLeaderRepository = SchedulerLeaderRepository(dbConfig),
      sessionRepository = QuerySessionRepository(dbConfig)
    )
}
```

**DAO 是仓储聚合**，让上层服务通过单一入口注入所有 Repository，避免构造参数爆炸。

## 6. 异步 / 并发

### Future 优先

- 所有返回异步结果的方法用 `Future[T]` 或 `Future[Either[Err, T]]`
- **严禁** `Await.result`（测试除外）
- `ExecutionContext` 通过 `using ec: ExecutionContext`（Scala 2: `implicit ec: ExecutionContext`）传递

### Pekko Streams 做定时 / 长期任务

```scala
import org.apache.pekko.stream.scaladsl.{RestartSource, Source}
import org.apache.pekko.stream.RestartSettings

val scheduledFlow =
  RestartSource.withBackoff(
    RestartSettings(minBackoff = 1.second, maxBackoff = 30.seconds, randomFactor = 0.2)
  ) { () =>
    Source
      .tick(initialDelay = 0.seconds, interval = 10.seconds, tick = ())
      .mapAsync(parallelism = 1)(_ => doPeriodicWork())
  }
```

**为什么 Pekko Streams 而不是 `Timer` / `ScheduledExecutor`**：
- 内置错误重试（`RestartSource`）
- `KillSwitch` 支持安全停止（见 `SchedulerLeaderElection`）
- 背压、并行度控制原生支持
- 与 Pekko HTTP / actor 同一 runtime

## 7. 日志

```scala
import com.typesafe.scalalogging.StrictLogging

class QueryService(...) extends StrictLogging {
  def submitQuery(req: SubmitQueryRequest): Future[SubmitQueryResponse] = {
    logger.info(s"[QueryModule] submit: user = ${req.user}, namespace = ${req.namespace}")
    // ...
  }
}
```

**规则**：
- 一律 `StrictLogging` trait，不手写 `LoggerFactory.getLogger(...)`
- `key = value` 格式，**`=` 两边必须有空格**：`s"user = $user"` 而不是 `s"user=$user"`
- 模块前缀用方括号：`[QueryModule]` / `[LeaderElection]` / `[Proxy]`
- 异常打 `logger.error("msg", ex)`，不要 `logger.error(ex.getMessage)`（丢了 stack trace）
- 不要在纯函数里打业务日志（日志是副作用）

### logback.xml：Console + 文件落盘

日志配置模板详见 `${SKILL_DIR}/references/logback-template.xml`，核心要点：

- **Console appender**：本地开发默认彩色输出，设置环境变量 `CONSOLE_LOG_NO_COLOR=1` 切换为无色（生产环境用）
- **Rolling file appender**：写入 `logs/app.log`，按小时滚动（`logs/app.yyyy-MM-dd-HH.log`），保留最近 60 个文件
- **日志级别**通过环境变量 `LOGBACK_LEVEL` 控制（默认 `INFO`）
- `<if>` 条件语法依赖 `janino`，需在 `build.mill` / `build.sbt` 中添加该依赖

## 8. 测试（MUnit）

### 命名

- 测试文件命名：`XxxSpec.scala`（不是 `Test` 或 `Suite`）
- 被测 `QueryService` → 测试 `QueryServiceSpec`
- 一律 `extends munit.FunSuite`

### 模板

```scala
package com.example.base.json

import com.typesafe.scalalogging.StrictLogging
import io.circe.derivation.Configuration
import io.circe.parser.decode
import io.circe.syntax.*

class ExampleJsonSpec extends munit.FunSuite with StrictLogging {

  import ExampleJsonSpec.*

  given Configuration = Configuration.default.withDiscriminator("type").withDefaults   // Scala 2: implicit val config: Configuration = ...

  test("WithDiscriminators") {
    val datePeriod = DatePeriod(LocalDate.of(2023, 1, 1), LocalDate.of(2023, 1, 31))
    assertEquals(
      (datePeriod: ExamplePeriod).asJson.noSpacesSortKeys,
      """{"endTime":"2023-01-31","startTime":"2023-01-01","type":"DatePeriod"}"""
    )
  }
}

object ExampleJsonSpec {
  sealed trait ExamplePeriod derives ConfiguredCodec    // Scala 2: remove "derives ConfiguredCodec", use import io.circe.generic.auto._
  final case class DatePeriod(startTime: LocalDate, endTime: LocalDate) extends ExamplePeriod
  // ...
}
```

### 异步测试

需要 `Future` 返回值时可以：

```scala
// 选项 A：Await（仅测试允许，主代码严禁）
test("async") {
  val result = Await.result(service.call(), 5.seconds)
  assertEquals(result, expected)
}

// 选项 B：munit.CatsEffectSuite（如果引入 cats-effect）
class AsyncSpec extends munit.CatsEffectSuite {
  test("async") {
    service.call().map(assertEquals(_, expected))
  }
}
```

推荐使用选项 A。是否引入 cats-effect 是另一回事，暂不强制。

### 测试覆盖重点

- 业务逻辑（`core` 模块）：必须有测试
- Repository：可选（接触真实 DB 的测试另走集成测试）
- HTTP 路由：建议用 Tapir test 接口测，不要启整个 Pekko HTTP
- 纯函数尤其要测（反正没副作用，测起来最简单）

## 9. OpenAPI / Swagger 文档自动生成

**凡是使用 Tapir 的项目，必须自动生成 OpenAPI 文档并挂载 Swagger UI。** 这不仅仅是"顺手加一个页面"——端点定义和文档同源，杜绝"代码改了文档没改"的 drift。

### 依赖

```scala
// build.mill 或 build.sbt
mvn"com.softwaremill.sttp.tapir::tapir-swagger-ui-bundle:${Versions.tapirV}"
```

`tapir-swagger-ui-bundle` 自带 Swagger UI 3.x 的静态资源（HTML/CSS/JS），无需额外引入。

### 收集所有 Endpoint 为 AnyEndpoint 列表

模板：

```scala
// 每个 API 类暴露 endpoints: List[AnyEndpoint]
class SessionApi(...) extends BaseApi(...):
  private val createSession: Endpoint[..., ..., ..., ..., Any] = ...
  private val listSessions: Endpoint[..., ..., ..., ..., Any] = ...

  val endpoints: List[AnyEndpoint] = List(createSession, listSessions, /* ... */)
```

然后在装配层汇拢：

```scala
class Apis(sessionApi: SessionApi, adminApi: AdminApi, healthApi: HealthApi):
  val allEndpoints: List[AnyEndpoint] =
    sessionApi.endpoints ++ adminApi.endpoints ++ healthApi.endpoints
```

### Opaque / 复杂类型的 Schema 声明

Tapir 自动为 case class 派生 `Schema`（`import sttp.tapir.generic.auto.*`）。但以下类型需要**显式**提供 `given Schema[T]`，否则 OpenAPI 文档里会变成空 object：

```scala
// 复杂 JSON / sum-type → Schema.string（文档里标为 string，描述里说明结构）
given Schema[Wire.ContentBlock]  = Schema.string[Wire.ContentBlock]       // Scala 2: implicit val
given Schema[Wire.StoredSession] = Schema.string[Wire.StoredSession]      // Scala 2: implicit val

// JDK 类型 → Schema.string（tapir 不自带 java.time.Instant 的 Schema）
given Schema[java.time.Instant] = Schema.string                           // Scala 2: implicit val

// 简单 record 可派生
given Schema[Wire.Message] = Schema.derived                               // Scala 2: implicit val
```

**规则**：
- `given Schema[T]`（Scala 2: `implicit val schema: Schema[T]`）放在 endpoint 定义文件的顶层
- 复杂类型走 `Schema.string`（文档里标注实际 JSON 结构），简单 record 走 `Schema.derived`
- 防止 OpenAPI spec 出现空 `{}` → Swagger UI 上"Example Value"一片空白

### ⚠️ 坑：Map/List 等泛型容器自动生成 ugly 名称

**现象**：当 model 中使用了 `Map[String, String]`、`Map[String, List[String]]`、`List[Map[String, String]]` 等泛型容器字段时，Tapir 的 Magnolia 自动推导会生成 `Map_String`、`Map_List_String`、`List_Map_String` 这样的拼接名作为 OpenAPI components/schemas 的 key：

```yaml
# ❌ 自动生成 — ugly
components:
  schemas:
    Map_List_String:
      title: Map_List_String
      type: object
      additionalProperties:
        type: array
        items:
          type: string
    Map_String:
      title: Map_String
      type: object
      additionalProperties:
        type: string
```

虽然底层 OpenAPI 类型（`object` + `additionalProperties`）是正确的，但命名非常难看，且会污染 OpenAPI spec 的全局命名空间。

**修复**：对这些泛型容器类型显式提供 `given Schema[T]`，使用 `Schema.string` 并附上清晰的 `description` 和 `format`：

```scala
// models/PingSchemas.scala
object PingSchemas:

  given Schema[Map[String, String]] =                                       // Scala 2: implicit val headersMapSchema: Schema[Map[String, String]] =
    Schema.string[Map[String, String]]
      .description("HTTP headers as key-value pairs (e.g. {\"Content-Type\": \"application/json\"})")
      .format("map[string, string]")

  given Schema[Map[String, List[String]]] =                                 // Scala 2: implicit val multiValuedQuerySchema
    Schema.string[Map[String, List[String]]]
      .description("Multi-valued query parameters (e.g. {\"filter\": [\"active\", \"premium\"]})")
      .format("map[string, list[string]]")

  given Schema[List[Map[String, String]]] =                                 // Scala 2: implicit val cookiesSchema
    Schema.string[List[Map[String, String]]]
      .description("Cookies as list of name-content pairs")
      .format("list[map[string, string]]")

end PingSchemas
```

然后在 endpoint 定义文件中导入：

```scala
import com.example.ping.models.PingSchemas.given   // Scala 2: import com.example.ping.models.PingSchemas._
```

**效果**：

```yaml
# ✅ 显式 Schema — 干净
headers:
  type: string
  format: map[string, string]
  description: >-
    HTTP headers as key-value pairs (e.g. {"Content-Type": "application/json"})

queryParams:
  type: string
  description: >-
    Multi-valued query parameters (e.g. {"filter": ["active", "premium"]})
```

**要点**：
- 所有 `given`（Scala 2: `implicit val`）必须**显式命名**（如 `headersMapSchema`、`queryParamsMapSchema`），否则 Scala 会因类型擦除产生 anonymous given 冲突
- `Schema.string[T]` 把类型在文档里标为 `string`，通过 `description` 和 `format` 说明真实结构；这是 SKILL 推荐的"复杂类型走 string"策略
- 把这些 given（Scala 2: implicit val）集中放在一个 `XxxSchemas` object 中，统一 import
- 触发场景：`Map[K, V]`、`List[Map[K, V]]`、`Option[Map[K, V]]` 等**任何嵌套泛型组合**都会产生 ugly 名称；只要是 model 中的字段类型，都要检查

### 挂载 Swagger UI 路由

**Pekko HTTP 后端：**

```scala
import sttp.tapir.swagger.bundle.SwaggerInterpreter
import sttp.tapir.server.pekkohttp.PekkoHttpServerInterpreter

class Apis(...):
  val allEndpoints: List[AnyEndpoint] = ...

  // Swagger UI 挂载到 /docs
  private val swaggerEndpoints = SwaggerInterpreter().fromEndpoints(
    allEndpoints,
    title = "My Service API",
    version = "1.0"
  )
  // swaggerEndpoints: List[ServerEndpoint[Any, Future]] —— 含 /docs 与 /docs/openapi.json

  val route: Route =
    PekkoHttpServerInterpreter().toRoute(
      allEndpoints ++ swaggerEndpoints
    )
```

**http4s 后端：**

```scala
import sttp.tapir.swagger.bundle.SwaggerInterpreter
import sttp.tapir.server.http4s.Http4sServerInterpreter
import cats.effect.IO

class Apis[F[_]: Async](...):
  val allEndpoints: List[AnyEndpoint] = ...

  private val swaggerEndpoints = SwaggerInterpreter().fromEndpoints(
    allEndpoints,
    title = "My Service API",
    version = "1.0"
  )

  val routes: HttpRoutes[F] =
    Http4sServerInterpreter[F]().toRoutes(
      allEndpoints ++ swaggerEndpoints
    )
```

### 启动时打印文档地址

启动日志里打一条 Swagger UI 的 URL，方便开发者：

```scala
val host = cfg.web.bindHost
val port = cfg.port
logger.info(s"Swagger UI: http://$host:$port/docs")
```

### 文档补全清单

- [ ] `tapir-swagger-ui-bundle` 加到依赖
- [ ] 所有 API 类有 `val endpoints: List[AnyEndpoint]`
- [ ] 装配层汇拢 `allEndpoints` 并传给 `SwaggerInterpreter`
- [ ] opaque / 非标准类型有 `given Schema[T]`（Scala 2: `implicit val schema: Schema[T]`）
- [ ] ⚠️ 检查 model 中是否有 `Map[K,V]` / `List[Map[K,V]]` 等泛型容器字段——它们会生成 `Map_String` 等 ugly 名，必须显式 given Schema（Scala 2: implicit val Schema）（详见上方"坑"）
- [ ] Swagger UI 挂到 `/docs`
- [ ] 启动日志打印 Swagger UI URL
- [ ] 废弃端点链 `.deprecated()`（Swagger UI 会标 `[deprecated]` 并添加 `Deprecation: true` 头）

### 参考

关键实践：显式声明 `given Schema[T]`（Scala 2: `implicit val`）保证 OpenAPI 质量；穷举 `def all: List[AnyEndpoint]` 保证文档收敛。在此基础之上补挂载 Swagger UI 路由即可。
