---
title: Scala-with-cats中文翻译（五）：Monoid的定义
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-tyhjuki889d
tags:
  - scala
  - cats
  - type class
---

### 前言

在本章中，我们将学习两个type class：**monoid**和**semigroup**。他们的主要功能是对值进行相加或者组合，并且很多基础类型都有对应这两个type class的实现，比如Int，String，List，Option等，接下来让我们先了解一些简单的类型和操作，尝试理解其背后原理。

> **学习程度：需要完全掌握**

##### Int类型的加法

Int的加法是满足**封闭性**的，也就是说两个Int相加会生一个新的Int：

```scala
 2+1
// res0: Int = 3
```

同时还存在一个**“幺元0”**满足a + 0 == 0 + a == a：

```scala
2+0
// res1: Int = 2
0+2
// res2: Int = 2
```

另外它也满足**结合律**：

```scala
(1 + 2) + 3
// res3: Int = 6
1 + (2 + 3)
// res4: Int = 6
```

**Int类型的乘法**

Int的乘法同样满足上面我们描述加法的三个特性，只不过它的幺元由0变为了1：

```scala
1*3
// res5: Int = 3
3*1
// res6: Int = 3
```

也满足**结合律**：

```scala
(1 * 2) * 3
// res7: Int = 6

1 * (2 * 3)
// res8: Int = 6
```

##### 字符串与序列的拼接

String类型同样也可以相加，可以使用字符串拼接作为一个操作符：

```scala
 "One" ++ "two"
// res9: String = Onetwo
```

幺元为一个空字符串:

```scala
"" ++ "Hello"
// res10: String = Hello

"Hello" ++ ""
// res11: String = Hello
```

同样，拼接也满足结合律：

```scala
("One" ++ "Two") ++ "Three"
// res12: String = OneTwoThree

"One" ++ ("Two" ++ "Three")
// res13: String = OneTwoThree
```

注意，这里我们是使用++来进行序列拼接，而不是+，除了字符串（可以看出Char类型的sequence）以外，我们还可以去其他类型的sequence执行相同的操作，拼接作为操作符，空sequence为单位元。

#### 2.1 声明一个 Monoid

现在我们已经看过好几个**“相加”**的例子，它们都有一个添加的操作符和一个单位元，其实这就是Monoid，对于类型A来说：

- 有一个操作满足(A,A)=>A
- A类型有一个empty的值

这个定义用Scala表示非常容易，我们来看一下在Cats中的一个简单定义：

```scala
trait Monoid[A] {
  def combine(x: A, y: A): A
  def empty: A
}
```

monoid除了拥有这两个最基本的定义外，还需要满足一些法则，就是我们前面提到的“**结合律**”和“**幺元**”，对于A类型中的值x，y，z，必须满足以下两个函数：

```scala
def associativeLaw[A](x: A, y: A, z: A)
      (implicit m: Monoid[A]): Boolean = {
  m.combine(x, m.combine(y, z)) ==
    m.combine(m.combine(x, y), z)
}

def identityLaw[A](x: A)
      (implicit m: Monoid[A]): Boolean = {
  (m.combine(x, m.empty) == x) &&
    (m.combine(m.empty, x) == x)
}
```

很显然，整数减法不是一个Monoid，因为它不满足**结合律**：

```scala
(1 - 2) - 3
// res15: Int = -4

1 - (2 - 3)
// res16: Int = 2
```

如果自己去实现一个Monoid instance的话，一定要遵循对应的法则，非法的instance非常危险，因为可能出现不可预测的结果。大多数时候我们只需要使用Cats中已经实现的instance，当然你最好去看一下它们的源码，了解一下它们背后的实现。