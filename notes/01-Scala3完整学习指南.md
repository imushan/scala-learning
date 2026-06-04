# Scala 3 完整学习指南

> 基于 Scala 3 官方参考文档整理  
> 适用于 Scala 2 开发者迁移及 Scala 3 新手学习

---

## 目录

1. [新类型系统](#1-新类型系统)
2. [枚举与代数数据类型](#2-枚举与代数数据类型)
3. [上下文抽象](#3-上下文抽象)
4. [元编程](#4-元编程)
5. [其他新特性](#5-其他新特性)
6. [变更的特性](#6-变更的特性)
7. [最佳实践总结](#7-最佳实践总结)

---

## 1. 新类型系统

### 1.1 交集类型 (Intersection Types)

#### 核心概念

交集类型 `A & B` 表示同时满足类型 `A` 和 `B` 的值。它是**复合类型**的一种形式。

#### 语法与语义

```scala
trait Resettable:
  def reset(): Unit

trait Growable[T]:
  def add(t: T): Unit

// 交集类型：参数 x 必须同时拥有 reset 和 add 方法
def f(x: Resettable & Growable[String]) =
  x.reset()
  x.add("first")
```

**关键特性：**
- `&` 运算符是**可交换的**：`A & B` 等于 `B & A`
- 交集类型的成员包含两个类型的所有成员
- 重叠成员的类型是两者类型的交集

#### 重叠成员处理

```scala
trait A:
  def children: List[A]

trait B:
  def children: List[B]

// children 方法的类型为 List[A & B]
val x: A & B = new C
val ys: List[A & B] = x.children

class C extends A, B:
  def children: List[A & B] = ???
```

#### 与 Scala 2 对比

| Scala 2 | Scala 3 |
|---------|---------|
| `A with B` | `A & B` |
| 语法较冗长 | 更简洁、数学化 |

#### 使用场景

- 需要**多个 trait 组合**作为参数类型
- 表达**复合能力要求**（如同时可重置可增长）
- 替代 Scala 2 的 `with` 关键字

---

### 1.2 联合类型 (Union Types)

#### 核心概念

联合类型 `A | B` 表示值为类型 `A` 或类型 `B` 之一。它是交集类型的**对偶**。

#### 基本语法

```scala
trait ID
case class UserName(name: String) extends ID
case class Password(hash: Hash) extends ID

def help(id: UserName | Password) =
  val user = id match
    case UserName(name) => lookupName(name)
    case Password(hash) => lookupPassword(hash)
  // ...
```

#### 类型推断行为

```scala
val password = Password(123)
val name = UserName("Eve")

// 默认：推断为共同超类型 ID
if true then name else password  // 类型: ID

// 显式标注：保留联合类型
val either: Password | UserName = if true then name else password
```

#### Transparent Trait 的影响

```scala
transparent trait ID  // 透明 trait 阻止类型拓宽

case class UserName(name: String)
case class Password(hash: Hash)

// 现在自动推断为联合类型
if true then UserName("Eve") else Password(123)
// 类型: UserName | Password
```

#### 使用场景

- **参数多态**：接受多种相关类型
- **结果类型**：返回可能的不同类型
- **模式匹配**：配合 match 表达式处理分支

---

### 1.3 类型 Lambda (Type Lambdas)

#### 核心概念

类型 Lambda 允许**直接定义高阶类型构造器**，无需单独的类型定义。

#### 语法

```scala
[X, Y] =>> Map[Y, X]
```

这定义了一个二元类型构造器，将参数 `X` 和 `Y` 映射到 `Map[Y, X]`。

#### 使用示例

```scala
// 传统方式：需要单独定义
type MyMap[X, Y] = Map[Y, X]

// 类型 Lambda：直接内联定义
type Flip = [X, Y] =>> Map[Y, X]

// 应用
type FlippedMap = Flip[Int, String]  // 等价于 Map[String, Int]
```

#### 限制

- 类型参数可以有边界
- 类型参数**不能携带变体注解**（`+` 或 `-`）

#### 与多态函数类型的区别

| 类型 Lambda | 多态函数类型 |
|------------|-------------|
| **类型层面**的操作 | **值层面**的操作 |
| `[A] =>> List[A]` | `[A] => List[A] => List[A]` |
| 用于类型表达式 | 用于函数值 |

---

### 1.4 匹配类型 (Match Types)

#### 核心概念

匹配类型根据 scrutinee 的类型**在编译时缩减**为其右侧类型之一。

#### 基本语法

```scala
type Elem[X] = X match
  case String => Char
  case Array[t] => t
  case Iterable[t] => t
```

#### 缩减结果

```scala
Elem[String]        =:= Char
Elem[Array[Int]]    =:= Int
Elem[List[Float]]   =:= Float
Elem[Nil.type]      =:= Nothing
```

#### 递归匹配类型

```scala
type LeafElem[X] = X match
  case String => Char
  case Array[t] => LeafElem[t]
  case Iterable[t] => LeafElem[t]
  case AnyVal => X

// 使用
LeafElem[Array[List[Int]]] =:= Int
```

#### 带上界的递归定义

```scala
type Concat[Xs <: Tuple, +Ys <: Tuple] <: Tuple = Xs match
  case EmptyTuple => Ys
  case x *: xs => x *: Concat[xs, Ys]

// 使用
Concat[("a", "b"), ("c", "d")] =:= ("a", "b", "c", "d")
```

#### 依赖类型方法

```scala
def leafElem[X](x: X): LeafElem[X] = x match
  case x: String      => x.charAt(0)
  case x: Array[t]    => leafElem(x(0))
  case x: Iterable[t] => leafElem(x.head)
  case x: AnyVal      => x

// 类型推断精确
val c: Char = leafElem("hello")      // Char
val i: Int = leafElem(Array(1,2,3))  // Int
```

#### 使用场景

- **类型级计算**：根据输入类型推导输出类型
- **泛型方法返回类型**：精确推断
- **类型安全的数据转换**

---

### 1.5 依赖函数类型

#### 核心概念

依赖函数类型的返回类型**依赖于参数值**。

#### 示例

```scala
trait Entry:
  type Key
  val key: Key

// 依赖函数类型
def extractKey(e: Entry): e.Key = e.key

// 多个依赖参数
def extractKeys(e1: Entry, e2: Entry): (e1.Key, e2.Key) =
  (e1.key, e2.key)
```

---

### 1.6 多态函数类型

#### 核心概念

多态函数类型是**接受类型参数的函数值**，可作为参数传递或作为结果返回。

#### 语法

```scala
[A] => List[A] => List[A]
```

#### 代码示例

```scala
// 多态方法
def foo[A](xs: List[A]): List[A] = xs.reverse

// 多态函数值
val bar: [A] => List[A] => List[A] =
  [A] => (xs: List[A]) => foo[A](xs)

// 使用
bar[Int](List(1,2,3))  // List(3,2,1)
```

#### 实际应用

```scala
enum Expr[A]:
  case Var(name: String)
  case Apply[A, B](fun: Expr[B => A], arg: Expr[B]) extends Expr[A]

def mapSubexpressions[A](e: Expr[A])(f: [B] => Expr[B] => Expr[B]): Expr[A] =
  e match
    case Apply(fun, arg) => Apply(f(fun), f(arg))
    case Var(n) => Var(n)

// 调用：传入多态函数
val e0 = Apply(Var("f"), Var("a"))
val e1 = mapSubexpressions(e0)(
  [B] => (se: Expr[B]) => Apply(Var[B => B]("wrap"), se))
```

---

## 2. 枚举与代数数据类型

### 2.1 枚举 (Enums)

#### 基本语法

```scala
enum Color:
  case Red, Green, Blue
```

这定义了一个 `sealed` 类 `Color`，包含三个值：`Color.Red`、`Color.Green`、`Color.Blue`。

#### 参数化枚举

```scala
enum Color(val rgb: Int):
  case Red   extends Color(0xFF0000)
  case Green extends Color(0x00FF00)
  case Blue  extends Color(0x0000FF)
```

#### 内置方法

```scala
val red = Color.Red

red.ordinal              // 0 - 序号
Color.valueOf("Blue")    // Color.Blue - 按名称获取
Color.values             // Array(Red, Green, Blue) - 所有值
Color.fromOrdinal(0)     // Color.Red - 按序号获取
```

#### 带自定义成员的枚举

```scala
enum Planet(mass: Double, radius: Double):
  private final val G = 6.67300E-11
  def surfaceGravity = G * mass / (radius * radius)
  def surfaceWeight(otherMass: Double) = otherMass * surfaceGravity

  case Mercury extends Planet(3.303e+23, 2.4397e6)
  case Venus   extends Planet(4.869e+24, 6.0518e6)
  case Earth   extends Planet(5.976e+24, 6.37814e6)
  case Mars    extends Planet(6.421e+23, 3.3972e6)
  case Jupiter extends Planet(1.9e+27,   7.1492e7)
  case Saturn  extends Planet(5.688e+26, 6.0268e7)
  case Uranus  extends Planet(8.686e+25, 2.5559e7)
  case Neptune extends Planet(1.024e+26, 2.4746e7)

object Planet:
  def main(args: Array[String]) =
    val earthWeight = args(0).toDouble
    val mass = earthWeight / Earth.surfaceGravity
    for p <- values do
      println(s"Your weight on $p is ${p.surfaceWeight(mass)}")
```

#### 与 Java 枚举兼容

```scala
enum Color extends Enum[Color]:
  case Red, Green, Blue

// 可使用 Java 枚举方法
Color.Red.compareTo(Color.Green)  // -1
```

---

### 2.2 代数数据类型 (ADTs)

#### 核心概念

Scala 3 的 `enum` 概念足够通用，可支持**代数数据类型**及其广义版本。

#### Option 类型实现

```scala
enum Option[+T]:
  case Some(x: T)
  case None
```

等价于：

```scala
enum Option[+T]:
  case Some(x: T) extends Option[T]
  case None       extends Option[Nothing]
```

#### 带方法的 Option

```scala
enum Option[+T]:
  case Some(x: T)
  case None

  def isDefined: Boolean = this match
    case None => false
    case _    => true

object Option:
  def apply[T >: Null](x: T): Option[T] =
    if x == null then None else Some(x)
```

#### 混合枚举与 ADT

```scala
enum Color(val rgb: Int):
  case Red   extends Color(0xFF0000)
  case Green extends Color(0x00FF00)
  case Blue  extends Color(0x0000FF)
  case Mix(mix: Int) extends Color(mix)  // 参数化 case
```

#### 类型变体处理

```scala
enum View[-T, +U] extends (T => U):
  case Refl[R](f: R => R) extends View[R, R]

  final def apply(t: T): U = this match
    case refl: Refl[r] => refl.f(t)
```

**类型推断规则：**
- 所有协变类型参数在 `extends` 子句中**最小化**
- 所有逆变类型参数在 `extends` 子句中**最大化**

---

## 3. 上下文抽象

### 3.1 Given 实例

#### 核心概念

Given 实例定义"**规范值**"，用于为上下文参数合成参数。

#### 基本定义

```scala
trait Ord[T]:
  def compare(x: T, y: T): Int
  extension (x: T)
    def < (y: T) = compare(x, y) < 0
    def > (y: T) = compare(x, y) > 0

given intOrd: Ord[Int]:
  def compare(x: Int, y: Int) =
    if x < y then -1 else if x > y then +1 else 0

given listOrd[T: Ord]: Ord[List[T]]:
  def compare(xs: List[T], ys: List[T]): Int = (xs, ys) match
    case (Nil, Nil) => 0
    case (Nil, _) => -1
    case (_, Nil) => +1
    case (x :: xs1, y :: ys1) =>
      val fst = summon[Ord[T]].compare(x, y)
      if fst != 0 then fst else compare(xs1, ys1)
```

#### 匿名 Given

```scala
given Ord[Int]:
  def compare(x: Int, y: Int) =
    if x < y then -1 else if x > y then +1 else 0

// 编译器合成名称：given_Ord_Int
```

> **注意**：公开库应使用命名实例以保证二进制兼容性。

#### 别名 Given

```scala
given global: ExecutionContext = ForkJoinPool()

// 首次访问时创建，后续返回相同实例
```

#### 初始化行为

| Given 类型 | 初始化方式 |
|-----------|-----------|
| 无参数无条件 given | 按需初始化，首次访问时初始化 |
| 别名指向不可变值 | 简单转发器，无缓存字段开销 |
| 条件 given | 每次引用创建新实例 |

---

### 3.2 Using 子句

#### 核心概念

Using 子句解决**参数传递问题**：当长调用链中多个函数需要相同参数时，避免显式重复传递。

#### 基本语法

```scala
def max[T](x: T, y: T)(using ord: Ord[T]): T =
  if ord.compare(x, y) < 0 then y else x

// 调用方式
max(2, 3)(using intOrd)  // 显式传递
max(2, 3)                 // 省略（通常用法）
```

#### 匿名上下文参数

```scala
def maximum[T](xs: List[T])(using Ord[T]): T =
  xs.reduceLeft(max)

// 参数名未使用，可省略
```

#### 类上下文参数

```scala
class GivenIntBox(using val usingParameter: Int):
  def myInt = summon[Int]

val b = GivenIntBox(using 23)
import b.usingParameter
summon[Int]  // 23
```

#### 多个 Using 子句

```scala
def f(u: Universe)(using ctx: u.Context)(using s: ctx.Symbol, k: ctx.Kind) = ...

// 调用
f(global)
f(global)(using ctx)
f(global)(using ctx)(using sym, kind)
```

#### 召唤实例

```scala
summon[Ord[List[Int]]  // 归约为 listOrd(using intOrd)

// summon 定义
def summon[T](using x: T): x.type = x
```

---

### 3.3 上下文边界

#### 核心概念

上下文边界是**上下文参数的简写**，常用于类型类建模。

#### 基本语法与展开

```scala
def maximum[T: Ord](xs: List[T]): T = xs.reduceLeft(max)

// 展开为
def maximum[T](xs: List[T])(using Ord[T]): T = ...
```

#### 与子类型边界组合

```scala
def f[T <: B : C](x: T): R = ...  // 子类型边界在前
```

#### 命名上下文边界 (Scala 3.6+)

```scala
trait Monoid[A]:
  def unit: A

def reduce[A: Monoid as m](xs: List[A]): A =
  xs.foldLeft(m.unit)(_ `combine` _)

// 展开为
def reduce[A](xs: List[A])(using m: Monoid[A]): A
```

#### 聚合上下文边界

```scala
def showMax[X : {Ord, Show}](x: X, y: X): String

// 命名版本
def showMax[X : {Ord as ordering, Show as show}](x: X, y: X): String =
  show.asString(ordering.max(x, y))
```

---

### 3.4 扩展方法

#### 核心概念

扩展方法允许**在类型定义后添加方法**。

#### 基本语法

```scala
extension (c: Circle)
  def circumference: Double = c.radius * math.Pi * 2

// 调用
circle.circumference
```

#### 运算符扩展

```scala
extension (x: String)
  def < (y: String): Boolean = ...

extension (x: Elem)
  def +: (xs: Seq[Elem]): Seq[Elem] = ...  // 右结合

extension (x: Number)
  infix def min (y: Number): Number = ...
```

#### 泛型扩展

```scala
extension [T](xs: List[T])
  def second = xs.tail.head

extension [T: Numeric](x: T)
  def + (y: T): T = summon[Numeric[T]].plus(x, y)
```

#### 集合扩展

```scala
extension (ss: Seq[String])
  def longestStrings: Seq[String] =
    val maxLength = ss.map(_.length).max
    ss.filter(_.length == maxLength)
  def longestString: String = longestStrings.head
```

---

### 3.5 类型类

#### 核心概念

类型类是**参数化 trait**，定义一组通用操作。

#### 完整示例：Eq 类型类

```scala
trait Eq[T]:
  def eqv(x: T, y: T): Boolean

object Eq:
  given Eq[Int]:
    def eqv(x: Int, y: Int) = x == y

  given Eq[String]:
    def eqv(x: String, y: String) = x == y

  // 条件实例
  given [T: Eq]: Eq[List[T]]:
    def eqv(xs: List[T], ys: List[T]): Boolean =
      xs.length == ys.length &&
      xs.lazyZip(ys).forall((x, y) => summon[Eq[T]].eqv(x, y))
```

---

### 3.6 类型类派生

#### 核心概念

类型类派生**自动生成 given 实例**。

#### 基本语法

```scala
enum Tree[T] derives Eq, Ordering, Show:
  case Branch(left: Tree[T], right: Tree[T])
  case Leaf(elem: T)

// 自动生成
given [T: Eq]       => Eq[Tree[T]]       = Eq.derived
given [T: Ordering] => Ordering[Tree[T]] = Ordering.derived
given [T: Show]     => Show[Tree[T]]     = Show.derived
```

#### Mirror 类型类

```scala
sealed trait Mirror:
  type MirroredType
  type MirroredElemTypes
  type MirroredMonoType
  type MirroredLabel <: String
  type MirroredElemLabels <: Tuple

object Mirror:
  trait Product extends Mirror:
    def fromProduct(p: scala.Product): MirroredMonoType

  trait Sum extends Mirror:
    def ordinal(x: MirroredMonoType): Int
```

#### 完整 Eq 派生实现

```scala
import scala.deriving.*
import scala.compiletime.{erasedValue, summonInline}

trait Eq[T]:
  def eqv(x: T, y: T): Boolean

object Eq:
  given Eq[Int]:
    def eqv(x: Int, y: Int) = x == y

  inline def derived[T](using m: Mirror.Of[T]): Eq[T] =
    lazy val elemInstances = summonInstances[T, m.MirroredElemTypes]
    inline m match
      case s: Mirror.SumOf[T]     => eqSum(s, elemInstances)
      case p: Mirror.ProductOf[T] => eqProduct(p, elemInstances)

  private def eqSum[T](s: Mirror.SumOf[T], elems: List[Eq[?]]): Eq[T] =
    new Eq[T]:
      def eqv(x: T, y: T): Boolean =
        val ordx = s.ordinal(x)
        (s.ordinal(y) == ordx) && elems(ordx).eqv(x, y)

  private def eqProduct[T](p: Mirror.ProductOf[T], elems: List[Eq[?]]): Eq[T] =
    new Eq[T]:
      def eqv(x: T, y: T): Boolean =
        x.productIterator.lazyZip(y.productIterator).lazyZip(elems)
          .forall((a, b, e) => e.eqv(a, b))
```

---

### 3.7 多宇宙相等性

#### 核心概念

Scala 3 引入**类型安全的相等性检查**，防止跨类型比较。

#### CanEqual 类型类

```scala
@implicitNotFound("Values of types ${L} and ${R} cannot be compared")
sealed trait CanEqual[-L, -R]

object CanEqual:
  object derived extends CanEqual[Any, Any]
```

#### 启用严格相等

```scala
import scala.language.strictEquality

// 或命令行
-language:strictEquality
```

#### 定义可比较类型

```scala
class T derives CanEqual

// 或手动定义
given CanEqual[T, T] = CanEqual.derived
```

#### 派生实例

```scala
class Box[T](x: T) derives CanEqual

// 生成
given [T, U] => CanEqual[T, U] => CanEqual[Box[T], Box[U]] = CanEqual.derived

// 使用
new Box(1) == new Box(1L)   // OK：Int 和 Long 可比较
new Box(1) == new Box("a")  // 错误：不可比较
```

---

## 4. 元编程

### 4.1 Inline

#### 核心概念

`inline` 保证定义在使用点会被**内联展开**。

#### Inline 值

```scala
object Config:
  inline val logging = false

  // 右侧必须是常量表达式
  inline val four = 4  // 类型为单例类型 4
```

#### Inline 方法

```scala
object Logger:
  private var indent = 0

  inline def log[T](msg: String, indentMargin: =>Int)(op: => T): T =
    if Config.logging then  // 常量条件被简化
      println(s"${"  " * indent}start $msg")
      indent += indentMargin
      val result = op
      indent -= indentMargin
      println(s"${"  " * indent}$msg = $result")
      result
    else op
```

#### 递归 Inline

```scala
inline def power(x: Double, n: Int): Double =
  if n == 0 then 1.0
  else if n == 1 then x
  else
    val y = power(x, n / 2)
    if n % 2 == 0 then y * y else y * y * x

power(expr, 10)
// 展开为无循环代码：
//   val x = expr
//   val y1 = x * x   // ^2
//   val y2 = y1 * y1 // ^4
//   val y3 = y2 * x  // ^5
//   y3 * y3          // ^10
```

#### Inline 参数

```scala
inline def funkyAssertEquals(actual: Double, expected: =>Double, inline delta: Double): Unit =
  if (actual - expected).abs > delta then
    throw new AssertionError(s"difference between ${expected} and ${actual} was larger than ${delta}")

// inline 参数：代码直接内联，可能重复求值
```

#### Transparent Inline

```scala
transparent inline def choose(b: Boolean): A =
  if b then new A else new B

val obj1 = choose(true)   // 静态类型：A
val obj2 = choose(false)  // 静态类型：B（精确类型）

obj2.m  // OK：B 有方法 m
```

#### Inline 匹配

```scala
transparent inline def g(x: Any): Any =
  inline x match
    case x: String => (x, x)
    case x: Double => x

g(1.0d)    // 类型：1.0d
g("test")  // 类型：(String, String)
```

---

### 4.2 编译时操作

#### constValue / constValueOpt

```scala
import scala.compiletime.constValue

transparent inline def toIntC[N]: Int =
  inline constValue[N] match
    case 0 => 0
    case _ => 1 + toIntC[N - 1]

inline val ctwo = toIntC[2]  // 值：2
```

#### erasedValue

```scala
import scala.compiletime.erasedValue

transparent inline def defaultValue[T] =
  inline erasedValue[T] match
    case _: Byte    => Some(0: Byte)
    case _: Int     => Some(0)
    case _: Boolean => Some(false)
    case _          => None

val dInt: Some[Int] = defaultValue[Int]
```

#### error

```scala
import scala.compiletime.error

inline def fail(inline p1: Any) =
  error("failed on: " + codeOf(p1))

fail(identity("foo"))  // 编译错误：failed on: identity[String]("foo")
```

#### summonFrom

```scala
import scala.compiletime.summonFrom

inline def setFor[T]: Set[T] = summonFrom {
  case given Ordering[T] => new TreeSet[T]
  case _                 => new HashSet[T]
}
```

#### 类型级运算

```scala
import scala.compiletime.ops.int.*

val conjunction: true && true = true
val multiplication: 3 * 5 = 15
val addition: 1 + 2 * 3 = 7  // 遵循运算优先级
```

---

### 4.3 宏

#### 核心概念

Scala 3 宏基于**引用和拼接**，类型安全且卫生。

#### 引号与拼接

```scala
import scala.quoted.*

// '{..} - 引用（延迟执行）
// ${..} - 拼接（求值并插入）

def unrolledPowerCode(x: Expr[Double], n: Int)(using Quotes): Expr[Double] =
  if n == 0 then '{ 1.0 }
  else if n == 1 then x
  else '{ $x * ${ unrolledPowerCode(x, n-1) } }
```

#### 内联宏定义

```scala
inline def powerMacro(x: Double, inline n: Int): Double =
  ${ powerCode('x, 'n) }

// 用户代码
def power2(x: Double) = powerMacro(x, 2)  // 展开为 x * x
```

#### 抽象类型处理

```scala
def singletonListExpr[T: Type](x: Expr[T])(using Quotes): Expr[List[T]] =
  '{ List[T]($x) }

def emptyListExpr[T](using Type[T], Quotes): Expr[List[T]] =
  '{ List.empty[T] }
```

#### 引用模式匹配

```scala
def fusedPowCode(x: Expr[Double], n: Expr[Int])(using Quotes): Expr[Double] =
  x match
    case '{ power($y, $m) } =>
      fusedPowCode(y, '{ $n * $m })
    case _ =>
      '{ power($x, $n) }
```

#### 类型模式

```scala
def empty[T: Type](using Quotes): Expr[T] =
  Type.of[T] match
    case '[String] => '{ "" }
    case '[List[t]] => '{ List.empty[t] }
```

---

### 4.4 反射

Scala 3 提供运行时反射能力（通过 TASTy），但主要通过宏在编译时进行类型检查和代码生成。

---

## 5. 其他新特性

### 5.1 Trait 参数

#### 核心概念

Trait 可以拥有参数，类似类的参数。

#### 基本语法

```scala
trait Greeting(val name: String):
  def msg = s"How are you, $name"

class C extends Greeting("Bob"):
  println(msg)
```

#### 三条规则

1. 若类 C 继承参数化 trait T，且其父类未继承，C **必须**向 T 传递参数
2. 若类 C 继承参数化 trait T，且其父类也已继承，C **不能**向 T 传递参数
3. Trait 永远不能向父 trait 传递参数

#### 错误示例

```scala
class D extends C, Greeting("Bill")  // 错误：参数传递两次

trait FormalGreeting extends Greeting:
  override def msg = s"How do you do, $name"

class E extends FormalGreeting  // 错误：缺少 Greeting 参数

// 正确写法
class E extends Greeting("Bob"), FormalGreeting
```

---

### 5.2 透明 Trait

```scala
transparent trait ID

case class UserName(name: String)
case class Password(hash: Hash)

// 类型不拓宽到 ID
if true then UserName("Eve") else Password(123)
// 类型：UserName | Password
```

---

### 5.3 Opaque 类型别名

#### 核心概念

Opaque 类型提供**零开销的类型抽象**。

#### 基本示例

```scala
object MyMath:
  opaque type Logarithm = Double

  object Logarithm:
    def apply(d: Double): Logarithm = math.log(d)
    def safe(d: Double): Option[Logarithm] =
      if d > 0.0 then Some(math.log(d)) else None

  extension (x: Logarithm)
    def toDouble: Double = math.exp(x)
    def + (y: Logarithm): Logarithm = Logarithm(math.exp(x) + math.exp(y))
    def * (y: Logarithm): Logarithm = x + y

// 使用
import MyMath.Logarithm

val l = Logarithm(1.0)
val l2 = Logarithm(2.0)
val l3 = l * l2

// 类型错误
val d: Double = l          // 错误
val l2: Logarithm = 1.0    // 错误
l * 2                      // 错误
```

#### 带边界的 Opaque 类型

```scala
object Access:
  opaque type Permissions = Int
  opaque type Permission <: Permissions = Int

  extension (granted: Permissions)
    def is(required: Permissions) = (granted & required) == required

  val Read: Permission = 1
  val Write: Permission = 2
  val ReadWrite: Permissions = Read | Write
```

---

### 5.4 Export 子句

#### 核心概念

Export 为对象的选定成员定义**别名**，实现转发功能。

#### 基本示例

```scala
class Printer:
  type PrinterType
  def print(bits: BitMap): Unit = ???
  def status: List[String] = ???

class Scanner:
  def scan(): BitMap = ???
  def status: List[String] = ???

class Copier:
  private val printUnit = new Printer { type PrinterType = InkJet }
  private val scanUnit = new Scanner

  export scanUnit.scan
  export printUnit.{status as _, *}

  def status: List[String] = printUnit.status ++ scanUnit.status

// 生成的别名
// final def scan(): BitMap = scanUnit.scan()
// final def print(bits: BitMap): Unit = printUnit.print(bits)
```

#### 语法格式

```scala
export path.{sel_1, ..., sel_n}

// 选择器类型
// x          - 为 x 创建别名
// x as y     - 重命名为 y
// x as _     - 阻止通配符为 x 创建别名
// given x    - 为给定类型的 given 实例创建别名
// *          - 为所有合格成员创建别名
```

---

### 5.5 命名元组

```scala
// 命名元组（实验性功能）
val person = (name = "Alice", age = 25)

person.name  // "Alice"
person.age   // 25
```

---

### 5.6 可选大括号/缩进语法

#### 核心规则

编译器在特定换行处插入 `<indent>` 或 `<outdent>` 标记。

#### 示例

```scala
// 传统语法
if (x < 0) {
  println(1)
  println(2)
}

// 缩进语法
if x < 0 then
  println(1)
  println(2)

// 模板体
trait A:
  def f: Int

class C(x: Int) extends A:
  def f = x

// 方法参数
xs.map: x =>
  val y = x - 1
  y * y

// Case 子句（可不缩进）
x match
case 1 => print("I")
case 2 => print("II")
```

#### End 标记

```scala
def largeMethod(...) =
  ...
  if ... then ...
  else
    ... // 大块代码
  end if
  ... // 更多代码
end largeMethod
```

---

### 5.7 顶层定义

Scala 3 允许在文件顶层直接定义类、trait、对象、方法、值等，无需包裹在包对象中。

```scala
// 直接在文件顶层
def topLevelMethod(x: Int): Int = x + 1

val topLevelValue = 42

class TopLevelClass

// 无需 package object
```

---

### 5.8 改进的 for 表达式

```scala
// 更清晰的 for 语法
for
  x <- xs
  y <- ys
  if x < y
do
  println(x, y)

// 替代传统
for {
  x <- xs
  y <- ys
  if x < y
} println(x, y)
```

---

## 6. 变更的特性

### 6.1 类型推断变化

Scala 3 类型推断更**精确**，倾向于保留更具体的类型。

```scala
// Scala 2：推断为 Product with Serializable
// Scala 3：推断为具体类型
if true then UserName("Eve") else Password(123)
```

---

### 6.2 隐式解析变化

| Scala 2 | Scala 3 |
|---------|---------|
| `implicit` | `given` / `using` |
| `implicitly` | `summon` |
| 隐式转换 | `given Conversion` |
| 隐式类 | `extension` |

---

### 6.3 模式匹配改进

```scala
// Scala 3：更精确的类型匹配
x match
  case _: List[t] =>  // t 是类型变量
  case _: Array[Int] =>

// Scala 2：类型模式需要 @unchecked
```

---

### 6.4 移除的特性

| 移除特性 | 替代方案 |
|---------|---------|
| `DelayedInit` | Trait 参数 |
| Scala 2 宏 | 新宏系统 |
| 存在类型 | 联合类型/匹配类型 |
| `do-while` | `while do` |
| 过程语法 | 显式 `: Unit =` |
| 包对象 | 顶层定义 |
| 早期初始化器 | Trait 参数 |
| 22 参数限制 | 无限制 |
| XML 字面量 | 库支持 |
| 符号字面量 | 字符串 |
| 自动应用 | 显式 `()` |
| `private[this]` | `private` |
| `_` 初始化器 | 显式初始化 |

---

## 7. 最佳实践总结

### 7.1 类型系统

- **优先使用联合类型**替代 `Either` 当只需要简单分支时
- **使用匹配类型**实现类型级计算
- **Opaque 类型**用于领域抽象，避免包装开销

### 7.2 上下文抽象

- **Given 实例**用于类型类和依赖注入
- **扩展方法**用于增强现有类型
- **命名上下文边界**提高可读性

### 7.3 元编程

- **Inline**用于性能优化和编译时配置
- **Transparent inline**用于精确类型推断
- **宏**用于复杂代码生成

### 7.4 语法风格

- **缩进语法**提高可读性，减少大括号嵌套
- **End 标记**标记大块代码结束
- **Export**替代继承实现组合

### 7.5 迁移建议

```scala
// Scala 2 风格
implicit val x: T = ...
implicit def f(...): R = ...
implicit class C(...)

// Scala 3 风格
given x: T = ...
given f(...): R = ...
extension (...) def method = ...
```

---

## 附录：参考资源

- [Scala 3 官方文档](https://docs.scala-lang.org/scala3/)
- [Scala 3 参考文档](https://docs.scala-lang.org/scala3/reference/index.html)
- [Scala 3 Book](https://docs.scala-lang.org/scala3/book/introduction.html)
- [Scala 2 to Scala 3 Migration Guide](https://docs.scala-lang.org/scala3/guides/migration/compatibility-classpath.html)

---

> 本文档基于 Scala 3 官方参考文档整理，涵盖核心新特性与最佳实践。