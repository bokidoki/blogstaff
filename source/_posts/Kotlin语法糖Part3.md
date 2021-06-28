---
title: Kotlin语法糖 Part3
tags: 基础
categories: Kotlin
thumbnail: https://dreamweaver.img.we1code.cn/Kotlin%E8%AF%AD%E6%B3%95%E7%B3%96Part3.jpg
date: 2019-04-05 00:00:00
top: 0
---


### 前言

在前面的两篇文章中，我们了解到了：

- sealed
- when()
- with()
- inline function and reified type

在这章中，我会给大家分享我是如何使用Kotlin委托机制的。

<!--more-->

## Kotlin的委托机制

Kotlin有一个内置的[委托模式](https://www.kotlincn.net/docs/reference/delegation.html)。在一些书中也提及委托模式是实现继承的一个很好的替代方式，在Kotlin使用它进行聚合非常容易：

```kotlin
interface Navigable {
    val onNavigationClick: (()->Unit)?
}

interface Searchable {
    val searchText:String
}

class Component(navigation: Navigable, searchable: Searchable): Navigable by navigable, Searchabe by searchable
```

使用by关键字，即可委托navigation、searchable的所有行为。比起Java，Kotlin减少了大量的模板代码。如果你把interface标记为internal，你会发现编译不能通过

> 'public' function exposes its 'internal' parameter type xxx  

Kotlin编译器不允许暴露模块的内部组件，如果想在不暴露Navigable，Searchable的前提下解决这个问题，你只需要做：

- 移除构造器
- 使用组合代替聚合
- 定义包含Navigable和Searchable方法名的ComponentInterface

```kotlin
interface ComponentInterface {
    val onNavigationClick: (()->Unit)?

    var searchText: String
}

class Component: ComponentInterface {
    private val navigable: Navigable = NavigableImpl()

    private val searchable: Searchable = SearchableImpl()

    override val onNavigationClick: (()->Unit)?
        get() = navigable.onNavigationClick

    override var searchText: String = ""
        get() = searchable.searchText
        set(value) {
            field = value
            searchable.searchText = value
        }
}
```

经过改造后，你发现代码从零模板的聚合变成了看上去很讨厌的组合方式。但是先别哭，Kotlin总能给你带来愉悦的编码体验。Kotlin不仅支持使用by关键字对指定对象进行方法委派，还具有委托属性的机制。你可能已经在使用lazy关键字初始化对象时接触到了它的这一机制了。

```kotlin
private val lazyProperty by lazy { "" }
```

怎么从使用lazy()上来改造上面组合的代码呢，请看代码：

```kotlin
class ReferencedProperty<T>(private val get: () -> T,
                            private val set: (T) -> Unit = {}) {

    operator fun getValue(thisRef: Any?,
                          property: KProperty<*>): T = get()

    operator fun setValue(thisRef: Any?,
                          property: KProperty<*>,
                          value: T) = set(value)
}

fun <T> ref(property: KMutableProperty0<T>) = ReferencedProperty(property::get,
                                                                 property::set)

fun <T> ref(property: KProperty0<T>) = ReferencedProperty(property::get)
```

ReferencedProperty用两个方法作为参数，并定义了两个函数：

- get函数返回泛型T
- set函数以T作为参数
- getValue()调用get()
- setValue()调用set()

最重要的一点是你需要知道操作符用到了属性代理机制。在ReferencedProperty类下，你会发现两个返回ReferencedProperty的泛型方法，第一个用于var，第二个用于val。  
现在让我们用ref()简化代码

```kotlin
class Component : ComponentInterface {

    private val navigable: Navigable = NavigableImpl()
    private val searchable: Searchable = SearchableImpl()

    override val onNavigationClick by ref(navigable::onNavigationClick)
    override var searchText by ref(searchable::searchText)
}
```

希望看完这三章你能有些许收获。

### 参考

[6 magic sugars that can make your Kotlin codebase happier — Part 3](https://medium.com/grand-parade/6-magic-sugars-that-can-make-your-kotlin-codebase-happier-part-3-6319a451cd5d)
