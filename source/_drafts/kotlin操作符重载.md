---
title: kotlin操作符重载
categories: Android
top: 0
date: 2020-07-02 14:29:42
tags:
thumbnail:
---


### kotlin类加载顺序

```kotlin
// 测试类
class Tester {
    constructor() {
        println("Tester对象constructor")
    }

    init {
        println("Tester对象init")
    }

    companion object {
        init {
            println("伴生对象init")
        }

        fun echo() {
            println("随便测试一下吧")
        }
    }
}

// 调用
class Main {
    companion object {
        @JvmStatic
        fun main(args: Array<String>) {
            val tester = Tester()
            Tester.echo()
        }
    }
}
```

```cmd
// 输出
伴生对象init
Tester对象init
Tester对象constructor
随便测试一下吧
```

总结：  
伴生对象总是跟随原外部对象的初始化而初始化，甚至，直接调用伴生对象的函数也能使init函数执行，而外部对象并不会执行初始化操作。从日志输出中也可以看出伴生对象的初始化时的优先级最高。

### 伴生对象的扩展