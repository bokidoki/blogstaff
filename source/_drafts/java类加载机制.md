---
title: java类加载机制
date: 2020-04-18 18:51:30
tags: jvm
categories: Java
img:
---

### 概念

这节想谈谈java的类加载机制，有人可能会说这有什么好谈的，不就是双亲委派机制吗？先由顶级父类ClassLoader加载，如果找不到就交给下一级的ClassLoader处理。嗯，是的这么说是没错，但是为什么要这么做呢？ClassLoader是如何将class文件加载到内存之中的呢？这两个问题将是这章我们要讨论的问题。

### 问题

1. ClassLoader双亲委派机制的好处
2. ClassLoader如何将class文件加载至内存中

要搞清楚这两个问题，首先要明确java中的classLoader到底有多少种，大概分为以下三种类型：

- BootStrap ClassLoader 负责加载rt.jar、resources.jar、charsets.jar等核心类库。
- Extension ClassLoader 默认加载JAVA_HOME/jre/lib/ext/目下的所有jar
- App ClassLoader 负责加载应用程序classpath目录下的所有jar
