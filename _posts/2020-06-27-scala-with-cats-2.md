---
title: Scala-with-cats中文翻译（二）：初遇Cats：Show type class
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-fbdsdc9ikiju
tags:
  - scala
  - cats
  - type class
---

### 前言

本篇内容主要是了解Cats中的几个简单Type class，基本用法以及它的组织结构，了解Cats的包组织形式能让你对Cats的使用更加清晰，因为Implicit有时并不是那么直观。

> **学习程度：需要完全掌握**

### 1.4 初遇 Cats

在先前的章节我们学习了如何在Scala中去实现一个type class，在本节中我们学习Cats中已经实现的type class。

Cats是的设计是模块化，我们可以自由选择自己想要的type class、instance、interface methods。让我们来看第一个例子：[cats.Show](http://typelevel.org/cats/api/cats/Show.html)。

Show的功能跟我们在上节实现的Printable基本一致。它的主要功能就是帮助我们将数据以更友好的方式输出的控制台，而不是通过toString方法，下面是它的一个简单声明：

```scala
package cats
trait Show[A] {
  def show(value: A): String
}
```

#### 1.4.1 导入Type Class

Show这个type class声明在[cats](http://typelevel.org/cats/api/cats/)这个包里，我们可以直接进行import：

```scala
import cats.Show
```

在Cats中，每个type class的伴生对象中都有一个apply方法，用于查找对应类型的instance：

```scala
val showInt = Show.apply[Int]
// <console>:13: error: could not find implicit value for parameter
//  instance: cats.Show[Int]
// val showInt = Show.apply[Int]
```

糟糕，竟然报错了，因为apply方法是通过implicit来查找对应的instance，所以我们需要导入相应的instance到implicit scope。

#### 1.4.2 导入默认的Instances实现

 [cats.instances](https://typelevel.org/cats/api/cats/instances/)这个包提供了很多默认实现的instance，我们可以通过以下方式来导入它们，每种类型的包都包含了该类型对于Cats中所有type class的instance实现：

- [cats.instances.int](https://typelevel.org/cats/api/cats/instances/package$$int$)提供所有Int的instances
- [cats.instances.string](https://typelevel.org/cats/api/cats/instances/package$$string$)提供所有Stirng的instances
- [cats.instances.list](https://typelevel.org/cats/api/cats/instances/package$$list$)提供所有List的instances
- [cats.instances.option](https://typelevel.org/cats/api/cats/instances/package$$option$)提供所有Option的instances
- [cats.instances.all](https://typelevel.org/cats/api/cats/instances/package$$all$)提供Cats中的所有instances

有关可用导入的详细信息，请参见[cats.instances](https://typelevel.org/cats/api/cats/instances/)包。

让我们来导入Int和String对应Show的instances：

```scala
import cats.instances.int._    // for Show
import cats.instances.string._ // for Show

val showInt: Show[Int] = Show.apply[Int] 
val showString: Show[String] = Show.apply[String]
```
很好，我们导入了Int和String对应Show的instance，现在可以使用它们来打印Int和String的数据：

```scala
val intAsString: String = showInt.show(123)
// intAsString: String = 123

val stringAsString: String = showString.show("abc")
// stringAsString: String = abc
```

#### 1.4.3 导入Interface Syntax

我们可以使用*interface syntax*让Show变的更容易使用，首先我们需要先导入[cats.syntax.show](https://typelevel.org/cats/api/cats/syntax/package$$show$)，它会为任意类型添加一个show的扩展方法，前提是implicit scope已经有了对应类型的instance：

```scala
import cats.syntax.show._ // for show

val shownInt = 123.show
// shownInt: String = 123

val shownString = "abc".show
// shownString: String = abc
```

Cats为每一个type class都提供了syntax，我们可以按需使用，在后面的章节，我们会继续介绍它们。

#### 1.4.4 导入所有内容

在这本书中，我们对于每个示例都是按需导入，只导入需要的instance和syntax。然而，有些时候这也是相当费时的，你可以通过以下方式简化导入：

- import cats._  导入Cats中所有的type class
- import cats.instances.all._ 导入Cats中所有的instances
- import cats.syntax.all._ 导入Cats中所有的syntax
- import cats.implicits._  导入Cats中所有的instances和syntax

大多数时候我们只需要全部导入即可：

```scala
import cats._
import cats.implicits._
```

但当遇到命名冲突或者implicit冲突的时候，我们就需要更具体导入。

#### 1.4.5 实现一个自定义的Instance

下面我们来实现Date类型对应Show的instance：

```scala
import java.util.Date

implicit val dateShow: Show[Date] =
  new Show[Date] {
    def show(date: Date): String =
      s"${date.getTime}ms since the epoch."
}
```

同时，Cats提供了一些更简洁的方法去创建instance。对于Show来说，在其伴生对象中有两个方法帮助我们创建自定义类型的instance：

```scala
object Show {
  
  // Convert a function to a `Show` instance:
  def show[A](f: A => String): Show[A] = ???
  
  // Create a `Show` instance from a `toString` method:
  def fromToString[A]: Show[A] = ???
}
```

使用这些方法会比传统创建instance的方式更加快速：

```scala
implicit val dateShow: Show[Date] = Show.show(date => s"${date.getTime}ms since the epoch.")
```

我们可以看到，确实简洁了不少，Cats为很多type class都提供了类似的辅助方法来创建instance，可以从头直接创建instance，也可以基于其他类型的instance创建新的instance，比如：基于Int类型的instance创建Option[Int]类型的instance。

#### 1.4.6 练习: 使用Cat Show

使用Show type class重写上面章节Printable的例子，代码见[示例](https://github.com/godpan/scala-with-cats-demo/tree/master/chapter1)


