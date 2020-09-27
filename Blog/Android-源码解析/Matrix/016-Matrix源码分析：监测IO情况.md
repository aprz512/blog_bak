---
title: 016-Matrix源码分析：监测IO情况
date: 2020-8-31
categories: Matrix
---

> 经过一个星期的 C++ 基本语法的学习，终于可以继续分析了，好不容易写了 15 篇，被个 C++ 拦住了就笑死人了。

IO的监测功能由以下3个库实现：

```
matrix-io-canary
matrix-android-lib
matrix-android-commons
```

matrix-io-canary 里面的是核心代码。

这个库里面提供了一下几个功能：

- 主线程IO，设定了一些阈值，超过就报警（最大读写时间超过 13ms，一次连续读写时间超过 500ms）
- 重复读IO（这个感觉有点鸡肋，可能是我没搞懂使用场景）
- buffer太小的IO行为



在分析这些功能是如何实现的之前，我们想一下，该如何监测应用中的的IO呢？

我们通常读取文件，都是通过 FileInputStream 等类来实现的，他们的内部是调用了 native 函数，最终会调用到 `libjavacore.so` 中的 `read/write` 方法，所以，我们只需要 hook 这俩个函数就好了。所以，最终的问题转化为如何hook 指定 .so 中的函数？？？

解决上面的问题需要用到 elf 文件格式的知识，以及 .so 的加载与链接知识。这些在《程序员的自我修养-链接、装载与库》中都有详细的描述，可以看看，我花了几天时间差不多看明白了。这里就简单的说一下这个过程。

### .so相关

> 没有阅读 《程序员的自我修养-链接、装载与库》 这本书的相关章节的话，这段内容几乎看不懂。

.so 是一个 ELF 文件格式的文件。

.so 是共享库，也就是说它加载到内存后，是可以多个进程共享的。但是这个文件的加载比较特殊，并不是与普通文件一样，一股脑的全部放到内存就行了。

简单的来说，它里面有多个段，指令（就是存放函数代码，它是只读的）放在一个段，这个是可以共享的。而有些段不是共享的，它是每个进程各自一份，比如全局偏移表（.got），这个里面存放的就是变量函数的地址，因为每个进程都有自己的虚拟内存，所以共享库函数的地址也是不一样的。

与 .got 相关的还有一个 .plt，它是用来懒绑定的，就是说函数到真正使用的时候，才会绑定地址，因为有的函数根本就不会被调用。

### hook 函数

这里的 hook 有两种选择，一种是 java 层，一种是 native 层。

但是 java 层有些缺点：

- 兼容性差。Java Hook 需要每个 Android 版本去兼容，特别是 Android P 增加对非公开 API 限制。
- 无法监控 Native 代码。
- I/O 操作调用非常频繁，因为使用动态代理和 Java 的大量字符串操作，导致性能比较差，无法达到线上使用的标准。

所以，采用的是 native 的方式。最终是从 libc.so 中的这几个函数中选定 Hook 的目标函数（当然，遍历所有已经加载的 library，全部替换更好）。

```c++
int open(const char *pathname, int flags, mode_t mode);
ssize_t read(int fd, void *buf, size_t size);
ssize_t write(int fd, const void *buf, size_t size); write_cuk
int close(int fd);
```

不同版本的 Android 系统实现有所不同，在 Android 7.0 之后，我们还需要替换下面这三个方法。

```
open64
__read_chk
__write_chk
```



爱奇艺开源的 xhook 可以 hook .so 中的函数，Github 上有相关信息，有兴趣的可以去看源码。

先上一张图，再看一下使用方式：

![https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/ba36f8e259427bde06bc44861905c63c.png?raw=true](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/ba36f8e259427bde06bc44861905c63c.png?raw=true)

#### open

> iocanary::Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_doHook

```c++
JNIEXPORT jboolean JNICALL
    Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_doHook(JNIEnv *env, jclass type) {

    for (int i = 0; i < TARGET_MODULE_COUNT; ++i) {
        const char* so_name = TARGET_MODULES[i];

        void* soinfo = xhook_elf_open(so_name);

        // 从 .so 里面 找到 open 函数，将open函数的地址指向 ProxyOpen，原来的函数地址保存到 original_open
        // void** 相当于一个泛型
        xhook_hook_symbol(soinfo, "open", (void*)ProxyOpen, (void**)&original_open);
        ...

        xhook_elf_close(soinfo);
    }

    return JNI_TRUE;
}
```

可以看到，使用还是挺简单的，先打开 .so 文件，然后传递需要hook的函数名，以及俩个函数指针就好了，我们看看 ProxyOpen 函数：

> iocanary::ProxyOpen

```c++
        int ProxyOpen(const char *pathname, int flags, mode_t mode) {
            if(!IsMainThread()) {
                return original_open(pathname, flags, mode);
            }

            int ret = original_open(pathname, flags, mode);

            if (ret != -1) {
                DoProxyOpenLogic(pathname, flags, mode, ret);
            }

            return ret;
        }
```

这里，它只监测了**主线程的IO**（其他的代理方法都有这个逻辑，可能是其他线程有bug还是什么）。然后调用原来的 open 函数打开文件，拿到返回的文件描述符后，执行插入的代理逻辑：

> iocanary::DoProxyOpenLogic

```c++
        static void DoProxyOpenLogic(const char *pathname, int flags, mode_t mode, int ret) {
            JNIEnv* env = NULL;
            kJvm->GetEnv((void**)&env, JNI_VERSION_1_6);
            if (env == NULL || !kInitSuc) {
                __android_log_print(ANDROID_LOG_ERROR, kTag, "ProxyOpen env null or kInitSuc:%d", kInitSuc);
            } else {
                // 获取 java 层的 JavaContext 对象
                jobject java_context_obj = env->CallStaticObjectMethod(kJavaBridgeClass, kMethodIDGetJavaContext);
                if (NULL == java_context_obj) {
                    return;
                }

                // 获取java层堆栈信息
                jstring j_stack = (jstring) env->GetObjectField(java_context_obj, kFieldIDStack);
                // 线程名
                jstring j_thread_name = (jstring) env->GetObjectField(java_context_obj, kFieldIDThreadName);

                char* thread_name = jstringToChars(env, j_thread_name);
                char* stack = jstringToChars(env, j_stack);
                // 创建 C++ 层的 JavaContex 对象
                JavaContext java_context(GetCurrentThreadId(), thread_name == NULL ? "" : thread_name, stack == NULL ? "" : stack);
                free(stack);
                free(thread_name);

                iocanary::IOCanary::Get().OnOpen(pathname, flags, mode, ret, java_context);

                env->DeleteLocalRef(java_context_obj);
                env->DeleteLocalRef(j_stack);
                env->DeleteLocalRef(j_thread_name);
            }
        }
```

kJavaBridgeClass 等变量在 JNI_OnLoad 的时候就已经初始化好了。这些都是用来创建 java 层的对象的。

该函数就是创建出了一些必要的参数，然后调用了 OnOpen 方法，最终调用到

> IOInfoCollector::OnOpen

```c++

    void IOInfoCollector::OnOpen(const char *pathname, int flags, mode_t mode, int open_ret,
                                 const JavaContext &java_context) {
        //__android_log_print(ANDROID_LOG_DEBUG, kTag, "OnOpen fd:%d; path:%s", open_ret, pathname);

        // 文件打开失败返回 -1，成功返回文件描述符，太奇葩了
        if (open_ret == -1) {
            return;
        }

        // 这里刚开始会觉得很奇怪啊，为啥要使用 返回值作为key？？？因为 open_ret 是文件的描述符
        // 文件已经被记录了
        if (info_map_.find(open_ret) != info_map_.end()) {
            //__android_log_print(ANDROID_LOG_WARN, kTag, "OnOpen fd:%d already in info_map_", open_ret);
            return;
        }

        // make_shared 会在堆上创建对象，返回一个智能指针
        std::shared_ptr<IOInfo> info = std::make_shared<IOInfo>(pathname, java_context);
        // 记录
        info_map_.insert(std::make_pair(open_ret, info));
    }

```

可以看到，每次打开一个文件的时候，都会将其记录到 map 中，key 是文件的路径，value 是一个IOInfo类型的指针。

#### read/write

我们使用同样的方式，可以追踪到 read/write 的相关hook逻辑：

> IOInfoCollector::CountRWInfo

```c++
    void
    IOInfoCollector::CountRWInfo(int fd, const FileOpType &fileOpType, long op_size, long rw_cost) {
        if (info_map_.find(fd) == info_map_.end()) {
            return;
        }

        const int64_t now = GetSysTimeMicros();

        info_map_[fd]->op_cnt_++;
        info_map_[fd]->op_size_ += op_size;
        info_map_[fd]->rw_cost_us_ += rw_cost;

        // 记录单次最大的读写时间
        if (rw_cost > info_map_[fd]->max_once_rw_cost_time_μs_) {
            info_map_[fd]->max_once_rw_cost_time_μs_ = rw_cost;
        }

        // 如果连续读写间隔小于   kContinualThreshold（8ms）
        if (info_map_[fd]->last_rw_time_μs_ > 0 &&
            (now - info_map_[fd]->last_rw_time_μs_) < kContinualThreshold) {
            // 本次连续读写时间
            info_map_[fd]->current_continual_rw_time_μs_ += rw_cost;
        } else {
            info_map_[fd]->current_continual_rw_time_μs_ = rw_cost;
        }
        if (info_map_[fd]->current_continual_rw_time_μs_ >
            info_map_[fd]->max_continual_rw_cost_time_μs_) {
            info_map_[fd]->max_continual_rw_cost_time_μs_ = info_map_[fd]->current_continual_rw_time_μs_;
        }
        // 记录读写时刻
        info_map_[fd]->last_rw_time_μs_ = now;

        // 记录操作的 buffer 大小，这里是记录的最大值
        if (info_map_[fd]->buffer_size_ < op_size) {
            info_map_[fd]->buffer_size_ = op_size;
        }

        // 如果对一个文件又读又写，记录第一次的读写类型？？？
        if (info_map_[fd]->op_type_ == FileOpType::kInit) {
            info_map_[fd]->op_type_ = fileOpType;
        }
    }
```

这里面是记录了一些读写的数据。

#### close

> IOInfoCollector::OnClose

```c++
    /**
     * 返回 fd 对应的文件信息，在 info_map_ 中
     */
    std::shared_ptr<IOInfo> IOInfoCollector::OnClose(int fd, int close_ret) {

        if (info_map_.find(fd) == info_map_.end()) {
            //__android_log_print(ANDROID_LOG_DEBUG, kTag, "OnClose fd:%d not in info_map_", fd);
            return nullptr;
        }

        // 从打开到关闭的耗时
        info_map_[fd]->total_cost_μs_ = GetSysTimeMicros() - info_map_[fd]->start_time_μs_;
        // 文件大小
        info_map_[fd]->file_size_ = GetFileSize(info_map_[fd]->path_.c_str());
        // 其他信息在读写时记录了
        std::shared_ptr<IOInfo> info = info_map_[fd];
        info_map_.erase(fd);

        return info;
    }
```

这里也记录了一些信息，注意，这里返回了指针，然后 map 里面的键值对被擦除了。

### 流程

经过上面对几个函数的hook，我们可以获取到读写文件时的详细信息。拿到这些信息之后，我们就可以进行相应的处理，判断该IO行为是否正常。

我们从 plugin 的 start 开始分析整个流程。

> com.tencent.matrix.iocanary.core.IOCanaryCore#initDetectorsAndHookers

```java
    private void initDetectorsAndHookers(IOConfig ioConfig) {
        assert ioConfig != null;

        if (ioConfig.isDetectFileIOInMainThread()
            || ioConfig.isDetectFileIOBufferTooSmall()
            || ioConfig.isDetectFileIORepeatReadSameFile()) {
            IOCanaryJniBridge.install(ioConfig, this);
        }

        // 监测 io 是否关闭了
        //if only detect io closeable leak use CloseGuardHooker is Better
        if (ioConfig.isDetectIOClosableLeak()) {
            mCloseGuardHooker = new CloseGuardHooker(this);
            mCloseGuardHooker.hook();
        }
    }
```

我们这里只分析第一个 if 里面的东西，第二个 if 里面的东西最后分析。

在第一个 if 里面，就是判断了是否开启某些监测，开启了才调用相应代码。

install 函数里面：

```java
    public static void install(IOConfig config, OnJniIssuePublishListener listener) {
        MatrixLog.v(TAG, "install sIsTryInstall:%b", sIsTryInstall);
        if (sIsTryInstall) {
            return;
        }

        // 加载 .so 文件
        //load lib
        if (!loadJni()) {
            MatrixLog.e(TAG, "install loadJni failed");
            return;
        }

        //set listener
        sOnIssuePublishListener = listener;

        try {
            //set config
            if (config != null) {
                if (config.isDetectFileIOInMainThread()) {
                    // 调用 native 方法
                    enableDetector(DetectorType.MAIN_THREAD_IO);
                    // ms to μs
                    setConfig(ConfigKey.MAIN_THREAD_THRESHOLD, config.getFileMainThreadTriggerThreshold() * 1000L);
                }

                if (config.isDetectFileIOBufferTooSmall()) {
                    enableDetector(DetectorType.SMALL_BUFFER);
                    setConfig(ConfigKey.SMALL_BUFFER_THRESHOLD, config.getFileBufferSmallThreshold());
                }

                if (config.isDetectFileIORepeatReadSameFile()) {
                    enableDetector(DetectorType.REPEAT_READ);
                    setConfig(ConfigKey.REPEAT_READ_THRESHOLD, config.getFileRepeatReadThreshold());
                }
            }

            //hook
            doHook();

            sIsTryInstall = true;
        } catch (Error e) {
            MatrixLog.printErrStackTrace(TAG, e, "call jni method error");
        }
    }
```

该函数做了这些事：

- 先加载对应的 .so 库

- 根据 config 设置对应的监听，enableDetector 是一个native 代码，最终会调用到

```c++
    void IOCanary::RegisterDetector(DetectorType type) {
        switch (type) {
            case DetectorType::kDetectorMainThreadIO:
                // 添加到detector容器集合
                detectors_.push_back(new FileIOMainThreadDetector());
                break;
            ...
        }
    }
```

- 进行 hook，这里的 hook 就是上面说的 hook 函数部分了。

这里，我们知道，监听设置了，但是监听什么时候被调用呢？我们看 IOCanary 的构造函数：

```c++
    IOCanary::IOCanary() {
        exit_ = false;
        // 创建一个线程，detect 是被调用的函数，其参数是 this（隐式参数）
        std::thread detect_thread(&IOCanary::Detect, this);
        // 分离线程，线程单独运行
        detect_thread.detach();
    }
```

这里，它开启了一个线程，该线程运行的是 Detect 函数。

```c++
    void IOCanary::Detect() {
        std::vector<Issue> published_issues;
        // 只要将 new 运算符返回的指针 p 交给一个 shared_ptr 对象“托管”，
        // 就不必担心在哪里写delete p语句——实际上根本不需要编写这条语句，
        // 托管 p 的 shared_ptr 对象在消亡时会自动执行delete p。
        // 有点 java 的味道了
        std::shared_ptr<IOInfo> file_io_info;
        while (true) {
            published_issues.clear();

            int ret = TakeFileIOInfo(file_io_info);

            // exit_ 为0， 就跳出了
            if (ret != 0) {
                break;
            }

            // detectors_ 是监听集合
            // 具体可见 IOCanary::RegisterDetector 方法
            // 在对一个文件操作完毕之后，open -> read/write -> close，才会回调
            for (auto detector : detectors_) {
                detector->Detect(env_, *file_io_info, published_issues);
            }

            // 调用回调方法
            // 该监听是在 iocanary::JNI_OnLoad 里面设置的
            if (issued_callback_ && !published_issues.empty()) {
                issued_callback_(published_issues);
            }

            // 释放指针
            file_io_info = nullptr;
        }
    }
```

这里面是一个生产者-消费者模式，Detect 扮演的是消费者的角色，它调用 TakeFileIOInfo 函数获取一个 IOInfo 的指针，然后对它进行处理，我们可以看到，在 for 里面，它将这个指针传递到了我们之前设置的监听函数的回调参数里面。这样，在监听的回调函数里面，我们就可以拿到本次IO的各种信息了。

这里是生产者-消费者模式，所以我们还要搞清楚，谁是生产者。搜索，queue_cv_ 变量，我们可以发现如下代码：

```c++
    void IOCanary::OnClose(int fd, int close_ret) {
        std::shared_ptr<IOInfo> info = collector_.OnClose(fd, close_ret);
        if (info == nullptr) {
            return;
        }

        OfferFileIOInfo(info);
    }

    void IOCanary::OfferFileIOInfo(std::shared_ptr<IOInfo> file_io_info) {
        std::unique_lock<std::mutex> lock(queue_mutex_);
        queue_.push_back(file_io_info);
        // 通知等待线程，添加进去了一个元素
        queue_cv_.notify_one();
        lock.unlock();
    }

    /**
     * 从 queue_ 里面获取队头的 file_io_info 对象
     */
    int IOCanary::TakeFileIOInfo(std::shared_ptr<IOInfo> &file_io_info) {
        // std::unique_lock对象以独占所有权的方式(unique owership)管理mutex对象的上锁和解锁操作，
        // 即在unique_lock对象的声明周期内，它所管理的锁对象会一直保持上锁状态；
        // 而unique_lock的生命周期结束之后，它所管理的锁对象会被解锁。
        std::unique_lock<std::mutex> lock(queue_mutex_);

        // 如果队列为空，那就一直等待，开起来像是一个生产者-消费者模式
        while (queue_.empty()) {
            // wait 会释放锁
            queue_cv_.wait(lock);
            if (exit_) {
                return -1;
            }
        }

        // 因为参数是引用，所以这里会改变传递进来的实参值
        file_io_info = queue_.front();
        // pop 居然没有返回 pop 出来的值，难怪要多加一句
        queue_.pop_front();
        return 0;
    }
```

OnClose 是我们的 hook 函数的代码，它在文件关闭的时候会被调用，所以结论就是文件关闭的时候，会返回（上面提到过）一个 IOInfo 的指针，然后将它放到队列中，通知等待线程开始执行，最后就回调到了我们的监听代码中。

### 主线程IO

我们看看主线程IO的监听代码：

```c++
    void FileIOMainThreadDetector::Detect(const IOCanaryEnv &env, const IOInfo &file_io_info,
                                          std::vector<Issue>& issues) {

        if (GetMainThreadId() == file_io_info.java_context_.thread_id_) {
            int type = 0;
            // 最大读写时间超过 13ms
            if (file_io_info.max_once_rw_cost_time_μs_ > IOCanaryEnv::kPossibleNegativeThreshold) {
                type = 1;
            }
            // 一次连续读写时间超过 500ms
            if(file_io_info.max_continual_rw_cost_time_μs_ > env.GetMainThreadThreshold()) {
                type |= 2;
            }

            if (type != 0) {
                Issue issue(kType, file_io_info);
                issue.repeat_read_cnt_ = type;  //use repeat to record type
                PublishIssue(issue, issues);
            }
        }
    }
```

没啥好说的，就是判断一些阈值而已。如果你想监测别的数据，可以自己设置一些监听。

### 重复读

这个监听我就不贴代码了，感觉它现在有点鸡肋，我的 note 分支里面有详细注释，有兴趣可以查看。

其实，我觉得可以统计一下App在一次运行过程中，每个文件被读取的次数。反正你可以获取到每次IO的所有信息，想做什么就做什么。



### 小 Buffer 的io

我们知道，对于文件系统是以block为单位读写，对于磁盘是以page 为单位读写，看起来即使我们在应用程序上面使用很小的Buffer，在底层应该差别不大，那是不是这样呢？

实际上不是的，因为如果 buffer 过小，会导致多次无用的系统调用，write与read的次数变多，这样性能就下降了，那么应该如何选择 buffer的大小呢？可以参考文件系统的 block size 的大小来决定。一般是 4K。

那么又来了一个问题？buffer搞很大会咋样呢？实际上如果你自己测试一下的话，会发现4K往上的话，收益就开始变小了，甚至会降低。所以一般推荐 4K 以上，也不要搞太大。

看一下监测小buffer的代码，很简单：

```c++
    void FileIOSmallBufferDetector::Detect(const IOCanaryEnv &env, const IOInfo &file_io_info,
                                           std::vector<Issue> &issues) {
        //__android_log_print(ANDROID_LOG_ERROR, "FileIOSmallBufferDetector", "Detect buffer_size:%d threshold:%d op_cnt:%d rw_cost:%d",
        //                  file_io_info.buffer_size_, env.GetSmallBufferThreshold(), file_io_info.op_cnt_, file_io_info.max_continual_rw_cost_time_μs_);

        // 读写次数 > 20
        // 读写总数量 / 读写次数 < 4096
        // 最大连续读写耗时 >= 13ms
        if (file_io_info.op_cnt_ > env.kSmallBufferOpTimesThreshold
            && (file_io_info.op_size_ / file_io_info.op_cnt_) < env.GetSmallBufferThreshold()
            && file_io_info.max_continual_rw_cost_time_μs_ >= env.kPossibleNegativeThreshold) {

            PublishIssue(Issue(kType, file_io_info), issues);
        }
    }
```



### 未关闭的文件

除了上面监测主线程的各种问题之外，该库还提供了一个特殊的功能，监测未关闭的文件操作。

其大致原理是使用了 CloseGuard 类，这个类的用法如下：

```java
    private final CloseGuard mCloseGuard = CloseGuard.get();

	...

    public InputQueue() {
        mPtr = nativeInit(new WeakReference<InputQueue>(this), Looper.myQueue());

        mCloseGuard.open("dispose");
    }

	...
        
	@Override
    protected void finalize() throws Throwable {
        try {
            dispose(true);
        } finally {
            super.finalize();
        }
    }

    public void dispose() {
        dispose(false);
    }

    public void dispose(boolean finalized) {
        if (mCloseGuard != null) {
            if (finalized) {
                mCloseGuard.warnIfOpen();
            }
            mCloseGuard.close();
        }

        if (mPtr != 0) {
            nativeDispose(mPtr);
            mPtr = 0;
        }
    }
```

用法还是挺简单的，就是利用了 finalize 方法。

它先在构造方法里面，调用了 CloseGuard 的 open 方法，open方法里面记录了一个标识。假设，我们使用完该类后，没有调用该类的 dispose 方法，那么 CloseGuard  里面的标识就还存在，那么当该类被回收的时候，会触发 finalize 方法，然后执行到 CloseGuard.warnIfOpen 方法，这样就会弹出警告。

所以它的原理，其实就是利用了 finalize 来监测我们是否成对的调用了某些方法。

我们可以利用这个类，因为系统的很多类都使用了 CloseGuard ，我们开启严格模式后能监测一些问题，就是利用的这个原理。

所以，我们需要 hook 这个类，监测到未关闭的文件，就提出警告。

具体代码如下：

```java
    private boolean tryHook() {
        try {
            // hook 系统的 CloseGuard 类，该类是用于监测某些类是否正常关闭的，比如 cursor
            // 具体是先看一下 android.view.InputQueue 的源码就好了，几十行
            // 大致原理就是依赖 finalize 方法来监测是否有调用对应的  close 方法
            // 所以，我们使用反射开启这个类的功能，然后 hook 它，在里面做我们的逻辑
            Class<?> closeGuardCls = Class.forName("dalvik.system.CloseGuard");
            Class<?> closeGuardReporterCls = Class.forName("dalvik.system.CloseGuard$Reporter");
            Method methodGetReporter = closeGuardCls.getDeclaredMethod("getReporter");
            Method methodSetReporter = closeGuardCls.getDeclaredMethod("setReporter", closeGuardReporterCls);
            Method methodSetEnabled = closeGuardCls.getDeclaredMethod("setEnabled", boolean.class);

            sOriginalReporter = methodGetReporter.invoke(null);

            methodSetEnabled.invoke(null, true);

            // open matrix close guard also
            MatrixCloseGuard.setEnabled(true);

            ClassLoader classLoader = closeGuardReporterCls.getClassLoader();
            if (classLoader == null) {
                return false;
            }

            methodSetReporter.invoke(null, Proxy.newProxyInstance(classLoader,
                new Class<?>[]{closeGuardReporterCls},
                    // hook 类
                new IOCloseLeakDetector(issueListener, sOriginalReporter)));

            return true;
        } catch (Throwable e) {
            MatrixLog.e(TAG, "tryHook exp=%s", e);
        }

        return false;
    }
```

这样，我们在 IOCloseLeakDetector 里面，就可以处理问题了：

```java
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        MatrixLog.i(TAG, "invoke method: %s", method.getName());
        if (method.getName().equals("report")) {
            ...

            if (isPublished(stackKey)) {
                ...
            } else {
                // 这里处理 issue
                Issue ioIssue = new Issue(SharePluginInfo.IssueType.ISSUE_IO_CLOSABLE_LEAK);
                ioIssue.setKey(stackKey);
                JSONObject content = new JSONObject();
                try {
                    content.put(SharePluginInfo.ISSUE_FILE_STACK, stackKey);
                } catch (JSONException e) {
//                e.printStackTrace();
                    MatrixLog.e(TAG, "json content error: %s", e);
                }
                ioIssue.setContent(content);
                publishIssue(ioIssue);
                MatrixLog.i(TAG, "close leak issue publish, key:%s", stackKey);
                markPublished(stackKey);
            }


            return null;
        }
        return method.invoke(originalReporter, args);
    }
```

这里，就触发了 publishIssue 方法。

有一点需要注意，这里是开启了 CloseGuard，所以使用了 CloseGuard 的类都可以监测到，比如：

```java
        try {
            Class<?> aClass = Class.forName("android.view.InputQueue");
            Object o = aClass.newInstance();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }


        //need to trigger gc to detect leak
        new Thread(new Runnable() {
            @Override
            public void run() {
                Runtime.getRuntime().gc();
                Runtime.getRuntime().runFinalization();
                Runtime.getRuntime().gc();
            }
        }).start();

```

这里我们使用了，InputQueue，它里面使用了 CloseGuard，而我们没有调用其 dispose 方法，所以这个函数会被检测出来使用有问题。



### 处理issue

上面检测出来的问题，最终都会汇总到 com.tencent.matrix.plugin.Plugin#onDetectIssue 这个方法里面，然后对 Issue 进行处理，其实就是转为json，用于可视化展示。

```java
    @Override
    public void onDetectIssue(Issue issue) {
        if (issue.getTag() == null) {
            // set default tag
            issue.setTag(getTag());
        }
        issue.setPlugin(this);
        JSONObject content = issue.getContent();
        // add tag and type for default
        try {
            if (issue.getTag() != null) {
                content.put(Issue.ISSUE_REPORT_TAG, issue.getTag());
            }
            if (issue.getType() != 0) {
                content.put(Issue.ISSUE_REPORT_TYPE, issue.getType());
            }
            content.put(Issue.ISSUE_REPORT_PROCESS, MatrixUtil.getProcessName(application));
            content.put(Issue.ISSUE_REPORT_TIME, System.currentTimeMillis());

        } catch (JSONException e) {
            MatrixLog.e(TAG, "json error", e);
        }

        // 这里的回调是整个 matrix 的一个回调，我们可以在这个回调里面进行个性化处理，比如上报后台之类的
        //MatrixLog.e(TAG, "detect issue:%s", issue);
        pluginListener.onReportIssue(issue);
    }
```



### 参考文档

https://zhuanlan.zhihu.com/p/36426206