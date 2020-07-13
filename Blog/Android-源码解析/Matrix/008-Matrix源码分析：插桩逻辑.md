---
title: 008-Matrix源码分析：插桩逻辑
date: 2020-7-12
categories: Matrix
---

前面几篇基本上将 matrix-trace-canary 的功能分析完毕了，只剩下插桩部分没有说，这里就来分析一下。

先上个图，了解一下流程。



正式开始之前，需要了解的预备知识：Transform。不清楚的这个的，看着会很蛋疼，天赋异禀除外。

从插件的配置开始，要使用这个插件，我们需要引用它：

```groovy
classpath ("com.tencent:matrix:1.0.1"){changing = true}
-------------------------------------------------------
apply plugin: 'com.tencent.matrix-plugin'
matrix {
    trace {
        enable = true
        baseMethodMapFile = "${project.projectDir}/matrixTrace/methodMapping.txt"
        blackListFile = "${project.projectDir}/matrixTrace/blackMethodList.txt"
    }
}
```

trace 里面有几个选项可以配置，就不说了，就是定义了一个 Extension。

```java
public class MatrixTraceExtension {
    boolean enable;
    String baseMethodMapFile;
    String blackListFile;
    String customDexTransformName;
}
```

然后，插件运行的时候，会读取这里值。

我们从源头看起：

```groovy
class MatrixPlugin implements Plugin<Project> {
    private static final String TAG = "Matrix.MatrixPlugin"

    @Override
    void apply(Project project) {

        ...
        // 创建 Extension
        project.matrix.extensions.create("trace", MatrixTraceExtension)

        project.afterEvaluate {
            def android = project.extensions.android
            def configuration = project.matrix
            android.applicationVariants.all { variant ->

                if (configuration.trace.enable) {
                    // trace 处理
                    // 读取 Extension 配置的值
                    com.tencent.matrix.trace.transform.MatrixTraceTransform.inject(project, configuration.trace, variant.getVariantData().getScope())
                }

                ...

            }
        }
    }
}
```

直接是调用了`com.tencent.matrix.trace.transform.MatrixTraceTransform#inject` 方法。

`project.afterEvaluate`算是一个回调，它表示所有的模块都已经配置完了，可以准备执行task了。这个时机可以 Hook 我们想要的 transform。

inject 方法较长，只展示重要的一段：

```java
    public static void inject(Project project, MatrixTraceExtension extension, VariantScope variantScope) {

        ...

        try {
            String[] hardTask = getTransformTaskName(extension.getCustomDexTransformName(), variant.getName());
            for (Task task : project.getTasks()) {
                for (String str : hardTask) {
                    // 找到指定的任务名
                    if (task.getName().equalsIgnoreCase(str) && task instanceof TransformTask) {
                        TransformTask transformTask = (TransformTask) task;
                        Log.i(TAG, "successfully inject task:" + transformTask.getName());
                        Field field = TransformTask.class.getDeclaredField("transform");
                        field.setAccessible(true);
                        // 替换为自己的 MatrixTraceTransform，对该task进行增强
                        field.set(task, new MatrixTraceTransform(config, transformTask.getTransform()));
                        break;
                    }
                }
            }
        } catch (Exception e) {
            Log.e(TAG, e.toString());
        }

    }
```

这里就是做了一件事：将指定的 transform 替换为我们自己的 transform。

extension.getCustomDexTransformName()， 说明我们自己也可以配置想要 hook 的 transform，因为 gradle 版本问题，transform 的名字会不一样。默认的两个 transform 是： transformClassesWithDexBuilderFor 与 transformClassesWithDexFor，都是在打 dex 的时候。

### MatrixTraceTransform

它只处理 CLASS，整个工程的，支持增量。

看它 transform 的逻辑，可以先大致猜猜：

```java
    @Override
    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation);
		...

            doTransform(transformInvocation); // hack
		...
        // 进行原来的处理逻辑
        origTransform.transform(transformInvocation);

    }
```

就是插入了一段自己的逻辑。

doTransform 分为3步，我们一步一步介绍。

#### 第一步：

```java
        futures.add(executor.submit(new ParseMappingTask(mappingCollector, collectedMethodMap, methodId)));

        // 储存 class 输入输出关系的
        Map<File, File> dirInputOutMap = new ConcurrentHashMap<>();
        // 储存 jar 输入输出关系的
        Map<File, File> jarInputOutMap = new ConcurrentHashMap<>();
        Collection<TransformInput> inputs = transformInvocation.getInputs();


        // 处理输入
        // 主要是将输入类的字段替换掉，替换到指定的输出位置
        // 里面做了增量的处理
        for (TransformInput input : inputs) {

            for (DirectoryInput directoryInput : input.getDirectoryInputs()) {
                futures.add(executor.submit(new CollectDirectoryInputTask(dirInputOutMap, directoryInput, isIncremental)));
            }

            for (JarInput inputJar : input.getJarInputs()) {
                futures.add(executor.submit(new CollectJarInputTask(inputJar, isIncremental, jarInputOutMap, dirInputOutMap)));
            }
        }
```

创建 ParseMappingTask，读取 mapping 文件，因为这个时候 class 已经被混淆了，之所以选在混淆后，是因为避免插桩导致某些编译器优化失效，等编译器优化完了再插桩。读取 mapping 文件有几个用处，第一，需要输出某些信息，肯定不能输出混淆后的class信息。第二，配置文件的类是没有混淆过的，读取进来需要能够转换为混淆后的，才能处理。

创建了两个 map，储存 class jar 文件的输入输出位置。

创建 CollectDirectoryInputTask，收集 class 文件到 map。

创建 CollectJarInputTask，收集 jar 文件到 map。

CollectDirectoryInputTask 与  CollectJarInputTask 里面还用到了反射，更改了其输出目录到 build/output/traceClassout。所以我们可以在这里看到插桩后的类。这两个类就做了这些事，就不贴代码了。

后面就是调用 future 的 get 方法，等待这里 task 执行完成，再进行下一步。

#### 第二步：

```java
        MethodCollector methodCollector = new MethodCollector(executor, mappingCollector, methodId, config, collectedMethodMap);
        methodCollector.collect(dirInputOutMap.keySet(), jarInputOutMap.keySet());
```

```java
    public void collect(Set<File> srcFolderList, Set<File> dependencyJarList) throws ExecutionException, InterruptedException {
        List<Future> futures = new LinkedList<>();

        for (File srcFile : srcFolderList) {
            // 将 文件/目录下 class ，全部放到 list 中
            ArrayList<File> classFileList = new ArrayList<>();
            if (srcFile.isDirectory()) {
                listClassFiles(classFileList, srcFile);
            } else {
                classFileList.add(srcFile);
            }

            // 每个 class 都分配一个 CollectSrcTask
            for (File classFile : classFileList) {
                futures.add(executor.submit(new CollectSrcTask(classFile)));
            }
        }

        // 每个 jar 分配一个 CollectJarTask
        for (File jarFile : dependencyJarList) {
            futures.add(executor.submit(new CollectJarTask(jarFile)));
        }

        // 等待任务完成
        for (Future future : futures) {
            future.get();
        }
        futures.clear();

        futures.add(executor.submit(new Runnable() {
            @Override
            public void run() {
                // 将被忽略的 方法名 存入 ignoreMethodMapping.txt 中
                saveIgnoreCollectedMethod(mappingCollector);
            }
        }));

        futures.add(executor.submit(new Runnable() {
            @Override
            public void run() {
                // 将被插桩的 方法名 存入 methodMapping.txt 中
                saveCollectedMethod(mappingCollector);
            }
        }));

        for (Future future : futures) {
            future.get();
        }
        futures.clear();

    }
```

collect 方法里面又开了两个任务：

CollectSrcTask 

CollectJarTask

它们的逻辑差不多，就介绍一个：

```java
        @Override
        public void run() {
            ...
                is = new FileInputStream(classFile);
                // ASM 的使用
                // 访问者模式，就是将对数据结构访问的操作分离出去
                // 代价就是需要将数据结构本身传递进来
                ClassReader classReader = new ClassReader(is);
                // 修改字节码，有时候需要改动本地变量数与stack大小，自己计算麻烦，可以直接使用这个自动计算
                ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
                // ASM5 api版本
                ClassVisitor visitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
                classReader.accept(visitor, 0);
             ...

        }
```

最后走到，TraceClassAdapter：

```java
        @Override
        public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
            super.visit(version, access, name, signature, superName, interfaces);
            this.className = name;
            // 是否抽象类
            if ((access & Opcodes.ACC_ABSTRACT) > 0 || (access & Opcodes.ACC_INTERFACE) > 0) {
                this.isABSClass = true;
            }
            // 保存父类，便于分析继承关系
            collectedClassExtendMap.put(className, superName);
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String desc,
                                         String signature, String[] exceptions) {
            // 跳过抽象类，接口
            if (isABSClass) {
                return super.visitMethod(access, name, desc, signature, exceptions);
            } else {
                if (!hasWindowFocusMethod) {
                    // 是否有 onWindowFocusChanged 方法，针对 activity 的
                    hasWindowFocusMethod = isWindowFocusChangeMethod(name, desc);
                }
                return new CollectMethodNode(className, access, name, desc, signature, exceptions);
            }
        }
```

这里面没有主要逻辑，主要逻辑在 CollectMethodNode：

```java
        @Override
        public void visitEnd() {
            super.visitEnd();
            TraceMethod traceMethod = TraceMethod.create(0, access, className, name, desc);

            // 是否构造方法
            if ("<init>".equals(name)) {
                isConstructor = true;
            }

            // 判断类是否 被配置在了 黑名单中
            boolean isNeedTrace = isNeedTrace(configuration, traceMethod.className, mappingCollector);
            // filter simple methods
            // 跳过 空方法，get/set 方法，没有调用其他方法的方法，可以理解为叶子方法
            if ((isEmptyMethod() || isGetSetMethod() || isSingleMethod())
                    && isNeedTrace) {
                ignoreCount.incrementAndGet();
                // 存入 ignore map
                collectedIgnoreMethodMap.put(traceMethod.getMethodName(), traceMethod);
                return;
            }

            // 不在黑名单中
            if (isNeedTrace && !collectedMethodMap.containsKey(traceMethod.getMethodName())) {
                traceMethod.id = methodId.incrementAndGet();
                // 存入 map
                collectedMethodMap.put(traceMethod.getMethodName(), traceMethod);
                incrementCount.incrementAndGet();
            } else if (!isNeedTrace && !collectedIgnoreMethodMap.containsKey(traceMethod.className)) {
                ignoreCount.incrementAndGet();
                // 存入 ignore map
                collectedIgnoreMethodMap.put(traceMethod.getMethodName(), traceMethod);
            }

        }
```

可以看到，最终是将 class 中满足条件的方法，都存入到了 collectedMethodMap，忽略的方法存入了 collectedIgnoreMethodMap。

#### 第三步：

```java
        MethodTracer methodTracer = new MethodTracer(executor, mappingCollector, config, methodCollector.getCollectedMethodMap(), methodCollector.getCollectedClassExtendMap());
        methodTracer.trace(dirInputOutMap, jarInputOutMap);
```

调用 trace 方法，trace 方法调用层次较深，最终会调用到 TraceClassAdapter，这个 TraceClassAdapter 与 第二步的 TraceClassAdapter 逻辑差不多，有一点点不一样，主要是它们关系的逻辑不同，这个 TraceClassAdapter 也是没有主要逻辑，主要逻辑在 TraceMethodAdapter 中：

```java
        @Override
        protected void onMethodEnter() {
            TraceMethod traceMethod = collectedMethodMap.get(methodName);
            if (traceMethod != null) {
                traceMethodCount.incrementAndGet();
                mv.visitLdcInsn(traceMethod.id);
                // 插入 i 方法
                mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "i", "(I)V", false);
            }
        }
        @Override
        protected void onMethodExit(int opcode) {
            TraceMethod traceMethod = collectedMethodMap.get(methodName);
            if (traceMethod != null) {
                // 如果该方法是 onWindowFocusChanged 方法
                // 还需要插桩 at 方法
                if (hasWindowFocusMethod && isActivityOrSubClass && isNeedTrace) {
                    TraceMethod windowFocusChangeMethod = TraceMethod.create(-1, Opcodes.ACC_PUBLIC, className,
                            TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS);
                    if (windowFocusChangeMethod.equals(traceMethod)) {
                        traceWindowFocusChangeMethod(mv, className);
                    }
                }


                // 插入 o 方法
                traceMethodCount.incrementAndGet();
                mv.visitLdcInsn(traceMethod.id);
                mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "o", "(I)V", false);
            }
        }
```

这样，就完成了插桩了。

更加详细的注释，我放在了 GitHub 上：

https://github.com/aprz512/matrix/tree/note

注意，这是 note 分支。