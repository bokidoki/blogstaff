---
title: Activity的启动流程（基于Android Q）
date: 2020-05-01 10:02:50
tags: framework
categories: Android
img:
---

### 前言

今天我们分析一下Android中四大组件的启动流程，分析之前我们先想一个问题

> 我们启动四大组件时只需要传入类名，那么系统在什么地方实例化它们的呢？

### Activity的启动流程(基于Android Q)

<!--more-->

涉及到的核心代码

```txt
\frameworks\base\core\java\android\content\ContextWrapper.java
\frameworks\base\core\java\android\app\ContextImpl.java
\frameworks\base\core\java\android\app\Instrumentation.java
\frameworks\base\services\core\java\com\android\server\wm\ActivityTaskManagerService.java
\frameworks\base\services\core\java\com\android\server\wm\ActivityStartController.java
\frameworks\base\services\core\java\com\android\server\wm\ActivityStarter.java
\frameworks\base\services\core\java\com\android\server\wm\ActivityStack.java
\frameworks\base\services\core\java\com\android\server\wm\ActivityStackSupervisor.java
\frameworks\base\core\java\android\app\servertransaction\ClientTransaction.java
\frameworks\base\core\java\android\app\ActivityThread.java
\frameworks\base\core\java\android\app\servertransaction\TransactionExecutor.java
```

常规操作从startActivity开始着手

```java
// ContextWrapper.java
@Override
// contextWrapper的startActivity，mBase就是ContextImpl啦
public void startActivity(Intent intent, Bundle options) {
    mBase.startActivity(intent, options);
}
```

```java
// ContextImpl.java
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
    // maintain this for backwards compatibility.
    // 就是说在activity之外启动activity Lanuch_mode不使用SingleTask是不允许的，但是在N-O之间是支持的，这里做了向下兼容处理
    final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

    if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && (targetSdkVersion < Build.VERSION_CODES.N
                    || targetSdkVersion >= Build.VERSION_CODES.P)
            && (options == null
                    || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                        + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                        + " Is this really what you want?");
    }
    // mMainThread就是ActivityThread啦，getInstrumentation就是Instrumentation类，它在两个地方可能被初始化attach/handleBindApplication
    // TODO: 这两个方法在哪调用的？
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}
```

```java
// instrumentation.java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
                intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
                synchronized (mSync) {
                        final int N = mActivityMonitors.size();
                        for (int i=0; i<N; i++) {
                                final ActivityMonitor am = mActivityMonitors.get(i);
                                ActivityResult result = null;
                                if (am.ignoreMatchingSpecificIntents()) {
                                        result = am.onStartActivity(intent);
                                }
                                if (result != null) {
                                        am.mHits++;
                                        return result;
                                } else if (am.match(who, null, intent)) {
                                        am.mHits++;
                                        if (am.isBlocking()) {
                                                return requestCode >= 0 ? am.getResult() : null;
                                        }
                                        break;
                                }
                        }
                }
        }
        try {
                intent.migrateExtraStreamToClipData();
                intent.prepareToLeaveProcess(who);
                // ActivityTaskManager.getService()返回的ActivityTaskManager
                // ATM在SystemServer.startBootstrapServices()中注册到SystemServiceManager中 TODO:SystemService中注册了哪几种类型的Service?
                int result = ActivityTaskManager.getService()
                        .startActivity(whoThread, who.getBasePackageName(), intent,
                                intent.resolveTypeIfNeeded(who.getContentResolver()),
                                token, target != null ? target.mEmbeddedID : null,
                                requestCode, 0, null, options);
                checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
                throw new RuntimeException("Failure from system", e);
        }
        return null;
}
```

```java
// ActivityTaskManagerService
    int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        // 从ActivityStarter中的DefaultFactory获取ActivityStarter，DefaultFactory有一个ActivityStarter池，大小为3
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }
```

```java
//ActivityStarter
    /**
     * Starts an activity based on the request parameters provided earlier.
     * @return The starter result.
     */
     // 这两个分支最终都会走到startActivityUnchecked
    int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                        mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            }
        } finally {
            onExecutionComplete();
        }
    }
```

中间都是一些校验和日志输出，在到realStartActivityLocked可以看到Activity的传递的消息被包装成了ClientTransaction，其中包含了回调列表和交易执行后，Activity应处于的最终生命周期状态。

```java
// ActivityStackSupervisor

boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
// 省略...
// Create activity launch transaction.
final ClientTransaction clientTransaction = ClientTransaction.obtain(
        proc.getThread(), r.appToken);

final DisplayContent dc = r.getDisplay().mDisplayContent;
clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
        System.identityHashCode(r), r.info,
        // TODO: Have this take the merged configuration instead of separate global
        // and override configs.
        mergedConfiguration.getGlobalConfiguration(),
        mergedConfiguration.getOverrideConfiguration(), r.compat,
        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
        r.icicle, r.persistentState, results, newIntents,
        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                r.assistToken));

// Set desired final state.
final ActivityLifecycleItem lifecycleItem;
if (andResume) {
        lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
} else {
        lifecycleItem = PauseActivityItem.obtain();
}
clientTransaction.setLifecycleStateRequest(lifecycleItem);

// Schedule transaction.
// 从这里绕到ClientLifecycleManager在绕回来，最后调用IApplicationThread mClient的scheduleTransaction，IApplicationThread对应的服务是ApplicationThread，它是ActivityThread的内部类，它也是个Binder，在哪注册的？
mService.getLifecycleManager().scheduleTransaction(clientTransaction);
// 省略...
}
```
