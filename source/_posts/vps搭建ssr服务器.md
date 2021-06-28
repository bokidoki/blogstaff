---
title: vps搭建ssr服务器
tags: 其他
categories: ssr
date: 2019-09-21 00:00:00
thumbnail:
top: 0
---

## 前言

发现最近一些搭建ssr服务器的教程都被迫下线了，心里慌的一匹，原先都是参照教程来搭建的，没有教程我可怎么办，赶紧备份一波。

## vps服务器选择

原先一直用搬瓦工，因为便宜啊，用了一年多ip被封了，花了将近10美元重置，一天不到又给我封了，遂换成vultr(1RMB一天)，这个比搬瓦工(9.99美元一年)贵上不少，但是好在能随时免费换ip，这个ip被封了，我再换一个。下面的服务商我就没用过了，先记录着，万一哪一天vultr也不好用了呢。  

<!--more-->

商家|价格(最低配)
----|---
https://digital-vm.com|$4/MONTHLY $41/YEARLY
https://www.onevps.com|$4/MONTHLY
www.fastcomet.com|$2.95/MONTHLY
https://www.hostkvm.com|$9.5/MONTHLY
https://www.locvps.com|68RMB/MONTHLY
https://zheye.io|88RMB/MONTHLY
https://www.jwdns.com|88RMB/MONTHLY
https://hxkvm.com|65RMB/MONTHLY
https://www.gke.cc|65RMB/MONTHLY
www.aoyouhost.com|48RMB/MONTHLY

>注1：这里只进行价格对比，详细配置还需自行仔细查看  
>注2：如果选择了vultr，一定要先[测试一下](ping.chinaz.com)服务器ip是否被墙了，然后再进行下面的操作，如果被墙了，请换一台服务器再试！

## 部署(非新手向)

- 安装脚本

```shell
wget --no-check-certificate https://freed.ga/github/shadowsocksR.sh; bash shadowsocksR.sh
```

>注：根据脚本提示选择密码，端口号等，最后请小心保存配置信息

- 安装锐速

留意看自己服务器所安装的系统，如果是centos6*64，执行如下命令：

```shell
wget --no-check-certificate -O appex.sh https://raw.githubusercontent.com/hombo125/doubi/master/appex.sh && bash appex.sh install '2.6.32-642.el6.x86_64'
```

如果是centos7*64则需先更换系统内核

```shell
wget --no-check-certificate -O rskernel.sh https://raw.githubusercontent.com/hombo125/doubi/master/rskernel.sh && bash rskernel.sh
```

然后再安装锐速

```shell
yum install net-tools -y && wget --no-check-certificate -O appex.sh https://raw.githubusercontent.com/0oVicero0/serverSpeeder_Install/master/appex.sh && bash appex.sh install
```

配置信息如下图所示(其实不是很明白第一项为啥选n)

![锐速配置](https://dreamweaver.img.we1code.cn/%E9%94%90%E9%80%9F%E9%85%8D%E7%BD%AE.jpg)

## 结束语

到这里，所有流程都已经走完了，最后提醒大家所有的配置信息一定要妥善保存哟。

## 参考

 [用VPS搭建SSR服务器教程](https://www.baishitou.cn/1524.html)(写的真的很详细，新手小伙伴可以过去学习一下)
