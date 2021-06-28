---
title: Android Jetpack系列其二livedata
tags: jetpack
categories: Android
date: 2019-11-07 14:44:46
thumbnail:
top: 0
---

```flow
1. livedata实现原理
2. livedata实现双向绑定
```

## 前言

livedata是被观察者的持有类，并能感应生命周期。此篇文章重在分析两点

- livedata如何实现观察者模式的
- livedata是如何感知数据流的变化的
- livedata是如何感知lifecycleOwner的生命周期的

<!--more-->

## 使用与分析

livedata一般是配合viewmodel使用的，首先看看下面的使用案例

```kotlin
class StoreViewModel: ViewModel {
    // 初始化
    val articles: MutableLiveData<MutableList<Article>> = MutableLiveData()

    fun loadArticles(page: Int) {
        // 网络请求应放在对应repository中，这里为了方便说明
        CoroutineScope(Dispatchers.IO).launch {
            val articlesWrapper = apiCenter().homeArticles(page.toString())
            // 从服务器获取数据
            articles.postValue(articlesWrapper.data.datas.toMutableList())
        }
    }
}

class HomeFragment : BaseFragment() {
    fun initialized() {
        vm.articles.observe(this,
            Observer<MutableList<Article>> {
                // 通知UI更新
            })
    }
}
```

接下来看看，在livedata内部是如何感知数据流变化的

```java
// livedata的构造方法
public LiveData(T value) {
    mData = value;
    mVersion = START_VERSION + 1;
}

public LiveData() {
    mData = NOT_SET;
    mVersion = START_VERSION;
}

// 其中mData就是存储的数据了，version可以看做存储数据的当前版本

protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

// 发送数据到主线程，如果在主线程执行完任务之前postValue多次，那么只会分发最后一次的value
// ArchTaskExecutor.getInstance().postToMainThread其实就是调用主线程的Handler将value传递到主线程

private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};

@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

// 在这里会给mData重新赋值，然后做分发
// 在MutableLiveData中的setValue其实就是调用这个方法，在主线程中如果要直接给livedata赋值，可以直接调用此方法

// 再来看看livedata内部的观察者
private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();
// 它其实是个双向列表 有如下优点
// 1.直接移动指针插入且无需执行hash算法效率高
// 2.可以一边遍历一遍删除元素而不会引起ConcurrentModifiedException
// 3.使用双向链表存储数据比HashMap(java8)更节省空间

// 向livedata添加一个观察者
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    // 如果Map中不存在就放置进去并返回Null，如果存在直接拿出
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
// LifecycleBoundObserver 生命周期边界的观察者
// LifecycleBoundObserver 它既是livedata事件的观察者又是生命周期变化的观察者，换句话来说，它既能感知livedata数据流的变化也能感知lifecycleOwner生命周期的变化
// 它被添加到了mObservers中也被添加到了LifecycleRegistry的mObserverMap: FastSafeIterableMap中

// 再来看看livedata是如何将数据变化下发到每一个观察者的
// 如果传值就是通知指定的observer，传Null就是通知所有的observer
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
// 这里跳过分析SafeIterableMap，FastSafeIterableMap，先把它们当作普通的迭代器，着重看下considerNotify

private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}

// 前面说了，这里的ObserverWrapper的实例是LifecycleBoundObserver
// 通知观察者之前首先会检查lifecycleOwner的生命周期然后再检测数据的版本，都符合要求才会通知数据更新
```

到这里，livedata内部数据流动与感知的过程已大体分析完成了，它内部用SafeInterableMap作为观察者的容器，在数据发生变化的时候分发给所有的观察者，在lifecycleOwner生命周期发生变化时也会通过LifecycleBoundObserver的onStateChanged通知数据分发。

梳理了一下流程图：  

![lifecycleDispatch](https://dreamweaver.img.we1code.cn/LifecycleDispatcher.png)

## livedata实际应用

### livedata事件总线简单实现

```kotlin
import androidx.annotation.MainThread
import androidx.lifecycle.LifecycleOwner
import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.Observer
import java.lang.Exception

class LiveDataBus private constructor() {

    private val mLock = Object()

    companion object {
        val instance: LiveDataBus = LiveDataBus()
    }

    private val bus: HashMap<String, BusLiveData<Any>> by lazy {
        HashMap<String, BusLiveData<Any>>()
    }

    fun subscribe(owner: LifecycleOwner, observer: Observer<Any>, event: String) {
        if (bus.containsKey(event)) {
            val wrapper = BusLiveDataWrapper(observer)
            val liveData = bus[event]
            if (liveData?.alreadySubmit == false) {
                wrapper.scrapPreEvent = true
                liveData.alreadySubmit = true
            }
            bus[event]?.observe(owner, wrapper)
        }
    }

    fun postEvent(event: String, addition: Any) {
        if (bus.containsKey(event)) {
            bus[event]?.postValue(addition)
        } else {
            synchronized(mLock) {
                bus[event] = BusLiveData(addition)
            }
        }
    }

    @MainThread
    fun setEvent(event: String, addition: Any) {
        if (bus.containsKey(event)) {
            bus[event]?.value = addition
        } else {
            bus[event] = BusLiveData(addition)
        }
    }

    inner class BusLiveData<T>(t: T) : MutableLiveData<T>(t) {
        var alreadySubmit: Boolean = false
    }

    inner class BusLiveDataWrapper<T> constructor(
        private val observer: Observer<T>,
        var scrapPreEvent: Boolean = false
    ) :
        Observer<T> {

        override fun onChanged(t: T) {
            if (scrapPreEvent) {
                scrapPreEvent = true
                return
            }
            try {
                observer.onChanged(t)
            } catch (e: Exception) {
                // catch ClassCastException etc.
                e.printStackTrace()
            }
        }
    }
}
```

## 结语

最后，做下总结，livedata的大致流程已经分析完了，但是比较重要的类LifecycleRegistry只是一笔带过了没有做分析，它是如何将生命周期变化事件分发到订阅者的呢？另外还有两个比较重要的数据结构FastSafeIterableMap和SafeIterableMap，在这里先记录一下啦，以后再填上吧。🖖

## 参考

- [LiveData Overview](https://developer.android.com/topic/libraries/architecture/livedata) by Android Developers
- [用LiveDataBus替代RxBus、EventBus——Android消息总线的演进之路](https://juejin.im/post/5b5ac0825188251acd0f3777) by 美团技术团队 海亮
