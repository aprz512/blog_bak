---
title: 001-Matrix源码分析：LooperMonitor 监测基石
index_img: /cover/06.jpg
banner_img: /cover/top.jpg
date: 2020-4-06
categories: Matrix
---



Matrix是一个APM库，它提供了许多功能供我们使用。我们从最简单的功能开始分析。

> com.tencent.matrix.trace.tracer.FrameTracer

这个类是用来展示app的帧率的，我们知道收集一个帧率还是相当容易的，但是这个库里面有些小细节以及扩展我们可以学习以下。

这里我们不从Matrix的入口函数开始分析，因为这样函数的调用栈会显得很长，不利于阅读，我们直接从最低层的类开始说起，慢慢往上走，就像搭建房子一样。

我们介绍的第一个类是 LooperMonitor，它的作用是监听**主线程**Looper的消息处理回调。

我们知道，Looper在分发消息的时候，会打印日志：

> android.os.Looper#loop

```java
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

			...
                
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
```

我们可以利用这个机制来监控消息的分发。首先，我们设置一个 Printer 进去：

> com.tencent.matrix.trace.core.LooperMonitor#resetPrinter

```java
    private synchronized void resetPrinter() {
        Printer originPrinter = null;
        try {
            if (!isReflectLoggingError) {
                originPrinter = ReflectUtils.get(looper.getClass(), "mLogging", looper);
                if (originPrinter == printer && null != printer) {
                    return;
                }
            }
        } catch (Exception e) {
            isReflectLoggingError = true;
            Log.e(TAG, "[resetPrinter] %s", e);
        }

        if (null != printer) {
            MatrixLog.w(TAG, "maybe thread:%s printer[%s] was replace other[%s]!",
                    looper.getThread().getName(), printer, originPrinter);
        }
        looper.setMessageLogging(printer = new LooperPrinter(originPrinter));
        if (null != originPrinter) {
            MatrixLog.i(TAG, "reset printer, originPrinter[%s] in %s", originPrinter, looper.getThread().getName());
        }
    }
```

这一段逻辑不复杂，首先是使用反射来获取原来的 Printer，然后将它包装一下，在重新设置回去。

我们看一下这个 Printer 的包装类的实现：

> com.tencent.matrix.trace.core.LooperMonitor.LooperPrinter

```java
    class LooperPrinter implements Printer {
        public Printer origin;
        boolean isHasChecked = false;
        boolean isValid = false;

        LooperPrinter(Printer printer) {
            this.origin = printer;
        }

        @Override
        public void println(String x) {
            if (null != origin) {
                origin.println(x);
                if (origin == this) {
                    throw new RuntimeException(TAG + " origin == this");
                }
            }

            if (!isHasChecked) {
                isValid = x.charAt(0) == '>' || x.charAt(0) == '<';
                isHasChecked = true;
                if (!isValid) {
                    MatrixLog.e(TAG, "[println] Printer is inValid! x:%s", x);
                }
            }

            if (isValid) {
                dispatch(x.charAt(0) == '>', x);
            }

        }
    }
```

对 println 方法进行了增强，也没做啥，就是先调用了原来的 println 方法，然后做了一下校验，校验是否有输出 ‘<’ 和 '>' 这两个特殊字符。

如果校验通过，则会进行分发处理。注意这里的校验只会进行一次。

> com.tencent.matrix.trace.core.LooperMonitor#dispatch

```java
    private void dispatch(boolean isBegin, String log) {

        for (LooperDispatchListener listener : listeners) {
            if (listener.isValid()) {
                if (isBegin) {
                    if (!listener.isHasDispatchStart) {
                        listener.onDispatchStart(log);
                    }
                } else {
                    if (listener.isHasDispatchStart) {
                        listener.onDispatchEnd(log);
                    }
                }
            } else if (!isBegin && listener.isHasDispatchStart) {
                listener.dispatchEnd();
            }
        }

    }
```

分发方法就是触发了 listener 的一些回调方法，里面的逻辑用了许多字段来判断，一下子不太好懂，有些逻辑理论上会重复，但是没错，建议多看几遍。

粗浅的理解就是遇到 '>' 字符，回调 onDispatchStart 方法，遇到 ‘<’ 字符回调 onDispatchEnd 方法。

我们看看 LooperDispatchListener 的代码：

> com.tencent.matrix.trace.core.LooperMonitor.LooperDispatchListener

```java
    public abstract static class LooperDispatchListener {

        boolean isHasDispatchStart = false;

        public boolean isValid() {
            return false;
        }


        public void dispatchStart() {

        }

        @CallSuper
        public void onDispatchStart(String x) {
            this.isHasDispatchStart = true;
            dispatchStart();
        }

        @CallSuper
        public void onDispatchEnd(String x) {
            this.isHasDispatchStart = false;
            dispatchEnd();
        }


        public void dispatchEnd() {
        }
    }
```

也没做什么，就是自身使用一个变量来控制了回调必须成对。然后将回调又封装了一层，子类只需要处理无参的方法就好了。

LooperMonitor类我们就分析完了，**主要是提供了一个监听，我们注册这个监听就可以知道消息分发的开始与结束**。

### 总结

每个消息在处理之前，Looper 会使用 printer 来打印一些消息，打印的消失是比较特殊的，我们可以利用打印出来的特殊字符串来做判断：

- 消息被处理之前
- 消息被处理之后

我们主要的目的是拿到这两个时间点。