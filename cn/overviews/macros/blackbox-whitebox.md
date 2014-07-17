---
layout: overview-large
title: 黑盒对白盒

disqus: true

partof: macros
num: 2
language: cn
---
<span class="label warning" style="float: right;">试验中</span>

**Eugene Burmako**

把宏氛围黑盒和白盒是在近期Scala2.11的里程碑构建实现的，从2.11.0-M7开始（不过，在2.11.0-M8语法经历了一些变化，所以本文档对2.11一些早期的里程碑构建不适用）。黑盒白盒分离在Scala2.10.x和宏乐园没有实现。遵照[http://www.scala-lang.org/download](http://www.scala-lang.org/download)的指示下载并使用2.11最新的里程碑构建。

## 为什么宏工作？

随着宏成为2.10官方发布的一部分，研究和工业界的程序员发现了以创新的方式使用宏解决各种问题，极大扩展了我们最初的期望。

实际上，在Scala2.10发布后的仅仅几个月宏就很快成为我们生态系统的一个重要不发，当宏被引入为实验性能力时，我们开了Scala语言团队会议并决定标准化宏并且在2.12之前使之成为Scala一个功能完善的特性。

因为宏的口味很丰富，所以我们决定自信检查以决定哪些应该放入标准。这意味着回答几个重要的问题。为什么宏工作的这么好？为什么人们用宏？

我们的推测是这是因为建立在熟悉的类型方法调用概念之上的定义宏所表达的元编程概念很难理解。感谢的是，用户写的代码能有更多的含义而不臃肿或丢失可理解性。

## 黑盒和白盒宏

不过，有时定义宏超越"仅仅是普通方法"的概念。例如，可能宏展开所生成表达式的类型比宏返回类型更具体。在Scala2.10，这种展开会返回它的具体类型，如Stack Overflow的文章["Static return type of Scala macros"](http://stackoverflow.com/questions/13669974/static-return-type-of-scala-macros)。 

这种奇怪的特点提供附加的灵活性，使[fake type providers](http://meta.plasm.us/posts/2013/07/11/fake-type-providers-part-2/), [extended vanilla materialization](/sips/pending/source-locations.html), [fundep materialization](/overviews/macros/implicits.html#fundep_materialization) and [extractor macros](https://github.com/paulp/scala/commit/84a335916556cb0fe939d1c51f27d80d9cf980dc), but it also sacrifices clarity - both for humans and for machines.

To concretize the crucial distinction between macros that behave just like normal methods and macros that refine their return types, we introduce the notions of blackbox macros and whitebox macros. Macros that faithfully follow their type signatures are called **blackbox macros** as their implementations are irrelevant to understanding their behaviour (could be treated as black boxes). Macros that can't have precise signatures in Scala's type system are called **whitebox macros** (whitebox def macros do have signatures, but these signatures are only approximations).

We recognize the importance of both blackbox and whitebox macros, however we feel more confidence in blackbox macros, because they are easier to explain, specify and support. Therefore our plans to standardize macros in Scala 2.12 only include blackbox macros. In future releases of Scala we might make whitebox macros non-experimental as well, but it is too early to tell.

## Codifying the distinction

In the 2.11 release, we take first step of standardization by expressing the distinction between blackbox and whitebox macros in signatures of def macros, so that `scalac` can treat such macros differently. This is just a preparatory step, so both blackbox and whitebox macros remain experimental in Scala 2.11.

We express the distinction by replacing `scala.reflect.macros.Context` with `scala.reflect.macros.blackbox.Context` and `scala.reflect.macros.whitebox.Context`. If a macro impl is defined with `blackbox.Context` as its first argument, then macro defs that are using it are considered blackbox, and analogously for `whitebox.Context`. Of course, the vanilla `Context` is still there for compatibility reasons, but it issues a deprecation warning encouraging to choose between blackbox and whitebox macros.

Blackbox def macros are treated differently from def macros of Scala 2.10. The following restrictions are applied to them by the Scala typechecker:

1. When an application of a blackbox macro expands into tree `x`, the expansion is wrapped into a type ascription `(x: T)`, where `T` is the declared return type of the blackbox macro with type arguments and path dependencies applied in consistency with the particular macro application being expanded. This invalidates blackbox macros as an implementation vehicle of [type providers](http://meta.plasm.us/posts/2013/07/11/fake-type-providers-part-2/).
1. When an application of a blackbox macro still has undetermined type parameters after Scala's type inference algorithm has finished working, these type parameters are inferred forcedly, in exactly the same manner as type inference happens for normal methods. This makes it impossible for blackbox macros to influence type inference, prohibiting [fundep materialization](/overviews/macros/implicits.html#fundep_materialization).
1. When an application of a blackbox macro is used as an implicit candidate, no expansion is performed until the macro is selected as the result of the implicit search. This makes it impossible to [dynamically calculate availability of implicit macros](/sips/pending/source-locations.html).
1. When an application of a blackbox macro is used as an extractor in a pattern match, it triggers an unconditional compiler error, preventing [customizations of pattern matching](https://github.com/paulp/scala/commit/84a335916556cb0fe939d1c51f27d80d9cf980dc) implemented with macros.

Whitebox def macros work exactly like def macros used to work in Scala 2.10. No restrictions of any kind get applied, so everything that could be done with macros in 2.10 should be possible in 2.11.
