---
title: 002-Matrix源码分析：UIThreadMonitor监测主线程
index_img: /cover/30.jpg
banner_img: /cover/top.jpg
date: 2020-6-30
categories: Matrix
---

常说的一个问题：为啥Looper有个死循环却不会阻塞主线程？？？

这是因为我们所谓的主线程就是这个死循环。我们的每一帧都是封装成了消息然后被分发，在主线程处理，主线程是不断的在处理这些消息，如果什么时候有个特别耗时的消息来了，那么主线程就会卡死。



上面说了一个题外话，我们现在来看看如何监测主线程。了解这个类还需要一点预备知识：

FrameDisplayEventReceiver 在收到 VSYNC 信号之后，会调用 doFrame 方法，而 doFrame 方法就会处理 Choreographer.CALLBACK_INPUT，

Choreographer.CALLBACK_ANIMATION，Choreographer.CALLBACK_TRAVERSAL这些东西。

![uithreadmonitor1.png](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/uithreadmonitor1.png?raw=true)

他们分别是 事件处理，动画，界面绘制相关的东西。比如对于属性动画，它注册了帧回调，会将相关代码添加到 Choreographer 的动画队列里面，然后下一帧就会被执行，动画也就得到了处理。

doFrame方法本身也是一个 message：

```java
Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
```

这里发送了一个消息，然后接受者会处理消息，处理这个消息的时候会调用 doFrame 方法，可以认为这个 doFrame 方法是在Looper的Printer的两个打印代码之间执行的。

所以相当于这个类将消息的处理过程更加细化了，我们看看代码：

> com.tencent.matrix.trace.core.UIThreadMonitor#init

```java
        choreographer = Choreographer.getInstance();
        callbackQueueLock = reflectObject(choreographer, "mLock");
        callbackQueues = reflectObject(choreographer, "mCallbackQueues");

        addInputQueue = reflectChoreographerMethod(callbackQueues[CALLBACK_INPUT], ADD_CALLBACK, long.class, Object.class, Object.class);
        addAnimationQueue = reflectChoreographerMethod(callbackQueues[CALLBACK_ANIMATION], ADD_CALLBACK, long.class, Object.class, Object.class);
        addTraversalQueue = reflectChoreographerMethod(callbackQueues[CALLBACK_TRAVERSAL], ADD_CALLBACK, long.class, Object.class, Object.class);
        frameIntervalNanos = reflectObject(choreographer, "mFrameIntervalNanos");
```

这里是使用反射获取了 Choreographer 类的一些字段与方法，后面会用于向队列里面添加回调。



> com.tencent.matrix.trace.core.UIThreadMonitor#init

```java
        LooperMonitor.register(new LooperMonitor.LooperDispatchListener() {
            @Override
            public boolean isValid() {
                return isAlive;
            }

            @Override
            public void dispatchStart() {
                super.dispatchStart();
                UIThreadMonitor.this.dispatchBegin();
            }

            @Override
            public void dispatchEnd() {
                super.dispatchEnd();
                UIThreadMonitor.this.dispatchEnd();
            }

        });
```

接下来就是注册了监听，这个监听的触发时机上一节我们分析过，分发消息的时候会成对的回调。



我们继续看 dispatchBegin 与 dispatchEnd：

> com.tencent.matrix.trace.core.UIThreadMonitor#dispatchBegin

```java
    private void dispatchBegin() {
        token = dispatchTimeMs[0] = SystemClock.uptimeMillis();
        dispatchTimeMs[2] = SystemClock.currentThreadTimeMillis();
        AppMethodBeat.i(AppMethodBeat.METHOD_ID_DISPATCH);

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (!observer.isDispatchBegin()) {
                    observer.dispatchBegin(dispatchTimeMs[0], dispatchTimeMs[2], token);
                }
            }
        }
    }
```

记录了一些时间：

dispatchTimeMs[0] 是手机从启动到现在的时间。

dispatchTimeMs[2] 是线程运行的时间。



然后是通知自己的 observers，**相当于又转了一下**，利用 LooperDispatchListener 来通知**自己的 LooperObserver**。

> com.tencent.matrix.trace.core.UIThreadMonitor#dispatchEnd

```java
    private void dispatchEnd() {

        if (isBelongFrame) {
            doFrameEnd(token);
        }

        long start = token;
        long end = SystemClock.uptimeMillis();

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (observer.isDispatchBegin()) {
                    observer.doFrame(AppMethodBeat.getVisibleScene(), token, SystemClock.uptimeMillis(), isBelongFrame ? end - start : 0, queueCost[CALLBACK_INPUT], queueCost[CALLBACK_ANIMATION], queueCost[CALLBACK_TRAVERSAL]);
                }
            }
        }

        dispatchTimeMs[3] = SystemClock.currentThreadTimeMillis();
        dispatchTimeMs[1] = SystemClock.uptimeMillis();

        AppMethodBeat.o(AppMethodBeat.METHOD_ID_DISPATCH);

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (observer.isDispatchBegin()) {
                    observer.dispatchEnd(dispatchTimeMs[0], dispatchTimeMs[2], dispatchTimeMs[1], dispatchTimeMs[3], token, isBelongFrame);
                }
            }
        }

    }
```

dispatchEnd 方法要稍微复杂一点，但是还是一样的，它也是记录了一些时间：

dispatchTimeMs[3] 是线程运行时间，与 dispatchTimeMs[2] 对应起来看就可以知道线程执行这个消息的耗时。

dispatchTimeMs[1] 是手机从启动到现在的时间，dispatchTimeMs[0] 对应就可以知道该方法现实时间的耗时。注意两个耗时的区别，现实耗时是大于线程耗时的，因为线程会切片运行。

这个方法，**也主要是回调了 observer.doFrame 和 observer.dispatchEnd 两个方法，这两个方法几乎是同时调用的，方法里面的参数是我们需要的**。

这里有个地方有点疑问：按照 LooperObserver 的3个方法来看，显然是要监测 doFrame 的运行情况，而 doFrame 只是一个特定的消息才会回调，假如我随便发送了一个普通的消息，也会触发这3个回调，那不是有问题吗？

我调试了一下这个回调，发现如下日志：

```
 activityName[com.example.sample.MainActivity] frame cost:0ms [104300|2480|218640]ns
 000000000
 activityName[com.example.sample.MainActivity] frame cost:0ms [104300|2480|218640]ns
```

我使用hander发送了一个message，打印出来的日志是这样的，就是说如果不是执行的 doFrame 的消息，frameCostMs 是 0，其余的是上一帧的值。我们注意一下就行了。

上面的函数中，开头就有一个 if 判断，这个很重要，里面涉及到我们上面所说的3个队列。

让我们从头道来，首先，外部会调用该类的 onStart 方法：

> com.tencent.matrix.trace.core.UIThreadMonitor#onStart

```java
    public synchronized void onStart() {
        if (!isInit) {
            throw new RuntimeException("never init!");
        }
        if (!isAlive) {
            this.isAlive = true;
            synchronized (this) {
                MatrixLog.i(TAG, "[onStart] callbackExist:%s %s", Arrays.toString(callbackExist), Utils.getStack());
                callbackExist = new boolean[CALLBACK_LAST + 1];
            }
            // queueStatus 存放状态，队列开始执行时，置为 DO_QUEUE_BEGIN，队列执行完毕时，置为 DO_QUEUE_END。
            queueStatus = new int[CALLBACK_LAST + 1];
            // queueCost 存放队列执行完毕的时间。
            queueCost = new long[CALLBACK_LAST + 1];
            addFrameCallback(CALLBACK_INPUT, this, true);
        }
    }
```

这里做了些初始化的判断，以及创建了一些数组，这个数组就是用来存放那3个队列的相关信息的。

最后一行代码，往  input 队列里面添加了一个 runnable，这个 runnable 就是自己，所以当input队列执行的时候会运行该类的 run 方法。 

> com.tencent.matrix.trace.core.UIThreadMonitor#run

```java
    public void run() {
        final long start = System.nanoTime();
        try {
            // 这个方法里面就做了一件事，就是将 isBelongFrame 置为 true
            doFrameBegin(token);
            // 开始执行 input 队列
            doQueueBegin(CALLBACK_INPUT);

            // 向 animation 队列添加一个 runnable
            addFrameCallback(CALLBACK_ANIMATION, new Runnable() {

                @Override
                public void run() {
                    // input队列执行结束
                    doQueueEnd(CALLBACK_INPUT);
                    // 开始执行 animation 队列
                    doQueueBegin(CALLBACK_ANIMATION);
                }
            }, true);

            // 向 traversal 队列添加一个 runnable
            addFrameCallback(CALLBACK_TRAVERSAL, new Runnable() {

                @Override
                public void run() {
                    // animation 队列执行结束
                    doQueueEnd(CALLBACK_ANIMATION);
                    // 开始执行 traversal 队列
                    doQueueBegin(CALLBACK_TRAVERSAL);
                }
            }, true);

        } finally {
            if (config.isDevEnv()) {
                MatrixLog.d(TAG, "[UIThreadMonitor#run] inner cost:%sns", System.nanoTime() - start);
            }
        }
    }
```

**这里其实就是向3个队列的头部插入 runnable，然后执行runnable的时候，计算出时差，调用对应的 begin与end方法。**

> com.tencent.matrix.trace.core.UIThreadMonitor#doQueueBegin
>
> com.tencent.matrix.trace.core.UIThreadMonitor#doQueueEnd

```java
    private void doQueueBegin(int type) {
        queueStatus[type] = DO_QUEUE_BEGIN;
        queueCost[type] = System.nanoTime();
    }

    private void doQueueEnd(int type) {
        queueStatus[type] = DO_QUEUE_END;
        queueCost[type] = System.nanoTime() - queueCost[type];
        synchronized (this) {
            callbackExist[type] = false;
        }
    }
```

这两方法啊就是设置队列的运行状态，计算队列的执行耗时。

最后总结，这个类做了两件事：

- 记录了3个队列的耗时
- 提供了 LooperObserver 回调，doFrame回调可以获取到 doFrame（Choreographer）的耗时，也可以知道每个消息处理的耗时（dispatchEnd）。
