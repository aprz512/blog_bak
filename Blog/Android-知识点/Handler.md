---
title: Handler
date: 2019-09-25
tags: Android-知识点
---


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

#### epoll

首先我们来定义**流**的概念，一个流可以是文件，socket，pipe等等可以进行I/O操作的内核对象。

不管是文件，还是套接字，还是管道，我们都可以把他们看作流。

之后我们来讨论I/O的操作，通过read，我们可以从流中读入数据；通过write，我们可以往流写入数据。现在假定一个情形，我们需要从流中读数据，但是流中还没有数据，（典型的例子为，客户端要从socket读如数据，但是服务器还没有把数据传回来），这时候该怎么办？

**阻塞**：阻塞是个什么概念呢？比如某个时候你在等快递，但是你不知道快递什么时候过来，而且你没有别的事可以干（或者说接下来的事要等快递来了才能做）；那么你可以去睡觉了，因为你知道快递把货送来时一定会给你打个电话（假定一定能叫醒你）。

**非阻塞忙轮询**：接着上面等快递的例子，如果用忙轮询的方法，那么你需要知道快递员的手机号，然后每分钟给他挂个电话：“你到了没？”

很明显一般人不会用第二种做法，不仅显很无脑，浪费话费不说，还占用了快递员大量的时间。

大部分程序也不会用第二种做法，因为第一种方法经济而简单，经济是指消耗很少的CPU时间，如果线程睡眠了，就掉出了系统的调度队列，暂时不会去瓜分CPU宝贵的时间片了。

为了了解阻塞是如何进行的，我们来讨论缓冲区，以及内核缓冲区，最终把I/O事件解释清楚。缓冲区的引入是为了减少频繁I/O操作而引起频繁的系统调用（你知道它很慢的），当你操作一个流时，更多的是以缓冲区为单位进行操作，这是相对于用户空间而言。对于内核来说，也需要缓冲区。

假设有一个管道，进程A为管道的写入方，Ｂ为管道的读出方。

假设一开始内核缓冲区是空的，B作为读出方，被阻塞着。然后首先A往管道写入，这时候内核缓冲区由空的状态变到非空状态，内核就会产生一个事件告诉Ｂ该醒来了，这个事件姑且称之为“缓冲区非空”。

但是“缓冲区非空”事件通知B后，B却还没有读出数据；且内核许诺了不能把写入管道中的数据丢掉，这个时候，Ａ写入的数据会滞留在内核缓冲区中，如果内核也缓冲区满了，B仍未开始读数据，最终内核缓冲区会被填满，这个时候会产生一个I/O事件，告诉进程A，你该等等（阻塞）了，我们把这个事件定义为“缓冲区满”。

假设后来Ｂ终于开始读数据了，于是内核的缓冲区空了出来，这时候内核会告诉A，内核缓冲区有空位了，你可以从长眠中醒来了，继续写数据了，我们把这个事件叫做“缓冲区非满”。

也许事件Y1已经通知了A，但是A也没有数据写入了，而Ｂ继续读出数据，知道内核缓冲区空了。这个时候内核就告诉B，你需要阻塞了！，我们把这个时间定为“缓冲区空”。

这四个情形涵盖了四个I/O事件，缓冲区满，缓冲区空，缓冲区非空，缓冲区非满（注都是说的内核缓冲区，且这四个术语都是我生造的，仅为解释其原理而造）。这四个I/O事件是进行阻塞同步的根本。（如果不能理解“同步”是什么概念，请学习操作系统的锁，信号量，条件变量等任务同步方面的相关知识）。

然后我们来说说阻塞I/O的缺点。但是阻塞I/O模式下，一个线程只能处理一个流的I/O事件。如果想要同时处理多个流，要么多进程(fork)，要么多线程(pthread_create)，很不幸这两种方法效率都不高。

于是再来考虑非阻塞忙轮询的I/O方式，我们发现我们可以同时处理多个流了（把一个流从阻塞模式切换到非阻塞模式再此不予讨论）：

```java
while true {
	for i in stream[]; {
		if i has data
			read until unavailable
	}
}
```

我们只要不停的把所有流从头到尾问一遍，又从头开始。这样就可以处理多个流了，但这样的做法显然不好，因为如果所有的流都没有数据，那么只会白白浪费CPU。这里要补充一点，阻塞模式下，内核对于I/O事件的处理是阻塞或者唤醒，而非阻塞模式下则把I/O事件交给其他对象（后文介绍的select以及epoll）处理甚至直接忽略。

为了避免CPU空转，可以引进了一个代理（一开始有一位叫做select的代理，后来又有一位叫做poll的代理，不过两者的本质是一样的）。这个代理比较厉害，可以同时观察许多流的I/O事件，在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有I/O事件时，就从阻塞态中醒来，于是我们的程序就会轮询一遍所有的流。代码长这样：

```java
while true {
	select(streams[])
		for i in streams[] {
			if i has data
				read until unavailable
	}
}
```

于是，如果没有I/O事件产生，我们的程序就会阻塞在select处。但是依然有个问题，我们从select那里仅仅知道了，有I/O事件发生了，但却并不知道是那几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。

但是使用select，我们有O(n)的无差别轮询复杂度，同时处理的流越多，没一次无差别轮询时间就越长。再次说了这么多，终于能好好解释epoll了。

epoll可以理解为event poll，不同于忙轮询和无差别轮询，epoll之会把哪个流发生了怎样的I/O事件通知我们。此时我们对这些流的操作都是有意义的。

一个epoll模式的代码大概的样子是：

```java
while true {
	active_stream[] = epoll_wait(epollfd)
	for i in active_stream[] {
		read or write till
	}
}
```

前文我们已经说过，next()中调用的nativePollOnce()起到了阻塞作用，保证消息循环不会在无消息处理时一直在那里“傻转”。那么，nativePollOnce()函数究竟是如何实现阻塞功能的呢？

其实是在 C++ 层的 Looper 的构造函数中，其内部除了创建了一个管道以外，还创建了一个epoll来监听管道的“读取端”。也就是说，**是利用epoll机制来完成阻塞动作的。每当我们向消息队列发送事件时，最终会间接向管道的“写入端”写入数据，于是epoll通过管道的“读取端”立即就感知到了风吹草动，epoll_wait()在等到事件后，随即进行相应的事件处理。**这就是消息循环阻塞并处理的大体流程。当然，因为向管道写数据只是为了通知风吹草动，所以写入的数据是非常简单的“W”字符串。

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

### 同步分割栏

所谓“同步分割栏”，可以被理解为一个特殊Message，它的target域为null。它不能通过sendMessageAtTime()等函数打入到消息队列里，而只能通过调用Looper的postSyncBarrier()来打入。

“同步分割栏”是起什么作用的呢？它就像一个卡子，卡在消息链表中的某个位置，当消息循环不断从消息链表中摘取消息并进行处理时，一旦遇到这种“同步分割栏”，那么即使在分割栏之后还有若干已经到时的普通Message，也不会摘取这些消息了。请注意，此时只是不会摘取“普通Message”了，如果队列中还设置有“异步Message”，那么还是会摘取已到时的“异步Message”的。

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

