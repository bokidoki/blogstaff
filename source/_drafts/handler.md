---
title: Handler机制剖析
date: 2020-04-18 11:41:17
tags:
categories:
img:
---

## Handler的实现原理

我们说Handler机制，通常包括三个部分，Handler，Looper，MessageQueue，一个线程只能绑定一个Looper，可以有多个Handler，一个Looper可以绑定多个Handler，一个Handler只能绑定一个Looper。
在子线程中通过Looper.prepare()创建一个绑定当前线程的Looper，并通过Looper.loop()，如果拿到了Message则会使用Message中的Handler去dispatchMessage。

## Handler中的ThreadLocal

ThreadLocal保证线程中的变量是该线程独有的

### 实现原理

ThreadLocal只是变量访问的入口，真正存储对象的是ThreadLocalMap，它是每个线程对象所持有的对象，以ThreadLocal为Key。

Looper通过myLooper()方法在ThreadLocalMap中获取当前线程内的Looper对象，保证了一个线程只有一个Looper对象。

## 主线程中的Looper

主线程中Looper可以通过Looper.getMainLooper()获取，它是在ActivityThread的main方法初始化的

## 为什么Looper.loop()不会造成ANR

首先ANR是如何造成的？

造成ANR的场景有：

- 前台服务在20s内未执行完成
- 后台服务超时设置为200s
- 前台广播在10s内未执行完成
- 后台广播超时设置为60s
- ContentProvider publish超过10s
- 输入事件分发超过5s，包括按键和触摸事件  
  
再来看看Looper.loop()，它只是开启了一个循环，从阻塞队列中去拿取消息（MessageQueue中也有个循环），在通过Handler去分发执行，真正可能造成ANR的操作在Handler的dispatchMessage中，比如在ActivityThread中，生命周期函数的回调是通过主线程的Handler mH执行的，如果在生命周期的回调函数中执行了耗时操作，才会引起ANR。

## 底层ANR是怎么实现的

### ANR的触发机制

以Service为例说明。BroadCastReciver与Service类似。输入事件的ANR情况则不一样。

Service进程attach到System_server进程的过程中会调用realStartServiceLocked()方法，该方法主要作用是在Service启动时发送延迟消息SERVICE_TIMEOUT_MSG，在Service启动完成时则会移除该消息。如果该延迟消息没有被移除，则会触发ANR。

UI线程不断处理各种Task，处理message，监听inputChannel文件句柄，监听到改变后会将Input事件分发给View，vsync信号，封装一个mesasge，重绘View。所以一个Message事件处理过长，会导致input事件无响应，界面无法刷新。

### ANR发生原因

- 主线程读写小文件，io过载的情况下，io调度是无法被及时执行的。‘
- binder通信被block，binder线程池满了
- 主线程死锁存在锁的竞争关系
- 手机cpu调度饥饿，cpu处在重载情况，进程优先级不高 /data/profile/load(adb shell top)

### ANR的处理流程

### ANR的弹窗

只有Activity中出现的ANR才会弹出ANR提示框
