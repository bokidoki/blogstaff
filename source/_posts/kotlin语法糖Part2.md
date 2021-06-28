---
title: Kotlin语法糖 Part2
tags: 基础
categories: Kotlin
date: 2019-03-30
thumbnail: https://dreamweaver.img.we1code.cn/Kotlin%E8%AF%AD%E6%B3%95%E7%B3%96Part2.jpg
top: 0
---

## 前言

在第一章中，我们学会了如何使用sealed classes，以及when()配合Pair或Triple使用做多重条件判断。  在这一章中，我想跟大家分享一下with()和inline reified的基本使用。  

<!--more-->

## 使用with()函数

with()函数位于[Standard.kt](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/util/Standard.kt)，是Kotlin标准函数之一，大家可以看看，掌握好这些函数对于我们简化编程有很大的帮助，有时间我会另开一章分享我平时使用到它们的地方，当然也非常欢迎大家评论分享使用技巧。  
假设你从未使用过with()函数，我们可以先看看源码：

```kotlin
inline fun <T, R> with(receiver: T, block: T.() -> R): R
```

> Calls the specified function block with the given receiver as its receiver and returns its result.

大体的意思是调用receiver中的方法然后返回它的结果：

```kotlin
val receiver: String = "Fructos"

val block: String() -> Unit = {
    println(toUpperCase())
    println(toLowerCase())
    println(capitalize())
}

val result: Unit = with<String, Unit>(receiver, block)

//简化一下代码
with(sugar) {
    println(toUpperCase())
    println(toLowerCase())
    println(capitalize())
}
```

在block中你可以调用到receiver的方法而且不需要任何限定符，在block的最后返回你需要的数据类型就可以了。在项目中我经常使用它在p层传递限定符像是view.show(),view.hide()

```kotlin

interface View {
    fun show()
    fun hide()
    fun reset()
    fun clear()
}

class Presenter(private val view: View) {

    fun present(isFructos:Boolean) = with(view) {
        if(isFructos) {
            show()
            hide()
        } else {
            hide()
            clear()
        }
    }
}
```

## inline reified

我们先分析一下下面的代码

```kotlin
abstact class Item

class MediaItem: Item() {
    val media = ...
}

class IconItem: Item() {
    val icon = ...
}

interface Renderer {
    fun render(view: View, item:Item)
}

class MediaItemRenderer: Renderer {

    override fun render(view: View, item: Item) {
        if(item !is MediaItem) {
            throw AssertionError("Item is not an instance of MediaItem")
        }

        view.showMedia(item.media)
        view.reset()
    }
}

class IconItemRenderer: Renderer {

    override fun render(view: Viewm, item: Item) {
        if(item !is IconItem) {
            throw AssertionError("Item is not an instance of IconItem")
        }

        view.showIcon(item.icon)
        view.reset()
    }
}
```

上述代码有很多冗余的地方，其实很明显你就能看出MediaItemRenderer和IconItemRenderer在render()函数中存在着相同的逻辑，现在，我们用inline reified来改造它，首先将render函数中的逻辑提取出来：

```kotlin
fun<T> withCorrectType(toBeChecked: Item, block: (T) -> Unit) {
    if(toBeChecked !is T) {
        throw IllegalArgumentException("invalid Type")
    }
    block.invoke(toBeChecked)
}
```

然而，这样做并不能通过编译，会报错

> Cannot check for instance of erased type: T  

产生这种错误是因为泛型机制。

> During the type erasure process, the Java compiler erases all type parameters and replaces each with its first bound if the type parameter is bounded,or Object if the type parameter is unbounded.-docs.oracle.com

so, 可以防止防止泛型T被清除吗？在Kotlin中一切都有可能，使用inline reified，就能修复这个问题。

```kotlin
class MediaItemRenderer: Renderer {

    override fun render(view: View, item: Item) = with(view) {
        withCorrectType<MediaItem>(item) {
            show {it.media()}
            reset()
        }
    }
}

class IconItemRenderer: Renderer {

    override fun render(view: View, item: Item) = with(view) {
        withCorrectType<IconItem>(item) {
            clear()
            show {it.icon()}
        }
    }
}

inline fun<reified T> withCorrectType(toBeChecked: Item, block: (T) -> Unit) {
    if(toBeChecked !is T) {
        throw ...
    }
    block.invoke(toBeChecked)
}
```

使用reified修饰符Kotlin compiler保留了你的类型，当然这是必须在使用inline的前提下。

以上就是第二章的全部了，在第三章中我们将展示更多的Kotlin小技巧。

### 参考

[6 magic sugars that can make your Kotlin codebase happier — Part 2](https://medium.com/grand-parade/6-magic-sugars-that-can-make-your-kotlin-codebase-happier-part-2-843bf096fa45)
