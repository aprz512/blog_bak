---
title: 011-Matrix源码分析：ApkChecker的其他小工具
date: 2020-8-3
categories: Matrix
---

本篇介绍一下 ApkChecker 里面的一些其他小工具，因为都比较简单，所以就合成一篇算了。

### CountClassTask

用于计算 apk 里面所有 dex 文件包含的的 class 的数量。

#### init 方法

```java
    @Override
    public void init() throws TaskInitException {
        super.init();
        String inputPath = config.getUnzipPath();

        ...

        inputFile = new File(inputPath);
        ...

        File[] files = inputFile.listFiles();
        try {
            if (files != null) {
                for (File file : files) {
                    if (file.isFile() && file.getName().endsWith(ApkConstants.DEX_FILE_SUFFIX)) {
                        dexFileNameList.add(file.getName());
                        RandomAccessFile randomAccessFile = new RandomAccessFile(file, "rw");
                        dexFileList.add(randomAccessFile);
                    }
                }
            }
        } catch (FileNotFoundException e) {
            throw new TaskInitException(e.getMessage(), e);
        }

        // 命令支持 GROUP-BY 参数，将结果分组
        if (params.containsKey(JobConstants.PARAM_GROUP)) {
            if (JobConstants.GROUP_PACKAGE.equals(params.get(JobConstants.PARAM_GROUP))) {
                group = JobConstants.GROUP_PACKAGE;
            } else {
                Log.e(TAG, "GROUP-BY '" + params.get(JobConstants.PARAM_GROUP) + "' is not correct!");
            }
        }

    }
```

init 方法里面，就是获取了所有 .dex 文件。注意，它使用了 RandomAccessFile。为啥呢？因为它读取的是 .dex 文件，它是有一定的结构的，所以读取的时候肯定需要跳来跳去。贴一张 dex 结构图，番外篇专门介绍，其实它与 class 文件的结构很像。

![img](https://img-blog.csdn.net/20170220085323266?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2luYXRfMTgyNjg4ODE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

上图比较简单，有详细点的：

![img](https://user-gold-cdn.xitu.io/2019/5/18/16acab9179e7b6a2?imageslim)

还有更详细点的，就不贴了，贴了也看不懂。

#### call 方法

```java
    /**
     * 读取dex文件，
     * 获取dex文件中的所有类
     * 进行分组
     * 然后输出
     *
     */
    @Override
    public TaskResult call() throws TaskExecuteException {
        try {
            ...

            for (int i = 0; i < dexFileList.size(); i++) {
                RandomAccessFile dexFile = dexFileList.get(i);
                DexData dexData = new DexData(dexFile);
                dexData.load();
                dexFile.close();
                // 获取 dex 中定义的 class
                // 注意 dex 中可能会有其他 dex 中的 class 的引用，这里是不包含的
                ClassRef[] defClassRefs = dexData.getInternalReferences();
                Set<String> classNameSet = new HashSet<>();
                for (ClassRef classRef : defClassRefs) {
                    // 将 "Ljava/lang/String;" 变成 "java.lang.String",
                    String className = ApkUtil.getNormalClassName(classRef.getName());
                    if (classProguardMap.containsKey(className)) {
                        // 获取混淆前的类名
                        // classProguardMap 是解压 task 中生成的 map
                        className = classProguardMap.get(className);
                    }
                    if (className.indexOf('.') == -1) {
                        continue;
                    }
                    classNameSet.add(className);
                }
                ...
            return taskResult;
        } catch (Exception e) {
            throw new TaskExecuteException(e.getMessage(), e);
        }
    }
```

for 循环里面的 DexData 是核心类，但是需要了解 dex 文件结构才能将解，会专门起一篇留到番外。

直接看输出的json，然后理解代码：

```json
{
  "taskType": 15,
  "taskDescription": "Count classes in dex file, output results group by package name.",
  "start-time": "2020-08-03 21:34:37:539",
  "end-time": "2020-08-03 21:34:39:381",
  "total-classes": 1423,
  "groups": [
    {
      "name": "android.support.v7.widget",
      "class-count": 187
    },
    {
      "name": "android.support.v4.app",
      "class-count": 175
    },
    {
      "name": "android.support.v4.view",
      "class-count": 92
    },
    ...
  ]
}
```

这里是将所有 dex 中的 class 按照包名分组，然后统计其数量。

那么，这个有什么用处呢？我个人的想法，其实是可以生成一个历史记录，看看每个版本各个包下的类的数量变化。



### CountRTask

这个 task 是用于计算所有 R 文件的字段数量。

如果没有特意的 keep R文件的话，由于 proguard 会将 R 文件的字段内联到代码里面，所以 release 版就不会有 R 文件。当然某些情况下需要keep R 文件，那么 最好的处理是只 keep 住需要的字段。

#### call 方法

```java
for (RandomAccessFile dexFile : dexFileList) {
    DexData dexData = new DexData(dexFile);
    dexData.load();
    dexFile.close();
    ClassRef[] defClassRefs = dexData.getInternalReferences();
    // 遍历dex中的类
    for (ClassRef classRef : defClassRefs) {
        // 获取类名
        String className = ApkUtil.getNormalClassName(classRef.getName());
        if (classProguardMap.containsKey(className)) {
            // 根据混淆文件，获取未混淆类名
            className = classProguardMap.get(className);
        }
        // 去除内部类的后半部分
        // com.example.sample.R$styleable -> com.example.sample.R
        String pureClassName = getOuterClassName(className);
        // 一个光 R 是个什么鬼？？？
        // 连包名都没有，java可以运行吗？
        if (pureClassName.endsWith(".R") || "R".equals(pureClassName)) {
            // 获取该类的字段长度
            if (!classesMap.containsKey(pureClassName)) {
                classesMap.put(pureClassName, classRef.getFieldArray().length);
            } else {
                // 累加内部类
                // com.example.sample.R$string
                // com.example.sample.R$layout
                classesMap.put(pureClassName, classesMap.get(pureClassName) + classRef.getFieldArray().length);
            }
        }
    }
}
```

计算 R 文件的字段数量与计算 dex 中 class 数量是差不多的。主要是如何找到 R 文件。

输出结果如下：

```
{
  "taskType": 9,
  "taskDescription": "Count the R class.",
  "R-count": 27,
  "Field-counts": 5449,
  "start-time": "2020-08-03 22:16:14:524",
  "end-time": "2020-08-03 22:16:16:318",
  "R-classes": [
    {
      "name": "com.example.resapk.R",
      "field-count": 1839
    },
    {
      "name": "android.support.v7.appcompat.R",
      "field-count": 1577
    },
    ...
  ]
}
```



### DuplicateFileTask

找出重复的文件。这个 task 还是很有用的，特别是对于一些资源文件。apk 解压后文件并不多，所以重复文件都是针对的 res 下的图片之类的。

#### call

```java
    @Override
    public TaskResult call() throws TaskExecuteException {
        TaskResult taskResult = null;
        try {
            ...
            computeMD5(inputFile);
            ...
        } catch (Exception e) {
            throw new TaskExecuteException(e.getMessage(), e);
        }
        return taskResult;
    }
```

为文件生成 md5 值，然后比较。

```java
// 计算出文件的 md5
final String md5 = Util.byteArrayToHex(msgDigest.digest());
String filename = file.getAbsolutePath().substring(inputFile.getAbsolutePath().length() + 1);
// 获取混淆之前的名字
if (entryNameMap.containsKey(filename)) {
    filename = entryNameMap.get(filename);
}
if (!md5Map.containsKey(md5)) {
    md5Map.put(md5, new ArrayList<String>());
    // 统计文件大小
    if (entrySizeMap.containsKey(filename)) {
        // 文件重复就使用之前的数据
        fileSizeList.add(Pair.of(md5, entrySizeMap.get(filename).getFirst()));
    } else {
        fileSizeList.add(Pair.of(md5, totalRead));
    }
}
// 统计所有文件的 md5
// 一个 md5 对应一个 list
md5Map.get(md5).add(filename);
```

就是使用一个map记录所有文件的 md5，发现有重复的则添加到 list 里面。

看看输出格式：

```
{
  "taskType": 10,
  "taskDescription": "Find out the duplicated files.",
  "files": [
    {
      "md5": "02aca4b1e0a80c3cf29ebefc103a506a",
      "size": 340,
      "files": [
        "res\\drawable-xxhdpi-v4\\bg_shadow_top.png",
        "res\\drawable-xxhdpi-v4\\bg_shadow_top_copy.png"
      ]
    },
    {
      "md5": "c9e47dbb0e1927076ed7b2e1ec157be7",
      "size": 6,
      "files": [
        "META-INF\\androidx.appcompat_appcompat.version",
        "META-INF\\androidx.asynclayoutinflater_asynclayoutinflater.version",
        "META-INF\\androidx.coordinatorlayout_coordinatorlayout.version",
        "META-INF\\androidx.core_core.version",
        "META-INF\\androidx.cursoradapter_cursoradapter.version",
        "META-INF\\androidx.customview_customview.version",
        "META-INF\\androidx.documentfile_documentfile.version",
        "META-INF\\androidx.drawerlayout_drawerlayout.version",
        "META-INF\\androidx.fragment_fragment.version",
        "META-INF\\androidx.interpolator_interpolator.version",
        "META-INF\\androidx.legacy_legacy-support-core-ui.version",
        "META-INF\\androidx.legacy_legacy-support-core-utils.version",
        "META-INF\\androidx.loader_loader.version",
        "META-INF\\androidx.localbroadcastmanager_localbroadcastmanager.version",
        "META-INF\\androidx.print_print.version",
        "META-INF\\androidx.slidingpanelayout_slidingpanelayout.version",
        "META-INF\\androidx.swiperefreshlayout_swiperefreshlayout.version",
        "META-INF\\androidx.vectordrawable_vectordrawable-animated.version",
        "META-INF\\androidx.vectordrawable_vectordrawable.version",
        "META-INF\\androidx.versionedparcelable_versionedparcelable.version",
        "META-INF\\androidx.viewpager_viewpager.version"
      ]
    }
    ...
  ],
  "start-time": "2020-08-03 22:29:58:809",
  "end-time": "2020-08-03 22:29:58:910"
}
```



### FindNonAlphaPngTask

找出不带透明通道的 png 图片，因为没有透明通道，png 就是浪费，可以使用 jpg 等别的格式。

```java
    private void findNonAlphaPng(File file) throws IOException {
        if (file != null) {
            if (file.isDirectory()) {
                File[] files = file.listFiles();
                for (File tempFile : files) {
                    findNonAlphaPng(tempFile);
                }
            } else if (file.isFile() && file.getName().endsWith(ApkConstants.PNG_FILE_SUFFIX) && !file.getName().endsWith(ApkConstants.NINE_PNG)) {
                // png 文件，但是不是 .9
                // .9 图片需要透明度，所以直接跳过
                BufferedImage bufferedImage = ImageIO.read(file);
                // 如果 没有 alpha 通道
                if (bufferedImage != null && bufferedImage.getColorModel() != null && !bufferedImage.getColorModel().hasAlpha()) {
                    // 获取文件名
                    String filename = file.getAbsolutePath().substring(inputFile.getAbsolutePath().length() + 1);
                    // 获取混淆前的文件名
                    if (entryNameMap.containsKey(filename)) {
                        filename = entryNameMap.get(filename);
                    }
                    long size = file.length();
                    if (entrySizeMap.containsKey(filename)) {
                        // 获取文件大小
                        size = entrySizeMap.get(filename).getFirst();
                    }
                    // 大于阈值，记录下来
                    if (size >= downLimitSize * ApkConstants.K1024) {
                        nonAlphaPngList.add(Pair.of(filename, file.length()));
                    }
                }
            }
        }
    }
```

是一个递归函数，直接看 else if 的逻辑就好了。

我还是第一个次见到 BufferedImage 这个 API，学到了。



### ManifestAnalyzeTask

分析 manifest 文件。这里有个有意思的东西，在 eclipse 年代，我们的版本号都是写在  manifest 中的，现在写在了 gradle 中，那么问题来了，当编译之后，可以从 manifest  获取到版本号吗？

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="1"
    android:versionName="1.0"
    android:compileSdkVersion="29"
    android:compileSdkVersionCodename="10"
    package="com.example.resapk"
    platformBuildVersionCode="1"
    platformBuildVersionName="1065353216.000000">

    <uses-sdk
        android:minSdkVersion="19"
        android:targetSdkVersion="29" />
```

上面的 xml 是我从 apk 里面解压出来，拖到 as 里面展示的信息，里面是可以获取到版本号的。

而且，我们也可以在 intermediates 中看最终的 manifest 文件：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.resapk"
    android:versionCode="1"
    android:versionName="1.0" >

    <uses-sdk
        android:minSdkVersion="19"
        android:targetSdkVersion="29" />
```

解析代码其实就是一个 xml 解析器，就不贴代码了，不过有一个疑问，就是这个解析器还使用到了 arsc 文件，很奇怪，肯能与 api 的关系。

```java
    public ManifestParser(File manifestFile, File arscFile) throws IOException, AndrolibException {
        if (manifestFile != null) {
            this.manifestFile = manifestFile;
        }
        resourceParser = ApkResourceDecoder.createAXmlParser(arscFile);
    }
```

输出如下：

```
{
  "taskType": 2,
  "taskDescription": "Read package info from the AndroidManifest.xml.",
  "start-time": "2020-08-03 22:29:55:208",
  "end-time": "2020-08-03 22:29:55:217",
  "manifest": {
    "package": "com.example.resapk",
    "android:minSdkVersion": "19",
    "android:targetSdkVersion": "29",
    "android:versionCode": "1",
    "android:versionName": "1.0"
  }
}
```



### MethodCountTask

统计 apk 中的方法数量，可以分组。这个task的逻辑与 CountClassTask 很像。

```java
    private void countDex(RandomAccessFile dexFile) throws IOException {
        classInternalMethod.clear();
        classExternalMethod.clear();
        pkgInternalRefMethod.clear();
        pkgExternalMethod.clear();
        DexData dexData = new DexData(dexFile);
        dexData.load();
        // 获取 dex 中的方法，在 struct method_id_list dex_method_ids 中储存
        MethodRef[] methodRefs = dexData.getMethodRefs();
        // 获取不属于该 dex 的类
        ClassRef[] externalClassRefs = dexData.getExternalReferences();
        Map<String, String> proguardClassMap = config.getProguardClassMap();
        String className = null;
        for (ClassRef classRef : externalClassRefs) {
            className = ApkUtil.getNormalClassName(classRef.getName());
            if (proguardClassMap.containsKey(className)) {
                className = proguardClassMap.get(className);
            }
            if (className.indexOf('.') == -1) {
                continue;
            }
            // 将外部类的方法数置为0
            classExternalMethod.put(className, 0);
        }
        for (MethodRef methodRef : methodRefs) {
            // 该方法所属的类
            className = ApkUtil.getNormalClassName(methodRef.getDeclClassName());
            if (proguardClassMap.containsKey(className)) {
                className = proguardClassMap.get(className);
            }
            if (!Util.isNullOrNil(className)) {
                if (className.indexOf('.') == -1) {
                    continue;
                }
                // 将dex内部方法放到 classInternalMethod
                // 将dex外部方法方法 classExternalMethod
                if (classExternalMethod.containsKey(className)) {
                    classExternalMethod.put(className, classExternalMethod.get(className) + 1);
                } else if (classInternalMethod.containsKey(className)) {
                    classInternalMethod.put(className, classInternalMethod.get(className) + 1);
                } else {
                    classInternalMethod.put(className, 1);
                }
            }
        }

        // 移除外部类（引用的方法为0的）
        //remove 0-method referenced class
        Iterator<String> iterator = classExternalMethod.keySet().iterator();
        while (iterator.hasNext()) {
            if (classExternalMethod.get(iterator.next()) == 0) {
                iterator.remove();
            }
        }
    }
```

统计 dex 中的方法数量，这里由于 class 分为两种，一种是 dex 中定义的，一种是引用的别的 dex 的。所以这里区分了一下，这个指标可以作为分包的一个指标，因为如果一个dex里面引用的别的 dex 的 class 非常多的话，说明分包没分好，会导致 dex 体积变大，从而导致 apk 变大。

看看输出结果：

```json
[
    {
        "dex-file":"classes.dex",
        "internal-packages":[
            {
                "name":"android.support.v7.widget",
                "methods":2390
            },
            ...
            {
                "name":"com.example.resapk",
                "methods":24
            },
            ...
        ],
        "total-internal-classes":1316,
        "total-internal-methods":12459,
        "external-packages":[
            {
                "name":"android.view",
                "methods":692
            },
            {
                "name":"android.widget",
                "methods":563
            },
            {
                "name":"android.app",
                "methods":271
            },
            ...
        ],
        "total-external-classes":481,
        "total-external-methods":3598
    }
]
```

可以看出我们的 dex 中，使用 framework 的方法有 3598 个。

这里原始的输出，但是经过格式化之后，只能看到这样的信息，不太清楚为啥要减少字段：

```json
{
  "taskType": 4,
  "taskDescription": "Count methods in dex file, output results group by class name or package name.",
  "start-time": "2020-08-03 23:19:26:927",
  "end-time": "2020-08-03 23:19:28:421",
  "total-methods": 16057,
  "groups": [
    {
      "name": "Android System",
      "method-count": 15486
    },
    {
      "name": "java system",
      "method-count": 531
    },
    {
      "name": "[others]",
      "method-count": 40
    }
  ]
}
```

关于 dex 的 class 的信息没有了。



### MultiSTLCheckTask

这个检测是与 so 有关，但是这方面我不熟，里面的检测主要是使用 nm 工具的输出信息，所以我就不介绍了。

```java
    // 这里不熟悉 nm 工具。所以不太清楚在做什么
    // 使用 arm-linux-androideabi-nm 分析指定的 so 文件，-D -C 参数
    // 参数意义：https://manned.org/arm-linux-androideabi-nm/07ad85eb
    // 判断输出的每一行内容
    // 0001df70 T std::get_unexpected()
    // 0001df50 T std::set_unexpected(void (*)())
    // 00016780 T std::get_new_handler()
    // 00016760 T std::set_new_handler(void (*)())
    // 0001dd90 T std::current_exception()
    private boolean isStlLinked(File libFile) throws IOException, InterruptedException {
        ProcessBuilder processBuilder = new ProcessBuilder(toolnmPath, "-D", "-C", libFile.getAbsolutePath());
        Process process = processBuilder.start();
        BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        String line = reader.readLine();
        while (line != null) {
            String[] columns = line.split(" ");
            Log.d(TAG, "%s", line);
            if (columns.length >= 3 && columns[1].equals("T") && columns[2].startsWith("std::")) {
                return true;
            }
            line = reader.readLine();
        }
        reader.close();
        process.waitFor();
        return false;
    }
```



### MultiLibCheckTask

这个很简单，就是判断  libs 下是否有超过1个目录，很简单，就不贴代码了。

```json
            // 这个 task 很简单，就是输出 lib 下的各个目录
            //   "lib-dirs": [
            //    "arm64-v8a",
            //    "armeabi",
            //    "armeabi-v7a",
            //    "mips",
            //    "mips64",
            //    "x86",
            //    "x86_64"
            //  ],
            //  "multi-lib": true,
```



### ResProguardCheckTask

这个task是判断apk的资源是否被混淆了。

咋判断呢？

第一种情况，如果存在 r 目录，则肯定是被混淆了，因为是配合的 AndResGuard 使用的。

第二种情况，如果存在 res 目录，需要判断厘米的各个目录是否被混淆了：

```java
fileNamePattern = Pattern.compile("[a-z_0-9]{1,3}");

if (dir.isDirectory() && !fileNamePattern.matcher(dir.getName()).matches()) {
    hasProguard = false;
    Log.i(TAG, "directory " + dir.getName() + " has a non-proguard name!");
    break;
}
```

判断方法很简单，看看混淆后的目录名字是否超过了3个即可。



### ShowFileSizeTask

这个 task 是用来找出超过一定大小的文件的。这个还是挺有用，可以找出ui切的大图。

```json
{
  "taskType": 3,
  "taskDescription": "Show files whose size exceed limit size in order.",
  "files": [
    {
      "entry-name": "resources.arsc",
      "entry-size": 265548
    },
    {
      "entry-name": "res/mipmap-xxxhdpi-v4/ic_launcher_round.png",
      "entry-size": 16570
    },
    {
      "entry-name": "res/mipmap-xxhdpi-v4/ic_launcher_round.png",
      "entry-size": 11873
    },
    {
      "entry-name": "res/mipmap-xxxhdpi-v4/ic_launcher.png",
      "entry-size": 10652
    }
  ],
  "start-time": "2020-08-03 23:22:35:851",
  "end-time": "2020-08-03 23:22:35:863"
}
```

因为，解压的时候记录了每个文件的大小，所以直接进行分组就好了。这里是因为过滤了指定后缀文件，所以才这么少。

```java
for (Map.Entry<String, Pair<Long, Long>> entry : entrySizeMap.entrySet()) {
    // 文件后缀
    final String suffix = getSuffix(entry.getKey());
    Pair<Long, Long> size = entry.getValue();
    // 文件大小超过下限
    if (size.getFirst() >= downLimit * ApkConstants.K1024) {
        // 有在参数指定该后缀，则统计
        // 没有指定任何后缀，也统计
        // 否则，忽略
        if (filterSuffix.isEmpty() || filterSuffix.contains(suffix)) {
            entryList.add(Pair.of(entry.getKey(), size.getFirst()));
        } else {
            Log.d(TAG, "file: %s, filter by suffix.", entry.getKey());
        }
    } else {
        Log.d(TAG, "file:%s, size:%d B, downlimit:%d KB", entry.getKey(), size.getFirst(), downLimit);
    }
}
```



#### UncompressedFileTask

该任务是输出压缩前后大小仍然相等的文件，apk 实质上是一个压缩文件，所以里面的文件都是经过压缩的才对。

```java
for (Map.Entry<String, Pair<Long, Long>> entry : entrySizeMap.entrySet()) {
    final String suffix = getSuffix(entry.getKey());
    Pair<Long, Long> size = entry.getValue();
    // 将压缩前的文件放入 uncompressSizeMap
    // 将压缩后的文件放入 compressSizeMap
    if (filterSuffix.isEmpty() || filterSuffix.contains(suffix)) {
        if (!uncompressSizeMap.containsKey(suffix)) {
            uncompressSizeMap.put(suffix, size.getFirst());
        } else {
            uncompressSizeMap.put(suffix, uncompressSizeMap.get(suffix) + size.getFirst());
        }
        if (!compressSizeMap.containsKey(suffix)) {
            compressSizeMap.put(suffix, size.getSecond());
        } else {
            compressSizeMap.put(suffix, compressSizeMap.get(suffix) + size.getSecond());
        }
    } else {
        Log.d(TAG, "file: %s, filter by suffix.", entry.getKey());
    }
}
```

```java
// 判断压缩前后的文件大小是否相等
if (uncompressSizeMap.get(suffix).equals(compressSizeMap.get(suffix))) {
    ...
}
```

是因为解压apk的时候，统计了每个文件的大小，压缩前后的大小，所以可以直接使用。



#### UnStrippedSoCheckTask

这个任务，同样使用到了 nm 工具，所以我不熟。因为使用到了 nm 工具，所以，需要配置该工具的路径。

```java
private boolean isSoStripped(File libFile) throws IOException, InterruptedException {
    // 使用 arm-linux-androideabi-nm 分析指定的 so 文件，不带参数
    ProcessBuilder processBuilder = new ProcessBuilder(toolnmPath, libFile.getAbsolutePath());
    Process process = processBuilder.start();
    BufferedReader reader = new BufferedReader(new InputStreamReader(process.getErrorStream()));
    String line = reader.readLine();
    boolean result = false;
    // aarch64-linux-android-nm: libcrashlytics.so: no symbols
    // 如果输出上面的信息，则通过，当然这里我是不太懂到底是个啥
    if (!Util.isNullOrNil(line)) {
        Log.d(TAG, "%s", line);
        String[] columns = line.split(":");
        if (columns.length == 3 && columns[2].trim().equalsIgnoreCase("no symbols")) {
            result = true;
        }
    }
    reader.close();
    process.waitFor();
    return result;
}
```

