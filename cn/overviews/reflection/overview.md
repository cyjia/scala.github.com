---
layout: overview-large
title: 概要

partof: reflection
num: 1
outof: 7
language: cn
---

<span class="label important" style="float: right;">试验中</span>

**希瑟米勒, 尤金博麦寇, 菲利普哈勒**

*反射*是程序检查乃至自我修改的一种能力。反射在面向对象、函数式和逻辑等编程模式中都有悠久的历史。大多数编程语言是在进化过程中逐渐发展出反射的能力，但也有些编程语言是围绕着反射这个指导原则构建的。

反射意味着一种使程序中的隐含元素具体化的能力。这些元素可能是类、方法、表达式之类的静态元素，也可能是诸如方法调用、字段访问等当前执行时或当前执行事件之类的动态元素。人们通常按照处理反射的时机把反射分为编译期反射和运行时反射。**编译期反射**用于开发强大的程序转换器和生成器，而**运行时反射**通常用于适配语义或支持软件组件间的很后期的绑定。

在2.10之前，Scala没有任何自己的反射能力。相反，人们可以使用部分来自Java的反射编程接口，这些接口提供动态检查类、对象以及访问类、对象内部成员的能力。然而，单独的Java反射只能暴露Java的元素和类型，不能复原Scala特有的元素（比如，函数、特征）和类型（比如，存在性existential、higher-kinded、路径依赖和抽象类型）。此外，Java反射也不能复原编译期是泛型的那些Java类型在运行时的类型信息（Scala在泛型类型上加的要传递到运行时的类型限定条件）。

为了弥补Java运行时反射的缺陷并给Scala增加一套更加强大的具备通用反射能力的工具，在Scala2.10中引入了一个新的反射库。 除了支持Scala类型和泛型的全功能运行时反射，Scala2.10还以[宏]({{site.baseurl }}/overviews/macros/overview.html)的形式提供了编译期的反射能力以及*转换*Scala表达式为抽象语法树的能力。

## 运行时反射

什么是运行时反射? 对于一个**运行时**的类型或实例对象，反射是指如下能力:

- 检查对象类型（包括泛型类型）,
- 实例化新对象,
- 访问或调用对象的成员。

让我们通过一些例子来看看如何实现上述每一种能力。

### 例子

#### 检查对象类型（包括泛型类型）

同其他基于Java虚拟机的语言一样，Scala的类型信息也会在编译期被*擦除*。这意味着当你想检查实例的运行时类型信息时，你可能得不到Scala编译器在编译期间具有的所有类型信息。

我们可以把`类型标记`看作是把类型的编译期信息传递到运行时的对象。不过，重要的是要注意`类型标记`是由编译器生成的。每当隐含参数或上下文边界需要`类型标记`时都会触发生成过程。也就是说，通常只能通过使用隐含参数或上下午边界才能获得`类型标记`。

例如, 使用上下文边界:

    scala> import scala.reflect.runtime.{universe => ru}
    import scala.reflect.runtime.{universe=>ru}

    scala> val l = List(1,2,3)
    l: List[Int] = List(1, 2, 3)

    scala> def getTypeTag[T: ru.TypeTag](obj: T) = ru.typeTag[T]
    getTypeTag: [T](obj: T)(implicit evidence$1: ru.TypeTag[T])ru.TypeTag[T]

    scala> val theType = getTypeTag(l).tpe
    theType: ru.Type = List[Int]

上例中, 我们首先引入 `scala.reflect.runtime.universe`（必须引入它以使用`类型标记`），我们还创建了一个名为`l`的`List[Int]` 
。接着，我们定义了一个方法`getTypeTag`，这个方法有一个具有上下文边界的类型参数`T`（正如REPL显示的那样，这等同于定义了一个隐含的"evidence"参数，这个隐含参数会导致编译器生成`T`的`类型标记`）。最后，我们以`l`为参数调用方法并调用返回值的`tpe`方法，`tpe`方法返回`类型标记`中的类型信息。不难发现，我们得到了正确的完整信息（包括`List`的实际类型参数）`List[Int]`。

当我们获得了期望的`类型`实例，我们就可以检查它，比如:

    scala> val decls = theType.declarations.take(10)
    decls: Iterable[ru.Symbol] = List(constructor List, method companion, method isEmpty, method head, method tail, method ::, method :::, method reverse_:::, method mapConserve, method ++)

#### 运行时实例化一个类型

对于通过反射获得的类型，我们可以通过使用合适的"调用"映射（映射的进一步信息在[下面]({{ site.baseurl }}/cn/overviews/reflection/overview.html#mirrors)）来调用他们的构造器从而进行实例化。我们看一个用REPL展示的例子:

    scala> case class Person(name: String)
    defined class Person

    scala> val m = ru.runtimeMirror(getClass.getClassLoader)
    m: reflect.runtime.universe.Mirror = JavaMirror with ...

第一步，我们获得映射`m`，通过`m`可以获得当前类加载器已经加载的所有类和类型（包括`Person`）。

    scala> val classPerson = ru.typeOf[Person].typeSymbol.asClass
    classPerson: reflect.runtime.universe.ClassSymbol = class Person

    scala> val cm = m.reflectClass(classPerson)
    cm: reflect.runtime.universe.ClassMirror = class mirror for Person (bound to null)

第二步，通过`reflectClass`方法获得`Person`类的`ClassMirror`。`ClassMirror`提供了获得`Person`类构造函数的访问方法。

    scala> val ctor = ru.typeOf[Person].declaration(ru.nme.CONSTRUCTOR).asMethod
    ctor: reflect.runtime.universe.MethodSymbol = constructor Person

`Person`的构造方法只能通过运行时宇`ru`在类型`Person`的声明中查找。

    scala> val ctorm = cm.reflectConstructor(ctor)
    ctorm: reflect.runtime.universe.MethodMirror = constructor mirror for Person.<init>(name: String): Person (bound to null)

    scala> val p = ctorm("Mike")
    p: Any = Person(Mike)

#### 访问和调用运行时类型的成员

大体上，我们使用合适的"调用"映射（映射的进一步信息在[下面]({{ site.baseurl }}/cn/overviews/reflection/overview.html#mirrors)）来访问运行时类型的成员。我们看一个用REPL展示的例子:

    scala> case class Purchase(name: String, orderNumber: Int, var shipped: Boolean)
    defined class Purchase

    scala> val p = Purchase("Jeff Lebowski", 23819, false)
    p: Purchase = Purchase(Jeff Lebowski,23819,false)

在这个例子中，我们将尝试用反射的方式读和写`Purchase``p`的`shipped`字段。.

    scala> import scala.reflect.runtime.{universe => ru}
    import scala.reflect.runtime.{universe=>ru}

    scala> val m = ru.runtimeMirror(p.getClass.getClassLoader)
    m: reflect.runtime.universe.Mirror = JavaMirror with ...

和前一个例子一样，我们首先获取映射`m`，通过`m`能获取加载`p`的类（`Purchase`）类加载器所加载的所有类和类型，我们需要这些信息才能访问成员`shipped`。

    scala> val shippingTermSymb = ru.typeOf[Purchase].declaration(ru.newTermName("shipped")).asTerm
    shippingTermSymb: reflect.runtime.universe.TermSymbol = method shipped

现在我们看一下字段`shipped`的声明，这个声明给出一个`TermSymbol`（`Symbol`的一种）。后面，我们将需要用这个`Symbol`去获取能访问这个字段（针对一些实例）的映射。

    scala> val im = m.reflect(p)
    im: reflect.runtime.universe.InstanceMirror = instance mirror for Purchase(Jeff Lebowski,23819,false)

    scala> val shippingFieldMirror = im.reflectField(shippingTermSymb)
    shippingFieldMirror: reflect.runtime.universe.FieldMirror = field mirror for Purchase.shipped (bound to Purchase(Jeff Lebowski,23819,false))

要访问这个实例的`shipped`成员，我们需要这个实例的映射，`p`这个实例的映射是`im`。有了实例的映射，我们就能从任意`TermSymbol`获取表示`p`所在类型的字段的`FieldMirror`。

有了这个字段的`FieldMirror`，我们就能用`get`和`set`方法去读、写这个实例的`shipped`成员。我们来把`shipped`的状态修改为`true`。

    scala> shippingFieldMirror.get
    res7: Any = false

    scala> shippingFieldMirror.set(true)

    scala> shippingFieldMirror.get
    res9: Any = true

### Java的运行时类和Scala的运行时类型

一些习惯了用Java反射在运行时获取Java*Class*实例的人可能已经注意到，在Scala中我使用了运行时的*type*。

下面的REPL运行结果是一个非常简单的例子，展示了把Java反射用于Scala类可能会得到一些奇怪的或不正确的结果。

首先，我们定义一个具有抽象类型成员`T`的基类`E`，然后派生两个子类`C`、`D`。

    scala> class E {
         |   type T
         |   val x: Option[T] = None
         | }
    defined class E

    scala> class C extends E
    defined class C

    scala> class D extends C
    defined class D

然后，我们为`C`和`D`各创建一个实例，同时使成员的类型`T`具体化（都用`String`）。

    scala> val c = new C { type T = String }
    c: C{type T = String} = $anon$1@7113bc51

    scala> val d = new D { type T = String }
    d: D{type T = String} = $anon$1@46364879

现在，我们用Java反射的`getClass`和`isAssignableFrom`方法获取代表`c`和`d`的类的`java.lang.Class`实例，然后我们测试`d`的运行时类是`c`的运行时类的子类。

    scala> c.getClass.isAssignableFrom(d.getClass)
    res6: Boolean = false

因为从上面我们已经看到`D`继承`C`，所以这个结果有点令人费解。对于这个简单的运行时类型检查，大家可能期望问题"`d`的类是`c`的类的子类吗？"的答案是`true`。不过，你可能也注意到了，上面实例化`c`和`d`时，Scala编译器实际上创建了`C`
和`D`各自的匿名类。这是因为Scala编译器必须把Scala特有的（*例如*非Java的）语言特征转换成对等的Java字节码才能使之运行于JVM之上。所以，Scala编译器常常创建用于运行时的合成类（例如，自动生成的类）来代替用户自定义类。这在Scala中很常见，也见于使用Java反射处理Scala的特性，*比如*，闭包、类型成员、类型细化和局部类等等。

在诸如此类的场景，我们可以使用Scala反射去获取这些Scala对象的精确的运行时*类型*。Scala运行时类型携带了所有编译期的类型信息以避免类型在编译期和运行时的不匹配。

下面，我们定义一个方法，该方法使用Scala反射获取其参数的运行时类型信息并检查两个参数直接的子类型关系，如果第一个参数的类型是第二个参数的类型的子类型就返回`true`。

    scala> import scala.reflect.runtime.{universe => ru}
    import scala.reflect.runtime.{universe=>ru}

    scala> def m[T: ru.TypeTag, S: ru.TypeTag](x: T, y: S): Boolean = {
        |   val leftTag = ru.typeTag[T]
        |   val rightTag = ru.typeTag[S]
        |   leftTag.tpe <:< rightTag.tpe
        | }
    m: [T, S](x: T, y: S)(implicit evidence$1: reflect.runtime.universe.TypeTag[T], implicit evidence$2: reflect.runtime.universe.TypeTag[S])Boolean

    scala> m(d, c)
    res9: Boolean = true

我们可以看到，现在我们得到了期望的结果-- `d`的运行时类型确实是`c`的运行时类型的子类型。

## 编译期反射

Scala反射开启了一种形式的*元编程*，使得程序可以在编译期间修改*自己*。这种编译期反射以宏的形式实现，提供了在编译期执行那些操作抽象语法树的方法的能力。

宏特别有趣的是他们和Scala运行时用相同的编程接口（在包`scala.reflect.api`中）。这样宏和使用运行时反射的实现就能共享通用代码。

请注意[宏的指南]({{ site.baseurl }}/cn/overviews/macros/overview.html)关注宏的细节，而本指南关注反射编程接口的通用方面。尽管如此，很多概念也只用于宏，比如在[符号, 树, 和类型]({{site.baseurl }}/overviews/reflection/symbols-trees-types.html)进行详尽介绍的抽象语法树。

## 环境

所有反射任务都需要搭建合适的环境。这个环境取决于反射任务是在运行时进行还是在编译期。*宇*封装运行时和编译期环境的差别。反射环境另一个重要的方面是一套可以通过反射访问的实体，这套实体由*映射*决定。

映射不仅决定可以通过反射访问的实体，还提供施加于这些实体的反射操作。例如，运行时反射中*调用映射*可以用于调用类的方法或构造函数。

### 宇

`宇`是Scala反射的入口点。宇为反射中诸如`类型`、`树`、`注解`等所有原理性概念提供接口。 更多了解，请参考[宇]({{ site.baseurl}}/cn/overviews/reflection/environment-universes-mirrors.html)的指南或包`scala.reflect.api`中的[宇编程接口文档](http://www.scala-lang.org/api/{{ site.scala-version}}/scala/reflect/api/Universe.html)。

Scala反射大部分特性和本指南提供的大部分例子都需要导入`宇`或`宇`的成员。通常，如果使用运行时反射，你可以使用如下的通配导入来导入`scala.reflect.runtime.universe`的全部成员:

    import scala.reflect.runtime.universe._

### 映射

`映射`是Scala反射的中心。 反射所提供的所有信息都需要通过映射才能访问。视所需获取信息的类型或所执行的反射动作，将需要不同类型的映射。

更多了解，请参考[映射]({{ site.baseurl}}/overviews/reflection/environment-universes-mirrors.html)的指南或包`scala.reflect.api`中的[映射编程接口文档](http://www.scala-lang.org/api/{{ site.scala-version}}/scala/reflect/api/Mirrors.html)。
