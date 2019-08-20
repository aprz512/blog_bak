---
title: 0013-ThreadLocal
date: 2019-8-20
tags: Java-并发
categories: 并发
---



ThreadLocal类创建的变量会给每个线程都分配一份，虽然每个线程都持有的是执行同一个ThreadLocal对象的引用，但是获取的（调用 get 方法）确实不同的对象。



## 创建 ThreadLocal 对象

```java
private ThreadLocal myThreadLocal = new ThreadLocal();
```

这个好理解，与普通Java类一样使用即可。



## 储存值

当创建好了 ThreadLocal 之后，我们就可以往里面储存值了，就像 ThreadLocal 是一个容器一样。

```java
// ThreadLocal 支持泛型
private ThreadLocal myThreadLocal1 = new ThreadLocal<String>();

// 储存一个 string 对象
myThreadLocal1.set("Hello ThreadLocal");
String threadLocalValues = myThreadLocal.get();
```

嗯，获取用 String 来当作例子有点不恰当，没法看出它是不是同一个对象，我们稍微改变一下：

```java
// ThreadLocal 支持泛型
private ThreadLocal myThreadLocal1 = new ThreadLocal<Object>();

// 储存一个 string 对象
myThreadLocal1.set(new Object());
Object o = myThreadLocal.get();
System.out.println(o.hashcode())
```

在不同的线程里面执行，打印 hashCode 会发现是不同的值，即不同的线程 set 与 get 获取的都是属于自己的那一份，无法获取别的线程的，别的线程也获取不到自己的。



## 初始值

ThreadLocal 还可以指定一个初始值，即当没有执行 set 方法的时候，get 方法也能取出初始值来。

```java
private ThreadLocal<Object> threadLocal = ThreadLocal.withInitial(Object::new);
```

这个**初始值也是每个线程都有一份**，每个线程获取的也是不同的对象，而不是同一个对象。



## 示例

```java
public class ThreadLocalExample {

    public static class MyRunnable implements Runnable {

        private ThreadLocal<Integer> threadLocal =
            new ThreadLocal<Integer>();

        @Override
        public void run() {
            threadLocal.set( (int) (Math.random() * 100D) );

            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
            }

            System.out.println(threadLocal.get());
        }
    }

    public static void main(String[] args) {
        MyRunnable sharedRunnableInstance = new MyRunnable();

        Thread thread1 = new Thread(sharedRunnableInstance);
        Thread thread2 = new Thread(sharedRunnableInstance);

        thread1.start();
        thread2.start();

        thread1.join(); //wait for thread 1 to terminate
        thread2.join(); //wait for thread 2 to terminate
    }

}
```

上面创建了两个线程共享一个MyRunnable实例。

每个线程执行run()方法的时候，会给同一个ThreadLocal实例设置不同的值。

如果不使用ThreadLocal对象，那么第二个线程将会覆盖第一个线程所设置的值。

然而，由于是ThreadLocal对象，所以两个线程无法看到彼此的值。因此，可以设置或者获取不同的值。



## InheritableThreadLocal

InheritableThreadLocal类是ThreadLocal的子类。

为了解决ThreadLocal实例内部每个线程都只能看到自己的私有值，所以InheritableThreadLocal**允许一个线程创建的所有子线程访问其父线程的值**。



## 需要注意的地方

ThreadLocal 实际上是将要存放的对象放入到了 Thread 的 localValues 变量中.

```java
    /**
     * Normal thread local values.
     */
    ThreadLocal.Values localValues;

```



使用 set 方法的时候，是将 ThreadLocal 本身的弱引用做为 key，将要储存的对象做为 value。

```java
    /**
     * Sets the value of this variable for the current thread. If set to
     * {@code null}, the value will be set to null and the underlying entry will
     * still be present.
     *
     * @param value the new value of the variable for the caller thread.
     */
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
```

 看起来传递的是 this，其实真正put的时候，使用的是 key.reference. 这个 reference 是一个弱引用。

```java
    /** Weak reference to this thread local instance. */
    private final Reference<ThreadLocal<T>> reference
            = new WeakReference<ThreadLocal<T>>(this);
```

所以，网上多说，会有内存泄露的可能。是因为如果 ThreadLocal 本身如果没有再使用了，而当前线程迟迟不结束的话，会导致 Thread 的 localValues 变量里存的 key 被回收，values 却无法被回收（引用找不到了，但是却存在于 threa 的成员变量里面），但是只要线程结束就好了。



**所以，使用 ThreadLocal 推荐写成 private static 的**。

