---
layout: overview-large
title: 定义宏

disqus: true

partof: macros
num: 3
language: cn
---
<span class="label warning" style="float: right;">试验中</span>

**Eugene Burmako**

定义宏是Scala2.10.0以后的一个试验性特性。定义宏的一个子集正等待详尽的规范，暂时计划在Scala未来的一个版本中趋于稳定。

## 直觉 

这是一个原型化的宏定义：

    def m(x: T): R = macro implRef

乍看宏定义和普通的函数定义等价，除了宏定义体以条件关键字`macro`开始并且跟着一个指向静态宏实现方法的有效标识符。

如果，在类型检查中编译器遇到宏的申请`m(args)`，编译器会调用宏实现以展开这个申请，以参数表达式参数的抽象语法树作为参数。宏实现的结果是另一个抽象语法树，这个抽象语法树会被内联到调用放并接着进行类型检查。

下面代码片段声明宏定义assert引用宏实现Asserts.assertImpl（assertImpl的实现在下面）：

    def assert(cond: Boolean, msg: Any) = macro Asserts.assertImpl

调用`assert(x < 10, "limit exceeded")`会导致编译期调用

    assertImpl(c)(<[ x < 10 ]>, <[ “limit exceeded” ]>)

`c`是上下文参数，包含编译器在调用方收集的信息，另两个参数是表示表达式`x < 10`和`limit exceeded`的抽象语法树。

本文档中，`<[ expr ]>`表示代表表达式expr的抽象语法树。这个标记在我们建议的Scala语言扩展中没有对应项。实际上，可以用特性`scala.reflect.api.Trees`中的类型构造语法树，上面两个表达式应该是这样：

    Literal(Constant("limit exceeded"))

    Apply(
      Select(Ident(newTermName("x")), newTermName("$less"),
      List(Literal(Constant(10)))))

这里是宏`assert`的一个可能实现：

    import scala.reflect.macros.Context
    import scala.language.experimental.macros

    object Asserts {
      def raise(msg: Any) = throw new AssertionError(msg)
      def assertImpl(c: Context)
        (cond: c.Expr[Boolean], msg: c.Expr[Any]) : c.Expr[Unit] =
       if (assertionsEnabled)
          <[ if (!cond) raise(msg) ]>
          else
          <[ () ]>
    }

如例中所示，宏实现接受几个参数列表。首先是单个参数，类型为`scala.reflect.macros.Context`。接着是一个参数列表，列表中参数和宏定义参数有同样的名字。原始宏参数有类型`T`的地方，宏实现参数有类型`c.Expr[T]`。`Expr[T]`是`Context`中定义的类型用于包装类型`T`的抽象语法树。宏实现`assertImpl`的结果类型又是一个包装的树，类型为`c.Expr[Unit]`。

还要注意，宏是实验性的高级特性，需要明确启用。用文件级的`import scala.language.experimental.macros`或者用编译级的`-language:experimental.macros`(提供编译开关)。

### 泛型宏

宏定义和宏实现都可以是泛型。如果宏实现有类型参数，实际的类型参数必须在宏定义体中明确给出。实现中的类型参数会绑定到`WeakTypeTag`上下文。这种情况下，当宏展开时申请方实例化的描述实际类型参数对应的类型标记会一并传入。

下面的代码片段声明宏定义`Queryable.map`，引用宏实现`QImpl.map`：

    class Queryable[T] {
     def map[U](p: T => U): Queryable[U] = macro QImpl.map[T, U]
    }

    object QImpl {
     def map[T: c.WeakTypeTag, U: c.WeakTypeTag]
            (c: Context)
            (p: c.Expr[T => U]): c.Expr[Queryable[U]] = ...
    }

现在考虑一个类型为`Queryable[String]`的值`q`和宏调用

    q.map[Int](s => s.length)

这个调用展开成下面的反射宏调用

    QImpl.map(c)(<[ s => s.length ]>)
       (implicitly[WeakTypeTag[String]], implicitly[WeakTypeTag[Int]])

## 完整例子

本节提供一个端到端实现`printf`宏，这个宏在编译期校验并应用格式化字符串。为了简化，这个讨论用Scala编译器控制台，不过Maven和SBT也支持下面的宏。

写一个宏从宏定义开始，它代表宏的外观。宏定义是普通的函数，人们可以在签名中用任何自己喜欢的东西。定义体仅仅是对实现的引用。如上面提到，定义宏需要导入`scala.language.experimental.macros`或启用特殊的编译器开关`-language:experimental.macros`。

    import scala.language.experimental.macros
    def printf(format: String, params: Any*): Unit = macro printf_impl

宏实现必须对应使用它的宏定义（通常只有一个，也可能有多个）。简而言之，宏定义签名中的每一个类型`T`的参数必须对应宏实现中类型为`c.Expr[T]`的参数。规则的全部列表相当复杂，不过这从来都不是问题，因为如果编译器不高兴，它会在错误信息中打印它期望的签名。

    import scala.reflect.macros.Context
    def printf_impl(c: Context)(format: c.Expr[String], params: c.Expr[Any]*): c.Expr[Unit] = ...

编译器API暴露在`scala.reflect.macros.Context`。其中最重要的部分，反射API，通过`c.universe`访问。习惯上导入`c.universe._`，因为它包含很多常规使用的函数和类型。

    import c.universe._

首先，宏需要解析提供的格式化字符串。宏在编译期运行，所以他们操作树而不是值。这意味着宏`printf`的format参数会是一个编译期的文字而不是类型为`java.lang.String`的对象。
这也意味着下面的代码对`printf(get_format(), ...)`不工作，因为这时`format`不是字符常量，而是代表函数的AST。

    val Literal(Constant(s_format: String)) = format.tree

典型的宏（这个宏也不例外）需要创建代表Scala代码的AST（抽象语法树）。要学校更多有关生成Scala代码的内容，请看[反射概览](http://docs.scala-lang.org/cn/overviews/reflection/overview.html)。随着创建AST，下面的代码还操作类型。注意我们如何持有对应`Int`和`String`的Scala类型。上面的反射概览链接详细讲述类型操作。代码生成的最后一步是结合所有生成的代码为一个`Block`。注意对`reify`的调用，这提供了创建AST的快捷方式。

    val evals = ListBuffer[ValDef]()
    def precompute(value: Tree, tpe: Type): Ident = {
      val freshName = newTermName(c.fresh("eval$"))
      evals += ValDef(Modifiers(), freshName, TypeTree(tpe), value)
      Ident(freshName)
    }

    val paramsStack = Stack[Tree]((params map (_.tree)): _*)
    val refs = s_format.split("(?<=%[\\w%])|(?=%[\\w%])") map {
      case "%d" => precompute(paramsStack.pop, typeOf[Int])
      case "%s" => precompute(paramsStack.pop, typeOf[String])
      case "%%" => Literal(Constant("%"))
      case part => Literal(Constant(part))
    }

    val stats = evals ++ refs.map(ref => reify(print(c.Expr[Any](ref).splice)).tree)
    c.Expr[Unit](Block(stats.toList, Literal(Constant(()))))

下面的片段表示宏`printf`的完整定义。仿照这个例子，创建一个空目录，复制代码到名为`Macros.scala`的新文件。

    import scala.reflect.macros.Context
    import scala.collection.mutable.{ListBuffer, Stack}

    object Macros {
      def printf(format: String, params: Any*): Unit = macro printf_impl

      def printf_impl(c: Context)(format: c.Expr[String], params: c.Expr[Any]*): c.Expr[Unit] = {
        import c.universe._
        val Literal(Constant(s_format: String)) = format.tree

        val evals = ListBuffer[ValDef]()
        def precompute(value: Tree, tpe: Type): Ident = {
          val freshName = newTermName(c.fresh("eval$"))
          evals += ValDef(Modifiers(), freshName, TypeTree(tpe), value)
          Ident(freshName)
        }

        val paramsStack = Stack[Tree]((params map (_.tree)): _*)
        val refs = s_format.split("(?<=%[\\w%])|(?=%[\\w%])") map {
          case "%d" => precompute(paramsStack.pop, typeOf[Int])
          case "%s" => precompute(paramsStack.pop, typeOf[String])
          case "%%" => Literal(Constant("%"))
          case part => Literal(Constant(part))
        }

        val stats = evals ++ refs.map(ref => reify(print(c.Expr[Any](ref).splice)).tree)
        c.Expr[Unit](Block(stats.toList, Literal(Constant(()))))
      }
    }

要使用`printf`宏，在同一个目录下创建另一个文件`Test.scala`并把下面的代码放进去。注意，使用宏和调用函数一样简单。同样不需要导入`scala.language.experimental.macros`。

    object Test extends App {
      import Macros._
      printf("hello %s!", "world")
    }

宏技术的重要方面是隔离编译。为执行宏宽展，编译器需要可执行形式的宏实现。所以宏实现必须在主编译之前编译，否则你可能会见到如下的错误：

    ~/Projects/Kepler/sandbox$ scalac -language:experimental.macros Macros.scala Test.scala
    Test.scala:3: error: macro implementation not found: printf (the most common reason for that is that
    you cannot use macro implementations in the same compilation run that defines them)
    pointing to the output of the first phase
      printf("hello %s!", "world")
            ^
    one error found

    ~/Projects/Kepler/sandbox$ scalac Macros.scala && scalac Test.scala && scala Test
    hello world!

## 提示与技巧

### 和Scala编译器控制台一起用宏

这个场景出现在前一节。总之，用`scalac`的不同调用编译宏和他们的使用方就万事大吉了。如果你使用REPL，那更好，因为REPL用分离的编译处理每一行，所以你可以定义宏然后马上就用。

### 和Maven或SBT一起用宏

本指南的演练用尽可能简单的命令行编译，不过宏也能和Maven和SBT之类的构建工具一起使用。看看[https://github.com/scalamacros/sbt-example](https://github.com/scalamacros/sbt-example)或[https://github.com/scalamacros/maven-example](https://github.com/scalamacros/maven-example)的端到端例子，不过简单来说你只需知道两件事：
* 库依赖中需要scala-reflect.jar。
* 编译分离的现实要求宏放到不同的项目。

### 和Scala IDE或Intellij IDEA一起使用宏

在ScalaIDE和Intellij IDEA中宏都能很好工作，只要他们被移到不同的项目。

### 调试宏

调试宏（例如，驱动宏扩展的逻辑）很直接。因为宏是在编译器中扩展，所以你只需在调试器中运行编译器。要做到这一点，你需要：1)添加Scala主目录下lib文件夹内的所有(!)库(包括类似`scala-library.jar`，`scala-reflect.jar`和`scala-compiler.jar`等jar文件)到调试配置的类路径，2)设置`scala.tools.nsc.Main`为入口点, 3)给JVM提供系统属性`-Dscala.usejavacp=true`（非常重要!），4) 设置编译器的命令行参数`-cp <宏所在类的路径> Test.scala`，这里`Test.scala`表示包含需要扩展的宏调用的测试文件。这些都做完之后，你应该能在宏实现中添加断点并启动调试器。

真正需要工具特别支持的是调试宏展开的结果（例如，宏生成的代码）。因为这部分代码从未手动写出，所以你无从设置断点也无从单步调试。Scala IDE和Intellij IDEA团队将来很可能在他们调试器的某个地方增加支持，不过现在调试宏扩展的唯一方法是诊断性的打印：`-Ymacro-debug-lite`（如下所述），会打印宏生成的代码并打印以追踪所生成代码的执行。

### 查看生成的代码

用`-Ymacro-debug-lite`可以看宏扩展生成代码的伪Scala表示和原生AST表示。两者各有长处：前者对浅层分析有用，而后者对细粒度调试非常有用。

    ~/Projects/Kepler/sandbox$ scalac -Ymacro-debug-lite Test.scala
    typechecking macro expansion Macros.printf("hello %s!", "world") at
    source-C:/Projects/Kepler/sandbox\Test.scala,line-3,offset=52
    {
      val eval$1: String = "world";
      scala.this.Predef.print("hello ");
      scala.this.Predef.print(eval$1);
      scala.this.Predef.print("!");
      ()
    }
    Block(List(
    ValDef(Modifiers(), newTermName("eval$1"), TypeTree().setType(String), Literal(Constant("world"))),
    Apply(
      Select(Select(This(newTypeName("scala")), newTermName("Predef")), newTermName("print")),
      List(Literal(Constant("hello")))),
    Apply(
      Select(Select(This(newTypeName("scala")), newTermName("Predef")), newTermName("print")),
      List(Ident(newTermName("eval$1")))),
    Apply(
      Select(Select(This(newTypeName("scala")), newTermName("Predef")), newTermName("print")),
      List(Literal(Constant("!"))))),
    Literal(Constant(())))

### 宏抛出未处理的异常

如果宏抛出未处理的异常会发生什么？例如，我们通过提供无效输入来破坏`printf`宏。如输出所示，没什么戏剧性事件发生。编译器对抗坏事的宏，打印相关部分的堆栈并报告错误。

    ~/Projects/Kepler/sandbox$ scala
    Welcome to Scala version 2.10.0-20120428-232041-e6d5d22d28 (Java HotSpot(TM) 64-Bit Server VM, Java 1.6.0_25).
    Type in expressions to have them evaluated.
    Type :help for more information.

    scala> import Macros._
    import Macros._

    scala> printf("hello %s!")
    <console>:11: error: exception during macro expansion:
    java.util.NoSuchElementException: head of empty list
            at scala.collection.immutable.Nil$.head(List.scala:318)
            at scala.collection.immutable.Nil$.head(List.scala:315)
            at scala.collection.mutable.Stack.pop(Stack.scala:140)
            at Macros$$anonfun$1.apply(Macros.scala:49)
            at Macros$$anonfun$1.apply(Macros.scala:47)
            at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:237)
            at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:237)
            at scala.collection.IndexedSeqOptimized$class.foreach(IndexedSeqOptimized.scala:34)
            at scala.collection.mutable.ArrayOps.foreach(ArrayOps.scala:39)
            at scala.collection.TraversableLike$class.map(TraversableLike.scala:237)
            at scala.collection.mutable.ArrayOps.map(ArrayOps.scala:39)
            at Macros$.printf_impl(Macros.scala:47)

                  printf("hello %s!")
                        ^

### 报告警告和错误

与用户交互的典型方法是用`scala.reflect.macros.FrontEnds`的方法。`c.error`报告编译错误，`c.info`发布警告，`c.abort`报告错误并终止执行宏。

    scala> def impl(c: Context) =
      c.abort(c.enclosingPosition, "macro has reported an error")
    impl: (c: scala.reflect.macros.Context)Nothing

    scala> def test = macro impl
    defined term macro test: Any

    scala> test
    <console>:32: error: macro has reported an error
                  test
                  ^

注意，现在报告设施不支持一个位置上多个警告或错误，这在[SI-6910](https://issues.scala-lang.org/browse/SI-6910)有描述。这意味着只会报告一个位置上的第一个错误或警告，其他都会丢失（并且错误覆盖同一位置上的警告，即便警告更早报告）。

### 写更大的宏

当宏实现大到需要在实现体外进行模块化时，明显需要带上参数context，因为多数情况都依赖context。

一种方法是写一个带参数`Context`的类，然后把宏实现分解成那个类的一系列方法。这很自然也很简单，但是很难做对。这儿有个典型的编译错误。

    scala> class Helper(val c: Context) {
         | def generate: c.Tree = ???
         | }
    defined class Helper

    scala> def impl(c: Context): c.Expr[Unit] = {
         | val helper = new Helper(c)
         | c.Expr(helper.generate)
         | }
    <console>:32: error: type mismatch;
     found   : helper.c.Tree
        (which expands to)  helper.c.universe.Tree
     required: c.Tree
        (which expands to)  c.universe.Tree
           c.Expr(helper.generate)
                         ^

这个片段的问题是路径依赖类型不匹配。Scala编译器不理解`impl`中的`c`和`Helper`中的`c`是同一个对象，尽管helper是用原始`c`构造的。

幸运的是只需一个微调就能让编译器明白状况。一种可能的方法是使用精细的类型（下面例子是这个想法的最简单应用；例如，还可以写一个从`Context`到`Helper`的隐式转换器以避免隐式实例化病简化调用）。

    scala> abstract class Helper {
         | val c: Context
         | def generate: c.Tree = ???
         | }
    defined class Helper

    scala> def impl(c1: Context): c1.Expr[Unit] = {
         | val helper = new { val c: c1.type = c1 } with Helper
         | c1.Expr(helper.generate)
         | }
    impl: (c1: scala.reflect.macros.Context)c1.Expr[Unit]

另一种方法是用明确的类型参数传递context的标识。注意`Helper`的构造器如何使用`c.type`来表达`Helper.c`和原始`c`相同的事实。Scala的类型推断自己不能解决，所以我们需要帮助它。

    scala> class Helper[C <: Context](val c: C) {
         | def generate: c.Tree = ???
         | }
    defined class Helper

    scala> def impl(c: Context): c.Expr[Unit] = {
         | val helper = new Helper[c.type](c)
         | c.Expr(helper.generate)
         | }
    impl: (c: scala.reflect.macros.Context)c.Expr[Unit]
