---
title: Kotlin Lambda and Extension
tags: 基础
categories: Kotlin
thumbnail: https://dreamweaver.img.we1code.cn/kotlin-extension-function.png
date: 2019-03-22 00:00:00
top: 1
---

### Extension Function(扩展函数)

> Extension Function 能在已经存在的类中添加新的方法或者属性，即使这些类来自库或者SDK中。在函数内部，我们可以访问类的公共函数和属性而不需要任何限定符，就好像这个函数就在这个类的内部一样。（注意：从技术上将，它并没有修改现有类，只是在声明的类中创建了static public final函数）

<!--more-->

举个栗子

```kotlin
object KotMain {
    @JvmStatic
    fun main(args: Array<String>) {
        val person = "snoopy"
        person.say("hello")
    }

    fun String.say(sth: String) {
        println("$this say $sth")
    }
}
```

反编译后我们可以看到生成的java代码

```java
public final class KotMain {
    public static final KotMain INSTANCE;

    public static final void main(@NotNull String[] args) {
        Instrinsics.checkParameterIsNotNull(args, "args");
        String person = "snoopy";
        INSTANCE.say(person, "hello");
    }

    public final void say(@NotNull String $this$say, @NotNull String sth) {
        Instrinsics.checkParameterIsNotNull($this$say, "$this$say");
        Instrinsics.checkParameterIsNotNull(sth, "sth");
        String var3 = $this$say + ' ' + sth;
        System.out.println(var3);
    }
}
```

可以看到只是增加了一个final方法。

接下来看看如何在Android项目中运用它

- 可以生成任何Android View实例的函数
  
  ```kotlin
  inline fun<reified V: View> v(context: Context, init: V.() -> Unit): V{
      val instance = V::class.java.getConstructor(context::class.java)
      val view = instace.newInstance(context)
      view.init()
      return view
  }
  ```

- dp-px拓展
  
  ```kotlin
  fun View.dp2px(dp: Float) {
      return TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dp, context.resource.displayMetrics)
  }
  ```

- 添加fragment
  
  ```kotlin
  inline fun FragmentManager.inTransaction(func: FragmentTransaction.() -> Unit) {
      val fragmentTransaction = begainTransaction()
      fragmentTransaction.func()
      fragmentTransaction.commit()
  }

  //使用
  supportFragmentManager.inTransaction {
      add(R.id.container, fragment)
      //other operation
  }
  ```

### High Order Function

High Order Function 在 kotlin 的官网中有很明确的解释:
>[Higher-Order Functions](https://kotlinlang.org/docs/reference/lambdas.html#higher-order-functions)  
>A higher-order function is a function that takes functions as parameters, or returns a function.  
>高阶函数是将函数作为参数或返回函数的函数。

High Order Function 中函数作为参数的情况

```kotlin
inline fun test1(func:Int.() -> Unit) {
    func(1)
}

inline fun Int.test2(func:Int.() -> Unit) {
    func()
}
```

```java
public final void test1(int $this$test1, Function1 call) {
    call.invoke($this$test)
}

public final void test2(Function1 call) {
    call.invoke(1)
}
```

### Lambda with Receiver

什么是Lambda with Receiver? Extension Function + Lambda = Lambda with Receiver，它允许你在没有任何限定符的情况下调用lambda中对象的方法。

### inline function

在kotlin中，函数是一等公民，所以我们可以传递函数或者像其它普通类型一样返回它们。然而，这些函数在运行时可能会产生一些性能上的问题，它们作为对象存储造成了额外的内存开销，这时候就轮到inline登场了，在一些使用High Order Function的场景中，我们一般用inline（内联）去修饰它，这样可以减少调用开销。我们依然从源码出发，通过反编译，看看使用High Order Function编译成Java是什么样子的。  

```kotlin
object KotMain {
    @JvmStatic
    fun main(args: Array<String>) {
        noInline {
            println("调用中")
        }
        inlineFunc {
            println("调用中")
        }
    }

    fun noInline(call: ()->Unit) {
        println("调用前")
        call()
        println("调用后")
    }

    inline fun inlineFunc(call: ()->Unit) {
        println("调用前")
        call()
        println("调用后")
    }
}
```

再来看看java代码

```java
public final class KotMain {
    public static final KotlinMain INSTANCE;

    public static final void main(String[] args) {
        //no inline
        INSTANCE.noInline(new Function() {
            @Override
            public void invoke() {
                System.out.println("调用中");
            }
        })

        //inline
        System.out.println("调用前");
        System.out.println("调用中");
        System.out.println("调用后");
    }

    public final void noInline(Function func) {
        println("调用前");
        func.invoke()
        println("调用后");
    }
}
```

大家可以非常直观的看到结论，不使用内联修饰符，每次调用这个函数都会初始化一个Function实例，显然会造成内存开销，而使用内联修饰符，不会创建Function实例，而会将回调函数内部的代码复制到call site中。

### 参考

>[kotlin-extension function](https://www.jianshu.com/p/7496715ba2fd)  
>[Kotlin里的Extension Functions实现原理分析](http://hengyunabc.github.io/kotlin-extension-functions/)  
>[How to Add a Fragment the Kotlin way](https://medium.com/thoughts-overflow/how-to-add-a-fragment-in-kotlin-way-73203c5a450b)
