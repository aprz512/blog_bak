---
title: RxJava2的doOnSubscribe引发的一个问题
index_img: /cover/01.jpg
banner_img: /cover/top.jpg
date: 2020-04-01
tags: Android-Bugs
---

接到一个需求，需要对一个旧功能做改动，由于网络部分的逻辑都很模板化（想念retrofit），所以我直接从别的地方copy了一份，在这里改吧改吧就运行了。但是运行时报错，**报了在子线程访问UI的异常**。这个异常很常见，也很好理解。

我们的代码逻辑大致如下：

```java
Http.request(xxxx)
    // 这里面弹出一个加载框
    .doOnSubscribe(xxx)
    .subscribeOn(io())
    .observerOn(main());
```

对RxJava2比较熟悉的同学，立马就能知道问题出在了哪里！因该把 doOnSubscribe 放在 subscribeOn 后面才对。

这里贴一张图，便于理解RxJava2。

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-Bugs/20200401211915.jpg?raw=true)

但是这里又有另外一个问题！！！这段代码是我从别的地方copy过来的，我copy的地方为什么没有问题？？？

我特意的断点查看，发现copy代码的地方，确实是在子线程执行的，那么它为什么没有报错呢？

经过一番排查，发现我copy地方弹出的加载框是使用的 DialogFragment，而老的代码加载框使用的是 ProgressDialog。

查看源码发现，DialogFragment 的 show 方法，实际上是显示了一个 Fragment。

我们知道，对 Fragment 的操作，都只是一个很简单的 addOp 操作，然后在 commit 的时候，会将这些操作一起执行，而 commit 操作的内部逻辑都是通过 handler.post 执行的。所以在子线程操作没有问题！！！

另外，有一个控件可以在子线程更新：ProgressBar。如果你去查看源码，会发现里面有不少同步代码！！！

再说一句，不让UI在子线程更新，是为了避免同步问题，如果可以在子线程更新UI，那UI框架非常容易导致死锁。输入事件处理与任何GUI组件背后的对象模型之间存在偶发的交互。用户发起的动作总会冒泡似的从操作系统传递给应用程序—先是由os检测到一次鼠标点击，然后工具集把它转化为“鼠标点击”事件，最终它会作为一个高层事件（比如“buttonpressed”事件）转发给应用程序的监听器。另一方面，应用程序发起的动作又会以冒泡的形式传回操作系统—应用程序发起一个动作要改变某个组件的背景颜色，这会被转发给一个特定的组件类，最终转发给os进行渲染。两种动作以完全相反的顺序访问相同的GUI对象，需要保证让每一个对象都是线程安全的，这会导致一系列的锁顺序的不一致，这会直接引发死锁。

