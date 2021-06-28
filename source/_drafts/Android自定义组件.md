---
title: 深入理解View和ViewGroup的绘制流程
tags:
categories: Android
date: 2018/04/11
---
### 前言

最近想着做一些复杂的自定义控件练手，仿造了一些看上去比较炫酷的自定义控件，虽然做的效果和原版差不多，但是在制作过程中发现自己对自定义控件的流程并没有那么熟悉，需要查询资料反复琢磨，究其原因还是对这一套流程不是很熟悉，于是想借此文章总结一下流程，记录一些小细节，如果以后又忘了，就不用导出查找资料了。

### 界面加载流程

众所周知，在一个Activity中通过setContentView()方法去加载视图，但是在这个方法背后是系统做了什么呢?布局文件中的层级结构是如何显示在屏幕上的？就这两个问题我们在源码中找一下答案。首先看下Android窗口解构图：
![windows](http://dreamweaver.img.we1code.cn/window.png)
在setContentView()中调用的是Window对象的setContentView()

```java
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

而在Activity中attach()中mWindow初始化为PhoneWindow

```java
 final void attach(...) {
    ...
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
 }

```

PhoneWindow中的setContentView()

```java
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }

        @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            mContentParent.addView(view, params);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

上面为两处setContentView()的实现，大体上结构相同，先是判断父级容器是否存在，如果不存在installDecor()，再去判断是否存在过场动画，不存在removeAllViews()然后再添加，一种是通过布局文件加载，一种是直接添加View，获取Scene的方法有所不同，getSceneForLayout()将scene通过layoutId缓存，这样做的目的是相同的布局文件可以共享相同的scenes。然后到了LayoutInflater中，层层遍历xml布局，并将子View加入到contentParent中(contentParent是DecorView的android.R.id.content)

在ActivityThread中handleResumeActivity(),vm.addView(decor, l)，vm的实现类为WindowManagerImpl，而在WindowManagerImpl中，addView又是通过WindowManagerGlobal.addView()实现的，在WindowManagerGlobal中addView最终又是通过ViewRootImpl.setView()实现的

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                requestLayout();
}

@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

void doTraversal() {
    if (mTraversalScheduled) {
        ...
        performTraversals();
        ...
    }
}
private void performTraversals() {
    performMeasure()
    performLayout()
    performDraw()
}
```

### Android事件分发机制

### 其他注意事项

### 结语
