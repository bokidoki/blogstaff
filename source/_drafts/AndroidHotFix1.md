---
title: Android热更新系列(其一)
tags: hotfix
categories: Android
date: 2019/09/18
img:
---
## 前言

最近在接入腾讯的热更新框架tinker，由于tinker生成的差分包需要上传到后台，后端大哥没时间开发相关功能，遂直接用了bugly平台的热更新，其本质还是tinker，只是用到了bugly后台对差分包进行分发。bugly官方文档还是比较完备的，但是在实际开发中还是踩了不少坑，在此记录下。

## 使用说明

先贴一下官方的文档地址[Bugly Android 应用升级 SDK 使用指南](https://bugly.qq.com/docs/user-guide/instruction-manual-android-upgrade/?v=20181014122344)

接下来记录下我自己接入的流水作业，如下👇

主工程目录导入bugly gradle脚本

```gradle
dependencies {
    classpath "com.tencent.bugly:tinker-support:1.1.1"
}
```

app目录下引入依赖包

```gradle
dependencies {
    implementation 'com.tencent.bugly:crashreport_upgrade:latest.release'
    implementation 'com.tencent.bugly:nativecrashreport:latest.release'
}
```

## 问题记录

由于使用的是bugly的gradle脚本，它和tinker的gradle脚本版本的对应关系如下：

> tinker-support 1.1.3 对应 tinker 1.9.8  
tinker-support 1.1.2 对应 tinker 1.9.6  
tinker-support 1.1.1 对应 tinker 1.9.1  
tinker-support 1.0.9 对应 tinker 1.9.0  
tinker-support 1.0.8 对应 tinker 1.7.11  
tinker-support 1.0.7 对应 tinker 1.7.9  
tinker-support 1.0.4 对应 tinker 1.7.7  
tinker-support 1.0.3 对应 tinker 1.7.6  
tinker-support 1.0.2 对应 tinker 1.7.5（需配置tinker插件的classpath）

问题描述 | 版本号 | 解决方法
--------|---------|--------
Could not find method getAaptOptions() for arguments [] on task | tinker-support 1.1.5 | 使用tinker-support版本1.1.1可编译成功
Didn't find class "com.tencent.tinker.entry.DefaultApplicationLike" | tinker-android-lib 1.9.1 | 修改其版本号为1.9.9

发现绝大部分问题出现在版本号上面😴
