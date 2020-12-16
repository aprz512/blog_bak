---
title: 0014-volatile关键字
index_img: /cover/20.jpg
banner_img: /cover/top.jpg
date: 2019-8-20
tags: Java-并发
categories: 并发
---



Java 的 volatile 关键字表示修饰的这个变量的 **值储存在主存** 中，这里不要理解错了，并不是说 volatile 修饰的变量就直接在主内存中操作（这是不可能的），而是说：

- 修改volatile变量时会强制将修改后的值刷新的主内存中。
- 修改volatile变量后会导致其他线程工作内存中对应的变量值失效。因此，需要重新读取主内存中的值。

说的通俗一点，就是每次读取 volatile 变量都会从主存中读取，每次写入 volatile 变量都是写入到主存中。



前面我们说过可见性问题，而 volatile 关键字就是用来保证线程之间变量的可见性。可能还是有点抽象，那就再说一遍。

## volatile 对可见性的保证

现在的 CPU 为了性能，会将内存中的数据拷贝到 CPU 的高速缓存中。如果有多个 CPU 的话，每个线程会运行在不同的 CPU 上，即每个线程都会拷贝一个数据到高速缓存中。如下图：

![](http://tutorials.jenkov.com/images/java-concurrency/java-volatile-1.png)

对于非 volatile 的变量，在多线程的程序中，就会出现可见性问题，当一个线程更新了值之后，而另一个线程却看不到，就会导致程序错误：

```java
public class SharedObject {

    public int counter = 0;

}
```

假设，线程 1 会更新计数器的值， 线程 2 会时时的读取这个计数器的值。由于 counter 不是 volatile 的，所以在线程 1 更新其值之后，不一定会将值刷新到主存中，当线程 2 读取的时候，读取的还是未更新的值：

![](http://tutorials.jenkov.com/images/java-concurrency/java-volatile-2.png)



使用 volatile 之后，就不一样了。volatile 就是用来解决这个问题的，我们给 counter 加上关键字：

```java
public class SharedObject {

    public volatile int counter = 0;

}
```

当线程1更新 counter 的值后， 线程2就能读到最新值了。

但是如果 线程1 与 线程2 都更新 counter 的值的话，仅仅加上 volatile 关键字还是不够的，后面会说。



## volatile 对重排序的影响

volatile 可见性不仅会影响到它优化的变量，还会对重排序有一定影响。看下面这个例子：

```java
//x、y为非volatile变量
//flag为volatile变量

x = 2;         //语句1
y = 0;         //语句2
flag = true;   //语句3
x = 4;         //语句4
y = -1;        //语句5
```

由于 flag 变量为 volatile 变量，那么在进行指令重排序的过程的时候，**不会将语句3放到语句1、语句2前面，也不会讲语句3放到语句4、语句5后面**。但是要注意语句1和语句2的顺序、语句4和语句5的顺序是不作任何保证的。

并且 volatile 关键字能保证，**执行到语句3时，语句1和语句2必定是执行完毕**了的，且语句1和语句2的执行结果对语句3、语句4、语句5是可见的。



## volatile 与 Happens-Before

再来看一个例子：

```java
public class MyClass {
    private int years;
    private int months
    private volatile int days;

    public int totalDays() {
        int total = this.days;
        total += months * 30;
        total += years * 365;
        return total;
    }

    public void update(int years, int months, int days){
        // 1
        this.years  = years;
        // 2
        this.months = months;
        // 3
        this.days   = days;
    }
}
```

update 方法写了3个变量，只有 days 是 volatile 的。但是实际上在将 days 的值刷入主内存的时候，会将 years 与 months 的值也刷入主内存。

totalDays 方法读取了3个变量的值，当从主内存读取 days 的值的时候，也会从主内存读取 months 与 years 的值。

起初，我是无法理解的，于是去  stackOverFlow 上问了一下，很快就有了答案：

这个是由于 “Happens-Before” 原则引发的：

> 由于 days 被 volatile 修饰，所以代码 1 2 处 Happens-Before 代码 3，即代码 1 2 的结果对代码 3 是可见的。
>
> 同样，volatile 变量的写 Happens-Before volatile 变量的读。
>
> 在根据 Happens-Before 的传递性，所以某一个线程想要读取 days 的值的时候，months 与  years 的值也是最新的。



## volatile 使用注意

前面，我们说的多个线程更新 counter 的值，尽管 counter 是 volatile 的，但是还是会出问题，现在就来解释一下。

```java
counter++;
```

就拿这个举例，假设某个时刻，counter 的值为 4。

```
线程1从主内存中读取值为4，准备执行加一的指令，然后线程1被阻塞，切换到线程2

线程2从主内存中读取值为4，准备执行加一的指令，然后线程2被阻塞，切换到线程1

线程1执行加一的指令，最后将自增后的值赋值给counter，counter的值成了5，写入回了主内存。线程1阻塞，切换到线程2.

线程2执行加一的指令（将4加一的过程中不需要对counter进行读写，所以自增之后的值还是5），然后将 5 赋值给 counter，并写入回了主内存。

最后，counter 的结果还是 5.
```

看一张图：

![](https://user-gold-cdn.xitu.io/2017/9/30/ee48b1e38ea8819d6f06df279a819a1d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

在读取一个变量与给这个变量赋值之间是有一段间隔的，这段间隔表示它们不是一个原子操作，不是原子操作就会产生竟态条件。两个线程同时读取值之后，一个线程即使更新了值，也不管用了，因为读取操作已经完成了，后面的写操作不需要再次读取该值，也就看不到最新的值。

所以，volatile 应该用在不需要依赖变量的当前值的地方。反过来说：

> 运算结果并不依赖变量的当前值（即结果对产生中间结果不依赖），或者能够确保只有单一的线程修改变量的值

比如，用来更新标识变量，直接给标识赋值，`flag =  true 或者 flag = false `，不需要依赖当前的值，像  flag = !flag 就不行。



最后，由于 volatile 每次写操作将最新值刷入主存，每次读操作要从主存重新读取，所以效率不高，而且它还禁止了指令优化，所以一定要在确实需要的时候才使用。
