---
title: 007-Matrix源码分析：StartupTracer 监控冷启动时间
index_img: /cover/6.jpg
banner_img: /cover/top.jpg
date: 2020-7-6
categories: Matrix
---

StartupTracer 也继承了 Tracer，但是由于它监控的是冷启动，所以它需要知道第一个 activity 是什么时候启动的，故它也实现了 ActivityLifecycleCallbacks 接口。

我们先看它是如何判断冷启动还是温启动的：

```java
    private boolean isColdStartup() {
        return coldCost == 0;
    }

    private boolean isWarmStartUp() {
        return isWarmStartUp;
    }

	@Override
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        if (activeActivityCount == 0 && coldCost > 0) {
            isWarmStartUp = true;
        }
        activeActivityCount++;
    }
```

冷启动直接是使用的 coldCost 是否为0，因为后面会有改变该值的逻辑，所以就相当于一个boolean变量的用法。

温启动是activity没有了，但是进程还在，StartupTracer 还在运行，所以 coldCost 的值也还在，这是就会认为是温启动。

我们看一下该类的注释：

```java
 * <p>
 * firstMethod.i       LAUNCH_ACTIVITY   onWindowFocusChange   LAUNCH_ACTIVITY    onWindowFocusChange
 * ^                         ^                   ^                     ^                  ^
 * |                         |                   |                     |                  |
 * |---------app---------|---|---firstActivity---|---------...---------|---careActivity---|
 * |<--applicationCost-->|
 * |<--------------firstScreenCost-------------->|
 * |<---------------------------------------coldCost------------------------------------->|
 * .                         |<-----warmCost---->|
 *
```

这个图还是挺清楚的。

我们继续看该类的一些方法，该类的 `onActivityFocused` 会在activity的`onWindowFocusChanged`方法执行的时候调用（使用的是插桩，这部分后面再讲）：

```java
    @Override
    public void onActivityFocused(String activity) {
        // 冷启动
        if (isColdStartup()) {
            if (firstScreenCost == 0) {
                // 从 application 创建到第一个activity 回调 onActivityFocused 的时间
                // ActivityThreadHacker.getEggBrokenTime() 是 application 创建的时间，不知道为啥要起这么个蛋疼的名字
                this.firstScreenCost = uptimeMillis() - ActivityThreadHacker.getEggBrokenTime();
            }
            if (hasShowSplashActivity) {
                // coldCost 还算上了 splash 显示的时间，从 application 创建到 "mainActivity"
                coldCost = uptimeMillis() - ActivityThreadHacker.getEggBrokenTime();
            } else {
                if (splashActivities.contains(activity)) {
                    hasShowSplashActivity = true;
                } else if (splashActivities.isEmpty()) {
                    MatrixLog.i(TAG, "default splash activity[%s]", activity);
                    coldCost = firstScreenCost;
                } else {
                    MatrixLog.w(TAG, "pass this activity[%s] at duration of start up! splashActivities=%s", activity, splashActivities);
                }
            }
            if (coldCost > 0) {
                // 分析
                analyse(ActivityThreadHacker.getApplicationCost(), firstScreenCost, coldCost, false);
            }

        }
        // 温启动
        else if (isWarmStartUp()) {
            isWarmStartUp = false;
            // 计算的是第一个 activity 从启动到 onActivityFocused 的时间
            // ActivityThreadHacker hook 了 H 的 LAUNCH_ACTIVITY
            // 温启动，application 还在
            long warmCost = uptimeMillis() - ActivityThreadHacker.getLastLaunchActivityTime();
            if (warmCost > 0) {
                analyse(ActivityThreadHacker.getApplicationCost(), firstScreenCost, warmCost, true);
            }
        }

    }
```

无论是冷启动还是温启动，都需要分析这些数据，然后上报：

```java
    private void analyse(long applicationCost, long firstScreenCost, long allCost, boolean isWarmStartUp) {
        MatrixLog.i(TAG, "[report] applicationCost:%s firstScreenCost:%s allCost:%s isWarmStartUp:%s", applicationCost, firstScreenCost, allCost, isWarmStartUp);
        long[] data = new long[0];
        // 冷启动不得超过 10s
        if (!isWarmStartUp && allCost >= coldStartupThresholdMs) { // for cold startup
            // 分析 ApplicationCreateBeginMethodIndex 的方法栈
            data = AppMethodBeat.getInstance().copyData(ActivityThreadHacker.sApplicationCreateBeginMethodIndex);
            ActivityThreadHacker.sApplicationCreateBeginMethodIndex.release();

        }
        // 温启动不得超过 4s
        else if (isWarmStartUp && allCost >= warmStartupThresholdMs) {
            // 分析 LastLaunchActivityMethodIndex 的方法栈
            data = AppMethodBeat.getInstance().copyData(ActivityThreadHacker.sLastLaunchActivityMethodIndex);
            ActivityThreadHacker.sLastLaunchActivityMethodIndex.release();
        }

        MatrixHandlerThread.getDefaultHandler().post(new AnalyseTask(data, applicationCost, firstScreenCost, allCost, isWarmStartUp, ActivityThreadHacker.sApplicationCreateScene));

    }
```

这里就是从 sBuffer 里面截取对应的 index 段来分析里面涉及到的方法，对 sBuffer 这里就不分析了，跟前面是一样的，这里说一下几个变量的意义。

```java
        AnalyseTask(long[] data, long applicationCost, long firstScreenCost, long allCost, boolean isWarmStartUp, int scene) {
            this.data = data;
            // 当前 activity
            this.scene = scene;
            // application 耗时
            this.applicationCost = applicationCost;
            // 启动直到用户看到第一个 activity 耗时
            this.firstScreenCost = firstScreenCost;
            // 冷启动/温启动耗时
            this.allCost = allCost;
            // 是冷启动还是温启动
            this.isWarmStartUp = isWarmStartUp;
        }
```

传进来 data，是为了两件事：

- 修正启动时间，与 data 里面的方法耗时累积做比较
- 计算 data 的 key，方便聚合

那么，启动监测就分析完成了。
