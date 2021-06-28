---
title: webview加载速度优化
date: 2020-04-30 15:14:59
tags:
categories:
img:
---

解决方案

- H5的缓存机制（WebView自带）
- 资源预加载
- 资源拦截

### H5的缓存机制

Android WebView的缓存机制

- 浏览器缓存机制
- Application Cache 缓存机制
- Dom Storage 缓存机制
- Web SQL Database 缓存机制
- Indexed Database 缓存机制
- File System 缓存机制

#### 浏览器缓存机制

原理：浏览器会根据协议头里的Cache-Control(或Expires)和Last-Modified(或Etag)等字段来控制文件缓存机制

1. Cache-Control: 用于控制文件在本地缓存有效时长
2. Expires: 用于控制缓存的有效时间，与Cache-Control功能相同，Cache-Control优先级较高
3. Last-Modified: 标识文件在服务器上的最新更新时间
4. Etag: 功能同Last-Modified

常见的用法是

- Cache-Control 与Last-Modified一起使用
- Expires与Etag一起使用

### Application Cache 缓存机制

原理：

- 以文件为单位进行缓存，且文件有一定的更新机制
- 原理有两个关键点：manifest属性和manifest文件

```html
<!DOCTYPE html>
<html manifest="demo_html.appcache">
// HTML 在头中通过 manifest 属性引用 manifest 文件
// manifest 文件：就是上面以 appcache 结尾的文件，是一个普通文件文件，列出了需要缓存的文件
// 浏览器在首次加载 HTML 文件时，会解析 manifest 属性，并读取 manifest 文件，获取 Section：CACHE MANIFEST 下要缓存的文件列表，再对文件缓存
<body>
...
</body>
</html>

// 原理说明如下：
// AppCache 在首次加载生成后，也有更新机制。被缓存的文件如果要更新，需要更新 manifest 文件
// 因为浏览器在下次加载时，除了会默认使用缓存外，还会在后台检查 manifest 文件有没有修改（byte by byte)
发现有修改，就会重新获取 manifest 文件，对 Section：CACHE MANIFEST 下文件列表检查更新
// manifest 文件与缓存文件的检查更新也遵守浏览器缓存机制
// 如用户手动清了 AppCache 缓存，下次加载时，浏览器会重新生成缓存，也可算是一种缓存的更新
// AppCache 的缓存文件，与浏览器的缓存文件分开存储的，因为 AppCache 在本地有 5MB（分 HOST）的空间限制
```

> 专门为Web App离线使用而开发的缓存机制  
> 存储静态文件（JS CSS 字体文件）  
> AppCache是对浏览器缓存机制的补充

实现

```java
String cacheDirPath = context.getFilesDir().getAbsolutePath()+"cache/";
settings.setAppCachePath(cacheDirPath);
// 1. 设置缓存路径

settings.setAppCacheMaxSize(20*1024*1024);
// 2. 设置缓存大小

settings.setAppCacheEnabled(true);
// 3. 开启Application Cache存储机制

// 特别注意
// 每个 Application 只调用一次 WebSettings.setAppCachePath() 和
 WebSettings.setAppCacheMaxSize()
```

### Dom Storage 缓存机制

Dom Storage 分为sessionStorage 和 localStorage

- sessionStorage 具备临时性，即存储与页面相关的数据，它在页面关闭后无法使用
- localStorage 具备持久性，即保存的数据在页面关闭后也可以使用

> 用于存储临时、简单的数据

实现

```java
// 通过设置 `WebView`的`Settings`类实现
WebSettings settings = getSettings();
// 开启DOM storage
settings.setDomStorageEnabled(true);
```
