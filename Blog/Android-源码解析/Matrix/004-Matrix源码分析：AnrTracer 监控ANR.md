---
title:004-Matrix源码分析：AnrTracer 监控ANR
date: 2020-7-2
categories: Matrix
---

AnrTracer是用来监测ANR的，可以打印出ANR发生的具体位置。打印log如下：

```
    >>>>>>>>>>>>>>>>>>>>>>> maybe happens ANR(5001 ms)! <<<<<<<<<<<<<<<<<<<<<<<
    |* scene: com.example.sample.MainActivity
    |* [ProcessStat]
    |*		Priority: 10
    |*		Nice: -10
    |*		Foreground: true
    |* [Memory]
    |*		DalvikHeap: 2076kb	// 占用的堆内存
    |*		NativeHeap: 17137kb // 占用的本地内存，使用 Debug 得到
    |*		VmSize: 5256708kb	// 虚拟内存大小，从 proc/[pid]/stat 文件中取得
    |* [doFrame]
    |*		inputCost: 0
    |*		animationCost: 0
    |*		traversalCost: 0
    |* [Thread]
    |*		State: RUNNABLE
    |*		Stack:  
    |*		at android.view.View:performClickInternal(6574)
    |*		at android.view.View:access$3100(778)
    |*		at android.view.View$PerformClick:run(25885)
    |*		at android.os.Handler:handleCallback(873)
    |*		at android.os.Handler:dispatchMessage(99)
    |*		at android.os.Looper:loop(193)
    |*		at android.app.ActivityThread:main(6669)
    |* [Trace]
    |*		StackSize: 2
    |*		StackKey: 1|
    |*		TraceStack:
    |*			[id count cost]
    |*		1048574 1 5005
    |*		.1 1 5004
    ========================================================================= 
    postTime:1734934 curTime:1739939
```

第一项是进程状态。

从 proc/[pid\]/stat 文件中获取的，主要描述了进程的优先级与前后台状态。nice 值与 oom_adj 有关，越低越好。

进程前后台状态是根据 ActivityLifecycleCallbacks 来判断的，没啥稀奇的。有一个地方需要注意，它判断进入后台是使用的 `com.tencent.matrix.AppActiveMatrixDelegate#getTopActivityName` 这个方法，里面使用反射查找 ActivityThread 的 mActivities 集合中 activity 的状态。不清楚这样是否会更好一点。我们通常是直接在 onStop 里面直接做了处理，没有这么麻烦。

第二项是内存状态。

第三项是 doFrame 的状态，是看看 3 个队列分别耗时多少。由于我写了一个死循环，所以他们都是0。

第四项是线程状态与当前堆栈信息，可以看到堆栈信息只能在ANR的附近，并不能准确的指出ANR是哪个函数导致的。

第五项是 trace，就是被插桩的方法的调用链。

```
    |* [Trace]
    |*		StackSize: 2
    |*		StackKey: 1|
    |*		TraceStack:
    |*			[id count cost]
    |*		1048574 1 5005
    |*		.1 1 5004
```

这个堆栈的意义需要解释一下，我们先看demo中的methodMapping：

```
1,1,com.example.sample.MainActivity$2 onClick (Landroid.view.View;)V
1048574,1,android.os.Handler dispatchMessage (Landroid.os.Message;)V
```

数据格式的意义为：methodId，方法的访问符，类， 函数。实现的函数为 `com.tencent.matrix.trace.item.TraceMethod#toString`。

所以上面的 traceStack 我们逆推一下，就是 Handler#dispatchMessage 调用了 MainActivity$2#onClick，而 MainActivity$2#onClick 耗时 5004 毫秒，所以可以得出 MainActivity$2#onClick 这个方法里面有耗时操作。

实际上，我的demo里面确实是这样：

```java
        test.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                final long s = System.currentTimeMillis();
                while (true) {
                    if (System.currentTimeMillis() > s + 6000) {
                        break;
                    }
                }
            }
        });
```

所以，这个 AnrTracer 还是挺准确的。

我们看看它的实现代码吧。

Tracer 类都是继承了 LooperObserver，我们从这个类的3个方法入手。

```java
    public void dispatchBegin(long beginMs, long cpuBeginMs, long token) {
        super.dispatchBegin(beginMs, cpuBeginMs, token);
        anrTask = new AnrHandleTask(AppMethodBeat.getInstance().maskIndex("AnrTracer#dispatchBegin"), token);
        if (traceConfig.isDevEnv()) {
            MatrixLog.v(TAG, "* [dispatchBegin] token:%s index:%s", token, anrTask.beginRecord.index);
        }
        anrHandler.postDelayed(anrTask, Constants.DEFAULT_ANR - (SystemClock.uptimeMillis() - token));
    }
```

anrHandler 会将消息 post 到一个子线程，所以该类对插桩方法堆栈的分析都是在子线程。**注意这里延迟了大约 5s**。

```java
    public void doFrame(String focusedActivityName, long start, long end, long frameCostMs, long inputCost, long animationCost, long traversalCost) {
        if (traceConfig.isDevEnv()) {
            MatrixLog.v(TAG, "--> [doFrame] activityName:%s frameCost:%sms [%s:%s:%s]ns", focusedActivityName, frameCostMs, inputCost, animationCost, traversalCost);
        }
    }
```

doFrame 可以忽略。

```java
    @Override
    public void dispatchEnd(long beginMs, long cpuBeginMs, long endMs, long cpuEndMs, long token, boolean isBelongFrame) {
        super.dispatchEnd(beginMs, cpuBeginMs, endMs, cpuEndMs, token, isBelongFrame);
        if (traceConfig.isDevEnv()) {
            MatrixLog.v(TAG, "[dispatchEnd] token:%s cost:%sms cpu:%sms usage:%s",
                    token, endMs - beginMs, cpuEndMs - cpuBeginMs, Utils.calculateCpuUsage(cpuEndMs - cpuBeginMs, endMs - beginMs));
        }
        if (null != anrTask) {
            anrTask.getBeginRecord().release();
            anrHandler.removeCallbacks(anrTask);
        }
    }
```

这个与 dispatchBegin 对应，如果 dispatchEnd 在 5s 内执行完了，那么就不用处理 AnrHandleTask 了，如果在 5s 内该方法没有调用，就需要分析方法调用，看看是哪里出了问题。我们看看 AnrHandleTask 里面做了什么：

> com.tencent.matrix.trace.tracer.AnrTracer.AnrHandleTask#run

```java
        public void run() {
            long curTime = SystemClock.uptimeMillis();
            boolean isForeground = isForeground();
            // process
            int[] processStat = Utils.getProcessPriority(Process.myPid());
            long[] data = AppMethodBeat.getInstance().copyData(beginRecord);
            beginRecord.release();
            String scene = AppMethodBeat.getVisibleScene();

            // memory
            long[] memoryInfo = dumpMemory();

            // Thread state
            Thread.State status = Looper.getMainLooper().getThread().getState();
            StackTraceElement[] stackTrace = Looper.getMainLooper().getThread().getStackTrace();
            String dumpStack = Utils.getStack(stackTrace, "|*\t\t", 12);

            // frame
            UIThreadMonitor monitor = UIThreadMonitor.getMonitor();
            long inputCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_INPUT, token);
            long animationCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_ANIMATION, token);
            long traversalCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_TRAVERSAL, token);

            // trace
            LinkedList<MethodItem> stack = new LinkedList();
            if (data.length > 0) {
                // 将 buffer 中的 long 转为 MethodItem
                TraceDataUtils.structuredDataToStack(data, stack, true, curTime);
                //
                TraceDataUtils.trimStack(stack, Constants.TARGET_EVIL_METHOD_STACK, new TraceDataUtils.IStructuredDataFilter() {
                    @Override
                    public boolean isFilter(long during, int filterCount) {
                        return during < filterCount * Constants.TIME_UPDATE_CYCLE_MS;
                    }

                    @Override
                    public int getFilterMaxCount() {
                        return Constants.FILTER_STACK_MAX_COUNT;
                    }

                    @Override
                    public void fallback(List<MethodItem> stack, int size) {
                        MatrixLog.w(TAG, "[fallback] size:%s targetSize:%s stack:%s", size, Constants.TARGET_EVIL_METHOD_STACK, stack);
                        Iterator iterator = stack.listIterator(Math.min(size, Constants.TARGET_EVIL_METHOD_STACK));
                        while (iterator.hasNext()) {
                            iterator.next();
                            iterator.remove();
                        }
                    }
                });
            }

            StringBuilder reportBuilder = new StringBuilder();
            StringBuilder logcatBuilder = new StringBuilder();
            long stackCost = Math.max(Constants.DEFAULT_ANR, TraceDataUtils.stackToString(stack, reportBuilder, logcatBuilder));

            // stackKey
            String stackKey = TraceDataUtils.getTreeKey(stack, stackCost);
            MatrixLog.w(TAG, "%s \npostTime:%s curTime:%s",
                    printAnr(scene, processStat, memoryInfo, status, logcatBuilder, isForeground, stack.size(),
                            stackKey, dumpStack, inputCost, animationCost, traversalCost, stackCost), token, curTime); // for logcat

            if (stackCost >= Constants.DEFAULT_ANR_INVALID) {
                MatrixLog.w(TAG, "The checked anr task was not executed on time. "
                        + "The possible reason is that the current process has a low priority. just pass this report");
                return;
            }
            // report
            try {
                TracePlugin plugin = Matrix.with().getPluginByClass(TracePlugin.class);
                if (null == plugin) {
                    return;
                }
                JSONObject jsonObject = new JSONObject();
                jsonObject = DeviceUtil.getDeviceInfo(jsonObject, Matrix.with().getApplication());
                jsonObject.put(SharePluginInfo.ISSUE_STACK_TYPE, Constants.Type.ANR);
                jsonObject.put(SharePluginInfo.ISSUE_COST, stackCost);
                jsonObject.put(SharePluginInfo.ISSUE_STACK_KEY, stackKey);
                jsonObject.put(SharePluginInfo.ISSUE_SCENE, scene);
                jsonObject.put(SharePluginInfo.ISSUE_TRACE_STACK, reportBuilder.toString());
                jsonObject.put(SharePluginInfo.ISSUE_THREAD_STACK, Utils.getStack(stackTrace));
                jsonObject.put(SharePluginInfo.ISSUE_PROCESS_PRIORITY, processStat[0]);
                jsonObject.put(SharePluginInfo.ISSUE_PROCESS_NICE, processStat[1]);
                jsonObject.put(SharePluginInfo.ISSUE_PROCESS_FOREGROUND, isForeground);
                // memory info
                JSONObject memJsonObject = new JSONObject();
                memJsonObject.put(SharePluginInfo.ISSUE_MEMORY_DALVIK, memoryInfo[0]);
                memJsonObject.put(SharePluginInfo.ISSUE_MEMORY_NATIVE, memoryInfo[1]);
                memJsonObject.put(SharePluginInfo.ISSUE_MEMORY_VM_SIZE, memoryInfo[2]);
                jsonObject.put(SharePluginInfo.ISSUE_MEMORY, memJsonObject);

                Issue issue = new Issue();
                issue.setKey(token + "");
                issue.setTag(SharePluginInfo.TAG_PLUGIN_EVIL_METHOD);
                issue.setContent(jsonObject);
                plugin.onDetectIssue(issue);

            } catch (JSONException e) {
                MatrixLog.e(TAG, "[JSONException error: %s", e);
            }

        }

```

可以看出里面就是输出上面我们看到的一些日志信息。

这里只详细说一下方法堆栈的处理：

```java
            // trace
            LinkedList<MethodItem> stack = new LinkedList();
            if (data.length > 0) {
                // 将 buffer 中的 long 转为 MethodItem
                TraceDataUtils.structuredDataToStack(data, stack, true, curTime);
                //
                TraceDataUtils.trimStack(stack, Constants.TARGET_EVIL_METHOD_STACK, new TraceDataUtils.IStructuredDataFilter() {
                    @Override
                    public boolean isFilter(long during, int filterCount) {
                        return during < filterCount * Constants.TIME_UPDATE_CYCLE_MS;
                    }

                    @Override
                    public int getFilterMaxCount() {
                        return Constants.FILTER_STACK_MAX_COUNT;
                    }

                    @Override
                    public void fallback(List<MethodItem> stack, int size) {
                        MatrixLog.w(TAG, "[fallback] size:%s targetSize:%s stack:%s", size, Constants.TARGET_EVIL_METHOD_STACK, stack);
                        Iterator iterator = stack.listIterator(Math.min(size, Constants.TARGET_EVIL_METHOD_STACK));
                        while (iterator.hasNext()) {
                            iterator.next();
                            iterator.remove();
                        }
                    }
                });
            }

            StringBuilder reportBuilder = new StringBuilder();
            StringBuilder logcatBuilder = new StringBuilder();
            long stackCost = Math.max(Constants.DEFAULT_ANR, TraceDataUtils.stackToString(stack, reportBuilder, logcatBuilder));

```

第一个关键函数：`com.tencent.matrix.trace.util.TraceDataUtils#structuredDataToStack`

它就是将 sBuffer 转成一个 LinkedList。之前我们有一张图，这里再看一下：

![Alt text](https://github.com/Tencent/matrix/wiki/images/trace/stack.jpg)

还是只看上半部分，对于 4 5 6 7 这4个方法来说，经过转换后，就变为了 List list = {{4， 0}, {5，1}, {6， 1}, {7， 1}} 。

这个列表里面是一个对象，我只写了 methodId 与 depth：

```java
public class MethodItem {

    // 方法的id
    public int methodId;
    // 耗费的时间
    public int durTime;
    // 调用的深度
    public int depth;
    // 被调用的次数
    public int count = 1;

}
```

拿到了所有的调用栈之后，有可能调用栈特别大，所以需要裁剪： `TraceDataUtils.trimStack`

这个方法就是用来过滤一些不耗时的函数，过滤类是 IStructuredDataFilter ：

```java
    public interface IStructuredDataFilter {
        // 将满足过滤条件的删掉
        // filterCount  是当前的过滤次数，可以根据过滤次数来做动态的调整，更改过滤条件，使之更宽
        boolean isFilter(long during, int filterCount);

        // 最大的过滤次数
        int getFilterMaxCount();

        // 如果达到最大过滤次数后，还是太多了，则在这个方法里面处理，一般就是直接丢弃掉多余的
        void fallback(List<MethodItem> stack, int size);
    }
```

将方法堆栈裁剪完之后，就可以打印输出了。这个类的核心就介绍完毕了。

哦，对了，还有一个较重要的函数：`com.tencent.matrix.trace.util.TraceDataUtils#getTreeKey(java.util.List<com.tencent.matrix.trace.items.MethodItem>, long)`

这个就是为堆栈生成一个 key，因为上报到后台，没有一个 key 的话很麻烦，而且将堆栈简化为key，可以更容易的做处理。具体就是：分析出主要耗时的那一级函数，作为代表卡顿堆栈的key。就是上图的下半部分。