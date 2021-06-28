---
title: RecyclerView源码分析
tags: library
categories: Android
date: 2019-07-08 00:00:00
img: https://dreamweaver.img.we1code.cn/android-recyclerview-fastadapter-header.png
top: true
---

## 前言

RecyclerView想必大家都不陌生了，通过配合使用ListAdapter和LinearLayoutManager就能很容易的实现一个列表的展示，本篇文章将着重分析RecyclerView的这些部件是如何各司其职的，以及它的复用机制。

- RecyclerView是如何与Adapter和LayoutManager解耦的
- Recylcer是如何管理ViewHolder的复用的

带着这两个问题进入源码分析

## 源码分析

以setAadapter作为切入点，开始探索RecyclerView的源码。

```java
    /**
     * Set a new adapter to provide child views on demand.
     * <p>
     * When adapter is changed, all existing views are recycled back to the pool. 
     * If the pool has only one adapter, it will be cleared.
     *
     * @param adapter The new adapter to set, or null to set no adapter.
     * @see #swapAdapter(Adapter, boolean)
     */
    public void setAdapter(@Nullable Adapter adapter) {
        // 是否停止一切Touch事件和滚动事件
        setLayoutFrozen(false);
        //1.原先的adapter注销观察者
        //2.把recycler中所有容器清空
        //3.新的adapter注册观察者
        setAdapterInternal(adapter, false, true);
        processDataSetCompletelyChanged(false);
        requestLayout();
    }
```

由上可知，setAdapter改变了RecyclerView中mAadapter的指向，并将观察者注册到新的adapter，从而监听其数据变化，然后重新执行测量，布局和绘制流程。

### onMeasure

实际上没什么好说的，就是一些常规的判断逻辑，分情况测量宽高什么的，值的注意的点是当LayoutManager开启了自动测量后(一般默认开启)，会先将子View先测量一遍，并且分别执行dispatchLayoutStep1、dispatchLayoutStep2进行预加载。

### onLayout

如果自动测量没有开启则会在onLayout中测量子View，然后执行dispatchLayoutStep3，这个方法存储该View的相关，并做相关的清理工作。

### onDraw

onDraw方法中就做了画ItemDecoration的操作。

### 复用机制

RecyclerView之所以能够高效的复用ViewHolder得益于其中的Recycler，Recycler中包含着多种容器，分别对attchedScrap，changedScrap等类别的ViewHolder进行缓存，看看注释中是怎么对Recycler进行定义的吧：

- Recycle(view) 之前被使用展示数据，被放在缓存中等待被再次使用展示相同类型的数据。
- Scrap(view) 在布局期间进入临时分离状态的子view，它可能还没有完全分离就被复用了，或者在没有重新绑定的需求后变的不可修改，或是被adapter认为是dirty的被修改。
- Dirty(view) 在被展示前，子view必须被adapter复原

```java
final ArrayList<RecyclerView.ViewHolder> mAttachedScrap = new ArrayList();
ArrayList<RecyclerView.ViewHolder> mChangedScrap = null;
final ArrayList<RecyclerView.ViewHolder> mCachedViews = new ArrayList();
private final List<RecyclerView.ViewHolder> mUnmodifiableAttachedScrap;
private int mRequestedCacheMax;
int mViewCacheMax;
RecyclerView.RecycledViewPool mRecyclerPool;
private RecyclerView.ViewCacheExtension mViewCacheExtension;
static final int DEFAULT_CACHE_SIZE = 2;
```

可以看到Recycler中维护着大量的容器，它们分别管理自己的ViewHolder，这些容器的存储/提取条件是什么呢？弄清了这个过程，RecyclerView的复用原理就差不多能理解了。

```java
        /**
         * Mark an attached view as scrap.
         *
         * <p>
         * "Scrap" views are still attached to their parent RecyclerView but are eligible
         * for rebinding and reuse. Requests for a view for a given position may return a
         * reused or rebound scrap view instance.</p>
         *
         * @param view View to scrap
         */
        void scrapView(View view) {
            final ViewHolder holder = getChildViewHolderInt(view);
            if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
                    || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
                //...
                mAttachedScrap.add(holder);
            } else {
                //...
                mChangedScrap.add(holder);
            }
        }
```

RecyclerView对mAttachedScrap.add和mChangedScrap.add的调用就只有一处，在scrapView这个方法中，注释也说的很明白，把attached view 标记为scrap。

那么什么时候从缓存中获取ViewHolder的呢？通过追踪，最终调用tryGetViewHolderForPositionByDeadline，复用逻辑已经写在此处了：

- If there is a changed scrap, try to find from there
- Find by position from scrap/hidden list/cache
- Find from scrap/cache via stable ids, if exists
- fallback to pool
- 如果上述4步都拿不到ViewHolder则通过createViewHolder实例化一个

根据代码逻辑，可以大概的了解RecyclerView的ViewHolder复用顺序了：

- 在mChangedScrap中查找
- 在mAttachedScrap中查找
- 在hiddenViews中查找
- 在mCachedViews中查找
- 在mAttachedScrap中通过id查找
- 在自定义的缓存扩展ViewCacheExtension中查找
- 在mRecyclerPool中查找
- 实例化一个ViewHolder
