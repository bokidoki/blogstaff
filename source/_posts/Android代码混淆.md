---
title: Android代码混淆
categories: Android
date: 2018-05-29 00:00:00
tags: 基础
thumbnail:
top: 0
---

## 前言

最近在用Kotlin撸App，准备发版了，做下代码混淆，想用原来的混淆逻辑，但是发现各种报错，头大的很，觉得是自己关于混淆的知识积累不够多，是应该系统的学习一下了！顺便在此记录下遇到的坑。那下面我们开始吧。

<!--more-->

## 代码混淆

### 开启代码混淆

只要在app.gradle文件下配置proguardFiles

```groovy
    buildTypes {
        release {
            minifyEnabled true //是否开启混淆
            zipAlignEnabled true //对齐zip
            debuggable false // 是否debug
            versionNameSuffix "_release" // 版本命名后缀
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro' // 混淆文件
            signingConfig signingConfigs.release
        }
        ...
    }
```

proguard-android.txt 是android自带的混淆规则，我们只需要在proguard-rules.pro这个文件中配置我么的混淆规则就可以了。

### Proguard混淆流程

![proguard混淆流程](https://dreamweaver.img.we1code.cn/proguard%E6%B7%B7%E6%B7%86%E6%B5%81%E7%A8%8B.jpg "proguard混淆流程")

- 压缩（shrink）：检测并移除代码中无用的类、字段、方法和特性
- 优化（optimize）：对字节码进行优化，移除无用指令
- 混淆（obfuscate）：使用a，b，c，d这样简短而无意义的名称，对类、字段和方法进行重命名
- 预检（preveirfy）：在java平台上对处理后的代码进行预检，确保加载的class文件时可执行的

### 混淆规则

Proguard关键字|描述
--|:--:
keep|保留类和类中的成员，防止被混淆或移除
keepnames|保留类和类中的成员，防止被混淆，成员没有被引用会被移除
keepclassmembers|只保留类中的成员，防止被混淆或移除
keepclassmembernames|只保留类中的成员，防止被混淆，成员没有引用会被移除
keepclasseswithmembers|保留类和类中的成员，防止被混淆或移除，保留指明的成员，前提是指名的类中的成员必须存在，如果不存在则还是会混淆。
keepclasseswithmembernames|保留类和类中的成员，防止被混淆，保留指明的成员，成员没有引用会被移除，前提是指名的类中的成员必须存在，如果不存在则还是会混淆。

通配符|描述
--|:--:
field|匹配类中的所有字段
method|匹配类中的所有方法
init|匹配类中的所有构造函数
*|匹配任意长度字符，但不含包名分隔符（.）。
**|匹配任意长度字符，并且包含包名分隔符（.）。
***|匹配任意参数类型
...|匹配任意长度任意类型参数

举例：  
我们完整的包名是com.xxx.ui.MainAct，使用com.\*或者`com.xxx.\*`都是无法匹配的，正确的写法是`com.xxx.\*.\*，`或者`com.xxx.ui.*`

### 避免混淆的因素

- native method：因为native是根据方法名去调用的，若混淆后会导致找不到此方法名。
- 反射相关的方法和类：反射原理就是通过方法名和类名去实例化相应的对象，调用相关的方法。
- setXX和getXX方法：这里指的是通过配置文件直接生成相应的set和get方法的相关库，所以javaBean类很多情况下不能做混淆。
- 第三方jar包：这个需要具体情况具体分析，很多库都会提供默认的混淆配置，大多数情况可以不用做混淆。

### 处理混淆失败问题

通常混淆失败导致gradle构建项目失败，原因在输出的错误日志上并不明显，我们可以在Build Output中找到构建出错的task，例如我构建失败的任务是transformClassesAndResourcesWithProguardForBaiduRelease，因此我可以执行

```groovy
gradlew transformClassesAndResourcesWithProguardForBaiduRelease -- stacktrace
```

这样我们就能在shell中看清楚到底是什么地方出错啦。

## 参考

[ProGuard manual](https://www.guardsquare.com/en/products/proguard/manual/troubleshooting#descriptorclass)
[Android混淆](https://www.jianshu.com/p/b5b2a5dfaaf4)
[Android 代码混淆零基础入门](https://www.jianshu.com/p/86ee6ef970ef)
[ProGuard 最全混淆规则说明](https://www.jianshu.com/p/b471db6a01af)
[Android 混淆：proguard实践](https://blog.csdn.net/youyu_torch/article/details/78775100?utm_source=blogxgwz3)