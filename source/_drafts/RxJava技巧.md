---
title: 5分钟让你放弃RxJava
tags: library
date: 2017/03/15
categories: Java
img: http://dreamweaver.img.we1code.cn/rxjava.png
---

### 前言

>Rx即Reactive Extensions的缩写,它是一个使用可观察异步数据进行异步编程的编程接口,让开发者更方便的编写异步和基于事件的程序.
<!--more-->

### RxJava的Observable类型

#### Completable

##### 文档定义

    a flow without items but only a completion or error signal

```java
public interface CompletableEmitter {
    //只处理onComplete和onError
    void onComplete();
    void onError(@NonNull Throwable var1);

    void setDisposable(@Nullable Disposable var1);
    void setCancellable(@Nullable Cancellable var1);
    boolean isDisposed();
    @Experimental
    boolean tryOnError(@NonNull Throwable var1);
}
```

##### 使用案例

```java
Completable.create(completableEmitter ->
    completableEmitter.onError(new Exception("test exception"))
).subscribe(new CompletableObserver() {
    @Override
    public void onSubscribe(Disposable disposable) {
        System.out.println("onSubscribe");
    }

    @Override
    public void onComplete() {
        System.out.println("onComplete");
    }

    @Override
    public void onError(Throwable throwable) {
        System.out.println("onError");
    }
});
```

##### 使用场景

> 请求服务器时,如果只需要关心是否成功,无需处理返回的数据时,我们可以使用Completable

##### 常用操作符

|操作符|描述|使用场景|
|:---:|:---|---:|:---:|
|compose|对Observale进行封装(合并Observale相关的操作符)|组合线程变换相关的操作符|
|lift|对Observer进行封装，将下游订阅者包装成一个上游订阅者|框架内大多数的操作符|
|flatMap||多个相互依赖的网络请求|
|merge|将多个发射源合成一个，相继发出|合并页面的多个请求|
|concat|将两个源发射的item链接在一起发射出来|取数据先检查缓存的场景|
|throttleFirst(节流阀)|定期发射的第一项数据|界面按钮需要防止连续点击的情况|
|zip|将多个发射源合并成一个，转换成一个后发出|可以用户多个接口数据合并|
|timer|延迟执行|可用与闪频页面|
|interval|轮询|轮询任务|

#### Rxjava是如何切换线程的

总结
subscribeOn()  
订阅顺序当从下到上，上游的ObservableSource被订阅时，先切换线程，然后立即执行task;
当存在多个subscribeOn()方法时，仅第一个subscribeOn()有效。  
observerOn()  
订阅顺序当从下到上，上游的ObservableSource被订阅时，会将对应的worker创建并作为构造参数存储在Observer的装饰器中，并不会立即切换线程；
当数据由上游发送过来时，先将数据存储到队列中，然后切换线程，然后在新的线程中将数据发送给下游的Observer；
当存在多个observerOn()方法时，仅对距下游下一个observerOn()之前的observer有效

Rxjava的订阅顺序是从下往上执行的，每一次订阅会对下游的Observer进行包装，比如使用Observable调用SubscribeOn方法时，下游的Observer就会被包装成SubscribeOnObserver，调用ObserveOn时，会被包装成ObserveOnObserver，当订阅到SubscribeOnObserver时，ObserveOnObserver发送数据时切换，它就会用自己内部的scheduler创建出的work内执行source.subscribe(parent)，也就是说将上游的subscribeActual切换到了自己scheduler对应的线程中执行。
subscribeOn对上游的Obserable起作用(从下到上起效果)，observeOn对下游的Observer起作用（从上到下起效果）  
当存在多个subscribeOn()方法时，仅第一个subscribeOn()有效。  
当存在多个observerOn()方法时，仅对距下游下一个observerOn()之前的observer有效

#### Flowable

被观察者发送消息太快以至于它的操作符或者订阅者不能及时处理相关的消息，从而操作消息的阻塞现象。

背压策略

- MISSING 这种就是没有策略，出现背压则抛出异常
- ERROR 抛出 MissingBackpressureException
- BUFFER 缓存策略 缓存池大小为128超出则抛异常
- DROP onNext值会丢失
- LATEST 一直保留最新的值，直到被消费

    0..N flows, supporting Reactive-Streams and backpressure
    // 可以发射多个数据,以成功或错误终止,支持BackpressureStrategy

#### Maybe

    a flow with no items, exactly one item or an error
    // 只能够发射0到1个数据

#### Observable

    0..N flows, no backpressure
    // Observable非背压,发射多个数据

#### Single

    a flow of exactly 1 item or an error
    // Single只能发射一个数据或error

### RxJava原理总结

- 在subscribeActual()方法中，源头和终点关联起来。
- source.subscribe(parent);这句代码执行时，才开始从发送ObservableOnSubscribe中利用ObservableEmitter发送数据给Observer。即数据是从源头push给终点的。
- CreateEmitter 中，只有Observable和Observer的关系没有被dispose，才会回调Observer的onXXXX()方法
- Observer的onComplete()和onError() 互斥只能执行一次，因为CreateEmitter在回调他们两中任意一个后，都会自动dispose()。根据上一点，验证此结论。
- 先error后complete，complete不显示。 反之会crash
- 还有一点要注意的是onSubscribe()是在我们执行subscribe()这句代码的那个线程回调的，并不受线程调度影响。

订阅的过程，是从下游到上游依次订阅的。

- 即终点 Observer 订阅了 map 返回的ObservableMap。
- 然后map的Observable(ObservableMap)在被订阅时，会订阅其内部保存上游Observable，用于订阅上游的Observer是一个装饰者(MapObserver)，内部保存了下游（本例是终点）Observer，以便上游发送数据过来时，能传递给下游。
- 以此类推，直到源头Observable被订阅，根据上节课内容，它开始向Observer发送数据。
- 数据传递的过程，当然是从上游push到下游的，

- 源头Observable传递数据给下游Observer（本例就是MapObserver）
- 然后MapObserver接收到数据，对其变换操作后(实际的function在这一步执行)，再调用内部保存的下游Observer的onNext()发送数据给下游
- 以此类推，直到终点Observer。
