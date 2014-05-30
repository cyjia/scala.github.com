---
layout: overview-large
title: 注解，名，范围，以及更多

disqus: true

partof: reflection
num: 4
outof: 7
language: cn
---

<span class="label important" style="float: right;">试验中</span>

## 注解

Scala中，可以用`scala.annotation.Annotation`的子类型给给声明加注解。而且，因为Scala整合了[Java的注解系统](http://docs.oracle.com/javase/7/docs/technotes/guides/language/annotations.html#_top)，所以可以使用标准Java编译器生成的注解。

如果相应的注解已经持久化并且从类文件读取注解声明，就可以通过反射探查注解。可以通过继承`scala.annotation.StaticAnnotation`或`scala.annotation.ClassfileAnnotation`使自定义注解持久化。结果，注解类型的实例就作为特殊属性保存到类文件。注意，仅继承`scala.annotation.Annotation`不足以使相应的元数据持久化到运行时反射。而且，继承`scala.annotation.ClassfileAnnotation`不能使你的注解在运行时像Java注解那样可见，那需要用Java写注解类才可以。

API区别两种注解：

- *Java注解：* Java编译器生成的定义之上的注解，*例如，*附加到程序定义的`java.lang.annotation.Annotation`的子类。当Scala反射读取Java注解时，会为每一个Java注解自动添加特征`scala.annotation.ClassfileAnnotation`使之成为子类。
- *Scala注解：* Scala编译器生成的定义或类型之上的注解。

Java注解和Scala注解之间的区别在契约`scala.reflect.api.Annotations#Annotation`中有明显表示，这个契约暴露了`scalaArgs`和`javaArgs`。对于扩展`scala.annotation.ClassfileAnnotation`的Scala注解或Java注解，`scalaArgs`为空并且参数（如果有）保存在`javaArgs`。对所有其他Scala注解，参数保存在`scalaArgs`并且`javaArgs`为空。

`scalaArgs`中的参数表示成类型树。注意，在类型检查之后的任何阶段都不会转换这些树。`javaArgs`中的参数表示成从`scala.reflect.api.Names#Name`到`scala.reflect.api.Annotations#JavaArgument`的映射。`JavaArgument`的实例表示不同的Java注解参数：

- 文字（原子值和字符常量），
- 数组，
- 嵌套的注解。

## 名

名是字符串的简单包装。
[名](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Names$NameApi)
有两个亚型`词语名`和`类型名`，他们区别词语（像对象和成员）和类型（像类、特征和类型成员）的名字。词语和同名的类型可以在同一个对象中共存。换句话说，类型和词语的名字空间相互隔离。

名和宇关联。例如：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> val mapName = newTermName("map")
    mapName: reflect.runtime.universe.TermName = map

上面，我们创建了一个和运行时宇关联的`名`（从它的依赖路径类型`reflect.runtime.universe.TermName`也能看出）。

名通常用来查找类型的成员。例如，查找`List`类中声明的`map`方法，可以这样：

    scala> val listTpe = typeOf[List[Int]]
    listTpe: reflect.runtime.universe.Type = scala.List[Int]

    scala> listTpe.member(mapName)
    res1: reflect.runtime.universe.Symbol = method map

查找类型成员，也是同样的流程，只不过用`newTypeName`。还可以依靠隐含转换在字符串与词语名和类型名直接转换：

    scala> listTpe.member("map": TermName)
    res2: reflect.runtime.universe.Symbol = method map

### 标准名

特定的名，比如`_root_`，在Scala程序中有特别含义。因为他们是通过反射访问特定Scala构造函数必不可少的。例如，反射地调用构造函数需要使用*标准名*`universe.nme.CONSTRUCTOR`，表示JVM上构造函数名的词语名`<init>`。

有两个

- *标准词语名，* *例如，* "`<init>`"、"`package`"和"`_root_`"，
- *标准类型名，* *例如，* "`<error>`"、"`_`"、和"`_*`"。

有些名，比如"包"，既是类型名又是词语名。
标准名由类`Universe`中的成员`nme`和`tpnme`提供。欲了解所有标准名的完整内容，请参考[API文档](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.StandardNames).

## 范围

范围对象通常映射名到相应词法范围内的符号。范围可以嵌套。反射API中暴露的基类型只保留了最小接口，用[符号](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Symbols$Symbol)的迭代表示的范围。

另外的功能暴露在[scala.reflect.api.Types#TypeApi](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Types$TypeApi)中定义的`members`和`declarations`返回的*成员范围*中。
[scala.reflect.api.Scopes#MemberScope](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Scopes$MemberScope)支持`sorted`方法，这个方法对成员*按声明顺序*排列。

下面例子以声明顺序返回`List`类所有覆写成员的符号列表：

    scala> val overridden = listTpe.declarations.sorted.filter(_.isOverride)
    overridden: List[reflect.runtime.universe.Symbol] = List(method companion, method ++, method +:, method toList, method take, method drop, method slice, method takeRight, method splitAt, method takeWhile, method dropWhile, method span, method reverse, method stringPrefix, method toStream, method foreach)

## 表达式

除了In addition to type `scala.reflect.api.Trees#Tree`, the base type of abstract
syntax trees, typed trees can also be represented as instances of type
[`scala.reflect.api.Exprs#Expr`](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Exprs$Expr).
An `Expr` wraps
an abstract syntax tree and an internal type tag to provide access to the type
of the tree. `Expr`s are mainly used to simply and conveniently create typed
abstract syntax trees for use in a macro. In most cases, this involves methods
`reify` and `splice` (see the
[macros guide](http://docs.scala-lang.org/overviews/macros/overview.html) for details).

## 标志和标志集

标志用于通过`scala.reflect.api.Trees#Modifiers`的`flags`字段为抽象语法树提供修饰符。接受修饰符的树有：

- `scala.reflect.api.Trees#ClassDef`。类和特性。
- `scala.reflect.api.Trees#ModuleDef`。对象。
- `scala.reflect.api.Trees#ValDef`。值，变量，参数和自类型注解。
- `scala.reflect.api.Trees#DefDef`。方法和构造函数。
- `scala.reflect.api.Trees#TypeDef`。类型别名，抽象类型成员和类型参数。

例如，要创建名为`C`的类，可以这样写：

    ClassDef(Modifiers(NoFlags), newTypeName("C"), Nil, ...)

这里，标志集为空。要让`C`私有，可以这样写：

    ClassDef(Modifiers(PRIVATE), newTypeName("C"), Nil, ...)

还可以用竖线操作符（`|`）组合标志。例如，私有的final类写成这样：

    ClassDef(Modifiers(PRIVATE | FINAL), newTypeName("C"), Nil, ...)

所有可用标志的列表定义在`scala.reflect.api.FlagSets#FlagValues`，通过`scala.reflect.api.FlagSets#Flag`获得。 （通常用通配导入，*例如，*`import scala.reflect.runtime.universe.Flag._`。）

定义树编译成符号，所以这些树修饰符上的标志也转换成结果符号上的标志。不同于树，符号不暴露标志，却提供模式为`isXXX`的测试方法（*例如，*`isFinal`）。有时，这些测试方法还需要用`asTerm`、`asType`、或`asClass`进行转换，因为有些标志仅对特定类型的符号有意义。

*值得注意的：*正在考虑重新设计这部分反射API。很有可能在将来发布的反射API中标志集会被其他东西代替。

## 常量

Scala编译器会在编译期对某些Scala规范称为*常量表达式*的表达式进行求值计算。下面几种表达式是编译期常量（参考[Scala语言规范](http://www.scala-lang.org/docu/files/ScalaReference.pdf)的6.24节）：

1. 原子值类的字面常量（[Byte](http://www.scala-lang.org/api/current/index.html#scala.Byte), [Short](http://www.scala-lang.org/api/current/index.html#scala.Short), [Int](http://www.scala-lang.org/api/current/index.html#scala.Int), [Long](http://www.scala-lang.org/api/current/index.html#scala.Long), [Float](http://www.scala-lang.org/api/current/index.html#scala.Float), [Double](http://www.scala-lang.org/api/current/index.html#scala.Double), [Char](http://www.scala-lang.org/api/current/index.html#scala.Char), [Boolean](http://www.scala-lang.org/api/current/index.html#scala.Boolean) 以及 [Unit](http://www.scala-lang.org/api/current/index.html#scala.Unit)） - 用对应类型直接表示。

2. 字符串常量 - 表示为字符串实例。

3. 对类的引用，通常用[scala.Predef#classOf](http://www.scala-lang.org/api/current/index.html#scala.Predef$@classOf[T]:Class[T])构造 - 表示为[类型](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Types$Type)。

4. 对Java枚举值的引用 - 表示为[符号](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Symbols$Symbol).

常量表达式用于表示

- 抽象语法树中的文字 (见`scala.reflect.api.Trees#Literal`)，
- Java类文件中注解的文字参数(见 `scala.reflect.api.Annotations#LiteralArgument`)。

例子：

    Literal(Constant(5))

上面表达式创建了一个表示Scala源码中整数常量`5`的AST。

`Constant`是一种“虚拟分支类”，*例如，*能够建实例并像分之类那样进行匹配的类。`Literal`和`LiteralArgument`都有`value`方法返回文字代表的编译期常量。

例子：

    Constant(true) match {
      case Constant(s: String)  => println("A string: " + s)
      case Constant(b: Boolean) => println("A Boolean value: " + b)
      case Constant(x)          => println("Something else: " + x)
    }
    assert(Constant(true).value == true)

类的引用表示为`scala.reflect.api.Types#Type`的实例。这种引用可以用`RuntimeMirror`（比如`scala.reflect.runtime.currentMirror`）的`runtimeClass`方法转换成运行时类。（这种从类型到运行时类的转换是必须的，因为在Scala编译器处理类引用时，真正的运行时类可能还没编译出来。）

Java枚举值引用表示为符号（`scala.reflect.api.Symbols#Symbol`的实例），在JVM上指向返回相关枚举值的方法。可以用`RuntimeMirror`检查相关枚举或得到所引用枚举的运行时值。

例子：

    // Java source:
    enum JavaSimpleEnumeration { FOO, BAR }

    import java.lang.annotation.*;
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE})
    public @interface JavaSimpleAnnotation {
      Class<?> classRef();
      JavaSimpleEnumeration enumRef();
    }

    @JavaSimpleAnnotation(
      classRef = JavaAnnottee.class,
      enumRef = JavaSimpleEnumeration.BAR
    )
    public class JavaAnnottee {}

    // Scala source:
    import scala.reflect.runtime.universe._
    import scala.reflect.runtime.{currentMirror => cm}

    object Test extends App {
      val jann = typeOf[JavaAnnottee].typeSymbol.annotations(0).javaArgs

      def jarg(name: String) = jann(newTermName(name)) match {
        // Constant is always wrapped in a Literal or LiteralArgument tree node
        case LiteralArgument(ct: Constant) => value
        case _ => sys.error("Not a constant")
      }

      val classRef = jarg("classRef").value.asInstanceOf[Type]
      println(showRaw(classRef))         // TypeRef(ThisType(), JavaAnnottee, List())
      println(cm.runtimeClass(classRef)) // class JavaAnnottee

      val enumRef = jarg("enumRef").value.asInstanceOf[Symbol]
      println(enumRef)                   // value BAR

      val siblings = enumRef.owner.typeSignature.declarations
      val enumValues = siblings.filter(sym => sym.isVal && sym.isPublic)
      println(enumValues)                // Scope {
                                         //   final val FOO: JavaSimpleEnumeration;
                                         //   final val BAR: JavaSimpleEnumeration
                                         // }

      val enumClass = cm.runtimeClass(enumRef.owner.asClass)
      val enumValue = enumClass.getDeclaredField(enumRef.name.toString).get(null)
      println(enumValue)                 // BAR
    }

## 打印

能很好地打印[`树`](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Trees) and
[`类型`](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Types)的工具。

### 打印树

方法`show`展示反射对象的“美化”表示。这种表示提供脱掉Scala语法糖衣之后的Java表示。例如：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> def tree = reify { final class C { def x = 2 } }.tree
    tree: reflect.runtime.universe.Tree

    scala> show(tree)
    res0: String =
    {
      final class C extends AnyRef {
        def <init>() = {
          super.<init>();
          ()
        };
        def x = 2
      };
      ()
    }

方法`showRaw`展示给定反射对象作为Scala抽象语法树（AST）的内部结构，Scala类型检查器工作在AST之上。

注意，因为这个表示看上去像生成了正确的树，可能有人认为可以在宏实现中使用，通常并不是这样的。符号并没有完全展示出来（只有符号的名称）。所以，这个方法最适合于简单分析给定Scala代码的AST。

    scala> showRaw(tree)
    res1: String = Block(List(
      ClassDef(Modifiers(FINAL), newTypeName("C"), List(), Template(
        List(Ident(newTypeName("AnyRef"))),
        emptyValDef,
        List(
          DefDef(Modifiers(), nme.CONSTRUCTOR, List(), List(List()), TypeTree(),
            Block(List(
              Apply(Select(Super(This(tpnme.EMPTY), tpnme.EMPTY), nme.CONSTRUCTOR), List())),
              Literal(Constant(())))),
          DefDef(Modifiers(), newTermName("x"), List(), List(), TypeTree(),
            Literal(Constant(2))))))),
      Literal(Constant(())))

`showRaw`方法还可以在所检查的对象旁边打印`scala.reflect.api.Types`。

    scala> import scala.tools.reflect.ToolBox // requires scala-compiler.jar
    import scala.tools.reflect.ToolBox

    scala> import scala.reflect.runtime.{currentMirror => cm}
    import scala.reflect.runtime.{currentMirror=>cm}

    scala> showRaw(cm.mkToolBox().typeCheck(tree), printTypes = true)
    res2: String = Block[1](List(
      ClassDef[2](Modifiers(FINAL), newTypeName("C"), List(), Template[3](
        List(Ident[4](newTypeName("AnyRef"))),
        emptyValDef,
        List(
          DefDef[2](Modifiers(), nme.CONSTRUCTOR, List(), List(List()), TypeTree[3](),
            Block[1](List(
              Apply[4](Select[5](Super[6](This[3](newTypeName("C")), tpnme.EMPTY), ...))),
              Literal[1](Constant(())))),
          DefDef[2](Modifiers(), newTermName("x"), List(), List(), TypeTree[7](),
            Literal[8](Constant(2))))))),
      Literal[1](Constant(())))
    [1] TypeRef(ThisType(scala), scala.Unit, List())
    [2] NoType
    [3] TypeRef(NoPrefix, newTypeName("C"), List())
    [4] TypeRef(ThisType(java.lang), java.lang.Object, List())
    [5] MethodType(List(), TypeRef(ThisType(java.lang), java.lang.Object, List()))
    [6] SuperType(ThisType(newTypeName("C")), TypeRef(... java.lang.Object ...))
    [7] TypeRef(ThisType(scala), scala.Int, List())
    [8] ConstantType(Constant(2))

### 打印类型

`show`方法可用于产生类型的*可读*字符串表示：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> def tpe = typeOf[{ def x: Int; val y: List[Int] }]
    tpe: reflect.runtime.universe.Type

    scala> show(tpe)
    res0: String = scala.AnyRef{def x: Int; val y: scala.List[Int]}

类似用于`scala.reflect.api.Trees`的`showRaw`方法，用于`scala.reflect.api.Types`的`showRaw`方法提供AST（Scala类型检查器工作在AST之上）的可视化。

    scala> showRaw(tpe)
    res1: String = RefinedType(
      List(TypeRef(ThisType(scala), newTypeName("AnyRef"), List())),
      Scope(
        newTermName("x"),
        newTermName("y")))

方法`showRaw`还有定名参数`printIds`和`printKinds`，两者的默认值都是`false`。给这些参数传入`true`时，`showRaw`额外显示符号的唯一标识符和种类（包，类型，方法，读方法，等等）。

    scala> showRaw(tpe, printIds = true, printKinds = true)
    res2: String = RefinedType(
      List(TypeRef(ThisType(scala#2043#PK), newTypeName("AnyRef")#691#TPE, List())),
      Scope(
        newTermName("x")#2540#METH,
        newTermName("y")#2541#GET))

## 位置

位置(特征[Position](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.Position)的实例)
用于追踪符号和树节点的来源。位置常用于展示警告和错误，指出程序中的错误位置。位置表示源文件的行和列（从源文件开始的偏移量称为“点”，有时用起来不方便）。位置还携带所指向行的内容。不是所有的树和符号都有位置，用`NoPosition`对象表示位置缺失。

位置可以指向源文件中的单个字符或一个*区域*。后一种情况使用*区域位置*（不是区域位置的位置也称为*偏移位置*）。区域位置有附加的`开始`和`结束`偏移量。可以使用`focusStart`和`focusEnd`方法“关注”`开始`和`结束`偏移量，这两个方法返回位置（调用非区域位置时，会返回`自己`）。

可以比较位置，比如用方法`precedes`，这个方法给出是否两个区域都明确（*例如，*不是`NoPosition`），`这个`区域的结束点不大于给定位置的开始点。另外，还可以测试区域位置的包含情况（用`includes`方法）和交叉情况（用`overlaps`方法）。

区域位置或者透明或者不透明。区域是否透明对其允许的使用有影响，因为包含区域的树必须满足下面的不变性：

- 带偏移位置的树不能包含带区域位置的孩子
- 如果带区域位置的树还有带区域位置的孩子，那么孩子的区域必须包含在父的区域内。
- 同一个节点的孩子们的不透明区域不能交叉（这意味着他们的交叉最多有一个点）。

用`makeTransparent`方法可以把不透明区域转换成透明区域，其他区域不变。

