---
name: scala-hex-add
description: Use when user asks to add a new entity, domain, service, repository, or component to a Scala hexagonal project, or mentions add entity/create new domain/new service. Do NOT use for creating entirely new projects.
---

# Add new component to Scala hexagonal project

Guide for adding new domain components following hexagonal architecture.

## Usage
- User says: "add a new entity" or "create new domain" or "add Product entity"

## Steps

### 1. Add Domain Entity (Core Layer)
**Location**: `modules/core/src/scala/$package$/core/domain/`

```scala
// $package$/core/domain/Product.scala
case class Product(id: Long, name: String, price: BigDecimal, stock: Int)

sealed trait ProductError extends Product with Serializable
object ProductError:
  case object ProductNotFound extends ProductError
  case object InsufficientStock extends ProductError
  case object InvalidPrice extends ProductError
```

### 2. Add Port/Algebra (Core Layer)
**Location**: `modules/core/src/scala/$package$/core/algebra/`

```scala
// $package$/core/algebra/ProductRepository.scala
trait ProductRepository[F[_]]:
  def findById(id: Long): F[Option[Product]]
  def findByName(name: String): F[Option[Product]]
  def save(product: Product): F[Product]
  def delete(id: Long): F[Unit]
  def findAll: F[List[Product]]
```

### 3. Add Service (Logic Layer)
**Location**: `modules/logic/src/scala/$package$/logic/service/`

```scala
// $package$/logic/service/ProductService.scala
class ProductService[F[_]: Monad](repo: ProductRepository[F]):

  def create(name: String, price: BigDecimal, stock: Int): F[Either[ProductError, Product]] =
    for
      existing <- repo.findByName(name)
      result <- existing match
        case Some(_) => ProductError.ProductAlreadyExists.asLeft[Product].pure[F]
        case None =>
          if price < 0 then ProductError.InvalidPrice.asLeft[Product].pure[F]
          else repo.save(Product(0, name, price, stock)).map(_.asRight[ProductError])
      yield result

  def findById(id: Long): F[Either[ProductError, Product]] =
    repo.findById(id).map {
      case Some(product) => product.asRight[ProductError]
      case None => ProductError.ProductNotFound.asLeft[Product]
    }

object ProductService:
  def apply[F[_]: Monad](repo: ProductRepository[F]): ProductService[F] =
    new ProductService[F](repo)
```

### 4. Add Repository Implementation (Infra Layer)
**Location**: `modules/infra/src/scala/$package$/infra/persistence/`

```scala
// $package$/infra/persistence/ProductRepositoryImpl.scala
class InMemoryProductRepository extends ProductRepository[IO]:
  private var products: Map[Long, Product] = Map.empty
  private var nextId: Long = 1L

  override def findById(id: Long): IO[Option[Product]] =
    IO.pure(products.get(id))

  override def save(product: Product): IO[Product] = IO.delay {
    val toSave = if product.id == 0 then
      val saved = product.copy(id = nextId)
      nextId += 1
      saved
    else product
    products = products + (toSave.id -> toSave)
    toSave
  }

object InMemoryProductRepository:
  def apply(): IO[InMemoryProductRepository] = IO.pure(new InMemoryProductRepository)
```

### 5. Add HTTP Routes (API Layer)
**Location**: `modules/api/src/scala/$package$/api/http/`

```scala
// $package$/api/http/ProductRoutes.scala
object ProductRoutes:
  def apply(service: ProductService[IO]): HttpRoutes[IO] =
    val dsl = new Http4sDsl[IO] {}
    import dsl.*

    def errorResponse(error: ProductError): IO[Response[IO]] = error match
      case ProductError.ProductNotFound => NotFound()
      case ProductError.InsufficientStock => Conflict("Insufficient stock")
      case ProductError.InvalidPrice => BadRequest("Invalid price")

    HttpRoutes.of[IO]:
      case GET -> Root / "products" =>
        service.findAll.flatMap(products => Ok(products.asJson))

      case GET -> Root / "products" / LongVar(id) =>
        service.findById(id).flatMap {
          case Right(product) => Ok(product.asJson)
          case Left(err) => errorResponse(err)
        }

      case req @ POST -> Root / "products" =>
        for
          body <- req.as[CreateProductRequest]
          result <- service.create(body.name, body.price, body.stock)
          response <- result match
            case Right(product) => Created(product.asJson)
            case Left(err) => errorResponse(err)
        yield response
```

### 6. Wire Dependencies (App Layer)
**Location**: `app/src/scala/$package$/MainApp.scala`

Add to the app:
```scala
// In MainApp.scala
for
  productRepo <- Resource.eval(InMemoryProductRepository())
  productService = ProductService[IO](productRepo)
  // Add to router
  httpApp = Router(
    "/api" -> UserRoutes(userService),
    "/api" -> ProductRoutes(productService)  // Add this
  ).orNotFound
yield server
```

### 7. Update bleep.yaml (if needed)
Add dependencies to `bleep.yaml` if new libraries are needed.

## Architecture Checklist
- [ ] Core: No side effects, no external deps
- [ ] Logic: Tagless Final, dependency injection
- [ ] Infra: Implements ports, resource safety
- [ ] API: HTTP routes, error mapping
- [ ] App: Wire all dependencies