---
title: Binder
date: 2020-04-20 11:36:21
tags:
categories:
img:
---

Binder是一个虚拟驱动，它位于/dev/binder，dev目录包含了所有linux系统中使用的外部设备。
作为一个驱动，Binder提供了一系列系统调用的[实现](https://code.woboq.org/linux/linux/drivers/android/binder.c.html)，如下：

```c
const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = binder_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};
```

在用户空间调用系统方法mmap/ioctl等最终映射到了binder_mmap/binder_ioctl函数上。

<!--more-->

Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。Client、Server、ServiceManager位于用户控件，Binder位于内核控件，Client、Server、ServiceManager都是通过Binder Driver进行交互的。

![ams ipc](https://dreamweaver.img.we1code.cn/ams_ipc.jpg)

常用的系统服务AMS,IMS,PMS,WMS,运行在system_server进程中的线程

AMP activity manager的代理 AMS activity manager的系统服务

以ActivityManagerService为例，启动Activity，service等都需要用到ActivityManagerService，这个service在system_service进程，并在system_service启动时就已经注册到Service Manager中了，app调用ActivityManagerService返回的时AMP，也就是AMS的代理，在AMP中通过也就是一个IBinder对象执行一个事务START_ACTIVITY_TRANSACTION，

### ioctl函数说明

> 此函数专门向驱动层发送或者接收指令  

int ioctl(int d,int request, ...)

参数1 设备描述符
参数2 指令，对应驱动层的某一个功能
参数3 可变参数，跟命令有关，传入驱动层的参数或是接收数据的缓存
