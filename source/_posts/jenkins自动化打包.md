---
title: Jenkins自动打包Android应用
tags: tools
categories: Jenkins
thumbnail: https://dreamweaver.img.we1code.cn/jenkins-cover.jpg
date: 2018-04-28 00:00:00
top: 0
---

## 前言

已经用Jenkins做过很多Android自动化打包的配置了，无奈记性不咋地，每配一次就要查一次资料，踩同样的坑，浪费不少时间和精力，更是被一些莫名其妙的问题折磨到抓狂，于是我决定在此把Jenkins的配置流程和遇到的坑整理、记录下来（其实早就想这么做了，但是懒癌晚期），方便以后做一些查阅。

<!--more-->

## 基本步骤

### 全局工具配置

在系统管理中做全局工具配置，如下图
![全局工具配置](https://dreamweaver.img.we1code.cn/optional1.png)
配置 JAVA_HOME、GRADLE_HOME 指向JDK的安装目录和Gradle的解压目录，然后配置Jenkins的全局变量，这里我配置了python的路径，GRADLE_USER_HOME，这个变量用作gradle的缓存目录，还配置了ANDROID_HOME指向AndroidSdk的目录。

### 基础工程配置

基础工程配置分为配置构建参数、源码管理、配置触发器、配置构建工具、构建后的一些操作

### 构建任务重命名

![重命名](https://dreamweaver.img.we1code.cn/%E4%BB%BB%E5%8A%A1%E9%87%8D%E5%91%BD%E5%90%8D.png)

### 配置构建参数

选择参数化构建过程>选项参数
![构建参数](https://dreamweaver.img.we1code.cn/jenkins02.jpg)

### 源码管理

选择git作为版本控制工具
![源码管理](https://dreamweaver.img.we1code.cn/jenkins03.png)

### 配置触发器

解释下触发器的各个选项
>触发远程构建 (例如,使用脚本)  
GitHub hook trigger for GITScm polling  
其他工程构建后触发  
定时构建  
Help for feature: 定时构建  

#### 轮询 SCM

格式为 * * * * *  
第一个星号表示分钟，取值0~59
第二个星号表示小时，取值0~23
第三个星号表示一个月内的天数，取值1~31
第四个星号表示第几个月，取值1~12
第五个星号表示一周的第几天，取值0~7

### 多渠道打包配置

#### 配置参数

![添加渠道选项参数](https://dreamweaver.img.we1code.cn/Jenkins%E6%89%93%E5%8C%85%E6%B8%A0%E9%81%93%E5%8F%B7%E9%85%8D%E7%BD%AE.jpg)

#### 接入[友盟](https://developer.umeng.com/docs/66632/detail/101848)

```groove
//build.gradle 配置
    productFlavors {
        yingyongbao {
        }
        huawei {
        }
    }

    productFlavors.all { flavor ->
        flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
    }
```

### 加固

我在项目使用的是乐固加固，首先去下载他们的[jar包](https://leguimg.qcloud.com/ms-client/java-tool/1.0.3/ms-shield.jar)。进入项目配置文件开始配置：  

配置构建后操作，执行打包后再执行加固，如下图：
![加固步骤1](https://dreamweaver.img.we1code.cn/%E5%8A%A0%E5%9B%BA%E6%AD%A5%E9%AA%A41.jpg)

接下来转到加固项目的配置中，可以将下载下来的jar包做版本管理，也可以直接放在项目根目录中，配置构建步骤：
![加固步骤2](https://dreamweaver.img.we1code.cn/%E5%8A%A0%E5%9B%BA%E6%AD%A5%E9%AA%A42.jpg)

### 再签名

然鹅加固完之后并没有结束，需要进行再签名，
> [加固过程不可避免的会破坏签名，因此加固后的包需重签名，未签名应用将无法顺利安装。](https://cloud.tencent.com/document/product/283/3271)

这里我是又另外建了一个项目，应该还有比较好的做法比如构建后执行什么的(需要另装插件)  

主要看看签名脚本是怎么写的

```python
import sys, os
print('使用apksiger命令为apk签名')
files = os.listdir('./')
jks_file = None
apk_file = None
for file in files:
    if file.endswith('.jks'):
        jks_file = file
    elif file.endswith('.apk'):
        apk_file = file
    else:
        print(file)
if jks_file == None or apk_file == None:
    print('当前目录不存在签名文件或者apk文件，请确认签名文件在当前目录下')
    sys.exit(1)
file_name=apk_file
zipalign_name=file_name.split('.apk')[0]+'_zipalign.apk'
command='zipalign -v -p 4 {0} {1}'.format(file_name, zipalign_name)
os.system(command)
jks_name=jks_file
key_alias = 'bonadeTravel'
ks_pass = 'bonadetravel888'
key_pass = 'bonadetravel888'
apk_name=zipalign_name.split('.apk')[0]+'_signed.apk'
command='apksigner sign --ks {0} --ks-key-alias {1} --ks-pass pass:{2} --key-pass pass:{3} --out {4} {5}'.format(jks_name, key_alias, ks_pass, key_pass, apk_name, zipalign_name)
os.system(command)
```

### 执行构建脚本

```groovy
clean
assemble${channel}${buildType} --stacktrace
//如果需要打所有渠道包
assemble${buildType} --stacktrace
```

### 构建后的操作

构建完成后的操作:  

1. 提取apk文件
2. 上传到蒲公英
3. jenkins中生成二维码
4. 通知测试人员

## tips

想修改一下apk文件输出目录，于是修改build.gradle  

```groovy
applicationVariants.all { variant ->
    variant.outputs.each { output ->
        def outputFile = output.outputFile
        if (outputFile != null && outputFile.name.endsWith('.apk')) {
            if(!outputFile.name.contains("debug")){
                def fileName = outputFile.name.replace(".apk", "-${defaultConfig.versionName}.apk")
                output.outputFile = new File("C:\\Users\\user\\Desktop\\apk\\${defaultConfig.versionName}", fileName)
            }
        }
    }
}
```

在4.0+gradle方法稍有不同

```groovy
applicationVariants.all { variant ->
    variant.outputs.all {
        // 自定义输出路径 但是getPackageApplication()将在19年底被移除
        variant.getPackageApplication().outputDirectory = new File(project.rootDir.absolutePath + File.separator + "outputs")
        outputFileName = "AppName-${variant.flavorName}-${variant.buildType.name}-v${variant.versionName}_${time()}.apk"
    }
```

最终版本

```groovy
applicationVariants.all { variant ->
    variant.outputs.all {
        def newName
        def timeNow
        if ("true".equals(IS_JENKINS)) {
            timeNow = JENKINS_TIME
            variant.packageApplicationProvider.get().outputDirectory = new File(project.rootDir.absolutePath + File.separator + "apks")
            newName = "xxx-v${APP_VERSION}-${timeNow}-${variant.buildType.name}.apk"
        } else {
            timeNow = getDate()
            if (variant.buildType.name.equals('debug')) {
                newName = "xxx-v${APP_VERSION}-debug.apk"
            } else {
                newName = "xxx-v${APP_VERSION}-${timeNow}-${variant.buildType.name}.apk"
            }
        }
        outputFileName = "${newName}"
    }
}
```

## 遇到的坑

### 在jenkins中编译的时候报错找不到abc_ab_share_pack_mtrl_alpha.9.png

![error](https://dreamweaver.img.we1code.cn/jenkins04.png)  
wtf没见过这种错误啊，我估摸着会不会是路径太长的原因，于是在gradle.properties中配置了android.buildCacheDir=F\://androidCache，但是，并没有卵用，秉承着不解决问题不罢休的态度，我又浪费了一个下午。终于，在stackoverflow上，看到有个哥们提到在jenkins中设置GRADLE_USER_HOME这个环境变量，随便指向一个目录。然后就不报错了。我的内心是崩溃的，好吧，总算是解决了，但是为什么AndroidStudio下编译就不会报错呢。

### com.sun.org.apache.xerces.internal.impl.io.MalformedByteSequenceException: 3 字节的 UTF-8 序列的字节 3 无效

又碰到一个奇怪的问题，这个坑是databinding框架产生的，由于我是在linux上开发的，jenkins环境部署在本地的windows上，在xml中databinding的表达式中如果出现了中文字符，就会报编码错误，于是我只能硬着头皮修改布局文件，把中文字符移到资源文件中。

### 开启混淆后报错，proguard-rules.pro文件配置出错

```batch
Execution failed for task ':app:transformClassesAndResourcesWithProguardForRelease'.
```

通过执行

```batch
gradlew --stacktrace task xxx
```

可以看到具体的报错信息，主要是不能混淆的文件没有忽略掉，逐个干掉就行了。

### jenkins使用不了系统的环境变量

配置一下jenkins的环境变量然后重启生效

![环境变量配置](https://dreamweaver.img.we1code.cn/%E9%85%8D%E7%BD%AEJenkins%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F.png)

### jenkins控制台出现中文乱码

jenkins环境变量中添加 key: LANG value: zh.CH.UTF-8

## 参考

[用apksigner进行批量签名的脚本](http://www.aoaoyi.com/archives/1126.html)  
[乐固加固FAQ](https://cloud.tencent.com/developer/article/1135340)

## 结语

上面记录的问题只不过是诸多问题的冰山一角，以后我遇到的jenkins相关的问题都会记录于此。想要熟练运用Android打包，看样子还是要深入研究一下gradle才行呐。
