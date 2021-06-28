---
title: Glide原理浅析
date: 2018-12-04
categories: Android
thumbnail: 
tags: library
---

想必做Android开发的同学都用过glide,glide用法非常简单

```java
Glide.with(this).load(url).into(imageView);
```

用这一行代码就能简单的加载图片了
下面看看glide底层逻辑
Glide.with 可以接受Activity、Fragment、Context等对象,其方法内部都是调用getRetriever(Context)去得到RequestManagerRetriever对象
看看RequestManagerRetriever的作用

```java
/**
 * A collection of static methods for creating new {@link com.bumptech.glide.RequestManager}s or
 * retrieving existing ones from activities and fragment.
 */
public class RequestManagerRetriever implements Handler.Callback {
    ...
}
```

文档上表明这个对象提供了一系列的静态方法去创建一个RequestManager或者从activities、fragment中追踪到一个已经存在的RequestManager

```java
public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
}

public static RequestManager with(@NonNull Activity activity) {
    return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull Fragment fragment) {
    return getRetriever(fragment.getContext()).get(fragment);
}

public static RequestManager with(@NonNull android.app.Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

public static RequestManager with(@NonNull View view) {
    return getRetriever(view.getContext()).get(view);
}

// RequestManagerRetriever 用于创建一个RequestManager或者从activities或者fragment寻回一个已经存在的RequestManager
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // Context could be null for other reasons (ie the user passes in null), but in practice it will
    // only occur due to errors with the Fragment lifecycle.
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
            // Glide.get(context) 单例 @GuardedBy 作用于变量或者函数，给它们加锁
            // 举个栗子
            // GuardedBy("objectLock")
            // volatile Object object;

            // Object getObject() {
            //     synchronized (objectLock) {
            //         if (object == null) {
            //             object = new Object();
            //         }
            //     }
            //     return object;
            // }
            // Glide.glide赋值
    return Glide.get(context).getRequestManagerRetriever();
}

// RequestManager由RequestManagerRetriever.get
public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }

public RequestManager get(@NonNull Fragment fragment) {
    Preconditions.checkNotNull(
        fragment.getContext(),
        "You cannot start a load on a fragment before it is attached or after it is destroyed");
    if (Util.isOnBackgroundThread()) {
      return get(fragment.getContext().getApplicationContext());
    } else {
      FragmentManager fm = fragment.getChildFragmentManager();
      return supportFragmentGet(fragment.getContext(), fm, fragment, fragment.isVisible());
    }
  }

@Deprecation
@NonNull
public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }

@SuppressWarnings("deprecation")
@NonNull
public RequestManager get(@NonNull View view) {
    if (Util.isOnBackgroundThread()) {
      return get(view.getContext().getApplicationContext());
    }

    Preconditions.checkNotNull(view);
    Preconditions.checkNotNull(
        view.getContext(), "Unable to obtain a request manager for a view without a Context");
    Activity activity = findActivity(view.getContext());
    // The view might be somewhere else, like a service.
    if (activity == null) {
      return get(view.getContext().getApplicationContext());
    }

    // Support Fragments.
    // Although the user might have non-support Fragments attached to FragmentActivity, searching
    // for non-support Fragments is so expensive pre O and that should be rare enough that we
    // prefer to just fall back to the Activity directly.
    if (activity instanceof FragmentActivity) {
      Fragment fragment = findSupportFragment(view, (FragmentActivity) activity);
      return fragment != null ? get(fragment) : get((FragmentActivity) activity);
    }

    // Standard Fragments.
    android.app.Fragment fragment = findFragment(view, activity);
    if (fragment == null) {
      return get(activity);
    }
    return get(fragment);
  }

@Deprecated
@NonNull
@TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
public RequestManager get(@NonNull android.app.Fragment fragment) {
    if (fragment.getActivity() == null) {
      throw new IllegalArgumentException(
          "You cannot start a load on a fragment before it is attached");
    }
    if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
      return get(fragment.getActivity().getApplicationContext());
    } else {
      android.app.FragmentManager fm = fragment.getChildFragmentManager();
      return fragmentGet(fragment.getActivity(), fm, fragment, fragment.isVisible());
    }
  }

// 如果当前线程不是主线程 拿到最上层的requestManager applicationRequestManager
// 如果在主线程(当前SupportRequestManagerFragment在前台)通过supportFragmentGet拿到当前fragment

  @NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }

  @NonNull
  private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      current = pendingSupportRequestManagerFragments.get(fm);
      if (current == null) {
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        if (isParentVisible) {
          current.getGlideLifecycle().onStart();
        }
        pendingSupportRequestManagerFragments.put(fm, current);
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }
  // 从当前pendingSupportRequestManagerFragments取SupportRequestManagerFragment，如果没有则new一个，触发onStart()生命周期
  // 添加到pendingSupportRequestManagerFragments中，再发送Message从pendingSupportRequestManagerFragments中移除？不知道为啥这么做
  // 回到supportFragmentGet从Fragment中取RequestManager。如果没有则new一个存到fragment中。至此取得requestManager

  // 看看RequestManager.load() 方法重载
  public RequestBuilder<Drawable> load(@Nullable Bitmap bitmap) {
    return asDrawable().load(bitmap);
  }

  public RequestBuilder<Drawable> load(@Nullable Drawable drawable) {
    return asDrawable().load(drawable);
  }

  public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
  }

  public RequestBuilder<Drawable> load(@Nullable Uri uri) {
    return asDrawable().load(uri);
  }

  public RequestBuilder<Drawable> load(@Nullable File file) {
    return asDrawable().load(file);
  }

  public RequestBuilder<Drawable> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
    return asDrawable().load(resourceId);
  }

  public RequestBuilder<Drawable> load(@Nullable URL url) {
    return asDrawable().load(url);
  }

  public RequestBuilder<Drawable> load(@Nullable byte[] model) {
    return asDrawable().load(model);
  }

  public RequestBuilder<Drawable> load(@Nullable Object model) {
    return asDrawable().load(model);
  }

  // 返回的都是Drawable对象

  public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
  }

  public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
  }

  // asDrawable 做了一个RequestBuilder对象 调用RequestBuilder.loadGeneric
    @NonNull
  private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
  }

  @NonNull
  public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    BaseRequestOptions<?> requestOptions = this;
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }

  private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    Request previous = target.getRequest();
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      // If the request is completed, beginning again will ensure the result is re-delivered,
      // triggering RequestListeners and Targets. If the request is failed, beginning again will
      // restart the request, giving it another chance to complete. If the request is already
      // running, we can let it continue running without interruption.
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        // Use the previous request rather than the new one to allow for optimizations like skipping
        // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
        // that are done in the individual Request.
        previous.begin();
      }
      return target;
    }

    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
  }
```

```java
/** A request that loads a resource for an {@link com.bumptech.glide.request.target.Target}. */
public interface Request {
  /** Starts an asynchronous load. */
  void begin();

  /**
   * Prevents any bitmaps being loaded from previous requests, releases any resources held by this
   * request, displays the current placeholder if one was provided, and marks the request as having
   * been cancelled.
   */
  void clear();

  /**
   * Similar to {@link #clear} for in progress requests (or portions of a request), but does nothing
   * if the request is already complete.
   *
   * <p>Unlike {@link #clear()}, this method allows implementations to act differently on subparts
   * of a request. For example if a Request has both a thumbnail and a primary request and the
   * thumbnail portion of the request is complete, this method allows only the primary portion of
   * the request to be paused without clearing the previously completed thumbnail portion.
   */
  void pause();

  /** Returns true if this request is running and has not completed or failed. */
  boolean isRunning();

  /** Returns true if the request has completed successfully. */
  boolean isComplete();

  /** Returns true if the request has been cleared. */
  boolean isCleared();

  /**
   * Returns true if a resource is set, even if the request is not yet complete or the primary
   * request has failed.
   */
  boolean isAnyResourceSet();

  /**
   * Returns {@code true} if this {@link Request} is equivalent to the given {@link Request} (has
   * all of the same options and sizes).
   *
   * <p>This method is identical to {@link Object#equals(Object)} except that it's specific to
   * {@link Request} subclasses. We do not use {@link Object#equals(Object)} directly because we
   * track {@link Request}s in collections like {@link java.util.Set} and it's perfectly legitimate
   * to have two different {@link Request} objects for two different {@link
   * com.bumptech.glide.request.target.Target}s (for example). Using a similar but different method
   * let's us selectively compare {@link Request} objects to each other when it's useful in specific
   * scenarios.
   */
  boolean isEquivalentTo(Request other);
}
```
