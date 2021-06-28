---
title: Andriod input系统
date: 2020-04-18 13:21:38
tags: framework
categories: Android
img:
---

用户触发屏幕或者案件操作，首次触发的是硬件驱动，驱动收到事件后，将事件写入设备节点，经过层层封装后称为KeyEvent或者MotionEvent，最后交付给Window消费输入事件。
