---
title: Scala-with-cats中文翻译（四）：Type class的变型与不变
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-tg6785hyj56
tags:
  - scala
  - cats
  - type class
---

### 前言

本篇内容主要来学习Type class的变型与不变。

> **学习程度：了解即可**

在使用type class的时候，我们必须考虑以下两个问题，因为它们对于如何选择instance至关重要：

- 假设B是A的子类型，那么声明为A类型的instance能作用于B吗？

  举个例子，假如我们声明了一个JsonWriter[Option[Int]]的instance，那么Json.toJson(Some(1))能使用这个instance吗？（Some是Option的子类型）

- 同一个类型有多个type class instance的时候该如何选择？
  举个例子，比如我们声明了两个JsonWriter对应Person类型的instance，当我们在使用Json.toJson(aPerson)，该如何选择instance？

#### **1.6.1 变型**

当我们在声明type class时，可以使用可变的类型参数，这样可以让type class也有“变型”的能力。

在Essential scala中提到，variance跟子类型有关，假如可以在任意接收A类型值的地方用B类型值代替，那么可以说B是A的子类型。

当我们在定义类型构造器的时候，可以使用annotations来标明它是否是支持协变或者逆变的。我们可以使用**”+“**符号表示它是协变的：

```scala
trait F[+A] // the "+" means "covariant"
```

##### 协变

convariance代表着假如B是A的子类型，那么F[B]也是F[A]的子类型。这对很多类型的建模都很有用，比如List和Option：

```scala
trait List[+A]
trait Option[+A]
```

在Scala中支持协变的集合允许我们使用子类型的集合代替父类型的集合。比如我们可以在任何接收List[Shape]的地方使用List[Circle]代替，因为Circle是Shape的子类型：

```scala
sealed trait Shape
case class Circle(radius: Double) extends Shape

val circles: List[Circle] = ???
val shapes: List[Shape] = circles
```

那么什么是逆变呢？我们可以使用**”-“**符号表示它是逆变的：

```scala
trait F[-A]
```

##### 逆变

不同的是，contravariance代表着假如B是A的子类型，那么F[A]是F[B]的子类型。逆变在对”**processes（处理）**“建模的时候非常有用，比如我们定义一个JsonWriter：

```scala
trait JsonWriter[-A] {
  def write(value: A): Json
}
// defined trait JsonWriter
```

进一步看其中的原理，我们要知道variance其实就是用一个值替换另一个值的能力。考虑一个场景，我们有两个类型值，Shape和Circle类型，以及两个JsonWriters，同样分别是Shape和Circle类型的：

```scala
val shape: Shape = ???
val circle: Circle = ???

val shapeWriter: JsonWriter[Shape] = ???
val circleWriter: JsonWriter[Circle] = ???

def format[A](value: A, writer: JsonWriter[A]): Json = writer.write(value)
```

现在你可以思考一下：”format方法支持哪些参数组合呢？“。假如value为circle，那么writer可以为任一一个，因为所有Circle都是Shape。但反过来，shape不能与circleWriter组合，因为不是所有的Shape都是Circle。

这种情况下，我们就会使用逆变参数来进行建模。JsonWriter[Shape]是JsonWriter[Circle]子类型，因为Circle是Shape的子类型，这意味着任何接收JsonWriter[Circle]类型值的地方，都可以用shapeWriter代替。

##### 不变型

不变相对来说更容易理解，在类型构造的时候不需要指定”**+**“或者”**-**“的符号：

```scala
trait F[A]
```

这意味着不管A和B是什么关系，F[A]和F[B]都不再是对方的子类型，这也是Scala的默认方式。

当编译器在寻找implicit值的时候，除了自身类型implicit值以外，子类型的implicit值也在匹配范围之内，因此我们在一定程度上可以使用可变类型来控制type class instance的选择。

但这同样会导致一些问题，假设我们有以下代数数据类型：

```scala
sealed trait A
final case object B extends A
final case object C extends A
```

思考以下两个问题：

1. 子类型的值是否可以使用父类型的Instance？举个例子，我们声明了一个A类型的instance，那么它是否可以作用于B或者C？
2. 同时存在子类型以及父类型的instance的时候，优先选择哪个？比如现在我们分别声明A和B的instance，这时有一个B类型的值，它是否会优先选择B instance吗？

事实上，我们无法同时拥有两者，我们可以对以下三种情况进行归纳：

| Type Class Variance  | 不可变 | 协变 | 逆变 |
| -------------------- | ------ | ---- | ---- |
| 是否可以使用父类型？ | No     | No   | Yes  |
| 优先选择子类型？     | No     | Yes  | No   |

很明显，没有完美的类型系统。Cats更倾向于使用不变的类型，这意味着我们需要指定更具体的instance，举个例子一个Some[Int]的值直接使用Option[Int]类型的instance，如果需要的话，可以将Some[Int]类型的值声明为Option[Int]，比如 Some(1) : Option[Int]，或者使用一些更便捷的方法，比如我们在之前1.5.3章节中看到的Option.apply, Option.empty, some, none等方法。