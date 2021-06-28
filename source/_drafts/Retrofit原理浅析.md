---
title: 从Retrofit到动态代理解析
tags: library
date: 2016-08-28
categories: Android
---

## 前言

提及Retrofit就不得不说到java的动态代理机制，在Android中的动态代理与Java中的动态代理有什么区别呢？这篇文章就从Retrofit入手来探讨一下动态代理。
<!--more-->

## 基本使用

Retrofit的核心逻辑在于通过动态代理将Service类中的注解解析成ServiceMethod类（包含生成http请求的完整信息）。代码如下：

```Java
public <T> T create(final Class<T> service) {
Utils.validateServiceInterface(service);
if (validateEagerly) {
    eagerlyValidateMethods(service);
}
return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
    new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
            // 如果是Object类定义的方法直接执行
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
            }
            // 如果是default方法直接执行
            if (platform.isDefaultMethod(method)) {
                return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 其他通一处理成ServiceMethod
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.adapt(okHttpCall);
        }
    });
}
```

## 动态代理

### 调用

```java
// 通过Proxy的静态函数生成动态代理类
// 为了阅读方法省略了一些代码
public class Proxy {

    private static final WeakCache<ClassLoader, Class<?>[], Class<?>> proxyClassCache = new WeakCache(new Proxy.KeyFactory(), new Proxy.ProxyClassFactory());

    private static final Class<?>[] constructorParams = new Class[]{InvocationHandler.class};

    // 动态代理方法
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        // 省略...
        Class proxyClass = getProxyClass0(loader, interfaces); // ①
        try {
            final Constructor proxyClassConstructor = proxyClass.getConstructor(constructorParams);
            return proxyClassConstructor.newInstance(h);
        } catch (InstantiationException e) {
            throw new InternalError(e, e);
        }
    }

    private static Class<?> getProxyClass0(ClassLoader classLoader, Class<?>... classes) {
        if (var1.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        } else {
            return (Class)proxyClassCache.get(classLoader, classes); // ②
        }
    }

    private static final class ProxyClassFactory implements BiFunction<ClassLoader, Class<?>[], Class<?>> {

        public Class<?> apply(ClassLoader classLoader, Class<?>[] classes) {
            // 省略 ...
            String var16 = null;
            for(int i = 0; i < classes.length; ++i) {
                Class claz = classes[i];
                int modifier = claz.getModifiers();
                if (!Modifier.isPublic(modifier)) {
                    String className = claz.getName();
                    // 46为.的assic码
                    int dotIndex = var11.lastIndexOf(46);
                    String packageName = dotIndex == -1 ? "" : var11.substring(0, dotIndex + 1);
                    if (var16 == null) {
                        var16 = packageName;
                    } else if (!packageName.equals(var16)) {
                        // 不在同一个包下面则会报错
                        throw new IllegalArgumentException("non-public interfaces from different packages");
                    }
                }
            }

            if (var16 == null) {
                var16 = "com.sun.proxy.";
            }

            long var19 = nextUniqueNumber.getAndIncrement();
            // 名称 var16 当前包名 + $Proxy + 自增数
            String name = var16 + "$Proxy" + var19;
            // 16 = final
            byte[] classBytes = ProxyGenerator.generateProxyClass(name, classes, 16); // ③

            try {
                // native 方法
                return Proxy.defineClass0(classLoader, name, classBytes, 0, classBytes.length); // ④
            } catch (ClassFormatError var14) {
                throw new IllegalArgumentException(var14.toString());
            }
        }
    }
}
```

动态代理步骤：

- getProxyClass0()从WeakCache中取值，如果没有Proxy.ProxyClassFactory生产一个；
- ProxyGenerator.generateProxyClass(String name, Class<?>[] classes, int accessFlags)自动生成class；
- ProxyGenerator.generateClassFile()将class写入byte数组。

ProxyGenerator.generateProxyClass用于生成class文件，如下图：向文件写入魔数CAFEBABE，最小版本和最大版本号

![ ](https://dreamweaver.img.we1code.cn/genClass.jpg)

我们可以直接调用generateProxyClass将生成的字节码输出看看文件结构

```java
public static void main(String[] args) throws IOException {
    byte[] classBtyes = ProxyGenerator.generateProxyClass("test-u", new Class[]{TestInter.class});
    FileOutputStream fos = null;
    try {
        fos = new FileOutputStream("test-u.class");
        fos.write(classBtyes);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } finally {
        if (null != fos) {
            try {
                fos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

// 生成的class
public final class test-u extends Proxy implements TestInter {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            // 接口定义的方法
            m3 = Class.forName("TestInter").getMethod("testU");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }

    public final void testU() throws  {
        try {
            // 调用我们传入的handler.invoke 第三个参数数方法需要的参数
            super.handler.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
}
```

分析了HotSpot对动态代理的实现，接下来看看在ART中的实现。

同样是调用Proxy.newProxyInstance但是在ProxyClassFactory的实现确有所不同，它直接通过一个native方法generateProxy来实现class文件的动态生成。

```java
private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // ...省略
        String proxyName = proxyPkg + proxyClassNamePrefix + num;
        return generateProxy(proxyName, interfaces, loader, methodsArray,
                            exceptionsArray);
    }
```

在C中是如何实现的呢？

> 目录 /art/runtime/native/java_lang_reflect_Proxy.cc

```c
static jclass Proxy_generateProxy(JNIEnv* env, jclass, jstring name, jobjectArray interfaces,
                                  jobject loader, jobjectArray methods, jobjectArray throws) {
  ScopedFastNativeObjectAccess soa(env);
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
  // 很显然通过ClassLinker->CreateProxyClass生成的jclass
  return soa.AddLocalReference<jclass>(class_linker->CreateProxyClass(
      soa, name, interfaces, loader, methods, throws));
}
```

> 目录 /art/master/runtime/scoped_thread_state_change.h

```h
/**
* Add a local reference for an object to the indirect reference table associated with the
* current stack frame.  When the native function returns, the reference will be discarded.
**/
// 返回obj
template<typename T>
T AddLocalReference(ObjPtr<mirror::Object> obj) const
    REQUIRES_SHARED(Locks::mutator_lock_);
```

> 目录 class_linker.cc /art/master/runtime/class_linker.cc
