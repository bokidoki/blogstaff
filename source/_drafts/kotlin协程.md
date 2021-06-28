---
title: 如何理解Kotlin协程
tags: syntax
categories: Kotlin
date: 2019/05/21
img: 
---

使用Coroutines可以写出可读性和可维护性高的异步代码。使用suspend关键字即可使用到Kotlin协程。  
在这篇文章中，你将简单了解到携程和suspend修饰函数的基本使用方法，意在分享我对协程的基本看法，所以对它的实现不做深入探讨。  

### coroutine到底是什么？

Kotlin团队将coroutines定义成轻量级的线程。它们是真正线程可执行的一系列的任务。最有趣的地方是线程可以停止执行一个协程，转而去做其它的工作，稍后再继续执行，甚至其它的线程也可以去执行它。  
因此更准确的说，协程不是一个任务而是一系列以特定的顺序执行的子任务。即使代码在顺序块中，每次调用suspending function都会在协程中定义新的子任务的开始。

### Suspending 函数

你可能会发现delay和HttpClient.post需要等待某事或在返回之前进行某些工作，他们都被suspend关键字标记了。

```kotlin
suspend fun delay(timeMillis: Long) {...}

suspend fun someNetWorkCall(): SomeType {...}
```

这些方法被称为suspending函数。正如我们所见：

>Suspending funcrions may suspend the execution of the current coroutine without blocking the current thread.

这意味着当调用suspending函数时

### 探讨coroutines替换Rxjava方案

### 什么是协程，进程，线程和协程之间又有什么关联呢？

进程是操作系统中最小的资源管理单元，线程则是进程中事务的最小执行单元，而协程则是存在于线程中的单元

### 由简入繁

先看看基础使用方法

```kotlin
object Syntax {
    @JvmStatic
    fun main(args: Array<String>) {
        val s = runBlocking {
            delay(1000L)
            "World!"
        }
        println(s)
        println("Hello,")
    }
}

//打印结果
//World!
//after 2 seconds
//Hello,
```

看看官方文档是怎么描述runBlocking

> Runs a new coroutine and **blocks** the current thread _interruptibly_ until its completion.

开始一个新的协程，阻塞线程一直到任务完成

我们总结一下创建协程的方式有哪些？

```kotlin

val main = GlobalScope.launch {}
```

GlobalScope的官方文档

> Global scope is used to launch top-level coroutines which are operating on the whole application lifetime and are not cancelled prematurely.
> Application code usually should use an application-defined [CoroutineScope]. Using [async][CoroutineScope.async] or [launch][CoroutineScope.launch] on the instance of [GlobalScope] is highly discouraged.

官方推荐我们使用CoroutineScope.async，CoroutineScope.launch，这两个方法需要传参CoroutineContext

辣么问题又来了CoroutineContext又是什么？

> Persistent context for the coroutine. It is an indexed set of [Element] instances.  
> An indexed set is a mix between a set and a map.  
> Every element in this set has a unique [Key]. Keys are compared _by reference_.

这个就是协程的上下文，它是Element实例的有序集合，有序集合是set和map的混合体，集合中的每个element都有一个独特的Key值。  
其中包含get(key: Key[E])方法返回一个Element，fold()不知道干啥的，plus()，minusKey()不知道干啥的

emm，先不管了，看看Dispatchers，在Dispatchers中可以看到Default，Main，Unconfined，IO这几个CoroutineDispatcher实例，它们的作用在注释中已有了说明，

- Default 默认的协程调度器，如果在协程作用域中

问题：

- GlobalScope 是什么？作用？ 全局作用域
- lanuch 做了什么操作

### Kotlin协程基础

#### 使用方法

```kotlin

GlobalScope

```

CoroutineScope

> Defines a scope for new coroutines. Every coroutine builderis an extension on [CoroutineScope] and inherits its [coroutineContext][CoroutineScope.coroutineContext] to automatically propagate both context elements and cancellation.The best ways to obtain a standalone instance of the scope are [CoroutineScope()] and [MainScope()] factory functions.Additional context elements can be appended to the scope using the [plus][CoroutineScope.plus] operator.Manual implementation of this interface is not recommended, implementation by delegation should be preferred instead.By convention, the [context of a scope][CoroutineScope.coroutineContext] should contain an instance of a [job][Job] to enforce structured concurrency.Every coroutine builder (like [launch][CoroutineScope.launch], [async][CoroutineScope.async], etc)and every scoping function (like [coroutineScope], [withContext], etc) provides _its own_ scopewith its own [Job] instance into the inner block of code it runs.By convention, they all wait for all the coroutines inside their block to complete before completing themselves,thus enforcing the discipline of structured concurrency.[CoroutineScope] should be implemented (or used as a field) on entities with a well-defined lifecycle that are responsiblefor launching children coroutines. Example of such entity on Android is Activity.Usage of this interface may look like this:

coroutines的官方注释如上，它透漏出了几点信息

- 可以用CoroutineScope()和MainScope()得到coroutines对象
- 不要去实现CoroutineScope，推荐使用代理方法
- CoroutineScope应该在明确定义生命周期的地方使用

CoroutineContext

> Persistent context for the coroutine. It is an indexed set of [Element] instances.An indexed set is a mix between a set and a map.Every element in this set has a unique [Key].

CoroutineScope.launch

> Launches a new coroutine without blocking the current thread and returns a reference to the coroutine as a [Job].

启动一个新的协程而不阻塞当前线程，以Job对象的形式返回协程的引用

launch方法实际上有三个参数，CoroutineContext，默认值为EmptyCoroutineContext，CoroutineStart默认值为CoroutineStart.DEFAULT，这个CoroutineStart是个枚举类，有四个值DEFAULT、LAZY、ATOMIC、UNDISPATCHED

Element实例的索引集，索引集是集和映射之间的混合，集合中的每一个element都是唯一的

#### coroutines with viewmodel

#### coroutines with lifecycle

## 参考

[漫画：什么是协程？](https://www.itcodemonkey.com/article/4620.html)
