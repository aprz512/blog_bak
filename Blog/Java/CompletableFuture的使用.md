---
title: CompletableFuture 的使用
date: 2019-08-18
tags: Java
---


遇到这样的一个实际问题：

写一个任务控制器，支持添加多个任务异步执行，任务间可以设置依赖，没有依赖的任务并行，有依赖的串行。

我刚开始一想，发现这个不是很简单么，写了包装类包装一下 Runnable，然后给它加一个添加依赖的功能不就好了么。后来发现没有我想的这么简单，首先循环依赖没有解决，这个问题这里就不深入了，就假设它没有循环依赖。

有了这个假设作为前提，我写出了第一版代码：

```kotlin
class Task(private var name: String) : Runnable {

    private var dependencies = mutableListOf<Task>()

    fun addDependency(vararg task: Task) {
        dependencies.addAll(task)
    }

    override fun run() {
        // 让 dependencies 先运行
        dependencies.forEach {
            it.run()
        }

        // 自己再运行
        doSome()
    }

    protected fun doSome() {
        println(name)
    }

}
```

当我，发给另一个小伙伴看的时候，他就说了，这个没有处理并发的执行啊，我一想，果然是这样。这里都没有多线程，全是一个线程。

问题的关键就是在于，需要将循环的那部分代码变为多线程代码。

本来，我想的是使用一些别的方法来做，比如，搞几个 future，等他们全部拿到结果了，在执行 doSome 的逻辑，但是这样写就有点麻烦了。突然，我想起了看《Java并发实践》这本书的时候，有提到过一个叫做 CompletableFuture  的类，它可以控制多个 Future，用在之类正好啊。



先看一下说明：

future接口可以构建异步应用，但依然有其局限性。它很难直接表述多个Future 结果之间的依赖性。实际开发中，我们经常需要达成以下目的：

- 将两个异步计算合并为一个——这两个异步计算之间相互独立，同时第二个又依赖于第
   一个的结果。
- 等待 Future 集合中的所有任务都完成。
- 仅等待 Future 集合中最快结束的任务完成（有可能因为它们试图通过不同的方式计算同
   一个值），并返回它的结果。
- 通过编程方式完成一个 Future 任务的执行（即以手工设定异步操作结果的方式）。
- 应对 Future 的完成事件（即当 Future 的完成事件发生时会收到通知，并能使用 Future
   计算的结果进行下一步的操作，不只是简单地阻塞等待操作的结果）

新的 CompletableFuture 将使得这些成为可能。



来看一下，改版之后的代码吧：

```kotlin
@RequiresApi(Build.VERSION_CODES.N)
    override fun run() {
        // 让 dependencies 先运行
        val cfs = dependencies.stream()
            .map {
                CompletableFuture.runAsync(it)
            }
            .collect(Collectors.toList())

        val array = cfs.toTypedArray()
        CompletableFuture.allOf(*array).join()
        // 自己再运行
        doSome()
    }
```

这里就是不直接执行 it.run() 了，而是交给 CompletableFuture 里面的 pool 去执行，这样就是并发的了。

拿到所有结果之后，自己的代码才能执行，所以我们使用了 join 方法。

逻辑还是非常清晰的，对 Stream 不了解的就需要去学习一下了。