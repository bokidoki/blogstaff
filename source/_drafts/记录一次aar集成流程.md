---
title: 记录一次aar包集成
tags: gradle
categories: Android
date: 2019/06/21
img:
---

## 前言

今天集成aar包，发现aar与项目中的包名重复了，并且重复的包都被编译到了classes.jar文件中，解决方法有两种，删除自己项目中相应的依赖包，删除aar文件中多余的依赖包

### 删除项目中重复依赖

删除之前，需要看看项目的依赖结构，不然找不到重复的依赖包，比如说我的项目中重复的依赖包有com.google.gson，但是我项目本身并没有做相关的依赖，因此我需要找到哪个依赖中同时依赖了gson，再把它exclude掉，我们可以找到Gradle中找到dependencies task双击执行它，就可以看到项目完整的依赖结构啦，或者也可以在命令行中输入(win环境)

```groovy
gradlew :app:dependencies
```

然后就可以去查找哪里有依赖到gson，在我的项目中是databinding-compiler

```groovy
\--- androidx.databinding:databinding-compiler:3.4.1
     +--- androidx.databinding:databinding-compiler-common:3.4.1
     |    +--- androidx.databinding:databinding-common:3.4.1
     |    +--- com.android.databinding:baseLibrary:3.4.1
     |    +--- org.antlr:antlr4:4.5.3
     |    +--- commons-io:commons-io:2.4
     |    +--- com.googlecode.juniversalchardet:juniversalchardet:1.0.3
     |    +--- com.google.guava:guava:26.0-jre (*)
     |    +--- com.squareup:javapoet:1.8.0
     |    +--- org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.3.31
     |    |    +--- org.jetbrains.kotlin:kotlin-stdlib:1.3.31 (*)
     |    |    \--- org.jetbrains.kotlin:kotlin-stdlib-jdk7:1.3.31
     |    |         \--- org.jetbrains.kotlin:kotlin-stdlib:1.3.31 (*)
     |    +--- com.google.code.gson:gson:2.8.0
     |    +--- com.android.tools:annotations:26.4.1
     |    \--- com.android.tools.build.jetifier:jetifier-core:1.0.0-beta04
     |         +--- com.google.code.gson:gson:2.8.0
     |         \--- org.jetbrains.kotlin:kotlin-stdlib:1.3.0 -> 1.3.31 (*)
```

找是找出来了但是，不知道怎么怎么去exclude掉它啊，如果有知道的朋友希望能指点一下。  
既然这样做不行，那么我们就要用第二种方法了。

### 删除aar包中的多余库

在linux环境下我们可以解压aar文件，然后删除重复文件再重新打包

```bash
unzip xxx.aar -d unzip/dir //解压文件目录
jar cvf xxx.aar -C zip/dir //打包目录
```

但是每次手动操作是不是觉得很沙雕，没事接下来介绍个更简便的方法，我们可以在Gradle构建过程中，解压aar，删除目标文件，重新打包，而这个脚本我们都不用写，因为已经有大神已经写好了，我们直接拿来用就好了。步骤也相当简单：

- 通过import module导入aar文件

此时的文件目录应该是这样

```bash
--- library
    +-- build
    +-- build.gradle
    +-- excludeAar.gradle
    +-- xxx.aar
```

这个excludeAar.gradle就是今天的主角了，不过首先还是要看看build.gradle

```groovy
configurations.maybeCreate("default")
artifacts.add("default", file('xxx.aar'))
apply from: "${project.projectDir.absoluteFile}\\excludeAar.gradle"
```

### 扩展

aar/jar打包流程

### 参考

[ExcludeAar](https://github.com/Siy-Wu/ExcludeAar/blob/master/library-exclude/excludeAar.gradle)  
[如何过滤aar和jar包中的类（class）](https://blog.csdn.net/baidu_34012226/article/details/80104771)
