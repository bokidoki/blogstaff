---
title: Android事件分发机制(一)
tags: 基础
categories: Android
date: 2019-09-21 15:14:42
thumbnail:
top: 0
---


## 前言

Android事件分发机制是Android开发中最基础的知识，在平时的开发中没有少用，但是确很少总结。温故而知新，为此我决定重新分析一下，也是对自己的经验做下总结。

<!--more-->

```shell
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================onInterceptTouchEvent==================
TestDispatchView: ==================dispatchTouchEvent==================
TestDispatchView: ==================onTouchEvent==================
TestDispatchView: MotionEvent { action=ACTION_DOWN }
TestDispatchGroup: ==================onTouchEvent==================
TestDispatchGroup: MotionEvent { action=ACTION_DOWN }
TestDispatchAct: ==================onTouchEvent==================
TestDispatchAct: MotionEvent { action=ACTION_DOWN }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchAct: ==================onTouchEvent==================
TestDispatchAct: MotionEvent { action=ACTION_MOVE }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchAct: ==================onTouchEvent==================
TestDispatchAct: MotionEvent { action=ACTION_UP }
```

当不做任何处理时，发现事件从最外层向最内层传递，最终将事件交回最外层处理，流程图如下:

![ ](https://dreamweaver.img.we1code.cn/android%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%281%29.png)

可以看到事件从最外层Activity传递到最内层View的onTouchEvent，这时View是把事件交回父类处理的，最终又回到Activity，并在Activity的onTouchEvent消费了ACTION_DOWN，接下来的ACTION_MOVE与ACTION_UP事件则直接在Activity被消费了，并不会再往下分发。

接下来看看如果在ViewGroup中如果消费了事件，流程又有什么改变。

```shell
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================onInterceptTouchEvent==================
TestDispatchView: ==================dispatchTouchEvent==================
TestDispatchView: ==================onTouchEvent==================
TestDispatchView: MotionEvent { action=ACTION_DOWN }
TestDispatchGroup: ==================onTouchEvent==================
TestDispatchGroup: MotionEvent { action=ACTION_DOWN }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================onTouchEvent==================
TestDispatchGroup: MotionEvent { action=ACTION_MOVE }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================onTouchEvent==================
TestDispatchGroup: MotionEvent { action=ACTION_MOVE }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================onTouchEvent==================
TestDispatchGroup: MotionEvent { action=ACTION_UP }
```

可以看到一旦事件被消费了，就不会再往上传递到Activity了，并且在接下来的事件中，事件分发也只会传递到ViewGroup并被它消费掉，流程图如下：

![ ](https://dreamweaver.img.we1code.cn/android%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%282%29.png)

接下来在ViewGroup的onInterceptTouchEvent中将事件拦截掉，看流程又有何变化。

```shell
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================dispatchTouchEvent==================
TestDispatchGroup: ==================onInterceptTouchEvent==================
TestDispatchGroup: ==================onTouchEvent==================
TestDispatchGroup: MotionEvent { action=ACTION_DOWN }
TestDispatchAct: ==================onTouchEvent==================
TestDispatchAct: MotionEvent { action=ACTION_DOWN }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchAct: ==================onTouchEvent==================
TestDispatchAct: MotionEvent { action=ACTION_MOVE }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchAct: ==================onTouchEvent==================
TestDispatchAct: MotionEvent { action=ACTION_MOVE }
TestDispatchAct: ==================dispatchTouchEvent==================
TestDispatchAct: ==================onTouchEvent==================
TestDispatchAct: MotionEvent { action=ACTION_UP }
```

通过日志，可以看到事件被ViewGroup拦截后，不再往下分发，直接执行的是ViewGroup的onTouchEvent，由于此时ViewGroup没有消费事件，所以所有的事件都交还给了Activity去处理，流程如下图所示：

![ ](https://dreamweaver.img.we1code.cn/android%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%283%29.png)

## 疑问

- 为什么ACTION_DOWN事件逐层分发，但是ViewGroup消费之后就不会继续向下分发了呢？

要弄清楚这个问题就必须更进一步阅读ViewGroup和View的源码，下一小节中，我将从源码的角度去分析事件分发的原理。

## 参考

[重学安卓：学习 View 事件分发，就像外地人上了黑车！by KunMinX](https://juejin.im/post/5d3140c951882565dd5a66ef)
