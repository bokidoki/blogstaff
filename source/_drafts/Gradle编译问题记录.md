---
title: Gradle编译问题记录
tags:
categories: gradle
date: 2019/06/04
img:
---
## 问题点

### gralde编译时报错Manifest merger failed with multiple errors, see logs

面对这种问题真的是毫无头绪，日志也毫无作用  
其实我们可以这样做打开AndroidStudio右上角的Gradle，可以看到所有gradle的任务，然后我们可以去找我们执行报错的任务，比如我执行出错的任务是

```groovy
processDebugManifest
```

我们可以直接点击执行任务，也可以在shell中执行

```groovy
gradlew processDebugManifest --stacktrace
```

这样我们就能在shell中看到具体的报错信息了

### Manifest merger failed : Attribute meta-data#android.support.VERSION@value value=(25.3.1) from

这个问题报错的原因在于AndroidX的兼容问题，需要在gradle.propties中加上配置

```groovy
android.useAndroidX=true
android.enableJetifier=true
```

### 使用material组件时报错Error inflating class

其实报错信息很明显了，眼瞎没有看见，提示信息为

```groovy
Caused by: java.lang.IllegalArgumentException: The style on this component requires your app theme to be Theme.MaterialComponents (or a descendant).
```

换下manifests中application 的theme就Ok了

### Invoke-customs are only supported starting with Android O (--min-api 26)

在build.gradle中加入

```groovy
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
```
