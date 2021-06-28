---
title: startup
tags: jetpack
categories: Android
top: 0
date: 2020-11-11 10:20:57
thumbnail:
---


> 版本号androidx.startup:startup-runtime:1.0.0

## 使用方法

App Startup提供了简单、搞笑的方法在应用启动时初始化组件。使用Startup可以简化启动顺序并显示的设置初始化顺序。

- 自定义Initializer实现Initializer接口

```kotlin
class FirstInitializer : Initializer<FirstInitializer.FirstObj> {
    override fun create(context: Context): FirstObj {
        // 组件的初始化操作
        Log.d("initializer", "the first initializer done")
        return FirstObj()
    }

    override fun dependencies(): MutableList<Class<out Initializer<*>>> {
        // 该组件初始化需要依赖的组件
        return mutableListOf()
    }

    class FirstObj {}
}
```

- 自定义Initializer实现Initializer接口

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="cn.we1code.startup.FirstInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="cn.we1code.startup.SecondInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="cn.we1code.startup.ThirdInitializer"
        android:value="androidx.startup" />
</provider>
```

> 如果为指定dependencies所有组件的初始化都是无序的

以上，在Application启动时就能做到Initializer的初始化工作。

- 手动设置

官方还提供了手动初始化的方法。

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data android:name="com.example.FirstInitializer"
    // remove掉这个节点信息
              tools:node="remove" />
</provider>
```

```kotlin
// 在application中调用初始化函数
AppInitializer.getInstance(context)
    .initializeComponent(FirstInitializer::class.java)
```

- tips

1. 如果某个节点依赖了另一个节点，则被依赖的节点就不用在meta-data中声明了.

```kotlin
class FirstInitializer : Initializer<FirstInitializer.FirstObj> {
    override fun create(context: Context): FirstObj {
        Log.d("initializer", "the first initializer done")
        return FirstObj()
    }

    override fun dependencies(): MutableList<Class<out Initializer<*>>> {
        // 在这里依赖了SecondInitializer
        return mutableListOf(SecondInitializer::class.java)
    }

    class FirstObj {}
}
```

```xml
        <provider
            android:name="androidx.startup.InitializationProvider"
            android:authorities="${applicationId}.androidx-startup"
            android:exported="false"
            tools:node="merge">
            <meta-data
                android:name="cn.we1code.startup.FirstInitializer"
                android:value="androidx.startup" />
                // 无须再做声明直接去掉即可
<!--            <meta-data-->
<!--                android:name="cn.we1code.startup.SecondInitializer"-->
<!--                android:value="androidx.startup" />-->
            <meta-data
                android:name="cn.we1code.startup.ThirdInitializer"
                android:value="androidx.startup" />
        </provider>
```

以上，会先初始化SecondInitializer然后初始化FirstInitializer

## 原理分析

startup的功能比较简单，因此逻辑并不是很复杂。

- InitializationProvider

InitializationProvider继承ContentProvider，Application启动时ContentProvider优先初始化。

```java
public boolean onCreate() {
        Context context = getContext();
        if (context != null) {
            // 初始化AppInitializer
            AppInitializer.getInstance(context).discoverAndInitialize();
        } else {
            throw new StartupException("Context cannot be null");
        }
        return true;
    }
```

- AppInitializer

主要函数在discoverAndInitialize与doInitialize，简单来说discoverAndInitialize就是检索出所有的Initializer，让它们有序或者无序的（取决于是否有dependencies）执行初始化函数。代码如下：

```java
void discoverAndInitialize() {
        try {
            // 找到组件名称为InitializationProvider的ContentProvider
            ComponentName provider = new ComponentName(mContext.getPackageName(),
                    InitializationProvider.class.getName());
            // 拿到meta-data
            ProviderInfo providerInfo = mContext.getPackageManager()
                    .getProviderInfo(provider, GET_META_DATA);
            Bundle metadata = providerInfo.metaData;
            // value = androidx.startup
            String startup = mContext.getString(R.string.androidx_startup);
            if (metadata != null) {
                Set<Class<?>> initializing = new HashSet<>();
                Set<String> keys = metadata.keySet();
                for (String key : keys) {
                    String value = metadata.getString(key, null);
                    // 匹配value = androidx.startup的meta-data
                    if (startup.equals(value)) {
                        Class<?> clazz = Class.forName(key);
                        // 判断Initializer是否为clazz的超类
                        if (Initializer.class.isAssignableFrom(clazz)) {
                            Class<? extends Initializer<?>> component =
                                    (Class<? extends Initializer<?>>) clazz;
                            mDiscovered.add(component);
                            // 开始初始化
                            doInitialize(component, initializing);
                        }
                    }
                }
            }
        }
        // 省略
    }
```

```java
<T> T doInitialize(
        @NonNull Class<? extends Initializer<?>> component,
        @NonNull Set<Class<?>> initializing) {
    synchronized (sLock) {
        try {
            Object result;
            // 判断是否已经初始化过了，若已经初始化过了直接返回结果
            if (!mInitialized.containsKey(component)) {
                initializing.add(component);
                try {
                    // 实例化Initializer对象
                    Object instance = component.getDeclaredConstructor().newInstance();
                    Initializer<?> initializer = (Initializer<?>) instance;
                    List<Class<? extends Initializer<?>>> dependencies =
                            initializer.dependencies();
                    if (!dependencies.isEmpty()) {
                        // 遍历所有的dependencies优先初始化
                        for (Class<? extends Initializer<?>> clazz : dependencies) {
                            if (!mInitialized.containsKey(clazz)) {
                                doInitialize(clazz, initializing);
                            }
                        }
                    }
                    // 没有dependencies则自己初始化
                    result = initializer.create(mContext);
                    initializing.remove(component);
                    // 将初始化的结果保存起来
                    mInitialized.put(component, result);
                } catch (Throwable throwable) {
                    throw new StartupException(throwable);
                }
            } else {
                result = mInitialized.get(component);
            }
            return (T) result;
        } finally {
            Trace.endSection();
        }
    }
}
```

## 总结

总的来说，startup使用起来比较容易，我觉得这一块统一起来挺好的，不然你第三方库中来一个ContentProvider，自己的app又来一个ContentProvider，这不都要消耗内存吗。
