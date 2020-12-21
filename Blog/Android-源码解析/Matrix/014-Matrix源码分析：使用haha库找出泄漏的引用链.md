---
title: 014-Matrix源码分析：使用haha库找出泄漏的引用链
index_img: /cover/18.jpg
banner_img: /cover/top.jpg
date: 2020-8-18
categories: Matrix
---

听说现在 LeakCanary 改用 shark 这个库来分析 hprof 了。

前面讲到，当检测到 Activity 泄漏之后，会将裁剪之后的 hprof 文件打包，然后储存到某个位置（这个过程自行实现），这里假设我们储存到了app的目录中，然后我们导出这个文件，使用 matrix-resource-canary-analyzer 来分析这个压缩文件。

matrix-resource-canary-analyzer 是一个命令行工具，我们从 main 方法入手，这里同样省略到命令参数的处理过程。

首先，肯定是要将压缩文件里面的 hprof 文件释放出来：

> com.tencent.matrix.resource.analyzer.CLIMain#doAnalyze

```java
// 将裁剪后的 hprof copy 到一个临时文件
// We would extract hprof entry into a temporary file.
tempHprofFile = new File(new File("").getAbsoluteFile(), "temp_" + System.currentTimeMillis() + ".hprof");
StreamUtil.extractZipEntry(zf, hprofEntry, tempHprofFile);
```

接下来，就需要使用 haha 来分析这个文件了。在开始之前，我们先说一个预备知识，如果你看过 LeakCanary 的部分源码，那么你应该知道这个类`AndroidExcludedRefs`，它的作用就是用来排除一些由  Android Sdk  自身（或者制造商）引起的一些泄漏。

我们看一个例子：

> com.tencent.matrix.resource.analyzer.model.AndroidExcludedRefs

```java
    ACTIVITY_CLIENT_RECORD__NEXT_IDLE() {
        @Override
        boolean accept(int sdkVersion, String manufacturer) {
            // 需要满足的条件
            return sdkVersion >= VersionCodes.KITKAT && sdkVersion <= VersionCodes.LOLLIPOP;
        }

        @Override
        void add(ExcludedRefs.Builder excluded) {
            excluded.instanceField("android.app.ActivityThread$ActivityClientRecord", "nextIdle")
                    .reason("Android AOSP sometimes keeps a reference to a destroyed activity as a"
                         + " nextIdle client record in the android.app.ActivityThread.mActivities map."
                         + " Not sure what's going on there, input welcome.");
        }
    },
```

可以看到，`ActivityThread$ActivityClientRecord` 这个字段有可能会导致误报，reason 里面是原因。还有些其他的，可以自行查看，对后面分析源码有较大帮助。

我们现在开始分析 hprof 文件：

> com.tencent.matrix.resource.analyzer.CLIMain#analyzeAndStoreResult

```java
        final HeapSnapshot heapSnapshot = new HeapSnapshot(hprofFile);
        // 添加需要排除的引用，主要是由Android的SDK引起的，在AndroidExcludedRefs这个类中
        final ExcludedRefs excludedRefs = AndroidExcludedRefs.createAppDefaults(sdkVersion, manufacturer).build();
        // 拿到泄漏的分析结果，根据 leakedActivityKey 找到了对象，就说明泄漏了，则返回引用链
        // 没有找到对象（或者排除了），则返回 com.tencent.matrix.resource.analyzer.model.ActivityLeakResult.noLeak
        final ActivityLeakResult activityLeakResult
                = new ActivityLeakAnalyzer(leakedActivityKey, excludedRefs).analyze(heapSnapshot);
```

就这几行代码，我们就拿到了 activity 的引用链（如果真的泄漏了，会在 hprof 里面找到该实例对象）。所有的分析代码，都是在 ActivityLeakAnalyzer 的 analyze 方法中，我们追踪下去，会发现下面这个方法：

> com.tencent.matrix.resource.analyzer.utils.ShortestPathFinder#findPath(com.squareup.haha.perflib.Snapshot, java.util.Collection<com.squareup.haha.perflib.Instance>)

ShortestPathFinder 就是专门用来分析引用链的，对象的引用关系与图是一样的，所以里面用到的分析方法就是广度遍历。

### 广度优先遍历-入队 gc roots

我们看一下大致的逻辑，由于是遍历，肯定需要先入队一些元素，这里肯定就是 gcRoots 了：

> com.tencent.matrix.resource.analyzer.utils.ShortestPathFinder#enqueueGcRoots

```java
    private void enqueueGcRoots(Snapshot snapshot) {
        for (RootObj rootObj : snapshot.getGCRoots()) {
            switch (rootObj.getRootType()) {
                case JAVA_LOCAL:
                    // Java棧幀中的局部變量
                    Instance thread = HahaSpy.allocatingThread(rootObj);
                    // 拿到线程名字
                    String threadName = threadName(thread);
                    // 如果线程在排除范围内，那么就不考虑
                    // 比如 main 线程，主线程堆栈一直在变化，所以局部变量不太可能长时间保存引用。
                    // 如果是真的泄漏，一定会有另外一条路径
                    Exclusion params = excludedRefs.threadNames.get(threadName);
                    if (params == null || !params.alwaysExclude) {
                        enqueue(params, null, rootObj, null, null);
                    }
                    break;
                case INTERNED_STRING:
                case DEBUGGER:
                case INVALID_TYPE:
                    // An object that is unreachable from any other root, but not a root itself.
                case UNREACHABLE:
                case UNKNOWN:
                    // An object that is in a queue, waiting for a finalizer to run.
                case FINALIZING:
                    break;
                case SYSTEM_CLASS:
                case VM_INTERNAL:
                    // A local variable in native code.
                case NATIVE_LOCAL:
                    // A global variable in native code.
                case NATIVE_STATIC:
                    // An object that was referenced from an active thread block.
                case THREAD_BLOCK:
                    // Everything that called the wait() or notify() methods, or that is synchronized.
                case BUSY_MONITOR:
                case NATIVE_MONITOR:
                case REFERENCE_CLEANUP:
                    // Input or output parameters in native code.
                case NATIVE_STACK:
                case JAVA_STATIC:
                    // 其他情况，直接入队列
                    enqueue(null, null, rootObj, null, null);
                    break;
                default:
                    throw new UnsupportedOperationException("Unknown root type:" + rootObj.getRootType());
            }
        }
    }
```

方法虽然有点长，但是还是挺简单的，就是将 gc roots 分为两部分了，对于 java_local 来说，有些字段是需要排除（参见AndroidExcludedRefs，只有 gc root 才这样处理），所以 Exclusion 可能会有值。其他的 gc roots ，Exclusion  都没有值，即不会排除，即使 AndroidExcludedRefs 设置了。

### 广度优先遍历-处理队列中的元素

入队之后，就需要对这个引用对象做处理了，处理这个元素的时候，还会往队列继续添加元素：

```
如果是 RootObj，那么将它引用的对象入队
如果是 ClassInstance，遍历该对象字段以及父类字段，然后入队
如果是 ClassObj，那么需要将它的静态字段入队
如果是 ArrayInstance，就将元素入队
```

具体的逻辑可以看代码，这里就贴一个 visitClassObj 的逻辑：

> com.tencent.matrix.resource.analyzer.utils.ShortestPathFinder#visitClassObj

```java
    private void visitClassObj(ReferenceNode node) {
        ClassObj classObj = (ClassObj) node.instance;
        Map<String, Exclusion> ignoredStaticFields =
                excludedRefs.staticFieldNameByClassName.get(classObj.getClassName());
        // 遍历静态字段
        for (Map.Entry<Field, Object> entry : classObj.getStaticFieldValues().entrySet()) {
            Field field = entry.getKey();
            // 不是 object（ref），就忽略
            if (field.getType() != Type.OBJECT) {
                continue;
            }
            /*
            一个Instance的field大致有这些：
            $staticOverhead 不知道是啥，猜测是静态类的大小？？？

            参考：https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/hprof/HprofHeap.cpp
            The static field-name for the synthetic object generated to account for class Static overhead.
            #define STATIC_OVERHEAD_NAME    "$staticOverhead"

            04-25 10:20:46.793 D/LeakCanary: * Class com.xiao.memoryleakexample.app.App
            04-25 10:20:46.793 D/LeakCanary: |   static $staticOverhead = byte[24]@314667009 (0x12c17001)
            04-25 10:20:46.793 D/LeakCanary: |   static sActivities = java.util.ArrayList@315492800 (0x12ce09c0)
            04-25 10:20:46.793 D/LeakCanary: |   static serialVersionUID = -920324649544707127
            04-25 10:20:46.793 D/LeakCanary: |   static $change = null
             */
            String fieldName = field.getName();
            if ("$staticOverhead".equals(fieldName)) {
                continue;
            }
            Instance child = (Instance) entry.getValue();
            boolean visit = true;
            // 排除某些静态字段
            if (ignoredStaticFields != null) {
                Exclusion params = ignoredStaticFields.get(fieldName);
                if (params != null) {
                    visit = false;
                    // 看了下AndroidExcludedRefs，现在的静态字段里面都没有设置alwaysExclude
                    if (!params.alwaysExclude) {
                        enqueue(params, node, child, fieldName, STATIC_FIELD);
                    }
                }
            }
            if (visit) {
                enqueue(null, node, child, fieldName, STATIC_FIELD);
            }
        }
    }
```

这里也是让我有点疑惑的位置，为啥要遍历 Class 对象的静态字段，是因为我们写的静态字段是属于 Class 的吗？

按照上面的规则，遍历完之后，如果有泄漏了，是肯定可以找到泄漏的对象的。当然不要忘记记录已经遍历过的对象。

### 广度优先遍历-防止重复遍历

> com.tencent.matrix.resource.analyzer.utils.ShortestPathFinder#findPath(com.squareup.haha.perflib.Snapshot, java.util.Collection<com.squareup.haha.perflib.Instance>)

```java
            // 找到了，跳出循环
            // Termination
            if (targetRefSet.contains(node.instance)) {
                results.put(node.instance, new Result(node, node.exclusion != null));
                targetRefSet.remove(node.instance);
                if (targetRefSet.isEmpty()) {
                    break;
                }
            }

            // 该节点被 visit 过了，跳过，像图的广度遍历
            if (checkSeen(node)) {
                continue;
            }
```

找到泄漏的对象之后，还需要找到引用链，这个就比较简单了，由于在入队的过程中，我们给每个对象都封装了一下：

> com.tencent.matrix.resource.analyzer.utils.ShortestPathFinder#enqueue

```java
        ReferenceNode childNode = new ReferenceNode(exclusion, child, parent, referenceName, referenceType);
```

里面记录了，child 与 parent。所以我们向上遍历 parent 直到 gc root 既可获取引用链。

> com.tencent.matrix.resource.analyzer.utils.ShortestPathFinder.Result#buildReferenceChain

```java
        public ReferenceChain buildReferenceChain() {
            List<ReferenceTraceElement> elements = new ArrayList<>();
            // We iterate from the leak to the GC root
            ReferenceNode node = new ReferenceNode(null,
                    null, referenceChainHead, null, null);
            // 不断的从 泄漏的对象 向上遍历，直到 gcRoots
            while (node != null) {
                ReferenceTraceElement element = buildReferenceTraceElement(node);
                if (element != null) {
                    elements.add(0, element);
                }
                node = node.parent;
            }
            // 就生成了一个引用链
            return new ReferenceChain(elements);
        }
```

这样，泄漏对象的引用链就找到了。

假设有两条泄漏路径的话，这里找到的是最短的那一条，因为是广度优先，所以最短的肯定先找完，然后就结束寻找了。
