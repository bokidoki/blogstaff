---
title: Android多渠道打包及加固方案
tags: android
categories: tools
date: 2019-09-25 15:36:28
thumbnail:
top: 0
---


## 前言

Android多渠道打包已经是老生常谈的问题了，各个大厂也先后开源了自己的打包方案，为我们开发者带来不少便捷。今天我就来谈谈美团的Walle，我在项目中也正是用到了它，也算做个总结和备忘吧。本篇中会提及Walle的基本使用方法以及如何在项目中配置加固使用，当然，最后也稍微会从源码的角度去分析一下这个方案的原理。那么，现在开始吧。

<!--more-->

## 如何使用

参考Walle项目[Github](https://github.com/Meituan-Dianping/walle)首页，操作如下：  
在工程目录引入

```groovy
buildscript {
    dependencies {
        classpath 'com.meituan.android.walle:plugin:1.1.6'
    }
}
```

在app目录下引入

```groovy
apply plugin: 'walle'

dependencies {
    // 用于读取渠道号
    compile 'com.meituan.android.walle:library:1.1.6'
}
```

配置信息呢，可以参考官方说明，我这就简单记录下(copy)了

```groovy
walle {
    // 指定渠道包的输出路径
    apkOutputFolder = new File("${project.buildDir}/outputs/channels");
    // 定制渠道包的APK的文件名称
    apkFileNameFormat = '${appName}-${packageName}-${channel}-${buildType}-v${versionName}-${versionCode}-${buildTime}.apk';
    // 渠道配置文件 一个渠道占一行
    channelFile = new File("${project.getProjectDir()}/channel")
}
```

不要忘记获取渠道信息

```kotlin
val channel = WalleChannelReader.getChannel(context)
UMConfigure.init(this, UMENG_APP_KEY, channel, UMConfigure.DEVICE_TYPE_PHONE, "")
```

接下来只需要在gradle任务执行channelRelease或是执行

```bash
gradlew clean assembleReleaseChannels
```

渠道包就能生成在你指定的目录下面了。

但是这样操作完之后就没问题了吗？显然不是。通常我们发布自己的应用之前，还需要进行应用加固(360或是乐固，本文用的是乐固)，加固后会清除apk的签名和渠道信息，需要重新签名然后写入渠道信息。因此，打渠道变成了如下流程：  
![ ](https://dreamweaver.img.we1code.cn/%E5%A4%9A%E6%B8%A0%E9%81%93%E6%89%93%E5%8C%85+%E5%8A%A0%E5%9B%BA%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

要满足上面的操作，项目中walle的配置显然就不太合适了，幸亏walle团队也有提供命令行工具walle-cli供我们自行打包，为了方便，我自己写了个简单的脚本，自动上传到乐固加固然后进行签名写入渠道信息，具体可以参考一下[autoReinforce](https://github.com/bokidoki/autoReinforce)的项目说明。

## 源码分析

有几个问题想问下大家。

1. 渠道信息写在哪里了呢？为什么写在这个位置呢？
2. 渠道信息是如何进行读写操作的呢？

第一个问题很简单啦，文档上也说了写在了Apk中的APK Signature Block区块，如下图，之所以写在这个位置是因为v2不会对该区域进行校验。

![ ](https://dreamweaver.img.we1code.cn/apk%E7%AD%BE%E5%90%8D%E7%A4%BA%E4%BE%8B%E5%9B%BE.png)  

在payload_reader可以找到写入渠道的逻辑，在讲述逻辑之前，我们首先要了解一下EOCD(End of Centtal Directory)的结构：

offset|Bytes|Description
-|-|-
0    |       4      | End of central directory signature = 0x06054b50
4    |       2      | Number of this disk
6    |       2      | Disk where central directory starts
8    |       2      | Number of central directory records on this disk
10   |       2      | Total number of central directory records
12   |       4      | Size of central directory (bytes)
16   |       4      | Offset of start of central directory, relative to start of archive
20   |       2      | Comment length (n)
22   |       n      | Comment

由于渠道信息是写在APK Signature Block，因此只要找到Center Directory的位置，那么往前就能找到Apk Signing Block的位置。在Walle中，通过循环找到魔数0x06054b50(假设Comment为空，通过增加Comment的长度，确定EOCD block的位置)，从而确定comment的长度，再将长度与Comment length对比，只要能确认Comment的长度，就能确认APK Signature Block的位置了。APK Signature Block结构如下表所示：

offset|Bytes|Description
-|-|-
@+0|8|block的长度(当前长度不计算在内)
@+8|n|ID-value值
@-24|8|block的长度
@-16|16|魔数"APK Sig Block 42"

walle渠道信息就是写在ID-value中，在上一步中已经拿到Center Directory的offset，再向前24bytes，取8bytes，就能拿到APK Signature Block的长度了，注意这个长度是不包括前面8个bytes的，在walle中向前多偏移了8个bytes，取首尾block长度对比进行校验，代码片段如下：

```java
        // Find the APK Signing Block. The block immediately precedes the Central Directory.
        if (centralDirOffset < APK_SIG_BLOCK_MIN_SIZE) {
            throw new SignatureNotFoundException(
                    "APK too small for APK Signing Block. ZIP Central Directory offset: "
                            + centralDirOffset);
        }
        // 后面16bytes就是魔数啦 加上前面8bytes的black长度信息，24bytes
        // * 16 bytes: magic
        fileChannel.position(centralDirOffset - 24);
        final ByteBuffer footer = ByteBuffer.allocate(24);
        fileChannel.read(footer);
        footer.order(ByteOrder.LITTLE_ENDIAN);
        // 这里不是很清楚为什么要将魔数拆开来对比？
        if ((footer.getLong(8) != APK_SIG_BLOCK_MAGIC_LO)
                || (footer.getLong(16) != APK_SIG_BLOCK_MAGIC_HI)) {
            throw new SignatureNotFoundException(
                    "No APK Signing Block before ZIP Central Directory");
        }
        // 尾部记录的block长度
        final long apkSigBlockSizeInFooter = footer.getLong(0);
        if ((apkSigBlockSizeInFooter < footer.capacity())
                || (apkSigBlockSizeInFooter > Integer.MAX_VALUE - 8)) {
            throw new SignatureNotFoundException(
                    "APK Signing Block size out of range: " + apkSigBlockSizeInFooter);
        }
        // 将总长度与头部记录的8bytes长度相加
        final int totalSize = (int) (apkSigBlockSizeInFooter + 8);
        final long apkSigBlockOffset = centralDirOffset - totalSize;
        if (apkSigBlockOffset < 0) {
            throw new SignatureNotFoundException(
                    "APK Signing Block offset out of range: " + apkSigBlockOffset);
        }
        fileChannel.position(apkSigBlockOffset);
        final ByteBuffer apkSigBlock = ByteBuffer.allocate(totalSize);
        fileChannel.read(apkSigBlock);
        apkSigBlock.order(ByteOrder.LITTLE_ENDIAN);
        // 头部和尾部的长度杜比校验
        final long apkSigBlockSizeInHeader = apkSigBlock.getLong(0);
        if (apkSigBlockSizeInHeader != apkSigBlockSizeInFooter) {
            throw new SignatureNotFoundException(
                    "APK Signing Block sizes in header and footer do not match: "
                            + apkSigBlockSizeInHeader + " vs " + apkSigBlockSizeInFooter);
        }
        // 拿到APK Signing Block了
```

再来看看ID-value区域结构

Bytes|Description
-|-
8|序列长度n(不包括其本身)
4|序列id
n-4|内容

了解了ID-value区域结构那么再贴一下获取custom ID-value的代码

```java
        // APK Sig Block 中的ID-value区域
        final ByteBuffer pairs = sliceFromTo(apkSigningBlock, 8, apkSigningBlock.capacity() - 24);

        final Map<Integer, ByteBuffer> idValues = new LinkedHashMap<Integer, ByteBuffer>(); // keep order

        int entryCount = 0;
        while (pairs.hasRemaining()) {
            entryCount++;
            if (pairs.remaining() < 8) {
                throw new SignatureNotFoundException(
                        "Insufficient data to read size of APK Signing Block entry #" + entryCount);
            }
            // 获取总长度 8bytes
            final long lenLong = pairs.getLong();
            if ((lenLong < 4) || (lenLong > Integer.MAX_VALUE)) {
                throw new SignatureNotFoundException(
                        "APK Signing Block entry #" + entryCount
                                + " size out of range: " + lenLong);
            }
            final int len = (int) lenLong;
            // id开始的位置
            final int nextEntryPos = pairs.position() + len;
            if (len > pairs.remaining()) {
                throw new SignatureNotFoundException(
                        "APK Signing Block entry #" + entryCount + " size out of range: " + len
                                + ", available: " + pairs.remaining());
            }
            // 获取id 4bytes
            final int id = pairs.getInt();
            idValues.put(id, getByteBuffer(pairs, len - 4));

            pairs.position(nextEntryPos);
        }
```

至此就分析完了如何在APK中去读取插入的渠道信息，顺带了解了一下APK包的结构。最后过一下如何写入渠道信息的吧，流程如下：

- 通过commentLength\centralDirStartOffset\apkSigningBlockAndOffset找到IdValues的位置
- 在IdValues block中找到V2签名的位置，判断是否已经签名
- 判断是否使用V3签名，如果有将长度补成4096的倍数(V3签名会校验)
- 写入渠道

## 结语

从多渠道打包，引申出了Apk的签名V2签名逻辑(V1类似，但是是放在EOCD的Comment中)，Apk(Zip)包的结构等问题。这里只是简单的做下自我总结，如有疑问欢迎留言，当然你也可以选择去看看官方的文档和大神们的博客。

## 参考

- [带你了解腾讯开源的多渠道打包技术 VasDolly源码解析 by 鸿洋](https://juejin.im/post/5ad47f466fb9a028d82c3e29)
- [APK文件结构详解](https://juejin.im/post/5c6d6d6a6fb9a049f06ad97e)  
- [Meituan-Dianping/walle](https://github.com/Meituan-Dianping/walle)
- [Tencent/VasDolly](https://github.com/Tencent/VasDolly/wiki/VasDolly%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)
