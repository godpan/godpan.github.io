---
title: Scala-with-cats中文翻译（六）：Semigroup定义以及基于Boolean与Set类型的Monoid实现
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-yhgtdbgt7783
tags:
  - scala
  - cats
  - type class
---

### 前言

本小节我们来看一下Semigroup的定义，然后基于Boolean与Set类型，实现一些Monoid。

> **学习程度：需要完全掌握**

#### 2.2 声明一个Semigroup

如何仅仅满足结合律，但是不存在幺元的结构我们称之为Semigroup。虽然很多时候Semigroup也可能是一个Monoid，但还是有一些类型我们无法定义它的empty值，也就说这种情况下Semigroup无法变成一个Monoid。举个例子，前面我们所看到的序列拼接以及整数加法都是Monoid，但比如现在我们的取值范围限制在不可用序列以及正整数上，我们将无法为它们声明一个empty值。Cats中就有一个非空列表[NonEmptyList](https://typelevel.org/cats/api/cats/data/NonEmptyList.html)，所以它仅仅实现了Semigroup接口而不是Monoid接口。

所以，在Cats中Monoid更加准确的定义是这样的：

```scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}

trait Monoid[A] extends Semigroup[A] {
  def empty: A
}
```

接下来我们将看到很多type class继承的例子，它让代码组织更加模块化以及拥有更好的复用性。假如现在我们为A类型声明了一个Monoid instance，我们能同时也能得到一个Semigroup instance，类似的，一个方法如果需要Semigroup[B]类型的参数，我们都可以用Monoid[B]的值代替。

#### 2.3 练习: Boolean类型的Monoid

之前我们只是看了Monoid的一些例子，比如整数加法，整数乘法等，其实还有很多Monoid等待我们发现，考虑一下Boolean类型，对于这个类型你能声明多少种Monoid？每个Monoid都有一个combine方法以及一个empty值，并且需要确保满足之前讲的两个法则，即结合律和存在幺元。我们先来声明基础的结构：

```scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}

trait Monoid[A] extends Semigroup[A] {
  def empty: A
}

object Monoid {
  def apply[A](implicit monoid: Monoid[A]) = monoid 
}
```

Boolean类型总共有4种Monoid，我们先来看两个比较熟悉和简单的：**&& 和 ||**：
```scala

class BooleanAndMonoid extends Monoid[Boolean] {
  override def empty: Boolean = true

  override def combine(x: Boolean, y: Boolean): Boolean = x && y
}

class BooleanOrMonoid extends Monoid[Boolean] {
  override def empty: Boolean = false

  override def combine(x: Boolean, y: Boolean): Boolean = x || y
}
```
接着我们来看两个不常见的操作：

```scala
/**
* 这个操作是指a与b同时为true或者false，返回fasle，否则返回true
*/
class BooleanEitherMonoid extends Monoid[Boolean] {
  override def empty: Boolean = false

  override def combine(x: Boolean, y: Boolean): Boolean =  (x && !y) || (!x && y)
}

/**
* 这个操作是指a与b同时为true或者false，返回true，否则返回false
*/
class BooleanXnorMonoid extends Monoid[Boolean] {
  override def empty: Boolean = false

  override def combine(x: Boolean, y: Boolean): Boolean =  (!x || y) && (x || !y)
}
```
前面我们说过Monoid需要同时满足封闭性以及结合律，大家有兴趣可以去验证，这里我们对**BooleanAndMonoid**来进行一下验证：
```scala
implicit val booleanAndMonoid = new BooleanAndMonoid

println(Monoid.identityLaw(true)) //expect true
println(Monoid.identityLaw(false)) //expect true

val data = Seq((true, true, true), (true, true, false), (true, false, true), (true, false, false), (false, true, true), (false, true, false), (false, false, true), (false, false, false))

println(data.map(d => Monoid.associativeLaw(d._1, d._2,d._3)).exists(_ == false) == false) //expect true
```
代码详情见[示例](https://github.com/godpan/scala-with-cats-demo/blob/master/chapter2/src/main/scala/monoid/BooleanMonoid.scala)

#### 2.4 练习: Set类型的Monoid**

对于Set类型来说，它可以有哪些Monoid和Semigroup结构？

Set类型有两个Monoid和一个Semigroup结构，我们先来看一下熟悉Set union操作，也可以看成 ++ 操作：
```scala
class SetUnionMonoid[A] extends Monoid[Set[A]] {
  override def empty: Set[A] = Set.empty[A]

  override def combine(x: Set[A], y: Set[A]): Set[A] = x union y
}
```
另一个Monoid是Set先进行diff，然后在union操作，简单理解就是元素都不在两个Set中的集合：
```scala
class SymDiffMonoid[A] extends Monoid[Set[A]] {
  override def empty: Set[A] = Set.empty[A]

  override def combine(x: Set[A], y: Set[A]): Set[A] = (x diff y) union (y diff x)
}
```
另外Set还有一个Semigroup，即Set的intersect操作，主要是取出两个Set公有的元素：
```
class SetIntersectionSemigroup[A] extends Semigroup[Set[A]] {
  override def combine(x: Set[A], y: Set[A]): Set[A] = x intersect y
}
```
因为我们无法为intersect找到一个empty的值满足identityLaw，所以它只能算是一个Semigroup。

最后我们来验证其中一个Monoid：
```scala
implicit val setUnionMonoid = new SetUnionMonoid[Int]

val setA = Set(1, 2, 3)
val setB = Set(2, 4, 5)
val setC = Set(3, 4, 8)

println(Monoid.identityLaw(setA)) //expect true

println(Monoid.associativeLaw(setA, setB, setC)) //expect true
```

代码详情见[示例](https://github.com/godpan/scala-with-cats-demo/blob/master/chapter2/src/main/scala/monoid/SetMonoid.scala)