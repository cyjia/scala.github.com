---
layout: overview-large
title: 类型标记和清单

disqus: true

partof: reflection
num: 5
outof: 7
language: cn
---

和其他JVM语言一样，Scala的类型信息也会擦除。这意味着，如果你想检查某些实例的运行时类型，你可能得不到Scala编译器在编译器能获得的所有信息。

类似`scala.reflect.Manifest`，可以把`TypeTags`认为是把编译器所有信息携带到运行时的对象。例如，`TypeTag[T]`封装了一些编译期类型`T`的运行时类型表示。不过要注意，应该仅仅把`TypeTag`当作2.10之前那个`Manifest`的更完善的替代，`Manifest`附加整合到Scala反射。

有三种不同的类型标记：

1. `scala.reflect.api.TypeTags#TypeTag`.
Scala类型的完整类型描述。例如，`TypeTag[List[String]]`包含类型`scala.List[String]`的所有类型信息。

2. `scala.reflect.ClassTag`.
Scala类型的不完全类型描述。例如，`ClassTag[List[String]]`仅包含类型`scala.collection.immutable.List`被擦除的类型信息。`ClassTag`仅提供对类型运行时类的访问，与`scala.reflect.ClassManifest`类似。

3. `scala.reflect.api.TypeTags#WeakTypeTag`.
抽象类型的类型描述符（见下面相关章节）。

## 获得`类型标记`

像`Manifest`一样，`类型标记`由编译器生成，有三种获取方式。

### 通过方法`typeTag`，`classTag`，或`weakTypeTag`。

可以简单地用`Universe`的`typeTag`方法直接获取某类型的`类型标记`。

例如，获取代表`Int`的`类型标记`，我们可以：

    import scala.reflect.runtime.universe._
    val tt = typeTag[Int]

类似的，获取代表`String`的`类标记`，我们可以：

    import scala.reflect._
    val ct = classTag[String]

这些方法构造给定类型参数`T`的`TypeTag[T]`或`ClassTag[T]`。

### 用类型`TypeTag[T]`,`ClassTag[T]`或`WeakTypeTag[T]`的隐含参数

和`Manifest`一样，可以达到*请求*编译器生成`类型标记`的效果。直接指定类型`TypeTag[T]`的隐含*证据*参数。如果在隐含查找阶段编译器不能找到匹配的隐含值，编译器就会自动生成`TypeTag[T]`。

_注意_: 通常仅在方法和类上使用隐含参数以达到此效果。

例如，我们可以写个方法接受某个对象，用`TypeTag`打印对象的类型参数信息：

    import scala.reflect.runtime.universe._

    def paramInfo[T](x: T)(implicit tag: TypeTag[T]): Unit = {
      val targs = tag.tpe match { case TypeRef(_, _, args) => args }
      println(s"type of $x has type arguments $targs")
    }

这里，我们写了一个泛型方法`paramInfo`, `T`是泛型参数，还提供了隐含参数`(implicit tag: TypeTag[T])`。我们接着直接用`TypeTag`的`tpe`方法访问`tag`表示的类型（类型为`Type`）。

然后我们就可以像下面这样用方法`paramInfo`：

    scala> paramInfo(42)
    type of 42 has type arguments List()

    scala> paramInfo(List(1, 2))
    type of List(1, 2) has type arguments List(Int)

### 用类型参数的上下文边界

实现上面更简洁的做法是用类型参数的上下文边界。直接像下面这样在类型参数中包含`类型标记`，而不是提供单独的隐含参数：

    def myMethod[T: TypeTag] = ...

给定类型边界`[T: TypeTag]`，编译器会直接生成类型为`TypeTag[T]`的隐含参数并且把方法改写成如上节那样带隐含参数。

上例重写为用上下文边界后如下：

    import scala.reflect.runtime.universe._

    def paramInfo[T: TypeTag](x: T): Unit = {
      val targs = typeOf[T] match { case TypeRef(_, _, args) => args }
      println(s"type of $x has type arguments $targs")
    }

    scala> paramInfo(42)
    type of 42 has type arguments List()

    scala> paramInfo(List(1, 2))
    type of List(1, 2) has type arguments List(Int)

## 弱类型标记

`WeakTypeTag[T]` 抽象化`TypeTag[T]`。与普通的`TypeTag`不同，它的类型表示可以是类型参数或抽象类型。不过，`WeakTypeTag[T]`试图尽可能地具体化，*例如,*如果引用类型参数或抽象类型的类型标记可用，他们就把具体类型嵌入到`WeakTypeTag[T]`。

继续上面的例子：

    def weakParamInfo[T](x: T)(implicit tag: WeakTypeTag[T]): Unit = {
      val targs = tag.tpe match { case TypeRef(_, _, args) => args }
      println(s"type of $x has type arguments $targs")
    }

    scala> def foo[T] = weakParamInfo(List[T]())
    foo: [T]=> Unit

    scala> foo[Int]
    type of List() has type arguments List(T)

## 类型标记和清单

与`类型标记`有点类似的是2.10之前的概念`scala.reflect.Manifest`。`scala.reflect.ClassTag`对应`scala.reflect.ClassManifest`，`scala.reflect.api.TypeTags#TypeTag`对应`scala.reflect.Manifest`，其他2.10之前的清单类型没有直接对应的"`标记`"类型。

- **不支持scala.reflect.OptManifest.**
因为`标记`能具体化任意类型，所以总有标记。

- **没有对等的scala.reflect.AnyValManifest.**
但是，可以和基础`标记`(在对应的伙伴对象中定义)比较以得知`标记`是否代表原生值类。此外，还可以直接用`<tag>.tpe.typeSymbol.isPrimitiveValueClass`。

- **没有与清单对等的对象提供工厂方法。**
但是，可以用Java(用于类)和Scala(用于类型)提供的反射API生成对应的类型。

- **有些清单操作（例如，`<:<`,`>:>`和`typeArguments`）不支持。

Scala2.10中，`scala.reflect.ClassManifests`被废弃，还计划在即将退出的版本中废弃`scala.reflect.Manifest`改用`TypeTag`和`ClassTag`。
