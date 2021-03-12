---
title: Handler
index_img: /cover/25.jpg
banner_img: /cover/top.jpg
date: 2019-09-25
categories: Android
---

## Handler是如何切换线程的

 handler这个线程切换的模型还是挺有意思的。想要让线程A中创建的一段代码，最终在线程B中执行，需要解决不少问题。

第一个问题，就是封装，创建的代码，肯定不能直接写到线程A中，否则这段代码肯定只能在线程 A 中执行。

第二个问题，封装之后，怎么将封装后的对象，传给线程B 呢？这个可以参考生产者消费者模式，就是使用一个仓库，存放封装后的对象，线程A放入，线程B取出执行。使用仓库只是解决了储存的问题，但是怎么确定我放入之后，是线程B来取，而不是别的线程。这里又涉及到一个叫做 ThreadLocal 的东西。

我们在使用 handler 的时候，首先需要创建一个对象出来，这里有两种方式：

- 在主线程创建一个 handler，直接 new 一个对象出来，然后在子线程使用
- 直接在子线程创建一个 handler，然后传递一个 主线程的 Looper 进去

这两种方式其实都是一样的，因为直接new一个handler对象的时候，默认传递的就是当前线程的 looper。而 looper 对象就是从 threadlocal 中取出来的，每个线程有一个，主线程的是在 activityThread 中创建的，所以，我们可以直接取就好了。如果是在子线程中，我们需要先 prepare，创建一个 looper 出来，才能使用。

looper 对象关联了一个 messageQueue，这个起的就是仓库的作用。所以每个线程都有一个自己的消息仓库，那么它是如何从仓库里面取出来消息并且执行的，也是在 ActivityThread 中，它调用了 looper 的 loop 方法，这个是一个循环，会不断的从仓库里面取消息，然后执行消息。这也就是我们常说的，主线程是由消息队列构成的，始终处于一个循环中的。其实这种设计方式很常见，比如一些游戏，它的画面UI更新，就是处于一个循环种，它只需要根据用户键盘的输入，计算各个元素的位置，然后重绘即可。

那么，整个模型应该就比较清楚了：

每个线程有一个 looper（子线程需要自己调用一些方法，参考 ActivityThread）。

Looper 里面有一个仓库，存放封装后的代码，Message。

Looper 开启了一个循环，不断的处理仓库里面的消息。

子线程，获取了主线程的 handler -> looper，这样就将封装后的 message，放到了主线程的message queue 中。







## ThreadLocal 的工作原理

### 文字版理解

- 每个线程都有一个 ThreadLocalMap 类型的 threadLocals 属性。
- ThreadLocalMap 类相当于一个Map，key 是 ThreadLocal 本身，value 就是我们的值。

- 当我们通过 threadLocal.set(new Integer(123)); ，我们就会在这个线程中的 threadLocals 属性中放入一个键值对，key 是 这个 threadlocal 自己，value 就是 new Integer(123)。
- 当我们通过 threadlocal.get() 方法的时候，首先会根据这个线程得到这个线程的 threadLocals 属性，然后由于这个属性放的是键值对，我们就可以根据键 threadlocal 拿到值。 注意，这时候这个键 threadlocal 和 我们 set 方法的时候的那个键 threadlocal 是一样的，所以我们能够拿到相同的值。

### 源码分析

这个类比较简单，我们直接从 java.lang.ThreadLocal#set 这个方法看起：

>  java.lang.ThreadLocal#set

```java
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

在 set 方法的第二行，由一个 getMap 方法获取了 ThreadLocalMap 的一个实例。

> java.lang.ThreadLocal#getMap

```java
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

看到了没，这里直接使用的是 Thread 类的变量。

> java.lang.Thread

```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

所以，我们储存的数据实际上时放入到了 Thread 类的成员变量中。
继续深入 ThreadLocalMap 这个类，上面的 set 方法调用它的 set 方法，所以我们直接看它的 set 方法。

> java.lang.ThreadLocal.ThreadLocalMap

```java
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

看这个代码，思路还是满清晰的。要说一些这个 Entry 类，它是继承了 WeakReference<ThreadLocal<?>>，注意这里是一个弱引用。有一个比较有趣的问题就是：网上有讨论说 ThreadLocal 有可能出现内存泄漏问题，就与它有关系。我们来看一下引用链：

- ThreadLocal 的引用链：Thread -> ThreadLocalMap -> Entry -> WeakReference -> ThreadLocal
- value 的引用链：Thread -> ThreadLocalMap -> Entry -> value

可以看到，value 是被强引用的，所以如果没有其他对象引用 ThreadLocal 对象的话，ThreadLocal 可能会被回收，但是 value 不会被回收。而且这个时候，我们也没法访问到 value 了。这样就造成了内存泄露。一般的使用方式都是使用 static 的或者手动调用 set null。看看官方的使用方式：

> android.os.Looper

```java
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

回到正题，分析 set 方法的流程：
首先，利用 key 的 hashCode 获取索引值，然后查看索引值执行的位置有没有数据，没有数据就创建一个新的放进去，有的话，就比较 key ，key一样就直接替换 value 的值，key不一样就看下一个位置的值，再比较。

set 方法说完了，我们在看看 get 方法。

> java.lang.ThreadLocal#get

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

了解了 set 方法，get 方法其实就是反过来的，与 set 的思路一样，由于 key 始终是自己（ThreadLocal<?>），所以总能取到正确的值。这里还有一个 setInitialValue 方法，它默认返回null，就是没有设置初始值的时候，就会触发这个方法，我们可以复写这个方法，返回一个默认值。

ThreadLocal 的大致原理就说完了，再说一下它的一般用法：
第一个比较普通，就是通过它来储存线程中的数据，只有当前线程可以获取到，其他线程获取的是另一份自己的数据。
第二个使用场景是复杂逻辑下的对象传递，比如监听器的传递，有些时候一个线程中的任务过于复杂，这可能表现为函数调用栈比较深以及代码入口的多样性，在这种情况下，我们又需要监听器能够贯穿整个线程的执行过程，这个时候可以怎么做呢？其实这个时候就可以采用 ThreadLocal，让监听器作为线程内的全局对象，在线程内部只要通过get方法就可以获取到监听器。如果不采用这种方式，那么一般会使用参数的传递或者使用静态变量。使用参数传递的话，如果方法调用栈不深还可以接收，如果调用栈很深，代码看起来就很糟糕了。使用静态变量的话，多个线程就难以维护了。

## 消息队列

这里不会走源码了，最多给个图。我只是梳理一下以前没有注意到的东西。

在讨论这个主题之前，我们先来认识一下GUI的单线程模型，嗯，想来了解一下问什么。
现代的 GUI 框架使用的模型：创建一个专门的线程，事件派发线程（event dispatch thread，RDT）来处理 GUI 事件。

有很多人都试图写出多线程的GUI框架，最终都由于竞争条件和死锁导致的稳定性问题，又回到了单线程化的事件队列模型的老路上来。多线程的GUI框架会尤其易受死锁的影响，部分原因在于：

> 用户发起的动作总会冒泡似的从操作系统传递给应用程序。先是由os检测到一次鼠标点击，然后工具集把它转化为“鼠标点击”事件，最终它会作为一个高层事件（比如“buttonpressed”事件）转发给应用程序的监听器。
>
> 另一方面，应用程序发起的动作又会以冒泡的形式传回操作系统。应用程序发起一个动作要改变某个组件的背景颜色，这会被转发给一个特定的组件类，最终转发给os进行渲染。两种动作以完全相反的顺序访问相同的GUI对象，需要保证让每一个对象都是线程安全的，这会导致一系列的锁顺序的不一致，这会直接引发死锁。

**虽然，单线程模式比较简单，但是单线程消息队列机制存在一个问题：**

> 消息响应函数中不能有耗时长的、计算密集型的操作，因为主线程在努力地处理这样的操作的时候就无法去处理其它的积压在消息队列中的绘制消息、事件消息了（一个消息处理完了主线程才会去队列中取下一个消息），这时候就会出现按键无响应、点击无反应的情况。

但这个问题有完美的解决方案，**我们可以在消息响应函数中启动另一个工作线程（Worker Thread）来执行耗时操作**，这样在线程启动起来后这个消息就算处理完了，主线程可以取下一个消息了，这时候主线程和还未执行完计算任务的工作线程就在操作系统的调度下并驾齐驱地狂奔了（调度算法会保证两个线程并发或并行地执行，不会专宠某个线程）。

Android 中也是采用的单线程消息队列，它是使用 Hanlder 来处理线程之间的消息传递的。一般我们在耗时任务执行完后还要更新界面展示计算的结果，正确的处理办法是将耗时任务改为异步通知机制，即工作线程向消息队列中添加消息以通知主线程耗时任务完成了，这样主线程在启动工作线程后就不需要主动地去调查任务的进展了。

了解了为什么，现在我们从几个问题来入手消息队列的运作过程。

1. 消息是如何延迟发送的？不同的延时长度的消息是如何排序的？
2. 没有消息时，MessageQueue 在干什么，Looper在做什么？从没有消息到有消息，MessageQueue 是如何被唤醒的？
3. Message 分发的3种渠道？
4. Looper是死循环，为什么 UI 线程不会ANR?
5. IdleHandler 是什么?
6. 异步消息与同步屏障了解不？

### 延时消息

放入Message时会根据msg.when这个时间戳进行顺序的排序,如果非延迟消息则msg.when为系统当前时间，延迟消息则为系统当前时间+延迟时间(如延迟发送3秒则为：SystemClock.uptimeMillis() + 3000)。

Message放入MessageQueue时会以msg.when对msg进行排序确认当前msg处于单链表中的位置,分为几种情况:

1. 头结点为null(代表MessageQueue没有消息),Message直接放入头结点。
2. 头结点不为null时开启死循环遍历所有节点
   - 遍历出的节点的when大于放入message的when(说明当前message是一个比放入message延迟更久的消息，将放入的Message放入当前遍历的Message节点之前)。
   - 遍历出的节点的next节点为null(说明当前链表已经遍历到了末尾，将放入的Message放入next节点)。

> android.os.MessageQueue#enqueueMessage

```java
    boolean enqueueMessage(Message msg, long when) {
        ...

        synchronized (this) {
        	// 退出循环
            if (mQuitting) {
                ...
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // 没有其他消息，把它作为头结点
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    // 根据时间来寻找节点的位置
                    if (p == null || when < p.when) {
                        break;
                    }
                    // 异步消息与屏障的处理，后面会说到
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            ...
        }
        return true;
    }
```

### 消息队列的阻塞与唤醒

对于消息队列而言，从里面取出消息需要考虑多个方面：

- 如果队列为空了，或者队列里面的消息没有可以取出的消息（时间都没到），那么应该阻塞消息队列。阻塞肯定不能用一般的方式，如果像流一样，直接阻塞了线程，浪费CPU，导致 ANR，那肯定是不行的，所以应该怎么办呢？
- 阻塞需要阻塞多长时间呢？怎么保存这个时间？ 这个比较简单，message 有个 when 字段保存了时间，由于 message 是排序了的，所以只需要去头部的 message 的 when 用来计算就好了。
- 阻塞后如何唤醒？

我们根据代码来分析：

> android.os.MessageQueue#next

```java
...
nativePollOnce(ptr, nextPollTimeoutMillis);
...
```

nativePollOnce 这个方法是一个 native 方法，它就是用来阻塞消息队列的，nextPollTimeoutMillis 就是阻塞的时间。在说明这个方法做了什么之前，我们需要先了解一下 epoll 是什么！

#### 为什么Android的Handler采用管道？

我们先来看一张图，图里展示的是Android整个Hander的设计。

![](https://pic2.zhimg.com/80/28a5f0d87457d432727270313cfec3a9_720w.jpg)

- 红色虚线关系：Java层和Native层的MessageQueue通过JNI建立关联，彼此之间能相互调用，搞明白这个互调关系，也就搞明白了Java如何调用C++代码，C++代码又是如何调用Java代码。

- 蓝色虚线关系：Handler/Looper/Message这三大类Java层与Native层并没有任何的真正关联，只是分别在Java层和Native层的handler消息模型中具有相似的功能。都是彼此独立的，各自实现相应的逻辑。

  

我们可以看出，Handler的设计不仅仅只是Java层，还有对应的C++层。这是因为Handler不仅仅要处理同一进程中别的线程发过来的消息，还要处理系统底层发过来的消息。在整个消息机制中，`MessageQueue`是连接Java层和Native层的纽带，换言之，Java层可以向MessageQueue消息队列中添加消息，Native层也可以向MessageQueue消息队列中添加消息。

如果只是Java层的话，使用 wait 与 notify 也可以做到唤醒。

### 分发优先级

当遍历出Message后Message会获取其中的Handler并调用Handler的dispatchMessage进行分发,这时也会有三个优先级。

> android.os.Handler#dispatchMessage

```java
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

- Message的回调方法：message.callback.run()，优先级最高； 对应handler.post(new Runnable)的方式发送消息。

- Handler的回调方法：Handler.mCallback.handleMessage(msg)，优先级仅次于上面； 对应新建Handler时传进CallBack接口

  ```java
   Handler handler=new Handler(new Handler.Callback());
  ```

  通常我们可以利用 Callback 这个拦截机制来拦截 Handler 的消息，场景如：Hook [ActivityThread.mH](http://activitythread.mh/)，在 ActivityThread 中有个成员变量 mH ，它是个 Handler，又是个极其重要的类，几乎所有的插件化框架都使用了这个方法。

- Handler的默认方法：Handler.handleMessage(msg)，优先级最低。对应新建Handler并复写handleMessage方法。

### Looper 的死循环

我们知道Android 的是由事件驱动的，looper.loop() 不断地接收事件、处理事件，**每一个点击触摸或者说Activity的生命周期都是运行在 Looper的控制之下，如果它停止了，应用也就停止了**。

真正的阻塞是因为轮询出message后在处理message消息的时候由于执行了耗时操作导致了ANR，而不是死循环导致的阻塞，没有消息处理的时候消息队列是阻塞在nativePollOnce方法中的，**这个方法使用的是epoll管道机制，Linux底层执行后会释放CPU避免不断死循环造成的CPU浪费。**

### IdleHandler 是什么

简而言之，IdleHandler 是一个接口，就是在looper里面的message暂时处理完了，这个时候会回调这个接口，返回false，那么就会移除它，返回true就会在下次message处理完了的时候继续回调。

IdleHandler 可以用来提升提升性能，主要用在我们希望能够在当前线程消息队列空闲时做些事情（譬如UI线程在显示完成后，如果线程空闲我们就可以提前准备其他内容）的情况下，不过最好不要做耗时操作。

当从消息链中取消息的时候，如果取到的消息还未到执行的时间或者没有取到消息，那么就会触发 IdleHandler （当然前提是你设置了），需要注意的是，这个回调只会执行一次，嗯，意思是如果在 5s 内，都没有消息需要处理，在这 5s 内，只会调用一次，而不会不断的调用。

> android.os.MessageQueue#next

```java
// Reset the idle handler count to 0 so we do not run them again.
pendingIdleHandlerCount = 0;
```

可以看到，调用一次之后，就将 count 设置为 0，下次就不会调用了，直到再次触发 next 方法。

可以看一下[这篇文章](https://blog.csdn.net/tencent_bugly/article/details/78395717)的使用。

有一个需要注意的地方，如果我们设置了 IdleHandler，而且在它的 `queueIdle`方法的返回值里面返回了 true。那么当消息队列为空的时候，这个方法会被不断的回调吗？

答案是不是，也只会被回调一次。但是当消息队列再次为空时，会被回调。我们看看源码：

```java
@UnsupportedAppUsage
Message next() {
    
	// 开始的时候，对这两个变量赋值
	// 表示没有 IdleHandler
    int pendingIdleHandlerCount = -1;
    // 睡眠时间为 0
    int nextPollTimeoutMillis = 0;
    for (;;) {
        
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 消息队列不为空
            if (msg != null) {
                ...
            } else {
                // 消息队列为空，表示无限制睡眠，等待唤醒
                nextPollTimeoutMillis = -1;
            }

            ...

            // 第一次循环进来，pendingIdleHandlerCount 的值为 -1
            // 然后判断消息队列是否为空（无消息或者无可执行消息）
            // 为空，就给 pendingIdleHandlerCount 赋值
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            // 没有 IdleHandler，直接继续循环
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            ...
        }

        // 调用 queueIdle 方法
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            ...
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
			// 如果 queueIdle 返回了 false，就移除监听
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // 重置变量，这里将 pendingIdleHandlerCount 赋值为 0
        // 那么，下一次循环的时候，由于将睡眠时间设置为了0
        // 但是仍然没有消息，但是 pendingIdleHandlerCount 为 0，那么就不会进入 queueIdle 的回调逻辑
        // 会进入无限制睡眠状态
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}
```



### 同步分割栏

所谓“同步分割栏”，可以被理解为一个**特殊Message**，它的target域为null。它不能通过sendMessageAtTime()等函数打入到消息队列里，而只能通过调用Looper的postSyncBarrier()来打入。

“同步分割栏”是起什么作用的呢？它就像一个卡子，卡在消息链表中的某个位置，当消息循环不断从消息链表中摘取消息并进行处理时，一旦遇到这种“同步分割栏”，那么即使在分割栏之后还有若干已经到时的普通Message，也不会摘取这些消息了。请注意，此时只是不会摘取“普通Message”了，如果队列中还设置有“异步Message”，那么还是会摘取已到时的“异步Message”的。

> 需要注意的一点是：
>
> 同步分隔栏插入到消息链表中的时候，也是按照时间的顺序来插入的。也就是说，如果分隔栏的时间是 10ms 后执行，那么从现在到10ms后的这个时间段，消息链表中的同步消息也能被执行。

在Android的消息机制里，“普通Message”和“异步Message”也就是这点儿区别啦，也就是说，如果消息列表中根本没有设置“同步分割栏”的话，那么“普通Message”和“异步Message”的处理就没什么大的不同了。

将普通消息变成异步消息，只需要调用一个方法就可以啦：

> android.os.Message#setAsynchronous

```java
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
```

### 参考文档

[带你真正攻克Handler](https://juejin.im/post/5c87c0ad6fb9a049f571fe3f)

[聊一聊Android的消息机制](https://my.oschina.net/youranhongcha/blog/492591)

[我读过的最好的epoll讲解–转自”知乎“](https://blog.51cto.com/yaocoder/888374)

