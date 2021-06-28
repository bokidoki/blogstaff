---
title: kotlin协程其一
categories: Android
top: 0
date: 2020-06-29 22:48:35
tags:
thumbnail:
---

> 协程被定义为轻量级的线程，线程内的任务调度单元，多个协程共享一份线程的内存资源，因此省去了切换线程的开销，更为高效。以前没有接触过协程，自从Kotlin出了协程库后，只浅尝辄止过，想通过从api的调用到对kotlinx.coroutines库源码分析，由浅入深的彻底理解协程。本文是koltin协程的第一章，总结一下Kotlin协程的基本用法。

```groovy
// 版本
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.3.0-M1"
```

### 基本类型

- CoroutineScope
- CoroutineContext
  - Element
    - Job
      - Deferred

#### CoroutineContext

> Persistent context for the coroutine. It is an indexed set of [Element] instances.
An indexed set is a mix between a set and a map.
Every element in this set has a unique [Key].  
> CoroutineContext 是协程持久性的上下文。它是一个有索引的set结构，它的索引和值都是独一无二的。它重写了[索引访问操作符和算术运算符+](https://www.kotlincn.net/docs/reference/operator-overloading.html)

CoroutineContext中的函数:

```kotlin
fun <R> fold(initial: R, operation: (R, Element) -> R): R
```

> Accumulates entries of this context starting with [initial] value and applying [operation]from left to right to current accumulator value and each element of this context.  
> 从[initial]值开始累加此上下文的条目，并从左到右对当前累加器值和此上下文的每个元素应用[operation]。

这两个函数的描述还是挺抽象的，不过没事，先放这里，待会回过头再看。

```kotlin
public operator fun plus(context: CoroutineContext): CoroutineContext
```

>Returns a context containing elements from this context and elements from  other [context].The elements from this context with the same key as in the other one are dropped.  
>返回一个上下文，其中包含该上下文中的元素以及其他[context]中的元素。该上下文中与其他元素具有相同键的元素将被删除。

#### CoroutineContext.Element

> An element of the [CoroutineContext]. An element of the coroutine context is a singleton context by itself.
> Element是[CoroutineContext]中的一个元素。协程上下文的元素本身就是单例上下文。

Elment有个内部变量Key<*>，并且重写几个方法如下：

```kotlin
// 返回与内部key值相同的元素，没有则返回null
public override operator fun <E : Element> get(key: Key<E>): E? =
    @Suppress("UNCHECKED_CAST")
    if (this.key == key) this as E else null

// 调用operation把自己传进去
public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
    operation(initial, this)

// 移除相关的key，如果有返回一个空的协程上下文，没找到返回自己
public override fun minusKey(key: Key<*>): CoroutineContext =
    if (this.key == key) EmptyCoroutineContext else this
```

这里遇到了第一个CoroutineContext的实现类EmptyCoroutineContext，这个不多说了，大家自己看看好了。

```kotlin
@SinceKotlin("1.3")
public object EmptyCoroutineContext : CoroutineContext, Serializable {
    private const val serialVersionUID: Long = 0
    private fun readResolve(): Any = EmptyCoroutineContext

    public override fun <E : Element> get(key: Key<E>): E? = null
    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R = initial
    public override fun plus(context: CoroutineContext): CoroutineContext = context
    public override fun minusKey(key: Key<*>): CoroutineContext = this
    public override fun hashCode(): Int = 0
    public override fun toString(): String = "EmptyCoroutineContext"
}
```

#### Job

Job继承于CoroutineContext.Element，说明它也是CoroutineContext的一个元素。它是对协程中的后台任务的抽象。

> A background job. Conceptually, a job is a cancellable thing with a life-cycle that
culminates in its completion.  
> 后台工作。从概念上讲，Job是可以取消的，它具有生命周期并将最终执行完成。

先放张官方的Job生命周期状态转换图

```kotlin
 * ```
 *                                       wait children
 * +-----+ start  +--------+ complete   +-------------+  finish  +-----------+
 * | New | -----> | Active | ---------> | Completing  | -------> | Completed |
 * +-----+        +--------+            +-------------+          +-----------+
 *                  |  cancel / fail       |
 *                  |     +----------------+
 *                  |     |
 *                  V     V
 *              +------------+                           finish  +-----------+
 *              | Cancelling | --------------------------------> | Cancelled |
 *              +------------+                                   +-----------+
 * ```
```

继续看文档

> Jobs can be arranged into parent-child hierarchies where cancellation of a parent leads to immediate cancellation of all its [children]  
>可以将Job安排到父子层次结构中，取消父级会立即取消所有[子级]  
> Failure or cancellation of a child with an exception other than [CancellationException] immediately cancels its parent.  
> 失败或取消除[CancellationException]以外的异常的子级会立即取消其父级。  
> This way, a parent can [cancel] its own children (including all their children recursively) without cancelling itself.  
> 这样，父母就可以[取消]自己的孩子（递归地包括所有孩子）而无需取消自己。  
> **Coroutine job** is created with [launch][CoroutineScope.launch] coroutine builder.  
> Cotoutine job可以通过CoroutineScope.launch创建（最基本构造Job实例的方法）。

Job的状态表

```kotlin
| **State**                        | [isActive] | [isCompleted] | [isCancelled] |
| -------------------------------- | ---------- | ------------- | ------------- |
| _New_ (optional initial state)   | `false`    | `false`       | `false`       |
| _Active_ (default initial state) | `true`     | `false`       | `false`       |
| _Completing_ (transient state)   | `true`     | `false`       | `false`       |
| _Cancelling_ (transient state)   | `false`    | `false`       | `true`        |
| _Cancelled_ (final state)        | `false`    | `true`        | `true`        |
| _Completed_ (final state)        | `false`    | `true`        | `false`       |
```

> 通常来说Job创建出来就是_active_状态，当coroutine的建造者的可选参数start为CoroutineStart.LAZY时，coroutine的状态为_new_，并可以通过start/join将状态转换为_active_。

#### Deferred

> Deferred value is a non-blocking cancellable future &mdash; it is a [Job] with a result.  
> Deferred继承Job并能返回一个结果。

### 使用方法

一般使用协程通过以下两种方式：

```kotlin
// 拖尾lambda表达式 总是忘记这个说法 记录一下
GlobalScope.launch {}
GlobalScope.async {}
```

这样子我们就使用了默认参数的协程，这两个函数的完全体如下：

```kotlin
// 创建一个不会阻塞当前的线程的协程
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}

// 创建一个coroutine将延迟返回它的结果
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

两者的区别，launch返回Job，async返回Deferred

> 类型解析

- Deferred  
    Deferred继承与Job并能返回一个结果。  
    通过CoroutineScope.async或者CompletableDeferred创建。
    Deferred与Job有相同的状态机并可以追踪成功或者失败的结果。结果在isCompleted为true时可用并可以通过await函数追踪到结果，如果deferred失败则会抛出异常(怎么才算是失败？)。所有与Deferred相关的方法或者由它派生的接口都是线程安全的。
