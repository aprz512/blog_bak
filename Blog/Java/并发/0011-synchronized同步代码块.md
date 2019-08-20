---
title: 0011-synchronized同步代码块
date: 2019-8-19
tags: Java-并发
categories: 并发
---

![](http://ifeve.com/wp-content/uploads/2013/04/keys_and_lock_holding250.jpg)

同步代码块就像一个锁一样，将代码给锁起来，线程需要执行同步块中的代码时需要先获取锁才能执行。一次只能有一个线程获得锁。

Java中的同步块用`synchronized`标记。同步块在Java中是同步在某个对象上。所有同步在同一个对象上的同步块在同时只能被一个线程进入并执行操作。所有其他等待进入该同步块的线程将被阻塞，直到执行该同步块中的线程退出。

有四种不同的同步块：

1. 实例方法
2. 静态方法
3. 实例方法中的同步块
4. 静态方法中的同步块

## 实例同步方法

```java
public synchronized void add(int value){
    this.count += value;
}
```

这个方法是同步在这个方法所属的对象上的。如果**有多个线程执行该对象的 add 方法**，就会出现阻塞，只有获取了锁的线程才能执行。但是如果是执行的不同对象上的 add 方法，因为获取的是不同的锁，所以不会阻塞。



## 静态同步方法

```java
public static synchronized void add(int value){
    this.count += value;
}
```

与实例同步方法一样。不过它同步的不是实例对象，而是类对象（XXX.class）。因为在Java虚拟机中一个类只能对应一个类对象，所以同时只允许一个线程执行同一个类中的静态同步方法。



## 实例方法中的同步块

有时你不需要同步整个方法，而是同步方法中的一部分。

```java
public void add(int value){

    synchronized(this){
        this.count += value;
    }
    
}
```

注意到，这里我们传递了this，说明同步代码块是作用在 this 这个对象上的，它的作用与实例同步方法一样。（在同步构造器中用括号括起来的对象叫做监视器对象）

我们出了传递 this，还可以传递别的对象，只要是你想让代码块作用在这个对象上就行。



## 静态方法中的同步块

```java
public class MyClass {
    public static synchronized void log1(String msg1, String msg2){
        log.writeln(msg1);
        log.writeln(msg2);
    }

    public static void log2(String msg1, String msg2){
        synchronized(MyClass.class){
            log.writeln(msg1);
            log.writeln(msg2);
        }
    }
}
```

log2 与 log1 是等价的，当然静态方法中的同步块我们**也可以传递别的类的 class**。



## 示例

在下面例子中，启动了两个线程，都调用Counter类同一个实例的add方法。因为同步在该方法所属的实例上，所以同时只能有一个线程访问该方法。

```java
public class Counter{
    long count = 0;

    public synchronized void add(long value){
        this.count += value;
    }
}
public class CounterThread extends Thread{

    protected Counter counter = null;

    public CounterThread(Counter counter){
        this.counter = counter;
    }

    public void run() {
        for(int i=0; i<10; i++){
            counter.add(i);
        }
    }
}
public class Example {

    public static void main(String[] args){
        Counter counter = new Counter();
        Thread  threadA = new CounterThread(counter);
        Thread  threadB = new CounterThread(counter);

        threadA.start();
        threadB.start();
    }
}
```

这里创建了两个线程。**他们的构造器引用同一个Counter实例**。因此每次只允许一个线程调用该方法。另外一个线程必须要等到第一个线程退出add()方法时，才能继续执行方法。

如果两个线程引用了两个不同的Counter实例，那么他们可以同时调用add()方法。这些方法调用了不同的对象，因此这些方法也就同步在不同的对象上。这些方法调用将不会被阻塞。如下面这个例子所示：

```java
public class Example {

    public static void main(String[] args){
        Counter counterA = new Counter();
        Counter counterB = new Counter();
        Thread  threadA = new CounterThread(counterA);
        Thread  threadB = new CounterThread(counterB);

        threadA.start();
        threadB.start();
    }
}
```

注意这两个线程，**threadA和threadB不再引用同一个counter实例**。CounterA和counterB的add方法同步在他们所属的对象上。调用counterA的add方法将不会阻塞调用counterB的add方法。