---
title: Scala-with-cats中文翻译（一）：Type class与Implicit
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-fbdsdc7yu89i
tags:
  - scala
  - cats
  - type class
---

### 前言

最近在学习Cats，发现Scala-with-cats这本书写的不错，所以有想法将其翻译成中文，另外也可以在翻译的过程中加深理解，另外我会对每部分内容建议需要了解的程度，帮助大家更好的学习整体内容（有部分内容理解起来比较晦涩且不常用，了解即可），同时我也将相关的练习代码放到github上了，大家可下载参考：[scala-with-cats](https://github.com/godpan/scala-with-cats-demo)，有翻译不准确的地方，也希望大家能指正🙏。

本篇内容主要为Type class与Implicit，这应该算是学习Cats需要了解的最基础的内容。

> **学习程度：需要完全掌握**

### 1.1 剖析Type class

Type class模式主要由3个模块组成：

- Type class 本身
- Type class Instances
- Type class interface

#### 1.1.1 Type Class

Type class 可以看成一个接口或者 API，用于定义我们想要实现功能。在 Cats 中，Type class相当于至少带有一个类型参数的 trait。比如以下定义代表将一个值转换为Json的行为：

```scala
// Define a very simple JSON AST 声明一些简单的JSON AST
sealed trait Json
final case class JsObject(get: Map[String, Json]) extends Json 
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json
case object JsNull extends Json

// The "serialize to JSON" behaviour is encoded in this trait 序列话JSON方法定义在这个Trait里
trait JsonWriter[A] {
  def write(value: A): Json
}
```

 这个例子中 `JsonWriter` 就是我们定义的一个 Type class，上述代码中还包含Json类型相关的代码。

#### 1.1.2 Type Class Instances

Type Class instance 就是特定类型的 Type Class实现，包括Scala的基本类型以及我们自己定义的类型。

在Scala中，Type Class  instance可以通过实现对应类型Type Class来声明，并用 **implicit** 这个关键词进行标记：

```scala
final case class Person(name: String, email: String)
object JsonWriterInstances {
  implicit val stringWriter: JsonWriter[String] =
    new JsonWriter[String] {
      def write(value: String): Json =
        JsString(value)
    }
  implicit val personWriter: JsonWriter[Person] =
    new JsonWriter[Person] {
      def write(value: Person): Json =
        JsObject(Map(
          "name" -> JsString(value.name),
          "email" -> JsString(value.email)
        ))
    }
// etc...
}
```

#### 1.1.3 Type Class Interfaces

Type Class  Interface 包含对我们想要对外部暴露的功能。interfaces是指接受 type class instance 作为 `implicit` 参数的泛型方法。

通常有两种方式去创建 interface：

- Interface Objects
- Interface Syntax

##### Interface Objects用法

创建 interface 最简单的方式就是将方法放在一个单例object中：

```scala
object Json {
  def toJson[A](value: A)(implicit w: JsonWriter[A]): Json = w.write(value)
}
```

在使用之前，我们需要导入我们所需的 type class instances，然后就可以调用相关的方法：

```scala
import JsonWriterInstances._
Json.toJson(Person("Dave", "dave@example.com"))
// res4: Json = JsObject(Map(name -> JsString(Dave), email -> JsString(dave@example.com)))
```

这里我们并没有指定对应的 implicit parameters，但是编译器会帮我们在导入的 type class instances 中寻找一个跟相应类型匹配的 type class instance，并插入对应的位置：

```scala
Json.toJson(Person("Dave", "dave@example.com"))(personWriter)
```

##### Interface Syntax用法

我们也可以使用扩展方法使已存在的类型拥有 interface methods，在 Cats 中将此称为 “*syntax*”：

```scala
object JsonSyntax {
  implicit class JsonWriterOps[A](value: A) {
    def toJson(implicit w: JsonWriter[A]): Json =
      w.write(value)
  } 
}
```

使用 interface syntax 之前，我们除了导入它本身以外，还需导入我们所需的 type class instance：

```scala
import JsonWriterInstances._
import JsonSyntax._
Person("Dave", "dave@example.com").toJson
// res6: Json = JsObject(Map(name -> JsString(Dave), email -> JsString(dave@example.com)))
```

 同样，编译器会自动帮我们寻找所需implicit parameters并插入对应的位置：

```scala
Person("Dave", "dave@example.com").toJson(personWriter)
```

##### 使用implicitly

Scala 标准库提供了一个泛型的 type class interface 叫做 implicitly，它的声明非常简单：

```scala
 def implicitly[A](implicit value: A): A = value
```

它接收一个 implicit 参数并返回该参数，我们可以使用 implicitly 调用 implicit scope 中的任意值，只需要指定对应的类型无需其他操作，便能得到对应的 instance 对象。

```scala
import JsonWriterInstances._
// import JsonWriterInstances._
implicitly[JsonWriter[String]]
// res8: JsonWriter[String] = JsonWriterInstances$$anon$1@38563298
```

在 Cats 中，大多数 type class 都提供了其他方式去调用对应的 instance。但是在代码调试过程中，implicitly 有着很大的用处。我们可以在代码中插入implicitly 相关代码，来确保编译器能找到对应的 type class instance（若无对应的 type class instance 则编译的时候会抱错）以及不会出现歧义性（比如 implicit scope 存在两个相同的 type class instance）。

### 1.2 使用 Implicits

对于 Scala 来说，使用 type class 就得跟 implicit values 和 implicit parameters 打交道，为了更好的使用它，我们需要了解以下几个点。

#### 1.2.1 组织 Implicits

奇怪的是,在Scala中任何标记为implicit的定义都必须放在object或trait中，而不是放在顶层。在上一小节的例子中，我们将所有的type class instances打包放在JsonWriterInstances中。同样我们也可以把它放在JsonWriter的伴生对象中，这种方式在Scala中有特殊的含义，因为这些instances会直接在*implicit scope*里面，无需单独导入。

#### 1.2.2 Implicit 作用域

正如我们看到的一样，编译器会自动寻找对应类型的type class instances，举个例子，下面这个例子就会编译器就会自动寻找**JsonWriter[String]**对应的instance：

```scala
Json.toJson("A string!")
```

编译器会从以下几个*implicit scope*中寻找适合的instance：

- 自身及继承范围内的 instance
- 导入范围内的 instance
- 对应 type class 以及参数类型的伴生对象中

只有用 implicit 关键词标注的instance才会在 *implicit scope*，而且如果编译器在引入的 implicit scope 中发现重复的 instance 声明，则会编译抱错：

```scala
implicit val writer1: JsonWriter[String] =
  JsonWriterInstances.stringWriter
implicit val writer2: JsonWriter[String] =
  JsonWriterInstances.stringWriter
Json.toJson("A string")

// <console>:23: error: ambiguous implicit values:
// both value stringWriter in object JsonWriterInstances of type => JsonWriter[String]
//  and value writer1 of type => JsonWriter[String]
// match expected type JsonWriter[String] 
// Json.toJson("A string")
//
```

但 Scala 中的 *implicit* 规则远比这复杂的多，但这些不在本书的讨论范围之内（如果你想对 *implicit* 有更深入的了解，可以参考这些内容：[this Stack Overflow post on implicit scope](https://stackoverflow.com/questions/5598085/where-does-scala-look-for-implicits)和[this blog post on implicit priority](http://eed3si9n.com/revisiting-implicits-without-import-tax)）。对于我们来说，通常把type class instances放在以下四个地方：

1. 一个单独的object中，比如上面提到的JsonWriterInstances；
2. 一个单独的trait中；
3. type class的伴生对象中；
4. 我们所使用类型的伴生对象中，比如JsonWriter[A]，即A的伴生对象中；

如果是第一种方式的，我们在使用之前通过import导入，第二种方式的话通过继承trait引入，另外两种方式的，无需单独导入，它们默认就在对应类型的implicit scope中。

#### 1.2.3 递归寻找Implicit

编译器除了能直接寻找对应类型type class instance，还拥有组合type class instance的能力。

之前我们都是通过 implicit val来声明type class instances ，这非常简单，实际上我们有两种方式去声明instances：

1. 通过 implicit val来声明具体类型的type class instances；
2. 利用 implicit methods通过其他类型的type class instances来生成新的instances；

我们为什么要通过其他类型的type class instances来生成新的instances呢？一个很明显的例子，我们如何让Option类型可以应用JsonWriter这个type class。对于系统中的任意类型的Option[A]，都得需要有对应的type class instance，我们可能会尝试通过声明所有instance：

```scala
implicit val optionIntWriter: JsonWriter[Option[Int]] = ???
implicit val optionPersonWriter: JsonWriter[Option[Person]] = ???
// and so on...
```

显然，这种方式是不易扩展的，对于系统中的任意类型A，我们都必须去声明两个instance，一个作用于A，一个作用于Option[A]。

幸运的是，我们可以基于A的instance来构造Option[A]的instance，而且这是一个通用逻辑：

- 假如option是Some(a: A)，则使用A的instance；
- 假如option是None，则返回JsNull；

我们通过implicit def来实现：

```scala
implicit def optionWriter[A](implicit writer: JsonWriter[A]): JsonWriter[Option[A]] =
  new JsonWriter[Option[A]] {
    def write(option: Option[A]): Json =
      option match {
        case Some(aValue) => writer.write(aValue)
        case None         => JsNull
		} 
 }
```

这个方法包含一个implicit参数writer，并通过它来构造一个Option[A]的JsonWriter instance。我们来看一个表达式：

```scala
Json.toJson(Option("A string"))
```

编译器首先会去寻找对应的type class instance，这里是optionWriter[String]，所以为表达式加上对应的implicit参数：

```scala
Json.toJson(Option("A string"))(optionWriter[String])
```

因为这里optionWriter是用implicit def声明的，而且需要一个implicit writer: JsonWriter[A]参数，所以编译器会继续寻找，这里的对应instance是stringWriter，最终完整的表达式：

```scala
Json.toJson(Option("A string"))(optionWriter(stringWriter))
```

通过这种方式，编译器会在引入的implicit scope中竟可能的寻找符合的instance，最终组合成所需要类型的type class instance。

> *Implicit Conversions*
>
> 在我们使用implicit def构建type class instance的时候，我们使用implicit参数，如果我们不使用implicit声明参数，编译器则不会自动去寻找填充参数。
>
> 使用implicit方法但是不使用implicit parameters在Scala中是另一种模式，叫做*implicit conversion*。跟之前内容中提到的Interface Syntax也是不同的，它是一个implicit class并使用扩展方法。implicit conversion是一种古老的编程模式，目前Scala已经不赞成使用了。而且当你使用该语法时，编译器会提出警告，如果你确定要使用，则需手动引入scala.language.implicitConversions：
>
> ```scala
> implicit def optionWriter[A]
> (writer: JsonWriter[A]): JsonWriter[Option[A]] =
> ???
> // <console>:18: warning: implicit conversion method optionWriter should be enabled
> // by making the implicit value scala.language.implicitConversions visible.
> // This can be achieved by adding the import clause 'import scala.language.implicitConversions'
> // or by setting the compiler option -language: implicitConversions.
> // See the Scaladoc for value scala.language.implicitConversions for a discussion
> // why the feature should be explicitly enabled.
> //
> //     implicit def optionWriter[A]
>                  ^
> // error: No warnings can be incurred under -Xfatal-warnings.
> ```
>
> 

### 1.3 练习: 实现一个Printable

Scala可以通过toString方法将一个任意一个值转换成String。但是这种方式有一些缺陷：

- 它对Scala中的每个类型都进行了实现，但是使用有很大限制；
- 不能对特定类型进行特定的实现；

让我们声明一个Printable type class去解决这些问题吧：

1. 声明一个type class Printable[A]包含一个方法format，该方法接受一个类型为A的参数并返回String。
2. 创建一个名为PrintableInstances的object，包含Printable[String]和Printable[Int]的instance声明。
3. 创建一个名为Printable的object，包含两个泛型方法：
   1. format方法：接受一个类型为A的参数和相关类型的Printable，使用Printable将参数转换为String。
   2. print方法：与format方法参数一致，但返回值时Unit，它执行的操作是通过println将类型为A的参数输出到控制台。

代码见[示例](https://github.com/godpan/scala-with-cats-demo/tree/master/chapter1/src/main/scala/A1)

#### 1.3.1 使用Printable

我们可以把Printable这个功能封装成类库，然后在使用的地方引入，我们先来定义一个case class：

```scala
final case class Cat(name: String, age: Int, color: String)
```

接下来我们实现一个Printable[Cat]类型的instance，对应format的返回结果应为：

```scala
NAME is a AGE year-old COLOR cat.
```

 代码见[示例](https://github.com/godpan/scala-with-cats-demo/tree/master/chapter1/src/main/scala/A1)

#### 1.3.2 更好的 Syntax语法

我们将使用前面介绍的**Interface Syntax**的语法，让Printable相关的功能更容易使用：

1. 创建一个PrintableSyntax的object。
2. 在PrintableSyntax中声明一个implicit class PrintableOps[A]对A类型的值进行包装。
3. 在PrintableOps[A]声明两个方法：
   - format接受一个implicit Printable[A]的参数，返回String；
   - print接受一个implicit Printable[A]的参数，返回Unit；
4. 使用扩展方法对上一个例子进行不一样实现；

代码见[示例](https://github.com/godpan/scala-with-cats-demo/tree/master/chapter1/src/main/scala/A1)

