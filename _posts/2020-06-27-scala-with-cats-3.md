---
title: Scala-with-cats中文翻译（三）：一个类型安全的判等Type class：Eq
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-hdgnjknd8ujy
tags:
  - scala
  - cats
  - type class
---

### 前言

本篇内容主要来学习Cats中另一个基础的type class：Eq。

> **学习程度：需要完全掌握**

### 1.5 Eq：一个类型安全的判等Type class

本章节我们继续来学习一个非常实用的type class：[cats.Eq](https://typelevel.org/cats/api/cats/kernel/Eq.html)。Eq主要是为了类型安全的判等设计的，因为Scala内置的 ==操作符有时会给我们带来困扰。

大多数Scala程序员都应该写过类似下面的代码：

```scala
List(1, 2, 3).map(Option(_)).filter(item => item == 1)
// res0: List[Option[Int]] = List()
```

可能很多人都不会犯这么简单的错误，但是这是可能存在的，filter里面的判断逻辑会一直返回false，因为Int和Option[Int]是不可能相等的。

这是开发者的错，我们应该用Some(1)去比较而不是1。然而这在技术上来说并不能说它是错的，因为==可以作用于任意的两个对象，不用关心具体的类型。Eq的设计，解决了这个问题，因为它是类型安全的。

#### 1.5.1 Equality, Liberty and Fraternity

我们可以使用Eq对任意给定类型的对象进行类型安全的判等：

```scala
package cats

trait Eq[A] {
  def eqv(a: A, b: A): Boolean
  // other concrete methods based on eqv...
}
```

与Show类似，Eq对应的interface syntax，声明在[cats.syntax.eq](https://typelevel.org/cats/api/cats/syntax/package$$eq$)这个包中，它提供了两个执行判等的方法，你可以直接使用，当然前提是在implicit scope中导入了对应的instance：

- === 比较两个对象相等
- =!= 比较两个对象不相等

#### 1.5.2 比较Int类型

让我们来看些例子，首先我们需要先导入Eq的type class:

```scala
import cats.Eq
```

接着，我们来获取一个Int的instance：

```scala
import cats.instances.int._ // for Eq

val eqInt = Eq[Int]
```

我们可以直接使用eqInt来进行判等：

```scala
eqInt.eqv(123, 123)
// res2: Boolean = true

eqInt.eqv(123, 234)
// res3: Boolean = false
```

 不同于Scala的==操作符，假如你试图用eqv去比较两个不同类型的对象时，编译将会报错：

```scala
eqInt.eqv(123, "234")
// <console>:18: error: type mismatch; // found : String("234")
// required: Int
// eqInt.eqv(123, "234")
// ^
```

我们同样可以使用interface syntax语法，但需要先导入[cats.syntax.eq](https://typelevel.org/cats/api/cats/syntax/package$$eq$)，然后我们就可以直接使用 === 和 =!=方法：

```scala
import cats.syntax.eq._ // for === and =!=

123 === 123
// res5: Boolean = true

123 =!= 234
// res6: Boolean = true
```

同样，我们尝试去比较两个不同类型的对象时也会编译报错：

```scala
123 === "123"
// <console>:20: error: type mismatch;
//  found   : String("123")
//  required: Int
//        123 === "123"
//         
```

#### 1.5.3 比较Option类型

接下来我们来看一个更有趣的例子—Option[Int]。如果要比较Option[Int]类型的值，我们需要先导入Option以及Int对应的instances：

```scala
import cats.instances.int._    // for Eq
import cats.instances.option._ // for Eq
```

现在我们来尝试进行一些比较：

```scala
Some(1) === None
// <console>:26: error: value === is not a member of Some[Int] // Some(1) === None
// 
```

编译发现了一个错误，原因是类型没匹配上，我们导入的是Int以及Option[Int]对应Eq的instances，所以Some[Int]是无法比较的，要解决这个问题我们需要将值的类型指定为Option[Int]：

```scala
 (Some(1) : Option[Int]) === (None : Option[Int])
// res9: Boolean = false
```

更友好的方式是利用标准库中的Option.apply和Option.empty方法：

```scala
 Option(1) === Option.empty[Int]
// res10: Boolean = false
```

或者使用[cats.syntax.option](https://typelevel.org/cats/api/cats/syntax/package$$option$)中特殊语法：

```scala
import cats.syntax.option._ // for some and none

1.some === none[Int]
// res11: Boolean = false
1.some =!= none[Int]
// res12: Boolean = true
```

#### 1.5.4 比较自定义类型

我们可以为自定义的类型创建一个关于Eq的instance，它接收一个(A, A) => Boolean 的方法返回一个Eq[A]：

```scala
import java.util.Date
import cats.instances.long._ // for Eq

implicit val dateEq: Eq[Date] =
  Eq.instance[Date] { (date1, date2) =>
    date1.getTime === date2.getTime
  }
val x = new Date() // now
val y = new Date() // a bit later than now

x === x
// res13: Boolean = true
x === y
// res14: Boolean = false
```

#### 1.5.5 练习: 实现Cat类型的判等

实现一个Cat类型关于Eq的instance：

```scala
final case class Cat(name: String, age: Int, color: String)
```

并对下面这些对象进行判等操作：

```scala
val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 33, "orange and black")

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]
```

 代码见[示例](https://github.com/godpan/scala-with-cats-demo/blob/master/chapter1/src/main/scala/A1/CatShowInstances.scala)