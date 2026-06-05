---
name: scala-hex-arch
description: Use when user asks to check architecture, verify hexagonal compliance, review architecture, or mentions architecture review/hexagonal check. Also use after adding new modules or code to Scala hexagonal project. Do NOT use for build/docker tasks.
---

# Check Scala hexagonal architecture compliance

Review code to ensure it follows hexagonal architecture principles.

## Usage
- User says: "check architecture" or "verify hexagonal" or "review architecture"
- Automatically after adding new modules or code

## Architecture Layers

### Core Layer (modules/core)
**Location**: `modules/core/src/scala/$package$/core/`

**Checklist**:
- [ ] No side effects: No IO, Task, F[_] types
- [ ] No external dependencies: Only Scala stdlib and cats-core
- [ ] ADT pattern: Use case class and sealed trait
- [ ] Contains: domain entities, value objects, domain errors, ports/algebras

**Correct patterns**:
```scala
// domain/User.scala
case class User(id: Long, name: String, email: String)

// domain/UserError.scala
sealed trait UserError extends Product with Serializable
object UserError:
  case object UserNotFound extends UserError
  case object UserAlreadyExists extends UserError

// algebra/UserAlgebra.scala
trait UserRepository[F[_]]:
  def findById(id: Long): F[Option[User]]
  def save(user: User): F[User]
```

**Violations to flag**:
- IO, Task, or F[_] in domain objects
- Database imports (doobie, slick, etc.)
- HTTP imports (http4s, akka-http, etc.)
- JSON imports (circe, play-json, etc.)

### Logic Layer (modules/logic)
**Location**: `modules/logic/src/scala/$package$/logic/service/`

**Checklist**:
- [ ] Tagless Final signature: Methods return F[Result]
- [ ] Dependency inversion: All external capabilities injected via constructor
- [ ] Combinator pattern: Use for-yield for orchestration
- [ ] Depends ONLY on core module

**Correct patterns**:
```scala
class UserService[F[_]: Monad](repo: UserRepository[F]):
  def create(name: String, email: String): F[Either[UserError, User]] =
    for
      existing <- repo.findByEmail(email)
      result <- existing match
        case Some(_) => UserError.UserAlreadyExists.asLeft[User].pure[F]
        case None => repo.save(User(0, name, email)).map(_.asRight[UserError])
    yield result
```

### Infra Layer (modules/infra)
**Location**: `modules/infra/src/scala/$package$/infra/persistence/`

**Checklist**:
- [ ] Implements outbound ports (Repository, Gateway)
- [ ] Resource safety: Use cats.effect.Resource for connections
- [ ] Exception translation: Convert low-level exceptions to domain errors
- [ ] Depends ONLY on core module

### API Layer (modules/api)
**Location**: `modules/api/src/scala/$package$/api/http/`

**Checklist**:
- [ ] Implements inbound ports (HTTP Routes)
- [ ] Exception translation: Convert domain errors to HTTP status codes
- [ ] DTO isolation: Request/Response objects separate from domain objects
- [ ] Depends on logic module

### App Layer (app)
**Location**: `app/src/scala/$package$/MainApp.scala`

**Checklist**:
- [ ] Dependency injection: Assemble all dependencies
- [ ] Resource management: Use Resource for lifecycle
- [ ] Configuration externalized: Sensitive config via environment variables
- [ ] Depends on api and infra modules

## Scala 3 Best Practices

### Enum (代替 sealed trait + case object)
优先使用 Scala 3 的 `enum` 定义代数数据类型：

```scala
// ✅ 推荐: 使用 enum
enum UserError:
  case UserNotFound(id: Long)
  case UserAlreadyExists(email: String)
  case InvalidInput(message: String)

enum OrderStatus:
  case Pending, Confirmed, Shipped, Delivered, Cancelled

// ❌ 避免: 旧的 sealed trait 模式 (除非需要更复杂的继承结构)
sealed trait UserError extends Product with Serializable
object UserError:
  case object UserNotFound extends UserError
```

### Extension Methods (扩展方法)
使用 `extension` 为现有类型添加方法：

```scala
// ✅ 推荐: 使用 extension
extension (user: User)
  def fullName: String = s"${user.firstName} ${user.lastName}"
  def isAdult: Boolean = user.age >= 18

extension [A](either: Either[UserError, A])
  def mapError[B](f: UserError => B): Either[B, A] =
    either.left.map(f)

// 使用示例
val user: User = ???
user.fullName  // 直接调用
```

### Given/Using (依赖注入)
使用 `given` 和 `using` 进行类型类定义和依赖注入：

```scala
// ✅ 推荐: 定义类型类实例
given Monad[IO] = cats.effect.IOInstances.ioMonad

given Encoder[User] with
  def encode(user: User): Json = ???

given Decoder[User] with
  def decode(json: Json): Result[User] = ???

// ✅ 推荐: 使用 using 进行依赖注入
class UserService[F[_]: Monad](using repo: UserRepository[F], 
                                    logger: Logger[F]):
  def findById(id: Long): F[Option[User]] = 
    repo.findById(id)

// ✅ 推荐: 使用 using 进行上下文参数
def validate[A](value: A)(using validator: Validator[A]): Either[Error, A] =
  validator.validate(value)
```

### Other Scala 3 Features

**Intersection Types**:
```scala
type UserWithAudit = User & Auditable
```

**Union Types**:
```scala
type Result = Success | Warning | Error
```

**Opaque Type Aliases** (类型安全的新类型):
```scala
object types:
  opaque type UserId = Long
  object UserId:
    def apply(value: Long): UserId = value
    extension (id: UserId) def value: Long = id
```

## Cats Library Best Practices

### EitherT 最佳实践

**核心原则**：永远不要在逻辑中间拆开 EitherT。把它当成一个整体进行 `flatMap` 和 `map`，只在最后调用 `.value`。

```scala
// ✅ 正确：for 推导保持 EitherT 完整性，只调用一次 .value
def createUser(name: String, email: String): F[Either[UserError, User]] =
  (for
    _    <- EitherT(validateEmail(email))
    opt  <- EitherT(repo.findByEmail(email)).leftMap(_ => UserError.DatabaseError)
    _    <- EitherT.cond[F](opt.isEmpty, (), UserError.UserAlreadyExists(email))
    user <- EitherT.right[UserError](repo.save(User(0, name, email)))
  yield user).value

// ✅ 正确：链式调用 .leftMap / .map
client.listTools()
  .leftMap(err => McpApiError.McpErr(err))
  .map(tools => tools.map(t => McpToolResponse(...)))
  .value

// ✅ 正确：EitherT.fromOption 替代 match
for
  tools <- client.listTools().leftMap(err => McpApiError.McpErr(err))
  tool  <- EitherT.fromOption(
    tools.find(_.name == name),
    McpApiError.McpErr(McpError.ToolNotFoundError(name))
  )
yield response.McpToolResponse(...)
```

```scala
// ❌ 错误：在逻辑中间拆开 EitherT，需要手动包装 Right/Left
def createUser(name: String, email: String): F[Either[UserError, User]] =
  validateEmail(email).value.flatMap {
    case Right(_) =>
      repo.findByEmail(email).value.flatMap {
        case Right(None) => repo.save(...).map(Right(_))
        case Right(Some) => (Left(UserError.UserAlreadyExists)).pure[F]
        case Left(err) => (Left(err)).pure[F]
      }
    case Left(err) => (Left(err)).pure[F]
  }
```

**快速参考**：

| 场景 | 推荐做法 |
|------|----------|
| 组合多个 Effect | for 推导 + `EitherT.liftF` |
| 错误转换 | `.leftMap(err => NewError(err))` |
| Option 转 EitherT | `EitherT.fromOption(opt, error)` |
| 成功值转换 | `.map(fn)` |
| 嵌套 Either | `.flatMap(fn)` |
| 返回给框架 | 最后调用 `.value` |

**例外**：当返回类型必须是 `F[A]` 而非 `EitherT` 时（如 traverse 中），可使用 `.value.map`。

### OptionT (嵌套 Option 处理)
```scala
import cats.data.OptionT

// ✅ 推荐: 使用 OptionT 处理 F[Option[A]]
def findActiveUser(id: Long): F[Option[User]] =
  (for
    user <- OptionT(repo.findById(id))
    _ <- OptionT.filterF(checkActive(user).pure[F])(identity)
  yield user).value
```

### Kleisli (Reader 模式)
```scala
import cats.data.Kleisli

// 用于依赖注入和组合
type UserRepoReader[A] = Kleisli[Option, UserRepository, A]

val findUser: UserRepoReader[User] = Kleisli(_.findById(1L).flatten)
val getUserName: UserRepoReader[String] = findUser.map(_.name)

// 组合多个操作
val processUser: UserRepoReader[String] = Kleisli { repo =>
  repo.findById(1L)
    .map(_.name)
}
```

### Validated (累积错误)
```scala
import cats.data.ValidatedNec
import cats.data.NonEmptyChain

type ValidationResult[A] = ValidatedNec[ValidationError, A]

def validateName(name: String): ValidationResult[String] =
  if (name.nonEmpty) name.validNec else ValidationError.EmptyName.invalidNec

def validateEmail(email: String): ValidationResult[String] =
  if (email.contains("@")) email.validNec else ValidationError.InvalidEmail.invalidNec

// ✅ 累积所有错误
def validateUser(name: String, email: String): ValidationResult[User] =
  (validateName(name), validateEmail(email))
    .mapN((n, e) => User(0, n, e))
```

### Resource (资源管理)
```scala
import cats.effect.Resource
import cats.effect.IO

def databaseConnection(url: String): Resource[IO, Connection] =
  Resource.make(IO(connect(url)))(conn => IO(conn.close()))

// 使用
def withDatabase[A](url: String)(f: Connection => IO[A]): IO[A] =
  databaseConnection(url).use(f)
```

### Ref (并发安全状态)
```scala
import cats.effect.concurrent.Ref

class UserCache[F[_]: Sync]:
  private val cache: Ref[F, Map[Long, User]] = Ref.of[F, Map[Long, User]](Map.empty)
  
  def get(id: Long): F[Option[User]] =
    cache.map(_.get(id))
    
  def put(user: User): F[Unit] =
    cache.update(_ + (user.id -> user))
```

## Output Format

When reviewing, provide:
1. Overall compliance status (PASS/FAIL)
2. Issues found by layer
3. Specific file:line references for violations
4. Suggested fixes