# Scala 3 上下文抽象与现代架构设计指南

> 基于 Scala 2 到 Scala 3 的重大范式转变，由浅入深地系统梳理 Scala 3 的核心现代特性。
>
> Scala 3 的核心哲学是**「意图导向 (Intent-driven)」**。它将 Scala 2 中功能过于混杂的 `implicit` 关键字解构并重塑，演变为一套逻辑清晰、专职专责的语言特性。

---

## 目录

1. [从瑞士军刀到专职专责（关键字重塑）](#第一章从瑞士军刀到专职专责关键字重塑)
2. [非侵入式扩展与零成本抽象（Extension Methods）](#第二章非侵入式扩展与零成本抽象extension-methods)
3. [现代架构的最佳实践（Export 机制）](#第三章现代架构的最佳实践export-机制)
4. [泛型地基与上下文界定的控场](#第四章泛型地基与上下文界定的控场)
5. [终极大招——高级类型类（Typeclass）模式](#第五章终极大招高级类型类typeclass模式的合流)
6. [Scala 2 ➔ Scala 3 全景对照速查表](#第六章scala-2--scala-3-全景对照速查表)

---

## 第一章：从瑞士军刀到专职专责（关键字重塑）

在 Scala 2 中，`implicit` 承载了过多的职责（修饰变量、参数、类、转换函数），导致代码意图模糊。Scala 3 将其拆解为更具可读性的专属关键字。

### 1.1 上下文的供给与接收：`implicit` ➔ `given` / `using`

Scala 3 严格区分了上下文的提供方与接收方，实现了语意上的顺畅。

**Scala 2（意图模糊）：**

```scala
implicit val runtimeContext: String = "Cluster-Mode"
def execute(implicit ctx: String) = println(s"Running in $ctx")
```

**Scala 3（意图清晰）：**

```scala
given runtimeContext: String = "Cluster-Mode"
def execute(using ctx: String) = println(s"Running in $ctx")
```

> 💡 **核心逻辑：** Scala 3 的显式哲学是："我 **给（given）你一个背景环境，你在需要的地方用（using）** 它就行。"

### 1.2 显式获取：`implicitly` ➔ `summon`

当需要在代码块中手动提取当前上下文中的隐式实例时，Scala 3 引入了更形象的 `summon`（召唤）。

```scala
// Scala 2
val ctx = implicitly[String]

// Scala 3
val ctx = summon[String]
```

### 1.3 隐式转换的收敛：`implicit def` ➔ `given Conversion`

Scala 2 的 `implicit def` 允许编译器在后台任意做类型转换，容易导致类型安全失控与编译速度下降。Scala 3 废弃了这种机制，要求必须显式定义一个 `Conversion` 实例。

```scala
// Scala 2
implicit def intToString(x: Int): String = x.toString

// Scala 3
given Conversion[Int, String] with
  def apply(x: Int): String = x.toString
```

---

## 第二章：非侵入式扩展与零成本抽象（Extension Methods）

在面向对象设计中，我们经常遇到需要为第三方库（或内置类型）添加方法的场景。Scala 3 引入了原生的 `extension` 机制，彻底终结了 Scala 2 复杂的 `implicit class` 包装套路。

### 2.1 基础与多方法扩展

`extension` 的核心语法极其符合直觉：先声明要扩展的对象及类型，再定义具体方法。

```scala
// 一次性为 String 扩展多个方法
extension (s: String) {
  def isEmail: Boolean = s.contains("@") && s.contains(".")
  def encrypt: String = s.reverse
}

// 使用层完全无感
"user@domain.com".isEmail  // true
```

### 2.2 底层原理：零运行时开销

有些开发者担心 `extension` 会破坏类型系统或带来性能损耗。实际上，它纯粹是**编译器的语法糖（视觉擦除）**。编译器在幕后会将扩展方法改写为高效的静态方法调用，避免了老式隐式类在运行时高频创建临时包装对象的内存开销：

```scala
// 编译器幕后生成的伪代码
object ExtensionOps {
  def isEmail(s: String): Boolean = s.contains("@") && s.contains(".")
}

// 你的调用 "abc".isEmail 被改写为：
ExtensionOps.isEmail("abc")
```

---

## 第三章：现代架构的最佳实践（Export 机制）

在经典设计模式中，「多用组合，少用继承（Composition over Inheritance）」是保障系统解耦的核心原则。然而在过去，组合模式伴随着大量死板的「胶水代码」。Scala 3 引入 `export` 彻底打破了这一困境。

### 3.1 老式组合的痛点（复读机模式）

当一个类内部组合了其他组件，并期望对外暴露这些组件的能力时，传统写法必须手动写代理转发：

```scala
class Printer {
  def printDoc(text: String): Unit = println(s"Printing: $text")
}

// 老式组合：架构健康，但编码极其痛苦
class SmartPrinter {
  private val printer = new Printer

  // 手动充当传话筒（胶水代码）
  def printDoc(text: String): Unit = printer.printDoc(text)
}
```

### 3.2 Scala 3 的解法：用 `export` 一键曝光

`export` 的方向与 `import` 正好相反。`import` 是将外部成员拉到内部使用；`export` 是将内部成员的能力公开曝光为自己的接口。

```scala
class SmartPrinter {
  private val printer = new Printer

  // 将 printer 的所有公开方法通配导出，外界可直接调用
  export printer.*

  def aiOptimize(): Unit = println("AI Optimizing...")
}

// 外界使用
val machine = new SmartPrinter()
machine.printDoc("Hello Data Pipeline")  // 实际执行了内部 printer.printDoc
```

此外，`export` 还支持极其灵活的**过滤与重命名**机制：

```scala
export printer.{printDoc => printLayout, *}  // 导出并重命名
export scanner.{scanDoc => _, *}             // 排除特定方法后导出其余全部
```

---

## 第四章：泛型地基与上下文界定的控场

要构建高级的抽象框架，必须将泛型与上下文机制结合。它们之间是**「地基」与「上层建筑」**的关系。

### 4.1 泛型 `[T]` 的本质

在声明方法时，方括号 `[T]` 的本质是给类型办一张**临时身份证（类型占位符）**。

```scala
def test[T](x: T): T = x
```

`[T]` 负责拓宽通用性，它告诉编译器这是一个通用方法，现在还不知道具体类型，由调用者传入。如果不写 `[T]` 直接写 `(x: T)`，编译器会误认为 `T` 是一个已经存在的基础类（如 `String`），从而报找不到类型的错误。

### 4.2 从无约束到有约束：引入上下文界定（Context Bounds）

泛型本身是完全自由的（白纸），但有些操作（如排序、序列化）要求类型必须具备特定的能力。

**传统 Using 参数写法（臃肿）：**

```scala
def max[T](x: T, y: T)(using ord: Ordering[T]): T =
  if (ord.compare(x, y) > 0) x else y
```

**Scala 2/3 经典上下文界定简写（丢失变量名）：**

```scala
def max[T: Ordering](x: T, y: T): T =
  if (summon[Ordering[T]].compare(x, y) > 0) x else y  // 必须用 summon 捞出
```

### 4.3 Scala 3.4+ 终极解法：命名上下文界定（Named Context Bounds）

为了兼顾「签名的紧凑性」与「内部调用的便利性」，Scala 3.4 允许在上下文界定中通过 `as` 直接为隐式实例命名。

```scala
// 既保持了泛型声明的紧凑，又直接拿到了实例变量名 ord
def max[T: Ordering as ord](x: T, y: T): T =
  if (ord.compare(x, y) > 0) x else y
```

---

## 第五章：终极大招——高级类型类（Typeclass）模式的合流

当我们把 **泛型 `[T]`、上下文抽象 `given/using`、扩展方法 `extension`** 和 **命名上下文界定 `as`** 全部融会贯通后，就可以写出 Scala 3 顶级的解耦神技：**现代类型类模式**。

假设我们要在不触动原有类源码的前提下，让任意类型具备「转 JSON 字符串」的能力：

### 1. 定义核心能力接口（Typeclass 原型）

```scala
trait JsonEncoder[A]:
  def toJson(a: A): String
```

### 2. 准备具体类型的上下文实现（Given 实例）

```scala
case class User(name: String)

given JsonEncoder[User] with
  def toJson(u: User): String = s"""{"name": "${u.name}"}"""
```

### 3. 一行代码合成天下：结合 Extension 与命名上下文界定

```scala
extension [T: JsonEncoder as encoder](x: T)
  def toJson: String = encoder.toJson(x)
```

**此处的架构美学解析：**

| 语法要素 | 含义 |
|---------|------|
| `[T: JsonEncoder as encoder]` | 向全世界敞开大门（支持任意类型 `T`），但设有门禁——后台必须备有转 JSON 的工具人，并将其命名为 `encoder` |
| `extension ... (x: T)` | 动态地为满足条件的类型挂载 `.toJson` 语法外壳 |
| `encoder.toJson(x)` | 方法外壳不包含具体业务，在被调用时，直接将活儿甩给后台抓取到的具体工具人 |

### 4. 丝滑的用户端调用

```scala
val user = User("张三")
user.toJson  // 返回 {"name": "张三"}，类型安全、按需扩展、零侵入
```

---

## 第六章：Scala 2 ➔ Scala 3 全景对照速查表

| 功能分类 | Scala 2 写法（多功能工具） | Scala 3 写法（意图导向） | 核心设计哲学 / 变更说明 |
|---------|--------------------------|-------------------------|----------------------|
| **上下文供给** | `implicit val ctx = ...` | `given ctx: T = ...` | 明确意图：清晰声明我是背景环境提供者 |
| **上下文接收** | `def foo(implicit ctx: T)` | `def foo(using ctx: T)` | 明确意图：清晰声明我需要背景环境配合 |
| **手动捞取实例** | `implicitly[T]` | `summon[T]` | 语意更具象、生动（召唤） |
| **隐式类型转换** | `implicit def aToB(x: A): B` | `given Conversion[A, B]` | 规范化转换通道，封杀失控的黑魔法 |
| **非侵入方法扩展** | `implicit class Ops(x: A)` | `extension (x: A)` | 回归语言直觉，消灭临时包装类，零开销 |
| **命名上下文界定** | 不支持（只能通过 `implicitly` 捞取） | `[T: Context as name]` | 重大升级：兼顾泛型签名的紧凑与代码引用 |
| **代码无感转发** | 手动写大量代理委托方法 | `export internalInstance.*` | 完美支持「组合优于继承」的干净架构 |

---

> 本文档基于 Scala 3 现代架构实践编写，涵盖上下文抽象、扩展方法、组合模式与类型类等核心内容。
