---
title: 006-Matrix源码分析：FrameTracer 监控帧率
index_img: /cover/6.jpg
banner_img: /cover/top.jpg
date: 2020-7-6
categories: Matrix
---

FrameTracer 与前面的两个 tracer 一样，都是继承的 Tracer。所以分析的套路也是一样的简单。

而且它没有覆盖 `dispatchBegin` 与 `dispatchEnd` 这两个方法。

```java
    @Override
    public void doFrame(String focusedActivityName, long start, long end, long frameCostMs, long inputCostNs, long animationCostNs, long traversalCostNs) {
        if (isForeground()) {
            notifyListener(focusedActivityName, end - start, frameCostMs, frameCostMs >= 0);
        }
    }
```

在前台才会进行帧率分析。这里的帧率采用的是累加的方式，即使

notifyListener 是 FrameTracer 内部可以注册监听，然后这里会回调这些监听。我们看看监听会回调一些什么信息过去：

```java
    private void notifyListener(final String visibleScene, final long taskCostMs, final long frameCostMs, final boolean isContainsFrame) {
        long start = System.currentTimeMillis();
        try {
            synchronized (listeners) {
                for (final IDoFrameListener listener : listeners) {
                    if (config.isDevEnv()) {
                        listener.time = SystemClock.uptimeMillis();
                    }
                    // 掉了几帧，这里是整除
                    final int dropFrame = (int) (taskCostMs / frameIntervalMs);

                    // taskCostMs 与 frameCostMs 差不多，多执行几行代码的时间差别
                    // 不过 frameCostMs 有可能为 0，当处理的message 是普通消息，而不是 doFrame 消息的时候
                    // visibleScene 是当前 activity 的名字
                    // isContainsFrame 我没搞懂，这个不是肯定为 true 吗？
                    listener.doFrameSync(visibleScene, taskCostMs, frameCostMs, dropFrame, isContainsFrame);
                    if (null != listener.getExecutor()) {
                        // 将回调转给 Executor 去执行
                        listener.getExecutor().execute(new Runnable() {
                            @Override
                            public void run() {
                                listener.doFrameAsync(visibleScene, taskCostMs, frameCostMs, dropFrame, isContainsFrame);
                            }
                        });
                    }
                    if (config.isDevEnv()) {
                        listener.time = SystemClock.uptimeMillis() - listener.time;
                        MatrixLog.d(TAG, "[notifyListener] cost:%sms listener:%s", listener.time, listener);
                    }
                }
            }
        } finally {
            long cost = System.currentTimeMillis() - start;
            if (config.isDebug() && cost > frameIntervalMs) {
                MatrixLog.w(TAG, "[notifyListener] warm! maybe do heavy work in doFrameSync! size:%s cost:%sms", listeners.size(), cost);
            }
        }
    }
```

notifyListener 就是将该消息的耗时，丢了多少帧，当前 activity 的名字传递过去。而且会在 listener 的 Executor 里面执行。

我们看一个 listener 的实现类：

> com.tencent.matrix.trace.tracer.FrameTracer.FPSCollector

```java
    private class FPSCollector extends IDoFrameListener {


        // 也是放到了MatrixHandlerThread中去执行
        private Handler frameHandler = new Handler(MatrixHandlerThread.getDefaultHandlerThread().getLooper());

        // 转到 MatrixHandlerThread 里面去执行
        Executor executor = new Executor() {
            @Override
            public void execute(Runnable command) {
                frameHandler.post(command);
            }
        };

        private HashMap<String, FrameCollectItem> map = new HashMap<>();

        @Override
        public Executor getExecutor() {
            return executor;
        }

        @Override
        public void doFrameAsync(String visibleScene, long taskCost, long frameCostMs, int droppedFrames, boolean isContainsFrame) {
            super.doFrameAsync(visibleScene, taskCost, frameCostMs, droppedFrames, isContainsFrame);
            if (Utils.isEmpty(visibleScene)) {
                return;
            }

            // 存放到 map 集合里面，以activity为单位
            FrameCollectItem item = map.get(visibleScene);
            if (null == item) {
                item = new FrameCollectItem(visibleScene);
                map.put(visibleScene, item);
            }

            // 进行数据收集
            item.collect(droppedFrames, isContainsFrame);

            // 当统计时间超过 10000 ms 时进行上报
            if (item.sumFrameCost >= timeSliceMs) { // report
                map.remove(visibleScene);
                item.report();
            }
        }
    }
```

我们看看它上报了什么东西：

```java
        void collect(int droppedFrames, boolean isContainsFrame) {
            long frameIntervalCost = UIThreadMonitor.getMonitor().getFrameIntervalNanos();
            // 总的帧数耗时
            sumFrameCost += (droppedFrames + 1) * frameIntervalCost / Constants.TIME_MILLIS_TO_NANO;
            // 总的掉帧数
            sumDroppedFrames += droppedFrames;
            // 总帧数
            sumFrame++;
            if (!isContainsFrame) {
                sumTaskFrame++;
            }

            // 按本次掉帧数来判断警戒级别
            // 在该页面，级别严重的越多，说明这个页面有问题，可以采取措施
            if (droppedFrames >= frozenThreshold) {
                dropLevel[DropStatus.DROPPED_FROZEN.index]++;
                dropSum[DropStatus.DROPPED_FROZEN.index] += droppedFrames;
            } else if (droppedFrames >= highThreshold) {
                dropLevel[DropStatus.DROPPED_HIGH.index]++;
                dropSum[DropStatus.DROPPED_HIGH.index] += droppedFrames;
            } else if (droppedFrames >= middleThreshold) {
                dropLevel[DropStatus.DROPPED_MIDDLE.index]++;
                dropSum[DropStatus.DROPPED_MIDDLE.index] += droppedFrames;
            } else if (droppedFrames >= normalThreshold) {
                dropLevel[DropStatus.DROPPED_NORMAL.index]++;
                dropSum[DropStatus.DROPPED_NORMAL.index] += droppedFrames;
            } else {
                dropLevel[DropStatus.DROPPED_BEST.index]++;
                dropSum[DropStatus.DROPPED_BEST.index] += (droppedFrames < 0 ? 0 : droppedFrames);
            }
        }
```

然后，将这些数据上报，`report`方法只是拼接了一个 JSONObject，就不说了。

最后看下来，这个类就只是做了上报，没有输出日志。但是我在运行demo的时候，有个悬浮窗显示出来了，看看是在哪里做的。再次戳一下 IDoFrameListener 的实现类，看看有哪些，果然发现了一个 `FrameDecorator`。这个类实际上与性能优化没啥关系，它只是提供一个实时限制帧率信息的一个悬浮窗，这里我们只介绍它展示了哪些信心，不去探究如何实现一个悬浮穿，画实时的线条之类的。

我们直接看它的 doFrameAsync 方法：

```java
    @Override
    public void doFrameAsync(String visibleScene, long taskCost, long frameCostMs, int droppedFrames, boolean isContainsFrame) {
        super.doFrameAsync(visibleScene, taskCost, frameCostMs, droppedFrames, isContainsFrame);
        sumFrameCost += (droppedFrames + 1) * UIThreadMonitor.getMonitor().getFrameIntervalNanos() / Constants.TIME_MILLIS_TO_NANO;
        sumFrames += 1;
        long duration = sumFrameCost - lastCost[0];

        long collectFrame = sumFrames - lastFrames[0];
        // 至少要累积到200ms才做一次更新
        if (duration >= 200) {
            // 拿到帧率
            final float fps = Math.min(60.f, 1000.f * collectFrame / duration);
            updateView(view.fpsView, fps);
            view.chartView.addFps((int) fps);
            lastCost[0] = sumFrameCost;
            lastFrames[0] = sumFrames;
            // 没有消息处理了，就显示 60FPS
            mainHandler.removeCallbacks(updateDefaultRunnable);
            mainHandler.postDelayed(updateDefaultRunnable, 130);
        }
    }
```

FrameDecorator 其实就显示了一个帧率，只不过使用悬浮窗现实的。



### 总结

如果一个 Message 处理的时候间隔小于 16.7ms，那么就可以认为它的帧率为 60fps，反之大于这个时间间隔，那么就说明掉帧了。

其实，还可以使用 Choreographer 的 FrameCallback 来检测帧率。