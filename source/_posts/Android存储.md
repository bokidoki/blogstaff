---
title: Android存储系统
categories: Android
date: 2020-04-01 15:29:07
tags: 基础
thumbnail:
top: 0
---


Android中的存储目录分为三块，内部存储，外部存储，系统存储目录

![图来自https://juejin.im/post/5de7772af265da3398561133#heading-16侵删](https://dreamweaver.img.we1code.cn/android%E5%AD%98%E5%82%A8%E7%9B%AE%E5%BD%95.jpg)

<!--more-->

### 内部存储

对于设备之每一个安装的APP都会在data/data/packagename/xxx目录下创建与之对应的文件夹，默认只能被此APP访问，当应用被卸载时，内部存储中的文件也会被删除。

根据手机厂商的不同，路径可能为

- data/data/packagename/xxx
- data/user/0/packagename/xxx

获取方法

```java
context.getFileDir(); // data/data/packagename/files
context.getCacheDir(); // data/data/packagename/cache 当内存不足时会被优先删除
```

### 外部存储

分为两部分

- SD卡，应用被卸载后，
- 扩展卡内存，在APP被卸载后，这些文件也会被删除

文件路径为

- 扩展卡外部存储的路径 /storage/emulated/0/Android/data/packagename/xxx
- SD卡外部存储的路径 /storage/extSdCard/Android/data/packagename/xxx

获取方法

以下的type类型为
> DIRECTORY_MUSIC  音乐目录  
DIRECTORY_PICTURES  图片目录  
DIRECTORY_MOVIES  电影目录  
DIRECTORY_DOWNLOADS  下载目录  
DIRECTORY_DCIM   相机拍照或录像文件的存储目录  
DIRECTORY_DOCUMENTS   文件文档目录  

```java
context.getExternalCacheDir(); // /storage/emulated/0/Android/data/packagename/cache
context.getExternalFilesDir(String type); // /storage/emulated/0/Android/data/packagename/files
context.getExternalStorageDirectory(); // /storage/emulated/0
context.getExternalStoragePublicDirectory(String type);

// SD卡外部存储可以通过Environment获取，获取之前要先判断SD是否存在
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            File[] files = getExternalFilesDirs(Environment.MEDIA_MOUNTED);
            for (File file : files) {
                Log.e("file_dir", file.getAbsolutePath());
            }
}
```

### 系统目录

```java
context.getRootDirectory(); // /system
context.getDataDirectory(); // /data
context.getDownloadCacheDirectory(); // /cache
```

### 参考

[一篇文章搞懂android存储目录结构](https://juejin.im/post/5de7772af265da3398561133)  
作者：crazyandcoder
