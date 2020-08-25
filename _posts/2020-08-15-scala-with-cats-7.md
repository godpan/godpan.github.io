---
title: Scala-with-cats中文翻译（七）：Cats中的Monoid
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-nhjku6ddgg678
tags:
  - scala
  - cats
  - type class
---

### 前言

本小节我们将来学习一些已经在Cats中实现的Monoid。

> **学习程度：需要完全掌握**

#### 2.5 Cats中的Monoid

现在我们已经知道什么是Monoid了，接下去就来看一下Cats中有哪些Monoid实现吧。我们还是先来看type class的三个主要部分：type class、instance、interface。

##### 2.5.1 The Monoid Type Class

Cats中的type class都定义在cats.kernel.Monoid这个包中，别名[cats.Monoid](https://typelevel.org/cats/api/cats/kernel/Monoid.html)。Monoid继承了cats.kernel.Semigroup（别名[cats.Semigroup](https://typelevel.org/cats/api/cats/kernel/Semigroup.html))，通常我们使用以下方式来进行导入：

```scala
import cats.Monoid
import cats.Semigroup
```

>*Cats Kernel?*
>
>Cats Kernel 是Cats的一个子项目，它仅仅包含了Cats中一些核心的type class，它们被定义在[cats.kernel](http://typelevel.org/cats/api/cats/kernel/)这个包，别名[cats](https://typelevel.org/cats/api/cats/)，但我们通常不需要关心这些。
>
>Cats Kernel包含了我们之前讲过的一些type class，比如[Eq](https://typelevel.org/cats/api/cats/kernel/Eq.html)，[Semigroup](https://typelevel.org/cats/api/cats/kernel/Semigroup.html)，[Monoid](https://typelevel.org/cats/api/cats/kernel/Monoid.html)等，其他属于Cats主项目的type class都声明在[cats](https://typelevel.org/cats/api/cats/)这个包中。

##### 2.5.2 Monoid Instances

Monoid同样遵循Cats的标准结构，在其伴生对象中有一个apply方法，用于返回特定类型的type class instance。以下两种获取instance是等价的：

```scala
import cats.Monoid
import cats.instances.string._ // for Monoid

Monoid[String].combine("Hi ", "there")
// res0: String = Hi there
Monoid[String].empty
// res1: String = ""
```

等价于：

```scala
Monoid.apply[String].combine("Hi ", "there") 
// res2: String = Hi there

Monoid.apply[String].empty
// res3: String = ""
```

我们知道，Monoid是继承Semigroup，如果不需要empty值，也可以这么写：

```scala
import cats.Semigroup

Semigroup[String].combine("Hi ", "there")
// res4: String = Hi there
```

Monoid相关的instance也是按照第一章中所讲的原则一样，根据类型定义在cats.instances这个包中。比如我们想要使用Int类型的Monoid instance，则需要导入[cats.instances.int](https://typelevel.org/cats/api/cats/instances/package$$int$)：

```scala
import cats.Monoid
import cats.instances.int._ // for Monoid

Monoid[Int].combine(32, 10)
// res5: Int = 42
```

同样，如果我们需要Monoid[Option[Int]]的instance，则需要同时导入[cats.instances.int](https://typelevel.org/cats/api/cats/instances/package$$int$)和[cats.instances.option](https://typelevel.org/cats/api/cats/instances/package$$option$)：

```scala
import cats.Monoid
import cats.instances.int._    // for Monoid
import cats.instances.option._ // for Monoid

val a = Option(22)
// a: Option[Int] = Some(22)
val b = Option(20)
// b: Option[Int] = Some(20)

Monoid[Option[Int]].combine(a, b)
// res6: Option[Int] = Some(42)
```

更多关于导入的详情请参考第一章的相关内容。

##### 2.5.3 Monoid Syntax

Cats通过interface syntax为combine提供了“|+|”操作符，因为combine方法是定一个Semigroup中的，所以要使用这个syntax需要导入[cats.syntax.semigroup](http://typelevel.org/cats/api/cats/syntax/package$$semigroup$)：

```scala
import cats.instances.string._ // for Monoid
import cats.syntax.semigroup._ // for |+|

val stringResult = "Hi " |+| "there" |+| Monoid[String].empty
// stringResult: String = Hi there

import cats.instances.int._ // for Monoid

val intResult = 1 |+| 2 |+| Monoid[Int].empty
// intResult: Int = 3
```

##### 2.5.4 练习: 实现一个可以累加任何类型的累加器

*SuperAdder v3.5a-32*加法器是对两数相加的首选工具，它主要功能由def add(items: List[Int]): Int这个方法提供，假如该方法的代码不幸被删除，现在要你去实现，你会怎么做？

代码见[示例](https://github.com/godpan/scala-with-cats-demo/blob/master/chapter2/src/main/scala/adding/Add.scala#L3)

很好，我们已经实现了基本的加法功能，但随着SuperAdder的发展，现在有了额外的需求，需要对List[Option[Int]]进行相加，现在就去实现吧，但要注意的是代码的质量要高，不要出现重复代码。

代码见[示例](https://github.com/godpan/scala-with-cats-demo/blob/master/chapter2/src/main/scala/adding/Add.scala#L18)

现在SuperAdder已经打进销售领域了，所以它现在必须拥有对订单相加的功能：

```scala
case class Order(totalCost: Double, quantity: Double)
```

如何不修改add方法的前提下做到这一点，开动脑筋吧！

代码见[示例](https://github.com/godpan/scala-with-cats-demo/blob/master/chapter2/src/main/scala/adding/Add.scala#L50)