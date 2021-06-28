---
title: Android面试QA
tags: interview
categories: Android
thumbnail: 
date: 2018-10-31 15:33:24
---

## 面试QA

- 简述java内存管理机制  
   jvm分成5个区，堆、栈、代码计数器、方法区、native区，栈中存储本地变量和对象的引用、所有new出来的对象都存储在堆中，计数器记录当前jvm执行到代码的位置。
- 关于gc的回收机制  
   在java中所有的内存都是由程序在后台自动分配的，内存的回收是由gc管理的，没有被引用到的对象都会被gc回收
   处理。
- activity的启动方式以及使用场景  
   1. standard          标准启动方式使用该方式会生成一个activity的实例放于栈顶
   2. singleTop         判断栈顶是否存在启动的activity,如有则不新建,若没有新建一个
   3. singleTask        判断任务栈中是否存在该页面,如果存在则将其提至栈顶,并使上面的页面全部出栈
   4. singleInstance    新建一个任务栈
   需要注意的是 singleTop如果没有新建,而是重启原页面则会调用onNewIntent(),singleTask、singleInstance同理
   并且需要在该函数中调用setIntent()重新给Intent赋值,不然调用getIntent()时得到的会是原来传入的Intent.
   调用onNewIntent activity的生命周期为 onPause -> onNewIntent -> onResume -> onPause -> onStop -> onDestroy
   正常的生命周期 onCreate -> onStart -> onResume -> onPause -> onStop -> onDestroy
   从另一个页面返回时 onRestart -> onStart -> onResume
- Service启动方式  
   1. startService()
   2. bindService()
   用startService 开启服务 生命周期为 onCreate -> onStartCommand -> onDestroy
   bindService() 生命周期为 onCreate -> onBind -> onUnbind -> onDestroy
   使用onBind时,必须通过返回IBinder提供一个接口,与服务端保持通信,定义class CustomBinder extends Binder,提供Service
   的方法
   通过startService启动的服务需要通过stopSelf或者stopService停止服务,bindService可以绑定多个客户端,当所有绑定全部
   取消时,系统会销毁该服务
   startService服务能在后台长期运行不能调用Service方法,bindService不行但是能调用Service方法,混合方式启动可以保证既能长期运行又能调用到Service方法
- BroadCast的注册方式与区别  
   广播的注册方式有动态注册与静态注册,所谓动态注册就是通过调用registerBroadCast在代码中注册,静态注册则是在manifest清单
   中配置,动态注册的广播必须在app退出时调用unRegisterBroadCast,否则会报 leaked IntentReceiver错误,而静态注册的广播是常驻型的(如果不设置action则系统接收不到广播)
- 循环语句步骤  
while循环  

    ```java
    while( 布尔表达式 ) {
        //循环内容
    }
    ```

    只要布尔表达式为 true，循环就会一直执行下去
do…while 循环  

    ```java
    do {
        //代码语句
    }while(布尔表达式);
    ```

    布尔表达式在循环体的后面,所以语句块在检测布尔表达式之前已经执行了.如果布尔表达式的值为 true,则语句块一直执行,直到布尔
    表达式的值为false
for循环  

    ```java
        for(初始化; 布尔表达式; 更新) {
        //代码语句
    }
    ```

    关于 for 循环有以下几点说明：
    最先执行初始化步骤。可以声明一种类型，但可初始化一个或多个循环控制变量，也可以是空语句
    然后，检测布尔表达式的值。如果为 true，循环体被执行。如果为false，循环终止，开始执行循环体后面的语句
    执行一次循环后，更新循环控制变量
    再次检测布尔表达式。循环执行上面的过程
- 进程保活策略
- 为什么单例模式可以用枚举来实现

## 鸿神每日一问

- recyclerview 的缓存机制是什么样的？和之前的listview 相比，最大的区别是什么？
- 为什么需要multidex?multidex对启动速度有什么影响，需要做什么优化？multidex的执行流程？multidex默认支持哪些配置？如果开启了multidex主dex依然爆掉，如何处理？
- intent 最大传递数据有限制吗？限制为多少？如果有限制，有什么好的方案传输数据？
- binder在通信时，为什么只需要一次拷贝？
- 匿名内部类，访问外部局域变量，为什么要final修饰？
- Android 事件分发机制能否控制时间给子view的传递规则（默认逆序），如何调整？
- 谈谈对intent的理解，说一下intent的匹配规则，对于intent启动activity，service，broadcast有无注意事项，以及区别。
- databinding的原理是什么？有什么坑吗？和butterknife对比有什么区别？
- Android进程间如何高效的传递较大数据块
- Android中创建进程的方式

```txt
        配置manifest文件中process属性，process=":remote"表示当前应用的私有进程，其它应用的组件不能和他跑在同一个进程中。进程名不以:开头的进程为全局进程，其它应用的组件可以通过设置shareUID和它跑在同一个进程。

        在底层可以通过fork函数创建一个新进程。
        操作系统会复制一个与父进程完全相同的子进程，虽说是父子关系，但是在操作系统看来，他们更像兄弟关系，这2个进程共享代码空间，但是数据空间是互相独立的，子进程数据空间中的内容是父进程的完整拷贝，指令指针也完全相同，子进程拥有父进程当前运行到的位置（两进程的程序计数器pc值相同，也就是说，子进程是从fork返回处开始执行的），但有一点不同，如果fork成功，子进程中fork的返回值是0，父进程中fork的返回值是子进程的进程号，如果fork不成功，父进程会返回错误。
可以这样想象，2个进程一直同时运行，而且步调一致，在fork之后，他们分别作不同的工作，也就是分岔了。这也是fork为什么叫fork的原因
```

- transform api和apt各有什么使用场景，能否相互替换，相互补充？
- Android p 限制访问hidden api大致原理？如何绕过？
- viewgroup中有个generateLayoutParams方法，什么情况下我们会考虑复写？具体有什么作用？举例说明
- 谈一下Android中lowmemorykiller机制
- 讲一下自定义recyclerview.layoutmanager的步骤，注意事项。
- 用笔画一下Android的打包流程，越细致越好？圈出自己不太熟悉的环节。
- 写一个任务控制器，支持添加多个任务异步执行，任务见可以什么依赖，没依赖任务并行，有依赖的串行
- 说到混淆，说说自己所有能想到的。实践，混淆流程，遇到的坑，做一些特殊事情时关注到的点，比如热修复等。
- 为什么主线程不会因为Loop.loop()死循环卡死？
- 说一下Android运行时权限的最佳实践；权限的底层实现原理，可以从申请权限，记录权限，权限弹框ui等流程回答；如果设计权限机制，你会怎么做？
- 平时有阅读framework源码的习惯吗？用什么方式阅读的？
- viewpager嵌套viewpager，会产生哪些问题，从事件分发以及vp源码上解释。可以从正常滑动，快滑，边界触摸滑动来分场景描述。
- 谈谈apk的安装流程
- 写一个简单的java类，可以涉及到简单的计算，try catch，反复编译它，查看它的字节码，尝试读懂这些字节码
- Parcelable通过bundle用于activity，fragment间参数传递，存取是同一个对象吗？
- intent与bundle的序列化和反序列化规则，顺便思考下序列化与反序列化的关键点
- 为什么parcelable比serializable效率高
- library中R.java中的资源id字段，为何不用final修饰？
- N的时候推出了FileProvider，你能说清楚到底哪些场景需要适配？系统是如何检测FileUriExposeException的？为什么在这个过程中会有权限相关参与进来？
- 自定义控件，考虑过多指触控么？没考虑有什么影响？如果现在有一个只考虑单指操作的控件，支持多指，需要做哪些？
- 谈下Android签名校验机制，v1，v2，v3各有什么不同？思考下为什么会有这样的发展趋势？
- 谈谈jvm的垃圾回收机制？思考为什么会有多种垃圾回收算法？
- 谈谈你对push了解的一切，接入，类型，提升到达率，原理，一些场景下需要注意的点等
- 谈谈对于rxjava的理解，可以尝试描述一些使用场景，以及原理
- JVM和Dalvik各有什么特点？思考Android为什么会选择使用Dalvik
- 简单描述下自定义gradle plugin的步骤，有哪些注意事项？自己定义过或者见过哪些plugin分别应用与哪些场景。
- .9图需要切多套吗?
- 聊一下平时用Android Studio的代码调试技巧？断点不阻塞直接加log，直接断点修改代码执行分支等类似经验
- 很多时候我们都在谈，RxJava会内存泄漏，我们要及时dispose，各种方案层出不穷。那么我想问，可否针对性的分析，哪些场景或者操作符会造成内存泄露，哪些内存泄漏是长期的？
- view.getContext返回的一定是activity吗？如果不一定，在什么场景下会发生变化。
- 考虑捕获用户在一个activity页面中的所有点击行为，有哪些方案？你会怎么设计实现？
- 谈谈对volatile，synchronize，cas的理解，可以考虑下使用场景？底层原理？以及happens before， lost wake up这些相关规则及原理
- 并发涉及到的关键词：原子性，可见性，有序性，死锁，活锁，饥饿你可能说清楚每个对应的知识点么，最好能举例并给出解决方案
- aop方案aspectj的原理是什么？能否注入第三方库，为什么？
- 顶部为webview，底部为recyclerview类似于一些新闻页面的详情页，如何做到连贯的滑动，fling，说出思路即可
- View中getContext一定返回的是Activity对象吗？
- 哪些 Context调用 startActivity 需要设置NEW_TASK，为什么？ 2019-07-18  
- Android 常见的制作圆角方案，有哪几种常见方式？ 在Android P上什么兼容性问题 2019-07-22

## 面经
