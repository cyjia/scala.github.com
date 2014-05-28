---
layout: overview-large
title: 环境、宇和映射

disqus: true

partof: reflection
num: 2
outof: 7
language: cn
---

<span class="label important" style="float: right;">试验中</span>

## 环境

反射环境的差别在于反射任务是在运行时还是在编译期。对运行时和反射期差别的封装称为*宇*。反射环境另一个重要的方面是我们能以反射的方式访问的一套实体。这套实体由*映射*决定。

例如，通过运行时反射能访问的实体由`ClassloaderMirror`提供。这个映射提供了访问某个类加载器加载的实体（包、类型和成员）。

映射不仅决定了能以反射方式访问的实体集合，还提供了施加于这些实体的反射操作。例如，*调用映射*可以在运行时调用类的方法或构造函数。

## 宇

因为同时有运行时和编译期的反射能力，所以理论上有两套宇，必须根据手头的任务选择相应的宇：

- `scala.reflect.runtime.universe`用于**运行时反射**，
- `scala.reflect.macros.Universe`用于**编译期反射**。

宇为反射中所有诸如`类型`、`树`和`注解`的原理性概念提供了接口。

## 映射

有了*映射*，反射提供的所有信息都能访问了。视所需获取的信息或所要执行的反射动作，必须使用不同类型的映射。*类加载器映射*用于获取类型和成员的表示。从类加载器映射那里可以得到更具体的*调用映射*（最常用的映射），调用映射实现了方法和构造函数的反射调用以及字段访问。

总结：

- **"类加载器"映射**.
这些映射把名称转换成符号（借助`staticClass`/`staticClass`/`staticModule`方法）。

- **"调用"映射**.
这些映射实现了反射调用（借助`MethodMirror.apply`、`FieldMirror.get`等方法）。这些“调用”映射是最常用的映射。

### 运行时映射

运行时获取映射的入口是`ru.runtimeMirror(<classloader>)`，其中`ru`是`scala.reflect.runtime.universe`。

调用`scala.reflect.api.JavaMirrors#runtimeMirror`的结果是一个类加载器映射，类型为`scala.reflect.api.Mirrors#ReflectiveMirror`，这个映射可以根据名字加载符号。

类加载器映射可以创建调用映射（包括`scala.reflect.api.Mirrors#InstanceMirror`、 `scala.reflect.api.Mirrors#MethodMirror`、`scala.reflect.api.Mirrors#FieldMirror`、`scala.reflect.api.Mirrors#ClassMirror`和 `scala.reflect.api.Mirrors#ModuleMirror`）。

下面是这两类映射如何交互的例子。

### 各种映射，使用场景和举例

`ReflectiveMirror`用于根据名称加载符号，也是调用映射的入口。入口：`val m = ru.runtimeMirror(<classloader>)`。例子：

    scala> val ru = scala.reflect.runtime.universe
    ru: scala.reflect.api.JavaUniverse = ...

    scala> val m = ru.runtimeMirror(getClass.getClassLoader)
    m: reflect.runtime.universe.Mirror = JavaMirror ...

`InstanceMirror`用于创建方法、字段、内部类以及内部对象（模块）的调用映射。入口： `val im = m.reflect(<value>)`。例子：

    scala> class C { def x = 2 }
    defined class C

    scala> val im = m.reflect(new C)
    im: reflect.runtime.universe.InstanceMirror = instance mirror for C@3442299e

`MethodMirror`用于调用实例方法（Scala只有实例方法-对象的方法是对象实例的实例方法，借助`ModuleMirror.instance`可以获得）。入口： `val mm = im.reflectMethod(<method symbol>)`。例子:

    scala> val methodX = ru.typeOf[C].declaration(ru.newTermName("x")).asMethod
    methodX: reflect.runtime.universe.MethodSymbol = method x

    scala> val mm = im.reflectMethod(methodX)
    mm: reflect.runtime.universe.MethodMirror = method mirror for C.x: scala.Int (bound to C@3442299e)

    scala> mm()
    res0: Any = 2

`FieldMirror`用于读、写实例的字段（和方法一样，Scala也只有实例字段，参见上面）。入口：`val fm = im.reflectMethod(<field or accessor symbol>)`。例子：

    scala> class C { val x = 2; var y = 3 }
    defined class C

    scala> val m = ru.runtimeMirror(getClass.getClassLoader)
    m: reflect.runtime.universe.Mirror = JavaMirror ...

    scala> val im = m.reflect(new C)
    im: reflect.runtime.universe.InstanceMirror = instance mirror for C@5f0c8ac1

    scala> val fieldX = ru.typeOf[C].declaration(ru.newTermName("x")).asTerm.accessed.asTerm
    fieldX: reflect.runtime.universe.TermSymbol = value x

    scala> val fmX = im.reflectField(fieldX)
    fmX: reflect.runtime.universe.FieldMirror = field mirror for C.x (bound to C@5f0c8ac1)

    scala> fmX.get
    res0: Any = 2

    scala> fmX.set(3)

    scala> val fieldY = ru.typeOf[C].declaration(ru.newTermName("y")).asTerm.accessed.asTerm
    fieldY: reflect.runtime.universe.TermSymbol = variable y

    scala> val fmY = im.reflectField(fieldY)
    fmY: reflect.runtime.universe.FieldMirror = field mirror for C.y (bound to C@5f0c8ac1)

    scala> fmY.get
    res1: Any = 3

    scala> fmY.set(4)

    scala> fmY.get
    res2: Any = 4

`ClassMirror`用于创建构造函数的调用映射。入口：对于静态类`val cm1 = m.reflectClass(<class symbol>)`，对于内部类 `val mm2 = im.reflectClass(<class symbol>)`。例子：

    scala> case class C(x: Int)
    defined class C

    scala> val m = ru.runtimeMirror(getClass.getClassLoader)
    m: reflect.runtime.universe.Mirror = JavaMirror ...

    scala> val classC = ru.typeOf[C].typeSymbol.asClass
    classC: reflect.runtime.universe.Symbol = class C

    scala> val cm = m.reflectClass(classC)
    cm: reflect.runtime.universe.ClassMirror = class mirror for C (bound to null)

    scala> val ctorC = ru.typeOf[C].declaration(ru.nme.CONSTRUCTOR).asMethod
    ctorC: reflect.runtime.universe.MethodSymbol = constructor C

    scala> val ctorm = cm.reflectConstructor(ctorC)
    ctorm: reflect.runtime.universe.MethodMirror = constructor mirror for C.<init>(x: scala.Int): C (bound to null)

    scala> ctorm(2)
    res0: Any = C(2)

`ModuleMirror`用于访问单例对象的实例。入口：对于静态对象`val mm1 = m.reflectModule(<module symbol>)`，对于内部对象`val mm2 = im.reflectModule(<module symbol>)`。例子：

    scala> object C { def x = 2 }
    defined module C

    scala> val m = ru.runtimeMirror(getClass.getClassLoader)
    m: reflect.runtime.universe.Mirror = JavaMirror ...

    scala> val objectC = ru.typeOf[C.type].termSymbol.asModule
    objectC: reflect.runtime.universe.ModuleSymbol = object C

    scala> val mm = m.reflectModule(objectC)
    mm: reflect.runtime.universe.ModuleMirror = module mirror for C (bound to null)

    scala> val obj = mm.instance
    obj: Any = C$@1005ec04

### 编译期映射

编译期映射只使用类加载器映射根据名称加载符号。

类加载器映射的入口是`scala.reflect.macros.Context#mirror`。类加载器的常用方法包括`scala.reflect.api.Mirror#staticClass`、`scala.reflect.api.Mirror#staticModule`和`scala.reflect.api.Mirror#staticPackage`。例如：

    import scala.reflect.macros.Context

    case class Location(filename: String, line: Int, column: Int)

    object Macros {
      def currentLocation: Location = macro impl

      def impl(c: Context): c.Expr[Location] = {
        import c.universe._
        val pos = c.macroApplication.pos
        val clsLocation = c.mirror.staticModule("Location") // get symbol of "Location" object
        c.Expr(Apply(Ident(clsLocation), List(Literal(Constant(pos.source.path)), Literal(Constant(pos.line)), Literal(Constant(pos.column)))))
      }
    }

*注意：*有一些高层次的手段能避免手动查找符号。例如，`typeOf[Location.type].termSymbol`（或 `typeOf[Location].typeSymbol`如果我们需要`ClassSymbol`），这能做到类型安全因为我们不用根据字符串查找符号。
