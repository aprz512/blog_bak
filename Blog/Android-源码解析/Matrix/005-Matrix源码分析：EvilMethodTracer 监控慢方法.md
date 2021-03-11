---
title: 005-Matrix源码分析：EvilMethodTracer 监控慢方法
index_img: /cover/5.jpg
banner_img: /cover/top.jpg
date: 2020-7-5
categories: Matrix
---

这里我们来分析一下如何监测主线程里面的一些慢方法。

我们先写一个例子，例子很简单，点击一个按钮，执行一个方法，这个方法里面又调用了许多别的方法：

```java
    Button test = findViewById(R.id.test);
    test.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            f();
        }
    });

    void f() {
        A();
        A();
        A();
        A();
        B();
        C();
        D();
        E();
    }

    void A() {
        SystemClock.sleep(100);
    }

    void B () {
        SystemClock.sleep(200);
    }

    void C() {
        SystemClock.sleep(300);
    }

    private void D() {
        SystemClock.sleep(1);
    }

    private void E() {
        SystemClock.sleep(5);
    }
```

这里可以想一下，如果是你自己的话，你希望得出一个什么样的足够直观的监测结果呢？

这里其实 A B C 三个方法都比较耗时，我们看看日志输出：

```
    >>>>>>>>>>>>>>>>>>>>> maybe happens Jankiness!(916ms) <<<<<<<<<<<<<<<<<<<<<
    |* scene: com.example.sample.MainActivity
    |* [ProcessStat]
    |*		Priority: 10
    |*		Nice: -10
    |*		Foreground: true
    |* [CPU]
    |*		usage: 0.33%
    |* [doFrame]
    |*		inputCost: 120000
    |*		animationCost: 418100
    |*		traversalCost: 6311100
    |* [Trace]
    |*		StackSize: 8
    |*		StackKey: 2|
    |*		TraceStack:
    |*			[id count cost]
    |*		1048574 1 916
    |*		.1 1 913
    |*		..2 1 913
    |*		...3 4 403
    |*		...4 1 198
    |*		...5 1 300
    |*		...6 1 6
    |*		...7 1 6
    =========================================================================
```

doFrame 项里面的参数都是 ns，所以他们都是正常范围的。

我们看 trace 项，发现有个方法耗时 913 毫秒，这个肯定就是不正常了。

看下 methodMapping：

```
1,4,com.example.sample.MainActivity onCreate (Landroid.os.Bundle;)V
2,0,com.example.sample.MainActivity f ()V
3,0,com.example.sample.MainActivity A ()V
4,0,com.example.sample.MainActivity B ()V
5,0,com.example.sample.MainActivity C ()V
6,2,com.example.sample.MainActivity D ()V
7,2,com.example.sample.MainActivity E ()V
1048574,1,android.os.Handler dispatchMessage (Landroid.os.Message;)V
```

与堆栈一对应，发现就是我们例子的方法。

仔细想一下，这个的工作原理应该是与 AnrTracer 是一样的，都是以单个 message 为单位，分析所有调用的函数，然后生成堆栈信息。

我们看看源码：

```java
    @Override
    public void dispatchBegin(long beginMs, long cpuBeginMs, long token) {
        super.dispatchBegin(beginMs, cpuBeginMs, token);
        indexRecord = AppMethodBeat.getInstance().maskIndex("EvilMethodTracer#dispatchBegin");
    }
```

创建一个 IndexRecord。

```java
    @Override
    public void doFrame(String focusedActivityName, long start, long end, long frameCostMs, long inputCostNs, long animationCostNs, long traversalCostNs) {
        queueTypeCosts[0] = inputCostNs;
        queueTypeCosts[1] = animationCostNs;
        queueTypeCosts[2] = traversalCostNs;
    }
```

记录 doFrame 三个队列的耗时。

```java
    @Override
    public void dispatchEnd(long beginMs, long cpuBeginMs, long endMs, long cpuEndMs, long token, boolean isBelongFrame) {
        super.dispatchEnd(beginMs, cpuBeginMs, endMs, cpuEndMs, token, isBelongFrame);
        long start = config.isDevEnv() ? System.currentTimeMillis() : 0;
        try {
            long dispatchCost = endMs - beginMs;
            // 消息处理耗时超过 700ms
            if (dispatchCost >= evilThresholdMs) {
                long[] data = AppMethodBeat.getInstance().copyData(indexRecord);
                long[] queueCosts = new long[3];
                System.arraycopy(queueTypeCosts, 0, queueCosts, 0, 3);
                String scene = AppMethodBeat.getVisibleScene();
                // 发送到子线程去分析
                MatrixHandlerThread.getDefaultHandler().post(new AnalyseTask(isForeground(), scene, data, queueCosts, cpuEndMs - cpuBeginMs, endMs - beginMs, endMs));
            }
        } finally {
            indexRecord.release();
            if (config.isDevEnv()) {
                String usage = Utils.calculateCpuUsage(cpuEndMs - cpuBeginMs, endMs - beginMs);
                MatrixLog.v(TAG, "[dispatchEnd] token:%s cost:%sms cpu:%sms usage:%s innerCost:%s",
                        token, endMs - beginMs, cpuEndMs - cpuBeginMs, usage, System.currentTimeMillis() - start);
            }
        }
    }
```

这个方法最重要，就是在每个 message 处理完成后，分析一下设计到的所有方法的执行时间。

我们看看 AnalyseTask 做了什么：

```java
        void analyse() {

            // process
            int[] processStat = Utils.getProcessPriority(Process.myPid());
            String usage = Utils.calculateCpuUsage(cpuCost, cost);
            LinkedList<MethodItem> stack = new LinkedList();
            if (data.length > 0) {
                TraceDataUtils.structuredDataToStack(data, stack, true, endMs);
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
            long stackCost = Math.max(cost, TraceDataUtils.stackToString(stack, reportBuilder, logcatBuilder));
            String stackKey = TraceDataUtils.getTreeKey(stack, stackCost);

            MatrixLog.w(TAG, "%s", printEvil(scene, processStat, isForeground, logcatBuilder, stack.size(), stackKey, usage, queueCost[0], queueCost[1], queueCost[2], cost)); // for logcat
			...

        }
```

这里差不多与 AnrTracer 一样的流程，因该很好看懂，就不多说了。

其实只要理清核心之处就好了，AnrTracer 是延迟5s发送分析任务，EvilMethodTracer 是每个消息处理结束后分析，但是它设定了阈值。

### 总结

在每个 Message 处理的时候，检测了两个方法调用的时间间隔：

- dispatchBegin 记录一个时间
- dispatchEnd 记录一个时间

如果这两个时间超过了 700ms，那么就分析这两个时间段之间的 long 数组，然后分析堆栈与耗时。

