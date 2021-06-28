---
title: Fragment原理浅析
tags: framework
categories: Android
date: 2019-09-29 19:02:46
thumbnail:
top: 0
---


## 前言

Fragment在日常开发中非常的常用，一版都是配合ViewPager或FrameLayout使用，我们基本不用担心操作它attachToActivity，因为FragmentManager都帮我们处理好了。那么Fragment是如何绑定Activity的生命周期的呢？系统是如何将Fragment添加到视图层的呢？Fragment的回退栈又是什么呢？带着这些问题我们开始探索Fragment的源码吧。

<!--more-->

## 基本操作

首先来回顾一下，我们如何添加Fragment的:

```kotlin
val fm = supportFragmentManager
val ts = fm.beginTransaction()
ts.add(fragment)
ts.commit()
```

首先弄清几个概念：

- FragmentController 主要作用是绑定Activity与Fragment的生命周期，在FragmentActivity可以看到在每个生命周期函数，FragmentController都有做分发，最终交给了FragmentManager处理
- FragmentManager Fragments的直接操作者，管理Fragment的内部状态以及添加\移除\隐藏\显示Fragment等操作
- FragmentTransaction 对Fragment操作的集合，各项操作会存储在Ops中，最终在FragmentManager中被执行

> tips: FragmentTransaction本身是一个抽象类，它包含着一个内部类Op，根据其构造函数可以看出来这个类用于记录Fragment的操作，并将这一系列操作存储在mOps，其中四个抽象方法commit/commitAllowingStateLoss/commitNow/commitNowAllowingStateLoss就是我们经常放在最后执行的方法了。

```java
// fm.beginTransaction()
@NonNull
@Override
public FragmentTransaction beginTransaction() {
    return new BackStackRecord(this);
}
```

BackStackRecord继承了FragmentTransaction，可以看到在这个类中最终还是调用了FragmentManager的enqueueAction方法，将所有的操作加入执行队列中。并对需要记录Fragment回退栈的操作做如下处理：

```java
public int allocBackStackIndex(BackStackRecord bse) {
    synchronized (this) {
        if (mAvailBackStackIndices == null || mAvailBackStackIndices.size() <= 0) {
            if (mBackStackIndices == null) {
                mBackStackIndices = new ArrayList<BackStackRecord>();
            }
            int index = mBackStackIndices.size();
            if (DEBUG) Log.v(TAG, "Setting back stack index " + index + " to " + bse);
            // 将当前操作添加到数组中
            mBackStackIndices.add(bse);
            return index;
        } else {
            // 找到一个可用的位置进行存储当前操作
            int index = mAvailBackStackIndices.remove(mAvailBackStackIndices.size()-1);
            if (DEBUG) Log.v(TAG, "Adding back stack index " + index + " with " + bse);
            mBackStackIndices.set(index, bse);
            return index;
        }
    }
}
```

最终调用了FragmentManager的enqueueAction/execSingleAction：

```java
public boolean execPendingActions() {
    // 校验准备工作
    ensureExecReady(true);
    boolean didSomething = false;
    // 初始化数据源
    while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
        mExecutingActions = true;
        try {
            removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
        } finally {
            // 清除执行程序
            cleanupExec();
        }
        didSomething = true;
    }
    updateOnBackPressedCallbackEnabled();
    // 等待加载延迟的Fragment
    doPendingDeferredStart();
    burpActive();
    return didSomething;
}

public void execSingleAction(OpGenerator action, boolean allowStateLoss) {
    if (allowStateLoss && (mHost == null || mDestroyed)) {
        // This FragmentManager isn't attached, so drop the entire transaction.
        return;
    }
    ensureExecReady(allowStateLoss);
    if (action.generateOps(mTmpRecords, mTmpIsPop)) {
        mExecutingActions = true;
        try {
            removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
        } finally {
            cleanupExec();
        }
    }

    updateOnBackPressedCallbackEnabled();
    doPendingDeferredStart();
    burpActive();
}
```

无论是执行execPendingActions还是execSingleAction，其核心方法还是removeRedundantOperationsAndExecute，这个方法可以移除冗余的操作，举个例子，如果两个事务一起执行，一个用于添加FragmentA，一个用于将FragmentA替换成FragmentB，实际上只有FragmentB会被添加，无法感应到FragmentA的创建/销毁生命周期。这个就是移除冗余操作的副作用了，Fragment的状态不会如预想那样变化。
*疑问，这个方法是如何去除冗余操作的呢？*

```java
// 移除冗余的回退栈操作再执行，需要设置setReorderingAllowed(true)
private void removeRedundantOperationsAndExecute(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop) {
    // 省略...
    final int numRecords = records.size();
        int startIndex = 0;
        for (int recordNum = 0; recordNum < numRecords; recordNum++) {
            final boolean canReorder = records.get(recordNum).mReorderingAllowed;
            // 所有事务如果设置了setReorderingAllowed(true)则全部跳过在最后一起执行
            if (!canReorder) {
                // execute all previous transactions
                // 如果中间有事务A没有设置setReorderingAllowed(true)，则从startIndex到事务A会被一起执行
                if (startIndex != recordNum) {
                    executeOpsTogether(records, isRecordPop, startIndex, recordNum);
                }
                // execute all pop operations that don't allow reordering together or one add operation
                // 上述注释说明此处执行所有不允许一起排序的pop操作
                // 在BackStackRecord中isRecordPop都为false，在PopBackStackState中isRecordPop都为true，这两个类分别对应着入栈和出栈，且仅当BackStackRecord设置了addToBackStack后才会被记录
                int reorderingEnd = recordNum + 1;
                if (isRecordPop.get(recordNum)) {
                    while (reorderingEnd < numRecords
                            && isRecordPop.get(reorderingEnd)
                            && !records.get(reorderingEnd).mReorderingAllowed) {
                        reorderingEnd++;
                    }
                }
                executeOpsTogether(records, isRecordPop, recordNum, reorderingEnd);
                startIndex = reorderingEnd;
                recordNum = reorderingEnd - 1;
            }
        }
        if (startIndex != numRecords) {
            executeOpsTogether(records, isRecordPop, startIndex, numRecords);
        }
    }
```

接着往下看executeOps

```java
private static void executeOps(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
    for (int i = startIndex; i < endIndex; i++) {
        final BackStackRecord record = records.get(i);
        final boolean isPop = isRecordPop.get(i);
        if (isPop) {
             record.bumpBackStackNesting(-1);
            boolean moveToState = i == (endIndex - 1);
            // 执行PopBackStackState
            record.executePopOps(moveToState);
        } else {
            record.bumpBackStackNesting(1);
            // 执行BackStackRecord
            record.executeOps();
        }
    }
}
```

然后是BackStackRecord的executeOps，最终这些ops由FragmentManager处理，将Fragment添加至mAdded或者从mAdded中移除，并对Fragment的内部状态进行修改。

最后也是最重要的方法moveToState，它主要负责修改Fragment的生命周期状态，在这我们可以看到Fragment是如何被添加至容器中的，在此Fragment中内部状态通过FragmentManger更新。

```java
case Fragment.CREATED:
    //省略...
    f.mContainer = container;
    f.performCreateView(f.performGetLayoutInflater(f.mSavedFragmentState), container, f.mSavedFragmentState);
    if (f.mView != null) {
        f.mInnerView = f.mView;
        f.mView.setSaveFromParentEnabled(false);
        if (container != null) {
            // 将fragment的视图添加到宿主的容器中
            container.addView(f.mView);
        }
        if (f.mHidden) {
            f.mView.setVisibility(View.GONE);
        }
        // 省略...
    } else {
        f.mInnerView = null;
    }
    // 省略...
```

至此，文章开始的疑问差不多都解决了，最后再梳理一下Fragment初始化流程。流程图大体如下：

![ ](https://dreamweaver.img.we1code.cn/fragment%E5%88%9D%E5%A7%8B%E5%8C%96%E8%BF%87%E7%A8%8B.png)

## 结语

Fragment的逻辑复杂，如果仅仅是靠读源码，是无法理清其复杂的逻辑关系的。此文的目的只是对Fragment做一次简单的探索，弄清楚它是如何被添加到视图的，如何去感知Activity的生命周期的，至于它的高级用法以及使用注意事项将会发布在其后的文章。
