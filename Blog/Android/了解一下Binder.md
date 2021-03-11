---

title: 了解一下Binder
index_img: /cover/22.jpg
banner_img: /cover/top.jpg
date: 2021-02-22
categories: Android
---

### 一个例子

Binder用于线程之间的通信，我们看一个例子：

![img](https://img-blog.csdnimg.cn/20190307214553683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MDM3MDQ3,size_16,color_FFFFFF,t_70)

这是点击桌面的图标，启动app的情景。我们可以看到这里面涉及到了4个进程，不算fork，有两个都使用了 Binder 通信，剩余一个却使用了 socket 通信，那么这是为什么呢？

###  Zygote 进程与 SystemService 进程的 socket 通信

我查了一些文章，一般都认为是 Binder 线程池导致的问题。

下面介绍一下 Binder 线程池。

Binder 有 C/S 两端，对于 Service 端来说，它需要处理来自 Client 的请求，那么问题来了，Service 一个线程够用的吗？显然是不够的，不然的话，多个 client 同时发出请求，那就需要排队了，这个体验显然是不好的。所以，会有一个线程池来处理请求。而 Binder 的双方一般都是互为 C/S 端，所以两边各有一个线程池。那么我们写 Service 端的时候，需要注意同步问题。

再回来说，为什么线程池会导致 Zygote 进程与 SystemService 进程之间的通信无法使用 Binder 而是要使用 Socket 呢？

因为应用进程是从 Zygote 进程 fork 出来的，而 Unix 的进程 fork 有规定：

> fork只能拷贝当前线程，不支持多线程的fork。

所以，如果 Zygote 采用 Binder 通信的话，fork 出来的应用进程就有问题，比如可能死锁（如果拷贝的当前线程还带等待别的线程的锁，但是别的线程没有拷贝过来）。

Zygote 在fork进程之前，会把多余的线程（包括Binder线程）都杀掉只保留一个线程。所以此时就无法把结果通过Binder把消息发送给system_server。fork()进程完成之后Zygote也会把其他线程重新启动，这时候即使有了Binder线程，也无法重新建立连接。

所以，就采用了 socket 通信，我觉得，这个解释还算比较合理。



### Binder 的优缺点

再看看 Binder 相比于 socket 通信的优缺点，毕竟是特意设计出来的，肯定要有两把刷子。

#### Binder一次拷贝原理

我们先说内核空间与用户空间：

内核空间（Kernel）是系统内核运行的空间，用户空间（User Space）是用户程序运行的空间。为了保证安全性，它们之间是隔离的。

![腾讯面试题——谈一谈Binder的原理和实现一次拷贝的流程](http://p9.pstatp.com/large/pgc-image/26eb41d898ca4c6d9467a9789f400005)

虽然从逻辑上进行了用户空间和内核空间的划分，但不可避免的用户空间需要访问内核资源，比如文件操作、访问网络等等。为了突破隔离限制，就需要借助系统调用来实现。系统调用是用户空间访问内核空间的唯一方式，保证了所有的资源访问都是在内核的控制下进行的，避免了用户程序对系统资源的越权访问，提升了系统安全性和稳定性。

Binder的一次拷贝的核心就是：

**当数据从用户空间拷贝到内核空间的时候，是直从当前进程的用户空间接拷贝到目标进程的内核空间，这个过程是在请求端线程中处理的，操作对象是目标进程的内核空间**。

然后，再通过内存映射，目标进程的用户空间就可以直接读取数据了。

内存映射可以简单的理解为，两个地址都指向物理内存的同一段区域。

![img](https://upload-images.jianshu.io/upload_images/1460468-1f61b4f411c35094.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

#### Binder传输数据的大小限制

比如在Activity之间传输BitMap的时候，如果Bitmap过大，就会引起问题，比如崩溃等，这其实就跟Binder传输数据大小的限制有关系，在上面的一次拷贝中分析过，mmap函数会为Binder数据传递映射一块连续的虚拟地址，这块虚拟内存空间其实是有大小限制的，不同的进程可能还不一样。

普通的由Zygote孵化而来的用户进程，所映射的Binder内存大小是不到1M的，准确说是 1*1024*1024) - (4096 *2) ：这个限制定义在ProcessState类中，如果传输说句超过这个大小，系统就会报错，因为Binder本身就是为了进程间频繁而灵活的通信所设计的，并不是为了拷贝大数据而使用的：

```cpp
#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))
```

