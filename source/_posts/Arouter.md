---
title: Arouter 原理浅析
tags: library
categories: Android
date: 2019-12-08 11:40:34
thumbnail:
top: 0
---


## 前言

想必大家都是用过arouter框架了，可以说arouter被广泛应用在组件化场景中，作为组件之间跳转的基石。在这篇中，我将主要分析arouter实现的原理，包括如何apt的部分以及使用时跳转的部分。

<!--more-->

## 项目结构

![项目结构](https://dreamweaver.img.we1code.cn/arouter1.jpg)

- 与注解相关
- arouter核心代码，读取apt生成的文件，api相关
- apt注解相关，提取注解信息生成java文件
- gradle 插件
- ide插件

## 分析

分析先从注解类开始入手，再分析apt提取注解信息逻辑，这部分包括android processor使用方法，以及java poet的基本使用方法，最后也是最简单的就是arouter-api这部分。

### arouter基本注解类型

![基础注解类型](https://dreamweaver.img.we1code.cn/arouter2.jpg)

- Autowired 变量相关(Param的替代，arouter跳转时携带的参数自动织入)
- Interceptor 路由拦截器注解
- Route 路由注解

### 相关注解处理器

![相关注解处理器](https://dreamweaver.img.we1code.cn/aouter3.jpg)

注解器处理注解生成java模板代码，到这里需要了解一下apt的基础知识，自定义注解处理器一般需要依赖下面两个库:

```groovy
implementation 'com.google.auto.service:auto-service:1.0-rc3'
implementation 'com.squareup:javapoet:1.11.1'
```

第一个主要用于注册自定义的注解处理器的，其本身也是一个注解处理器，一般来说注册自定义注解器需要将注解器的绝对路径写入如下目录结构的文件中

![注册处理器](https://dreamweaver.img.we1code.cn/router4.jpg)

而第二个库主要是用于生成java文件的，apt与javapoet分章再讲，先看看arouter这几个注解的逻辑

```java
@Override
public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
    if (CollectionUtils.isNotEmpty(set)) {
        try {
            // 获取所有被Autowired注解的element
            categories(roundEnvironment.getElementsAnnotatedWith(Autowired.class));
            // 生成模板java文件，详情可以去看arouter源码
            generateHelper();
        } catch (Exception e) {
            logger.error(e);
        }
        return true;
    }
    return false;
}

/**
* 例子
* s1被@Autowired注解，它的enclosingElement就是 A, parentAndChild收集的就是A中所有被 Autowired 注解的element
**/
class A {
    @Autowired
    String s1;

    @Autowired
    String s2;
}
```

InterceptorProcessor/RouteProcessor的处理逻辑则差不太多，都是先获得相应的注解类，然后生成java模板代码。

### 模板代码的调用

模板代码的调用逻辑在arouter-api中，所有的路由/拦截器信息都存在warehouse中，我们使用Arouter时，首先需要调用

```java
Arouter.init(applicaiotn);

protected static synchronized boolean init(Application application) {
    mContext = application;
    LogisticsCenter.init(mContext, executor);
    logger.info(Consts.TAG, "ARouter init success!");
    hasInit = true;
    mHandler = new Handler(Looper.getMainLooper());
    return true;
}
```

模板代码的读取逻辑就在LogisticsCenter中，可以看到arouter通过DexFile读取到对应包名，筛选出com.alibaba.android.arouter.routes目录下route文件的路径，缓存在sharePeferences中，最后通过反射加载class文件缓存至warehouse中。

```java
ARouter.getInstance().build("/test/activity2").navigation();
```

一般使用上面的方法进行ARouter跳转，逻辑大概是通过path在RouteMetaData中找到相关联的类信息存储在postcard对象中，最后在_ARoute中实现跳转逻辑，如果路由地址对应的是Activity，执行的就是ActivityCompat.startActivity/ActivityCompat.startActivityForResult.

## 结束语

arouter的实现逻辑差不多就分析完了，总的来说就是编译时提取注解信息，生成java模板代码，难点可能在于对apt/javapoet我们并不是非常熟练，这两部分需要自己亲自实践一下才能熟悉它们的api，因此我准备重新开一章具体实践一下。感兴趣的朋友可以继续关注一下。
