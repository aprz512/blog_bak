---
title: 003-Matrix源码分析:方法监控
date: 2020-7-1
categories: Matrix
---

在上一节中，我们看到过 AppMethodBeat 这个类，但是却没有介绍它，是因为这个玩意特别的难搞，需要单独的起一篇。

这个类其实是用来统计每个函数的耗时的，具体的可以看官方文档：

https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary

知道了每个函数的耗时，就可以找出卡顿的原因。

下面我们来分析这个类。



> com.tencent.matrix.trace.core.AppMethodBeat#status

```java
private static volatile int status = STATUS_DEFAULT;
```

这个字段还是挺重要的，它涉及到该类的状态的管理。

一开始是 STATUS_DEFAULT 状态

然后当我们调用了  i 方法之后，会变为 STATUS_READY 状态。

然后当我们调用了 onStart 方法之后，会变为 STATUS_STARTED 状态。

然后当我们调用了 onStop 方法之后，会变为 STATUS_STOPPED 状态。

需要注意一下调用的顺序，如果顺序不对，有些逻辑是不会走的。

由于，i 与 o 方法会插桩到我们的代码中，所以 i 肯定是会先于 onStart 方法执行，这个没有问题。



状态搞清楚了，我们看看 i 与 o 这两个方法。

> com.tencent.matrix.trace.core.AppMethodBeat#i

```java
    public static void i(int methodId) {
        if (status <= STATUS_STOPPED) {
            return;
        }
        if (methodId >= METHOD_ID_MAX) {
            return;
        }

        if (status == STATUS_DEFAULT) {
            synchronized (statusLock) {
                if (status == STATUS_DEFAULT) {
                    // 逻辑只会执行一次
                    realExecute();
                    status = STATUS_READY;
                }
            }
        }

        long threadId = Thread.currentThread().getId();
        if (sMethodEnterListener != null) {
            sMethodEnterListener.enter(methodId, threadId);
        }

        if (threadId == sMainThreadId) {
            if (assertIn) {
                android.util.Log.e(TAG, "ERROR!!! AppMethodBeat.i Recursive calls!!!");
                return;
            }
            assertIn = true;
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, true);
            } else {
                sIndex = 0;
                mergeData(methodId, sIndex, true);
            }
            ++sIndex;
            assertIn = false;
        }
    }
```

应用插件之后，在每个方法的前面都会加上这个 i 方法。该方法里面做了两件事：

- realExecute();
- mergeData

> com.tencent.matrix.trace.core.AppMethodBeat#realExecute

```java
    private static void realExecute() {
        MatrixLog.i(TAG, "[realExecute] timestamp:%s", System.currentTimeMillis());

        sCurrentDiffTime = SystemClock.uptimeMillis() - sDiffTime;

        sHandler.removeCallbacksAndMessages(null);
        // 开线程更新时间
        sHandler.postDelayed(sUpdateDiffTimeRunnable, Constants.TIME_UPDATE_CYCLE_MS);
        // 启动过期检查，可以忽略
        sHandler.postDelayed(checkStartExpiredRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (statusLock) {
                    MatrixLog.i(TAG, "[startExpired] timestamp:%s status:%s", System.currentTimeMillis(), status);
                    if (status == STATUS_DEFAULT || status == STATUS_READY) {
                        status = STATUS_EXPIRED_START;
                    }
                }
            }
        }, Constants.DEFAULT_RELEASE_BUFFER_DELAY);

        // hack H 
        ActivityThreadHacker.hackSysHandlerCallback();
        // 注册回调
        LooperMonitor.register(looperMonitorListener);
    }
```

realExecute 只会调用一次，里面启动了一个线程专门用来更新时间（隔5ms循环一次），原因是：

> 考虑到每个方法执行前后都获取系统时间（System.nanoTime）会对性能影响比较大，
>
> 而实际上，单个函数执行耗时小于 5ms 的情况，对卡顿来说不是主要原因，可以忽略不计，
>
> 如果是多次调用的情况，则在它的父级方法中可以反映出来，所以为了减少对性能的影响，
>
> 通过另一条更新时间的线程每 5ms 去更新一个时间变量，而每个方法执行前后只读取该变量来减少性能损耗。

还hack了 ActivityThread 的 H 的 callback，主要是用来拦截消息的处理，是一种很常用的 hook 方式，里面做了启动的耗时监测，暂时不深入，后面再说。

> com.tencent.matrix.trace.core.AppMethodBeat#mergeData

```java
    private static void mergeData(int methodId, int index, boolean isIn) {
        // 看注释这里是修复了一个bug，anr时间计算有问题，没看懂
        if (methodId == AppMethodBeat.METHOD_ID_DISPATCH) {
            sCurrentDiffTime = SystemClock.uptimeMillis() - sDiffTime;
        }
        long trueId = 0L;
        if (isIn) {
            trueId |= 1L << 63;
        }
        trueId |= (long) methodId << 43;
        trueId |= sCurrentDiffTime & 0x7FFFFFFFFFFL;
        // sBuffer 是一个long数组，long的结构：
        // 第1位是 1或者0，1是函数入口，0是函数出口
        // 2-21位是 methodId
        // 22-64位是 函数的执行前后距离 MethodBeat 模块初始化的时间，一个函数会有占两个问题，根据 methodId 就可以计算出函数耗时
        sBuffer[index] = trueId;
        // 该方法用于处理循环问题，sBuffer满了，会重头开始覆盖旧数据，主要是更新 indexRecord 链表头位置
        checkPileup(index);
        sLastIndex = index;
    }
```

这个方法也不难，就是往 sBuffer 里面添加数据，数据的结构注释也解释的很清楚了。

最终sBuffer大致长这样：

![Alt text](https://github.com/Tencent/matrix/wiki/images/trace/run_store.jpg)

我们再看看 o 方法：

> com.tencent.matrix.trace.core.AppMethodBeat#o

```java
    public static void o(int methodId) {
        if (status <= STATUS_STOPPED) {
            return;
        }
        if (methodId >= METHOD_ID_MAX) {
            return;
        }
        if (Thread.currentThread().getId() == sMainThreadId) {
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, false);
            } else {
                sIndex = 0;
                mergeData(methodId, sIndex, false);
            }
            ++sIndex;
        }
    }
```

就做了一件事就是 mergeData，这里就没啥好说的了。再看一下 sBuffer 的结构图，**只看上半部分**：

![Alt text](https://github.com/Tencent/matrix/wiki/images/trace/stack.jpg)

我们在 sBuffer 中找到 methodId 一致的，就可以拿到该函数的耗时。

一般情况下，我们需要获取的是 sBuffer 中的一段数据，比如执行 doFrame 消息的时候，我们想知道，它里面调用到了哪些函数，这个时候就需要记录一下 doFrame 前后的 sIndex，有一个内部类是专门做这个的：

> com.tencent.matrix.trace.core.AppMethodBeat.IndexRecord

```java
    public static final class IndexRecord {

        public int index;
        private IndexRecord next;
        public boolean isValid = true;
        public String source;

    }
```

index 字段是用来记录 sBuffer 中的位置的。next 说明它是一个链表。

用法如下：

比如我们在，分发消息之前，首先调用 `com.tencent.matrix.trace.core.AppMethodBeat#maskIndex` 方法，传递一个 source 作为参数，得到一个 IndexRecord 对象，然后在分发消息结束后，再获取拿到 sIndex 的值，这样就有了两个 sIndex。取出这个范围里面的数据就好了。

```java
AppMethodBeat.IndexRecord beginRecord = AppMethodBeat.getInstance().maskIndex("AnrTracer#dispatchBegin");
long[] data = AppMethodBeat.getInstance().copyData(beginRecord);
beginRecord.release();
```



AppMethodBeat 里面重要的方法就分析完了，我们看看上面忽略的 ActivityThreadHacker 这个类。这个类也很简单，主要就是 hook  ActivityThread的内部类 H 这个类。

hook Hanlder 一般使用 callback 的方式，不清楚的可以看下消息分发优先级。

>com.tencent.matrix.trace.hacker.ActivityThreadHacker#hackSysHandlerCallback

```java
    public static void hackSysHandlerCallback() {
        try {
            sApplicationCreateBeginTime = SystemClock.uptimeMillis();
            sApplicationCreateBeginMethodIndex = AppMethodBeat.getInstance().maskIndex("ApplicationCreateBeginMethodIndex");
            Class<?> forName = Class.forName("android.app.ActivityThread");
            Field field = forName.getDeclaredField("sCurrentActivityThread");
            field.setAccessible(true);
            Object activityThreadValue = field.get(forName);
            Field mH = forName.getDeclaredField("mH");
            mH.setAccessible(true);
            Object handler = mH.get(activityThreadValue);
            Class<?> handlerClass = handler.getClass().getSuperclass();
            Field callbackField = handlerClass.getDeclaredField("mCallback");
            callbackField.setAccessible(true);
            Handler.Callback originalCallback = (Handler.Callback) callbackField.get(handler);
            HackCallback callback = new HackCallback(originalCallback);
            callbackField.set(handler, callback);
            MatrixLog.i(TAG, "hook system handler completed. start:%s SDK_INT:%s", sApplicationCreateBeginTime, Build.VERSION.SDK_INT);
        } catch (Exception e) {
            MatrixLog.e(TAG, "hook system handler err! %s", e.getCause().toString());
        }
    }
```

那么，我们直接看 HackCallback 做了什么：

> com.tencent.matrix.trace.hacker.ActivityThreadHacker.HackCallback

```java
        public boolean handleMessage(Message msg) {

            if (!AppMethodBeat.isRealTrace()) {
                return null != mOriginalCallback && mOriginalCallback.handleMessage(msg);
            }

            boolean isLaunchActivity = isLaunchActivity(msg);
            if (hasPrint > 0) {
                MatrixLog.i(TAG, "[handleMessage] msg.what:%s begin:%s isLaunchActivity:%s", msg.what, SystemClock.uptimeMillis(), isLaunchActivity);
                hasPrint--;
            }
            if (isLaunchActivity) {
                ActivityThreadHacker.sLastLaunchActivityTime = SystemClock.uptimeMillis();
                ActivityThreadHacker.sLastLaunchActivityMethodIndex = AppMethodBeat.getInstance().maskIndex("LastLaunchActivityMethodIndex");
            }

            if (!isCreated) {
                if (isLaunchActivity || msg.what == CREATE_SERVICE || msg.what == RECEIVER) { // todo for provider
                    ActivityThreadHacker.sApplicationCreateEndTime = SystemClock.uptimeMillis();
                    ActivityThreadHacker.sApplicationCreateScene = msg.what;
                    isCreated = true;
                }
            }

            return null != mOriginalCallback && mOriginalCallback.handleMessage(msg);
        }
```

是对 activity 的启动做了监控。记录了 activity 启动的时间，记录了对应的 sIndex，以便后面进行启动分析。

还记录了 application 的启动情况，原理是，如果是第一次启动一个 activity，那么记录当前时间，这个时间就当成 application 创建完成的时间，还记录了启动场景，因为启动APP的，可以有 Activity，Service，BroadcastReceiver 等。

