---
title: Kotlin作用域函数(Scope Functions)
tags: 基础
categories: Kotlin
thumbnail: 'https://dreamweaver.img.we1code.cn/kotlin-base-syntax.jpg'
date: 2019-03-22 00:00:00
top: 1
---

## 前言

Kotlin中有5种[作用域函数](https://www.kotlincn.net/docs/reference/scope-functions.html)，分别是：  
> let, run, with, apply, and also  

它们并没有任何特性，但是使用他们可以让我们的代码更加简洁，具备更好的可读性。我们可以在[这](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/util/Standard.kt)找到它们的源码。下面我将分析这些方法的区别。

<!--more-->

## 分析

可以看到其实文档上已经有说明了，它们之间的区别在于：

- 引用上下文的方式
- 返回值

每个作用域函数使用两种访问上下文对象的方式之一：作为lambda接收器（this）或作为lambda自变量（it）。两者都提供相同的功能，因此我将描述在每种情况下的利弊，并提供有关其用法的建议。

### this

run, with, apply通过关键字this将上下文作为lambda的接收器。因此，在其lambda中，该对象的用法跟在普通类函数中一样。在大多数情况下，可以省略this直接使用接收器对象的成员变量，从而使代码更加简洁。但是如果省略了this，就很难区分出接收器对象的成员变量和外部变量或方法。因此，如果在lambda中操作对象，像是调用它的方法或是给属性赋值，推荐使用这三种作用域函数。

### it

let, also将上下文对象作为lambda表达式的参数。如果在作用域中没有定义参数名，则默认为it。it比this更短，使用it的表达式更加易读。但是在调用对象方法和参数时，不能像this一样隐式调用。因此，
当对象被用作方法参数时，推荐使用这两种作用域函数。

### 返回值

- apply, also 返回上下文对象
- let, run, with 返回lambda表达式的结果

## 如何选择使用

为了更方便我们选择使用，官方给出了不少示例

### let

let 可以被用在在调用链的结果上执行一个或者多个方法。

```kotlin
val numbers = mutableListOf("one", "two", "three", "four", "five")
numbers.map { it.length }.filter { it > 3 }.let {
    println(it)
    // and more function calls if needed
}
```

如果代码块中只有一个方法，并将it作为参数，可以将lambda简写成(::)

```kotlin
val numbers = mutableListOf("one", "two", "three", "four", "five")
numbers.map { it.length }.filter { it > 3 }.let(::println)
```

let经常被用在执行non-null values的代码块。要对非空对象进行操作，需要使用安全操作符?.

```kotlin
val str: String? = "Hello"
//processNonNullString(str)       // compilation error: str can be null
val length = str?.let {
    println("let() called on $it")
    processNonNullString(it)      // OK: 'it' is not null inside '?.let { }'
    it.length
}
```

### with

官方建议使用with不返回结果("with this object, do the following.")，如下：

```kotlin
val numbers = mutableListOf("one", "two", "three")
with(numbers) {
    println("'with' is called with argument $this")
    println("It contains $size elements")
}
```

### run

当你的lambda表达式中含有对象的初始化和返回值的计算时run就很有用了。

```kotlin
val service = MultiportService("https://example.kotlinlang.org", 80)

val result = service.run {
    port = 8080
    query(prepareRequest() + " to port $port")
}

// the same code written with let() function:
val letResult = service.let {
    it.port = 8080
    it.query(it.prepareRequest() + " to port ${it.port}")
}
```

### apply

主要用作操作接收对象的成员，最典型的例子就是对象的配置。可以理解成"apply the following assignments to the object"。

```kotlin
val adam = Person("Adam").apply {
    age = 32
    city = "London"
}
```

将接收对象作为返回值，你能很容易的将apply置入调用链中做更为复杂的处理。

### also

also适用于将上下文对象作为参数的操作。also可以用作不改变对象的额外操作，像是打印日志或打印调试信息。通常，你可以从调用链中移除also而不会破坏程序原有的逻辑。可以将also理解成"and also do the following"。

### 总结

可以参考下面的流程图  
![flow](http://dreamweaver.img.we1code.cn/kotlin_standard_function_use_flow.png)

官方给出的对比表格：  

Function | Object reference | Return value | Is extension function
-|-|-|-
let | it | Lambda result | Yes
run | this | Lambda result | Yes
run | - | Lambda result | No: called without the context object
with | this | Lambda result | No: takes the context object as an argument
apply | this | Context object | Yes
also | it | Context object | Yes
  
一句话总结：

- 对非空对象执行lambda：let
- 将表达式作为局部变量引入：let
- 对象配置：apply
- 对象配置并计算结果：run
- 需要表达式的运行语句：no-extension run
- 额外的操作：also
- 将对象的调用方法分组：with

## 总结语

尽管作用域方法可以使代码更为简洁，但是请避免过度使用它：这样会减少代码的可读性并且可能导致错误。避免嵌套作用域函数，链式调用它们的时候要格外小心。

## 参考

[作用域函数](https://www.kotlincn.net/docs/reference/scope-functions.html)  
[kotlin standard function](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/util/Standard.kt)
