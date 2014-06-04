---
layout: overview-large
title: 线程安全

disqus: true

partof: reflection
num: 6
outof: 7
language: cn
---

<span class="label important" style="float: right;">试验中</span>

**Eugene Burmako**

很遗憾，Scala2.10.0中反射不是线程安全。
有个JIRA问题[SI-6240](https://issues.scala-lang.org/browse/SI-6240)用于跟踪我们的进展以及查看技术细节，这里有最新进展的简明总结。

<p><span class="label success">新</span>&nbsp;线程安全问题已经在Scala2.11.0-RC1中解决，但我们还准备保持现有文件，因为这个问题在Scala2.10.x中依旧存在，并且我们现在还没有回迁相关修复的具体计划。</p>

当前，我们知道反射相关的两种竞争。首先，反射初始化（当首次访问`scala.reflect.runtime.universe`所调用的代码）不能安全地从多个线程调用。其次，符号初始化（首次访问符号标准或类型签名所调用的代码）也不是线程安全。下面是典型的清单：

    java.lang.NullPointerException:
    at s.r.i.Types$TypeRef.computeHashCode(Types.scala:2332)
    at s.r.i.Types$UniqueType.<init>(Types.scala:1274)
    at s.r.i.Types$TypeRef.<init>(Types.scala:2315)
    at s.r.i.Types$NoArgsTypeRef.<init>(Types.scala:2107)
    at s.r.i.Types$ModuleTypeRef.<init>(Types.scala:2078)
    at s.r.i.Types$PackageTypeRef.<init>(Types.scala:2095)
    at s.r.i.Types$TypeRef$.apply(Types.scala:2516)
    at s.r.i.Types$class.typeRef(Types.scala:3577)
    at s.r.i.SymbolTable.typeRef(SymbolTable.scala:13)
    at s.r.i.Symbols$TypeSymbol.newTypeRef(Symbols.scala:2754)

好消息是编译期反射（通过`scala.reflect.macros.Context`暴露给宏）没有运行期反射那样易受线程问题影响。第一个原因是宏开始运行时运行期反射的宇已经初始化，这也就排除了第一个竞争条件。第二个原因是编译器从来都不是线程安全，因此没有工具会期望并行运行。尽管如此，如果你的宏起多线程你还要小心。

对运行时反射情况就糟得多了。初始化`scala.reflect.runtime.universe`时首次调用反射初始化，这种初始化可能间接发生。最明显的例子是调用`TypeTag`上下文绑定的方法有潜在问题，因为调用那样的的方法Scala通常需要构建自动生成的类型标记，这需要创建一个类型，从而需要初始化反射宇。一个推论是如果你不采取特别的措施，你就不能放心地在测试中使用基于`TypeTag`的方法，因为很多工具，比如SBT，并行运行测试。

结果：
* 如果你写一个宏，它不明确地创建线程，你非常安全。
* 混杂着线程或actor的运行时反射可能危险。
* 多个线程调用`TypeTag`上下文绑定的方法可能导致不确定的结果。
* 查看[SI-6240](https://issues.scala-lang.org/browse/SI-6240)以获取我们对此问题的进展。
