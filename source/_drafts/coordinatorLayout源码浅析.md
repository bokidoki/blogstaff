---
title: coordinatorLayout源码浅析
tags:
categories:
date:
img:
---

自定义view的自定义属性

context.obtainStyledAttributes

自定义coordinatorLayout.behavoir报错This graph contains cyclic dependencies

带着这个问题看一下coordinatorLayout源码androidx.coordinatorLayout有三个类
CoordinatorLayout 3300
DirectedAcyclicGraph 217
ViewGroupUtils 99
加起来也就不到4000行代码，人家RecyclerView一个类就有14000行代码，所以了解一下它的机制不是什么问题。

so，先看看官方是怎么定义CoordinatorLayout这个类的
这个类做出来的目的有两个：
1作为顶级应用的装饰器或者chrome布局
2作为一个或者多个子view交互的容器

老规矩从自定义View/ViewGroup三步 onMeasure()->onLayout()->onDraw()

记录下Behavior的方法

onAttachedToLayoutParams()
这个方法将Behavoir绑定LayoutParams时被调用
当实例化一个LayoutParams或者是从子view读取LayoutParams时都会调用这个方法

onDetachedFromLayoutParams()
顾名思义，这个方法在Behavoir与LayoutParams解绑时被调用，这个方法只在getResolvedLayoutParams()被调用，意思是，xml中
定义的LayoutParams先解绑，再绑定

onInterceptTouchEvent()
