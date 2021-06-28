---
title: Android事件分发机制(二)
tags: 基础
categories: Android
date: 2019-09-23 10:14:48
thumbnail:
top: 0
---


## 前言

接上节，在这一小节中，我将着重从View和ViewGroup的源码中去探索事件分发的流程是否如上小节分析的那样，带着上一节所留下的疑问开始愉快的阅读源码吧。

<!--more-->

## 源码分析

先放上小节中总结的流程图：

![ ](https://dreamweaver.img.we1code.cn/android%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%283%29.png)

在Activity中最终也是走的ViewGroup.dispatchTouchEvent，所以直接看ViewGroup就可以了：

首先我们要明确一次完整的事件分发包括ACTION_DOWN，若干ACTION_MOVE，ACTION_UP事件

```java
final int action = ev.getAction();
final int actionMasked = action & MotionEvent.ACTION_MASK;
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
```

此处做重置工作，如果当前的MotionEvent是ACTION_DOWN，则cancel所有未执行完的事件，并清除所有的TouchTarget，这个TouchTarget是什么?它记录了所有被点击的View和MotionEvent的id，如果有设置
FLAG_DISALLOW_INTERCEPT也会被一并清除掉。

```java
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}

// If intercepted, start normal event dispatch. Also if there is already
// a view that is handling the gesture, do normal event dispatch.
if (intercepted || mFirstTouchTarget != null) {
    ev.setTargetAccessibilityFocus(false);
}
```

此处判断事件是否被拦截，在这里我们看到了熟悉的onInterceptTouchEvent，当前事件为ACTION_DOWN时，首先会判断有没有设置FLAG_DISALLOW_INTERCEPT标识，这个标识是通过requestDisallowInterceptTouchEvent设置的，一般是子View控制父类不去拦截事件，前面分析到了这个flag会在执行ACTION_DOWN事件时被重置，如果不允许被拦截，那么事件当然交予子View去处理啦，反之，则会执行onInterceptTouchEvent方法。如果是其他的事件，则需要考虑mFirstTouchTarget是否为null，在下面的代码中可以看到如果事件交予子控件处理，那么mFirstTouchTarget将被赋值，因此如果事件没有交予子View处理，mFirstTouchTarget就是null值，那么接下来的所有事件都不会交予子View处理了，而且也不会执行onInterceptTouchEvent。

```java
final View[] children = mChildren;
for (int i = childrenCount - 1; i >= 0; i--) {
    final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
    final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);

    if (childWithAccessibilityFocus != null) {
        if (childWithAccessibilityFocus != child) {
            continue;
        }
        childWithAccessibilityFocus = null;
        i = childrenCount - 1;
    }
    // 判断子View是否能收到点击事件和点击事件是否在View内发生
    if (!canViewReceivePointerEvents(child) || !isTransformedTouchPointInView(x, y, child, null)) {
        ev.setTargetAccessibilityFocus(false);
        // 如果子View收不到事件，进行下一从循环，一直到找到目标子View
        continue;
    }
    newTouchTarget = getTouchTarget(child);
    if (newTouchTarget != null) {
        // Child is already receiving touch within its bounds.
        // Give it the new pointer in addition to the ones it is handling.
        newTouchTarget.pointerIdBits |= idBitsToAssign;
        break;
    }
    resetCancelNextUpFlag(child);
    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
        // Child wants to receive touch within its bounds.
        mLastTouchDownTime = ev.getDownTime();
        if (preorderedList != null) {
            // childIndex points into presorted list, find original index
            for (int j = 0; j < childrenCount; j++) {
                if (children[childIndex] == mChildren[j]) {
                    mLastTouchDownIndex = j;
                    break;
                }
            }
        } else {
            mLastTouchDownIndex = childIndex;
        }
        mLastTouchDownX = ev.getX();
        mLastTouchDownY = ev.getY();
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }
    // The accessibility focus didn't handle the event, so clear
    // the flag and do a normal dispatch to all children.
    ev.setTargetAccessibilityFocus(false);
 }

// 判断子View能收到点击事件的条件
private static boolean canViewReceivePointerEvents(@NonNull View child) {
    return (child.mViewFlags & VISIBILITY_MASK) == VISIBLE
        || child.getAnimation() != null;
}
```

接下来分发事件到子View，如果子View能收到点击事件，并且点击事件在子View的范围之内，这里判断子View能否收到点击事件的条件在于它是否可见，或子View的mCurrentAnimation不为null。事件交由子View去处理，如果子View处理了该次事件，则会通过addTouchTarget记录起来。确定了事件被消费后，就会结束循环。

```java
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
} else {
    // Dispatch to touch targets, excluding the new touch target if we already
    // dispatched to it.  Cancel touch targets if necessary.
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    while (target != null) {
        final TouchTarget next = target.next;
        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
            handled = true;
        } else {
            final boolean cancelChild = resetCancelNextUpFlag(target.child) || intercepted;
            if (dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)) {
                handled = true;
            }
            if (cancelChild) {
                if (predecessor == null) {
                    mFirstTouchTarget = next;
                } else {
                    predecessor.next = next;
                }
                target.recycle();
                target = next;
                continue;
            }
        }
        predecessor = target;
        target = next;
    }
}
```

如果没有子View处理这次事件，则会执行super.dispatchTouchEvent，交给View的dispatchTouchEvent处理，代码如下：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    // If the event should be handled by accessibility focus first.
    if (event.isTargetAccessibilityFocus()) {
        // We don't have focus or no virtual descendant has it, do not handle the event.
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
    }

    boolean result = false;

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Defensive cleanup for new gesture
        stopNestedScroll();
    }

    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

在View中的事件分发逻辑就比ViewGroup少多了，咱们挑重点看

```java
ListenerInfo li = mListenerInfo;
if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED && li.mOnTouchListener.onTouch(this, event)) {
    result = true;
}

if (!result && onTouchEvent(event)) {
    result = true;
}
```

在这里，可以看到View会先判断OnTouch事件，如果有OnTouchListener成功处理了这次事件，那么就不会执行onTouchEvent方法了。

至此，Android的事件分发机制就基本分析完了，总的来说，ViewGroup将事件分发给子View，并询问子View是否能处理这次事件，如果事件被拦截了，或者没有子View处理，则执行自己的onTouchEvent，并将dispatchTouchEvent的结果反馈给父类。

下面有几个疑问，想问下读者，也顺便提醒下自己

- 如果ACTION_DOWN事件没有被处理过，那么mFirstTouchTarget一定为null吗？
- 可以看到在将事件分发给子View主要是通过dispatchTransformedTouchEvent方法的，在ViewGroup的dispatchTouchEvent中会遍历一次所有的子View，然后通过dispatchTransformedTouchEvent去询问子View是否有处理过事件，但是在dispatchTouchEvent的最后面可以看到，对于mFirstTouchTarget != null时，会再对它做一次事件分发，为什么要这么做呢？
