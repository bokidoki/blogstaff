---
title: Koltin委托属性
tags: 基础
categories: Kotlin
date: 2019-10-24 11:02:49
thumbnail:
top: 0
---

## 委托模式

在Kotlin中委托模式通过by关键字实现

```kotlin
interface Base {
    fun print()
}

class Derived(b: Base): Base by b
```

上述表达式表示b将代理Derived去实现interface Base的方法。

<!--more-->

## 委托属性

### 延迟属性(Lazy)

Kotlin标准库中有很多有用的委托属性(delegates，代理属性)，像使用lazy方法创建一个对象，只有在第一次使用的时候会被初始化。

lazy的重载方法

```kotlin
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

public actual fun <T> lazy(lock: Any?, initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer, lock)

public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }
```

由上可见，Lazy接口的实现类有三种SynchronizedLazyImpl，SafePublicationLazyImpl，UnsafeLazyImpl，前两者线程安全，但是实现方法有所不同，分别使用线程锁和原子类实现

```kotlin
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required to enable safe publication of constructed instance
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}

private class SafePublicationLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    @Volatile private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // this final field is required to enable safe publication of constructed instance
    private val final: Any = UNINITIALIZED_VALUE

    override val value: T
        get() {
            val value = _value
            if (value !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return value as T
            }

            val initializerValue = initializer
            // if we see null in initializer here, it means that the value is already set by another thread
            if (initializerValue != null) {
                val newValue = initializerValue()
                if (valueUpdater.compareAndSet(this, UNINITIALIZED_VALUE, newValue)) {
                    initializer = null
                    return newValue
                }
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)

    companion object {
        private val valueUpdater = java.util.concurrent.atomic.AtomicReferenceFieldUpdater.newUpdater(
            SafePublicationLazyImpl::class.java,
            Any::class.java,
            "_value"
        )
    }
}

internal class UnsafeLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE

    override val value: T
        get() {
            if (_value === UNINITIALIZED_VALUE) {
                _value = initializer!!()
                initializer = null
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}
```

至于3者的区别，官方文档上已有说明，如果该值只在一个线程中计算，并且所有线程会看到相同的值就使用SynchronizedLazyImpl；如果该值在多个线程可以同时执行，那么使用SafePublicationLazyImpl；如果初始化发生与使用在同一个线程就使用UnsafeLazyImpl，它不会有任何线程安全的保证和开销。

### 可观察属性(Observable)

在kotlin.properties.Delegates中，可以找到相关方法，分别是observable和vetoable，它们在被观察者的值发生改变时会执行回调，回调有三个值分别时被观察者的类型，旧值与新值；两者的区别在于，vetoable是否给被观察者赋值取决与回调函数的返回值。

```kotlin
var name: String by Delegates.observable("") { property, oldValue, newValue ->
}

var age: String by Delegates.vetoable("") { property, oldValue, newValue ->
    false
}
```

### map代理(map delegate)

暂时没有使用过这种代理。

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int by map
}
```

按照官方文档的例子，来看传入键值对，map代理会根据键值给相对的属性赋值，相关的实现方法如下

```java
@kotlin.internal.InlineOnly
public inline operator fun <V, V1 : V> Map<in String, @Exact V>.getValue(thisRef: Any?, property: KProperty<*>): V1 =
    @Suppress("UNCHECKED_CAST") (getOrImplicitDefault(property.name) as V1)

@kotlin.jvm.JvmName("getOrImplicitDefaultNullable")
@PublishedApi
internal fun <K, V> Map<K, V>.getOrImplicitDefault(key: K): V {
    if (this is MapWithDefault)
        return this.getOrImplicitDefault(key)
    return getOrElseNullable(key, { throw NoSuchElementException("Key $key is missing in the map.") })
}
```

如果传入的map没有实现MapWithDefault接口，key不存在时会抛出NoSuchElementException异常，因此初始化map时，需要做类似如下操作：

```kotlin
emptyMap<String, String>().withDefault { key -> "" }
```

这样找不到相对应的key-value时，就会使用默认值了，实现方法如下：

```kotlin
internal inline fun <K, V> Map<K, V>.getOrElseNullable(key: K, defaultValue: () -> V): V {
    val value = get(key)
    if (value == null && !containsKey(key)) {
        return defaultValue()
    } else {
        @Suppress("UNCHECKED_CAST")
        return value as V
    }
}
```

使用的场景没有遇到过，但是有相关博客说，如果需要预定义keys的时候可以使用，相关博文会贴在下面

## 使用范例

### Shared Preferences

```kotlin
private inline fun <T> SharedPreferences.delegate(
    defaultValue: T,
    key: String?,
    crossinline getter: SharedPreferences.(String, T) -> T,
    crossinline setter: Editor.(String, T) -> Editor
): ReadWriteProperty<Any, T> {
    return object: ReadWriteProperty<Any, T> {
        override fun getValue(thisRef: Any, property: KProperty<*>) =
            getter(key?:property.name, defaultValue)

        override fun setValue(thisRef: Any, property: KProperty<*>, value: T) =
            edit().setter(key?:property.name, value).apply()
    }
}
```

有了如上方法我们就可以改造SharedPreferences了：

```kotlin
// 存储Int
fun SharedPreferences.int(def: Int=0, key: String?=null) =
    delegate(def, key, SharedPreferences::getInt, Editor::putInt)

// 存储Long
fun SharedPreferences.long(def: Long=0L, key: String?=null) =
    delegate(def, key, SharedPreferences::getLong), Editor::putLong)
```

使用

```kotlin
var sthInt by prefs.int()

init {
    sthInt = 1
}

// sthInt的getter和setter都被代理了，取值时实际上调用的是SharedPreferences::getInt，被赋值的同时，也通过Editor::putInt将值存入了SharedPreferences中。
```

更多的使用案例会逐一的添加的此篇中。

## 参考

[Kotlin delegates in Android development - Part1]("https://medium.com/hackernoon/kotlin-delegates-in-android-development-part-1-50346cf4aed7") by Fabio Collini

[Kotlin 委托属性]("https://www.kotlincn.net/docs/reference/delegated-properties.html")