---
layout: overview-large
title: 符号、树和类型

disqus: true

partof: reflection
num: 3
outof: 7
language: cn
---

<span class="label important" style="float: right;">试验中</span>

## 符号

符号用于建立名称与所指实体（比如类或方法）之间的绑定。在Scala中，你能定义并给出名称的任何事物都有与之关联的符号。

符号含有实体（`class`、`object`、`trait`等）或成员（`val`、`var`、`def`等）声明的所有可用信息，因此也是运行时反射和编译期反射（宏）的虚拟中心。

符号能提供大量信息，从所有符号都有的基础方法`name`到`ClassSymbol`的`baseClasses`之类更隐秘的概念。符号的其他使用场景包括检查成员的签名、获取类的类型参数、获取方法的类型参数以及确定字段的类型。

### 符号所有者的层次结构

多个符号组织在一个层次结构内。例如，代表方法参数的符号*隶属*于对应方法的符号，方法的符号*隶属*于外层的类、特性或对象，类*隶属*于包含它的包，等等。

如果符号没有所有者（例如，因为它表示类似顶层包一样的顶层实体），那么它的所有者就是那个特殊的单例对象`NoSymbol`。表示缺失符号的`NoSymbol`常用在API中表示空或默认值。访问`NoSymbol`的`owner`会抛异常。参见API文档中类型[`Symbol`](http://www.scala-lang.org/api/{{ site.scala-version }}/scala/reflect/api/Symbols$SymbolApi.html)的完整接口。

### `类型符号`

类型符号表示类型、类和特征的声明以及类型参数。几个有趣的成员不适用于更具体的`ClassSymbol`，他们包括`isAbstractType`、`isContravariant`、和
`isCovariant`。

- `ClassSymbol`：提供对类或特征声明含有的所有信息的访问，例如，`name`, 修饰符 (`isFinal`，`isPrivate`，`isProtected`，`isAbstractClass`，等等), `baseClasses`和`typeParams`。

### `词语符号`

词语符号表示val、var、def、对象以及包、值参数的声明。

- `MethodSymbol`：方法符号表示def声明（`TermSymbol`的子类）。它支持一些查询，比如确认方法是不是（主）构造器，或者方法是否支持可变长度参数列表。
- `ModuleSymbol`：模块符号表示对象声明。他允许借助成员`moduleClass`查询对象定义隐含关联的类。可以通过`selfType.termSymbol`从模块类回到关联的模块符号。

### 符号转换

有些情况下可能会用到一些返回通用`Symbol`类型的方法，这时可以根据需要把通用`Symbol`类型转换成更具体的符号类型。

像`asClass`、`asMethod`之类的符号转换用于转换到一个合适的（比如，你想使用`MethodSymbol`接口）更具体的`Symbol`子类型。

例如，

    scala> import reflect.runtime.universe._
    import reflect.runtime.universe._

    scala> class C[T] { def test[U](x: T)(y: U): Int = ??? }
    defined class C

    scala> val testMember = typeOf[C[Int]].member(newTermName("test"))
    testMember: reflect.runtime.universe.Symbol = method test

这个例子，`member`返回`Symbol`实例而不是期望的`MethodSymbol`实例，所以我们必须使用`asMethod`以获得`MethodSymbol`

    scala> testMember.asMethod
    res0: reflect.runtime.universe.MethodSymbol = method test

### 自由符号

`FreeTermSymbol`和`FreeTypeSymbol`这两个特殊符号类型有附加状态，因为他们表示的符号可用信息不完全。这些符号在具体化过程的某些场合（欲知更多背景请参考具体化树的相关部分）生成。每当具体化过程不能定位一个符号（即这个符号不在相应类文件，比如，因为这个符号表示一个局部类），它就把这个符号具体化成所谓的“自由类型”（一个合成的假符号用于记住原始名字和所有者，而且在原始名称之后紧跟一个托管类型签名）。 你可以通过调用`sym.isFreeType`确认符号是否是自由类型。你还可以通过调用`tree.freeTypes`获取树及其孩子的所有自由类型列表。最后，你可以使用`-Xlog-free-types`来获得当具体化过程产生自由类型时的警告。

## 类型

从字面意义可以看出，`类型`的实例表示相应符号的类型信息。这包括它直接声明或继承的成员（方法、字段、类型别名、抽象类型、嵌套类、特征，等等），它的基类型，它的擦除器，等等。类型还提供测试类型一直或相等的方法。

### 实例化类型

通常，有三种实例化`类型`的方法。

1. 借助`scala.reflect.api.TypeTags`（已经混入`Universe`）的`typeOf`方法(最简单，最常用)。
2. 一些标准类型，比如`Int`、`Boolean`、`Any`或`Unit`，可以从宇中获得。
3. 用`scala.reflect.api.Types`的`typeRef`或`polyType`工厂方法手动实例化（不推荐）。

#### 用`typeOf`实例化类型

绝大多数情况下都可以用`scala.reflect.api.TypeTags#typeOf`方法实例化类型。它接受一个类型参数产生一个表示该参数的`类型`实例。例如：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> typeOf[List[Int]]
    res0: reflect.runtime.universe.Type = scala.List[Int]

这个例子中，返回了[`scala.reflect.api.Types$TypeRef`](http://www.scala-lang.org/api/{{ site.scala-version }}/scala/reflect/api/Types$TypeRef.html), 与之相应的是类型构造函数`List`和类型参数`Int`。

不过，值得注意的是这种方法需要手工指定我们试图实例化的类型。如果我们想某一特定实例的`类型`又该怎么办呢？可以顺手定义一个绑定了上下文类型参数的方法，使之为我们生成特定的`TypeTag`，然后我们就可以用这个方法来获取特定实例的类型：

    scala> def getType[T: TypeTag](obj: T) = typeOf[T]
    getType: [T](obj: T)(implicit evidence$1: reflect.runtime.universe.TypeTag[T])reflect.runtime.universe.Type

    scala> getType(List(1,2,3))
    res1: reflect.runtime.universe.Type = List[Int]

    scala> class Animal; class Cat extends Animal
    defined class Animal
    defined class Cat

    scala> val a = new Animal
    a: Animal = Animal@21c17f5a

    scala> getType(a)
    res2: reflect.runtime.universe.Type = Animal

    scala> val c = new Cat
    c: Cat = Cat@2302d72d

    scala> getType(c)
    res3: reflect.runtime.universe.Type = Cat

*注意:*方法`typeOf`不适用于类型参数，比如`typeOf[List[A]]`，`A`就是类型参数。这种情况，可以转而使用`scala.reflect.api.TypeTags#weakTypeOf`。欲知详情，请参考本指南[类型标记]({{ site.baseurl }}/cn/overviews/reflection/typetags-manifests.html)一节.

#### 标准类型

可以通过宇的`definitions`成员访问诸如`Int`、`Boolean`、`Any`、 `Unit`的标准类型。例如：

    scala> import scala.reflect.runtime.universe
    import scala.reflect.runtime.universe

    scala> val intTpe = universe.definitions.IntTpe
    intTpe: reflect.runtime.universe.Type = Int

标准类型列表定义在特征`StandardTypes`中，在[`scala.reflect.api.StandardDefinitions`](http://www.scala-lang.org/api/current/index.html#scala.reflect.api.StandardDefinitions$StandardTypes)。

### 类型的常见操作

类型通常用来测试类型一致性或查询成员。类型上三类主要操作是：

1. 确认两个类型之间的子类型关系。
2. 确认两个类型相等。
3. 查询给定类型的特定成员或子类型。

#### 子类型关系

给定两个类型实例，用`<:<`（有些特例用`weak_<:<`，下面会有解释）就可以很容易地测试一个是否另一个的子类型。

    scala> import scala.reflect.runtime.universe._
    import scala-lang.reflect.runtime.universe._

    scala> class A; class B extends A
    defined class A
    defined class B

    scala> typeOf[A] <:< typeOf[B]
    res0: Boolean = false

    scala> typeOf[B] <:< typeOf[A]
    res1: Boolean = true

注意，方法`weak_<:<`用于确认两个类型之间的*弱一致性*。这一点在处理数值类型时特别重要。

Scala的数值类型遵守如下的顺序（Scala语言规范第3.5.3节）：

> 有些场景下Scala用更一般的一致性关系。如果S<:T或S和T都是原始数值类型并且S和T遵循如下顺序，那么类型S与T有弱一致性，写作S <:w T。

| 弱一致性关系 |
 ---------------------------
| `Byte` `<:w` `Short` |
| `Short` `<:w` `Int` |
| `Char` `<:w` `Int` |
| `Int` `<:w` `Long` |
| `Long` `<:w` `Float` |
| `Float` `<:w` `Double` |

例如，若一致性用于确定下面if表达式的类型：

    scala> if (true) 1 else 1d
    res2: Double = 1.0

上面的if表达式，结果类型定义为两个类型的*最弱上限*（注，按弱一致性确定的最小上限）。

因此，既然按照弱一致性`Double`是`Int`和`Double`中的最小上限，那么例子中if表达式的类型就推断为`Double`。

注意到方法`weak_<:<`检查*弱一致性*（相反，`<:<`检查一致性时不考虑规范3.5.3节的弱一致性关系）因此在查看数值类型`Int`和`Double`的一致性关系时返回了正确结果：

    scala> typeOf[Int] weak_<:< typeOf[Double]
    res3: Boolean = true

    scala> typeOf[Double] weak_<:< typeOf[Int]
    res4: Boolean = false

如果用`<:<`将错误地报告`Int`和`Double`在仁一方向上都不具有一致性：

    scala> typeOf[Int] <:< typeOf[Double]
    res5: Boolean = false

    scala> typeOf[Double] <:< typeOf[Int]
    res6: Boolean = false

#### 类型相等

于类型一致性类似，检查两个类型的*相等*很容易。给定两个任意类型，可以用方法`=:=`查看是否两者表示相同的编译期类型。

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> def getType[T: TypeTag](obj: T) = typeOf[T]
    getType: [T](obj: T)(implicit evidence$1: reflect.runtime.universe.TypeTag[T])reflect.runtime.universe.Type

    scala> class A
    defined class A

    scala> val a1 = new A; val a2 = new A
    a1: A = A@cddb2e7
    a2: A = A@2f0c624a

    scala> getType(a1) =:= getType(a2)
    res0: Boolean = true

注意两个实例的*具体类型信息*必须相等。例如，下面的代码片段中，我们有两个不同类型参数的`List`实例。

    scala> getType(List(1,2,3)) =:= getType(List(1.0, 2.0, 3.0))
    res1: Boolean = false

    scala> getType(List(1,2,3)) =:= getType(List(9,8,7))
    res2: Boolean = true

还要注意应该*总是*用`=:=`比较类型的相等。换句话说，绝不要用`==`，因为如果有类型别名出现它就无法检查类型相等，而`=:=`可以：

    scala> type Histogram = List[Int]
    defined type alias Histogram

    scala> typeOf[Histogram] =:= getType(List(4,5,6))
    res3: Boolean = true

    scala> typeOf[Histogram] == getType(List(4,5,6))
    res4: Boolean = false

我们可以看到，`==`错误地报告`Histogram`和`List[Int]`有不同的类型。

#### 向类型查询成员和声明

有了`类型`，还可以向它查询具体的成员和声明。`类型`的*成员*包括所有的字段、方法、类型别名、抽象类型、嵌套类、嵌套对象、嵌套特征，等等。`类型`的*声明*仅包括该类型所表示的类或特征或对象定义在内部声明（非继承）的成员。

为获取某个成员或声明的`符号`，只需要使用`members`或`declarations`方法，这些方法提供该类型相关的定义列表。还有每个方法的单形式，方法`member`和`declaration`。四个方法的签名如下：

    /** The member with given name, either directly declared or inherited, an
      * OverloadedSymbol if several exist, NoSymbol if none exist. */
    def member(name: Universe.Name): Universe.Symbol

    /** The defined or declared members with name name in this type; an
      * OverloadedSymbol if several exist, NoSymbol if none exist. */
    def declaration(name: Universe.Name): Universe.Symbol

    /** A Scope containing all members of this type
      * (directly declared or inherited). */
    def members: Universe.MemberScope // MemberScope is a type of
                                      // Traversable, use higher-order
                                      // functions such as map,
                                      // filter, foreach to query!

    /** A Scope containing the members declared directly on this type. */
    def declarations: Universe.MemberScope // MemberScope is a type of
                                           // Traversable, use higher-order
                                           // functions such as map,
                                           // filter, foreach to query!

例如，要查找`List`的`map`方法，可以这样：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> typeOf[List[_]].member("map": TermName)
    res0: reflect.runtime.universe.Symbol = method map

注意，我们给方法`member`传入了一个`词语名`，因为我们想查找方法。如果我们想查找类型成员，比如`List`的自身类型`Self`，我们需要传入`类型名`：

    scala> typeOf[List[_]].member("Self": TypeName)
    res1: reflect.runtime.universe.Symbol = type Self

我们还有一些有意思的方式查询类型的所有成员和声明。我们可以用方法`members`获取一个表示给定类型继承或声明的所有成员的`Symbol`的`Traversable`，这意味着我们可以用流行的集合高阶函数（比如`foreach`、`filter`、`map`，等等）探索类型的成员。例如，要打印`List`的所有私有成员，只需这样：

    scala> typeOf[List[Int]].members.filter(_.isPrivate).foreach(println _)
    method super$sameElements
    method occCounts
    class CombinationsItr
    class PermutationsItr
    method sequential
    method iterateUntilEmpty

## 树

树是Scala抽象语法（用于表示程序）的基础。树也称为抽象语法树，通常缩写为AST。

Scala反射中生成和使用树的API如下：

1. Scala注解，用树表示参数，暴露在`Annotation.scalaArgs`中（更多信息，参考本指南[注解]({{ site.baseurl }}/overviews/cn/reflection/names-exprs-scopes-more.html)一章）。
2. `reify`，一个特殊方法，能接受表达式并返回该表达式的AST。
3. 宏的编译期反射(在[宏指南]({{ site.baseurl }}/macros/overview.html)有提到)和工具箱的运行期反射都用树作为程序表示的媒介。

需要特别注意树是不可变的除了`pos` (`Position`)、`symbol` (`Symbol`)和`tpe` (`Type`)三个字段，当对树进行类型检查时会对这三个字段赋值。

### `树`的种类

树主要分三类：

1. **`TermTree`的子类** 代表词语，*比如*方法调用表示为`Apply`节点，对象实例化表示为`New`节点，等等。
2. **`TypTree`的子类** 表示程序源码中明确指定的类型，*比如* `List[Int]`解析成`AppliedTypeTree`。*注意*：`TypTree`没拼错，概念上也不同于`TypeTree`-- `TypeTree`是不同的东西。亦即，对于编译器创建的`类型`（*比如*在类型推断阶段）可以用`TypeTree`树包装并集成到程序的AST中。
3. **`SymTree`的子类** 引入或引用定义。引入新定义的例子包括表示类和特征定义的`ClassDef`、表示字段和参数定义的`ValDef`。引用已有定义的例子包括`Ident`，它引用当前范围的已有定义，比如局部变量或方法。

其他可能遇到的树类型通常是句法或短暂的结构。比如`CaseDef`，它包装单个匹配条件，这种节点既不是词语也不是类型，也不携带任何符号。

### 考查树

Scala反射提供少量可视化树的方法，都通过宇提供。给定一棵树，可以：

- 用`show`或`toString`方法打印树所代表的Scala伪代码
- 用方法`showRaw`看类型检查器所操作的原生内部树。

例如，对于下面的树：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> val tree = Apply(Select(Ident(newTermName("x")), newTermName("$plus")), List(Literal(Constant(2))))
    tree: reflect.runtime.universe.Apply = x.$plus(2)

我们可以用方法`show`（或者`toString`也一样）看树代表什么：

    scala> show(tree)
    res0: String = x.$plus(2)

可以看到，`tree`只是把`2`加到词语`x`上。

我们还可以尝试另外的方向。给定Scala表达式，我们可以先获得树，然后用方法`showRaw`去看编译器和类型检查器操作的原生内部树。例如，给定表达式：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> val expr = reify { class Flower { def name = "Rose" } }
    expr: reflect.runtime.universe.Expr[Unit] = ...

这里，`reify`只是接收传入的Scala表达式并返回Scala`表达式`，`表达式`只是包装一个`树`和一个`类型标记`（更多信息请参见本指南[Expr]({{ site.baseurl }}/cn/overviews/reflection/names-exprs-scopes-more.html)
一章）。我们可以这样做来获取`expr`包含的树：

    scala> val tree = expr.tree
    tree: reflect.runtime.universe.Tree =
    {
      class Flower extends AnyRef {
        def <init>() = {
          super.<init>();
          ()
        };
        def name = "Rose"
      };
      ()
    }

我们还可以这样做来考察原生树：

    scala> showRaw(tree)
    res1: String = Block(List(ClassDef(Modifiers(), newTypeName("Flower"), List(), Template(List(Ident(newTypeName("AnyRef"))), emptyValDef, List(DefDef(Modifiers(), nme.CONSTRUCTOR, List(), List(List()), TypeTree(), Block(List(Apply(Select(Super(This(tpnme.EMPTY), tpnme.EMPTY), nme.CONSTRUCTOR), List())), Literal(Constant(())))), DefDef(Modifiers(), newTermName("name"), List(), List(), TypeTree(), Literal(Constant("Rose"))))))), Literal(Constant(())))

### 遍历树

理解了树的结构之后，通常下一步就是从中抽取信息。可以通过*遍历*树抽取信息，有两种做法：

- 通过模式匹配遍历。
- 用`Traverser`的子类。

#### 通过模式匹配遍历

通过模式匹配遍历是最简单也最常用的遍历树的方法。通常，当对树某个节点的状态感兴趣时会通过模式匹配遍历树。比如，假如我们只想获取如下树中`Apply`节点的函数和参数：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> val tree = Apply(Select(Ident(newTermName("x")), newTermName("$plus")), List(Literal(Constant(2))))
    tree: reflect.runtime.universe.Apply = x.$plus(2)

我们可以直接匹配`tree`，如果我们有`Apply`节点就返回`Apply`的函数和参数：

    scala> val (fun, arg) = tree match {
         |     case Apply(fn, a :: Nil) => (fn, a)
         | }
    fun: reflect.runtime.universe.Tree = x.$plus
    arg: reflect.runtime.universe.Tree = 2

我们可以更简洁地获得相同结果，把模式匹配放在左侧：

    scala> val Apply(fun, arg :: Nil) = tree
    fun: reflect.runtime.universe.Tree = x.$plus
    arg: reflect.runtime.universe.Tree = 2

注意`树`通常会很复杂，节点会深深嵌套在其他节点中。一个简单的例证，如果我们给上面的树增加第二个`Apply`节点以给和加`3`：

    scala> val tree = Apply(Select(Apply(Select(Ident(newTermName("x")), newTermName("$plus")), List(Literal(Constant(2)))), newTermName("$plus")), List(Literal(Constant(3))))
    tree: reflect.runtime.universe.Apply = x.$plus(2).$plus(3)

如果我们用和上面相同的模式匹配，我们获得外层`Apply`节点，这个节点包含的函数是上面我们看到的表示`x.$plus(2)`整棵树：

    scala> val Apply(fun, arg :: Nil) = tree
    fun: reflect.runtime.universe.Tree = x.$plus(2).$plus
    arg: reflect.runtime.universe.Tree = 3

    scala> showRaw(fun)
    res3: String = Select(Apply(Select(Ident(newTermName("x")), newTermName("$plus")), List(Literal(Constant(2)))), newTermName("$plus"))

有时需要做一些更丰富的任务，比如遍历整棵树而不停在某个节点，或者收集并考察某种类型的节点，这时使用`Traverser`可能更有利。

#### 用`Traverser`遍历

当需要从上到下遍历整棵树时，用模式匹配遍历不可行-- 那样就需要单独处理在模式匹配中可能遇到的每一种节点。所以，这种情况下通常使用`Traverser`。

`Traverser`确保以宽度优先的方式访问到树的每一个节点。

使用`Traverser`，直接继承`Traverser`并重载方法`traverse`。可以只提供处理感兴趣场景的逻辑。例如，对于前一节的树`x.$plus(2).$plus(3)`，如果我们想收集所有`Apply`节点，我们可以这样：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> val tree = Apply(Select(Apply(Select(Ident(newTermName("x")), newTermName("$plus")), List(Literal(Constant(2)))), newTermName("$plus")), List(Literal(Constant(3))))
    tree: reflect.runtime.universe.Apply = x.$plus(2).$plus(3)

    scala> object traverser extends Traverser {
         |   var applies = List[Apply]()
         |   override def traverse(tree: Tree): Unit = tree match {
         |     case app @ Apply(fun, args) =>
         |       applies = app :: applies
         |       super.traverse(fun)
         |       super.traverseTrees(args)
         |     case _ => super.traverse(tree)
         |   }
         | }
    defined module traverser

上面，我们意图构建给定树中`Apply`节点的列表。

我们通过让子类`traverser`重载`traverse`方法做到这一点，相当于给父类`Traverser`中已经是宽度优先的`traverse`方法*添加*特例。我们的特例只影响匹配模式`Apply(fun, args)`的节点，模式中`fun`是函数（表示为`树`）而`args`是参数列表（表示为`树`的列表）。

当树匹配模式时（*比如*，当我们有一个`Apply`节点），我们就把它加入到`List[Apply]`, `applies`并继续遍历。

注意到，在这个匹配中，对`Apply`中包含的`fun`我们调用`super.traverse`，而对参数列表`args`我们调用`super.traverseTrees`（实质上和`super.traverse`相同，只不过适用于`List[Tree]`而不是`Tree`）。这两个调用中，我们的目的都很简单---- 我们想确定使用`Traverser`中默认的`traverse`方法，因为我们不知道代表函数的那个`Tree`是否包含我们的`Apply`模式，也就是说，我们想遍历整颗子树。因为父类`Traverser`调用`this.traverse`，传入每一个嵌套的子树，对每个匹配我们`Apply`模式的子树，我们的自定义`traverse`方法一定会被调用。

为触发`traverse`并看到匹配`Apply`节点的结果`列表`，直接：

    scala> traverser.traverse(tree)

    scala> traverser.applies
    res0: List[reflect.runtime.universe.Apply] = List(x.$plus(2), x.$plus(2).$plus(3))

### 创建树

在运行时反射中，需要手动构建树。不过，工具箱的运行时编译和宏的编译期反射都使用树作为程序表示的媒介。在这些场景中，有三种值得推荐的树创建方法：

1. 通过方法`reify`（尽可能优先选择）。
2. 通过`ToolBox`的`parse`方法。
3. 手动构建（不推荐）。

#### 通过`reify`创建树

方法`reify`直接以Scala表达式作为参数，生成参数的`树`类型表示。

Scala反射中推荐使用方法`reify`创建树。让我们开始一个小例子去看看为什么：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> { val tree = reify(println(2)).tree; showRaw(tree) }
    res0: String = Apply(Select(Select(This(newTypeName("scala")), newTermName("Predef")), newTermName("println")), List(Literal(Constant(2))))

例子中，我们直接用`println(2)`调用`reify`方法-- 也就是说，我们把表达式`println(2)`转化成对应的树表示。然后，我们输出原生树。注意到方法`println`转换成了`scala.Predef.println`。这种转换确保不管在哪里使用`reify`的结果都不会影响这个片段的行为。

从它保持标识符的绑定的意义而言，这种树创建方法很*干净*。

##### 拼接树

使用`reify`还可以从小树构建大树，用`Expr.splice`。

*注意：* `Expr`是`reify`的返回类型，可以认为它是一个简单包装，包含*类型*`树`、`类型标记`和一些具体化相关的方法，比如`splice`。更多有关`Expr`的信息，参考[本指南相关章节]({{ site.baseurl}}/cn/overviews/reflection/annotations-names-scopes.html)。

例如，我们试着用`splice`构建一个表示`println(2)`的树：

    scala> val x = reify(2)
    x: reflect.runtime.universe.Expr[Int(2)] = Expr[Int(2)](2)

    scala> reify(println(x.splice))
    res1: reflect.runtime.universe.Expr[Unit] = Expr[Unit](scala.this.Predef.println(2))

例子中，我们分别`具体化``2`和`println`，直接`拼接`一个到另一个。

不过，要注意对`reify`有个要求，`reify`的参数必须有效并且是有类型的Scala代码。如果我们想抽象化`println`自己而不是`println`的参数，就行不通：

    scala> val fn = reify(println)
    fn: reflect.runtime.universe.Expr[Unit] = Expr[Unit](scala.this.Predef.println())

    scala> reify(fn.splice(2))
    <console>:12: error: Unit does not take parameters
                reify(fn.splice(2))
                                ^

我们可以看到，当我们实际想捕获所调用函数的名称时，编译器认为我们想具体化对`println`的无参调用。

这类用法现在用`reify`还无法表达。

#### 通过`ToolBox`上的`parse`创建树

`Toolbox`用于类型检查、编译和执行抽象语法树。工具箱还可以用来解析字符串为AST。

*注意：*使用工具箱需要`scala-compiler.jar`在类路径上。

我们看看`parse`如何处理前一节中的`println`例子：

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> import scala.tools.reflect.ToolBox
    import scala.tools.reflect.ToolBox

    scala> val tb = runtimeMirror(getClass.getClassLoader).mkToolBox()
    tb: scala.tools.reflect.ToolBox[reflect.runtime.universe.type] = scala.tools.reflect.ToolBoxFactory$ToolBoxImpl@7bc979dd

    scala> showRaw(tb.parse("println(2)"))
    res2: String = Apply(Ident(newTermName("println")), List(Literal(Constant(2))))

值得注意的是，不像`reify`，工具箱没有类型化要求的限制--尽管这种灵活性牺牲了健壮性。也就是说，这里我们可以看到，不像`reify`，`parse`没有反映`println`应该绑定到标准`println`方法这一事实。

*注意:*使用宏时，不应该用`ToolBox.parse`。这是因为在宏的上下文已经有一个`parse`方法。例如：

    scala> import language.experimental.macros
    import language.experimental.macros

    scala> def impl(c: scala.reflect.macros.Context) = c.Expr[Unit](c.parse("println(2)"))
    impl: (c: scala.reflect.macros.Context)c.Expr[Unit]

    scala> def test = macro impl
    test: Unit

    scala> test
    2

##### 工具箱的类型检查

前面提过，`ToolBox`不仅能从字符串构建树，还能用于类型检查、编译和执行树。

除了提领程序结构，树还持有程序语义有关的重要信息，程序编码成`symbol`（符号赋值给树，引入或引用定义）和`tpe`（树的类型）。这些字段默认为空，不过类型检查填充这些字段。

当使用运行时反射框架时，`ToolBox.typeCheck`实现了类型检查。当使用宏时，可以在运行期使用`Context.typeCheck`方法。

    scala> import scala.reflect.runtime.universe._
    import scala.reflect.runtime.universe._

    scala> val tree = reify { "test".length }.tree
    tree: reflect.runtime.universe.Tree = "test".length()

    scala> import scala.tools.reflect.ToolBox
    import scala.tools.reflect.ToolBox

    scala> val tb = runtimeMirror(getClass.getClassLoader).mkToolBox()
    tb: scala.tools.reflect.ToolBox[reflect.runtime.universe.type] = ...

    scala> val ttree = tb.typeCheck(tree)
    ttree: tb.u.Tree = "test".length()

    scala> ttree.tpe
    res5: tb.u.Type = Int

    scala> ttree.symbol
    res6: tb.u.Symbol = method length

这里我们创建代表调用`"test".length`的树，并用`ToolBox` `tb`的`typeCheck`方法对树进行类型检查。如我们所见，`ttree`获得了正确的类型`Int`，并且正确设置了`Symbol`。

#### 通过手工构造方式创建树

如果其他方法都不行，还可以手工构造树。这是最低级的树创建方法，应该在其他方法都不行时才尝试。这种方法有时比`parse`提供更多的灵活性，尽管这种灵活性需要大量描述并且很脆弱。

我们前面的`println(2)`例子可以如下进行手工构造：

    scala> Apply(Ident(newTermName("println")), List(Literal(Constant(2))))
    res0: reflect.runtime.universe.Apply = println(2)

这种技巧的典型例子是当目标树需要从动态创建的部分装配而得，这时候把各部分隔离起来没有意义。这种情况下，`具体化`很可能不适用，因为它要去它的参数是类型化的。`parse`可能也不行，因为通常树是在子表达式级别装配，单个的部分不能表达成Scala的源码。
