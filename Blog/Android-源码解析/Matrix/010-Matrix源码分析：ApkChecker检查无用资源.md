---
title: 010-Matrix源码分析：ApkChecker检查无用资源
categories: Matrix
date: 2020-7-30
---

分析无用资源，Android studio 本身就提供了lint工具。但是有个缺点就是无法配合proguard使用，无用代码引用的资源是不会被计算出来的。

ApkChecker里面提供了两个小工具，可以直接分析apk包里面的无用资源（res目录与asset目录）。

### UnusedAssetsTask 

我们先看简单一些的 UnusedAssetsTask 类，该类用于寻找出没有使用的 asset 资源。

```java
public class UnusedAssetsTask extends ApkTask {...}
```

ApkChecker里面提供了十来个小工具，都是继承于 ApkTask，但是 ApkTask 里面啥都没有，只是做了输入参数处理，与提供一个统一处理核心逻辑的地方。



Task 的分析，主要是3部分

- 第一部分是构造函数，里面有个 type 比较重要，输出 task 分析的结果会用到，后面会有展示。

- 第二部分是 init 函数，用于初始化一些变量。

- 第三部分是 call 函数，里面是 task 的核心逻辑。

后面的 task 都会按照这个模板来分析。



#### 构造函数

```java
public class UnusedAssetsTask extends ApkTask {

    private static final String TAG = "Matrix.UnusedAssetsTask";

    public UnusedAssetsTask(JobConfig config, Map<String, String> params) {
        super(config, params);
        type = TaskFactory.TASK_TYPE_UNUSED_ASSETS;
        dexFileNameList = new ArrayList<>();
        ignoreSet = new HashSet<>();
        assetsPathSet = new HashSet<>();
        assetRefSet = new HashSet<>();
    }
```

JobConfig 是从命令行输入读取的配置信息解析而成的对象。ApkChecker 除了提供了十几个小工具用于分析与优化APK，由于它是一个命令行工具，所以还有一部分代码都是与读取参数有关，就比如我们平时使用的 `ls -l` 命令，会带些参数，还是一套比较完整的解析代码，而且对于结果输出，也有json与html两个格式，这也涉及到些设计模式，与重构还是代码大全的某个例子有点类似，有兴趣的可以研究研究。



####  init 函数

```java
    @Override
    public void init() throws TaskInitException {
        super.init();

        String inputPath = config.getUnzipPath();
        inputFile = new File(inputPath);
        ...

        // 收集 dex 文件
        File[] files = inputFile.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.isFile() && file.getName().endsWith(ApkConstants.DEX_FILE_SUFFIX)) {
                    dexFileNameList.add(file.getName());
                }
            }
        }
    }
```

`config.getUnzipPath`是获取解压的路径，09 篇说了 `UnzipTask` 任务，它将需要分析的 apk 解压，这里的路径就是解压目录路径。

这个函数主要是获取所有 .dex 文件。

#### call 函数

```    @Override
    @Override
    public TaskResult call() throws TaskExecuteException {
        try {
            ...
            
            // 收集 asset 目录下文件
            findAssetsFile(assetDir);
            
            // 收集相对路径，里面去除了配置的忽略文件
            // 因为写忽略配置文件，肯定是相对 assets 目录的
            // --ignoreAssets
            //  assetsPathSet 里面存放的是 asset 目录下的所有文件
            generateAssetsSet(assetDir.getAbsolutePath());
            
            Log.i(TAG, "find all assets count: %d", assetsPathSet.size());
            decodeCode();
            // assetRefSet 里面存放的是 配置的忽略的资源 + smali 中引用到的资源
            Log.i(TAG, "find reference assets count: %d", assetRefSet.size());
            assetsPathSet.removeAll(assetRefSet);
            ...
        } catch (Exception e) {
            throw new TaskExecuteException(e.getMessage(), e);
        }
    }
```

可以自己先想一下，如何才能找出 assets 目录下的未使用的资源？

访问 asset 资源的方法：

```java
webView.loadUrl("file:///android_asset/win8_Demo/index.html");
getAssets().open("wpics/0ZR424L-0.jpg");
```

可以看到，我们都是使用的字符串来表示 assets 目录下的某一特定资源。

当 .dex 反编译为 .smail 代码的时候，对于常量字符串，都是以 `const-string` 开头的。

```smail
// const-string 的语法：
// const-string v0, "LOG"        # 将v0寄存器赋值为字符串常量"LOG"
// 这算是一种简单的分析方式
```

所以，以这种方式，我们可以得出一个大致正确的结果。

所以思路就是：**以读取文件的方式，遍历 smali 代码文件，找到所有使用常量字符串的地方。**

需要先反编译 .dex 文件：

```java
private void decodeCode() throws IOException {
    for (String dexFileName : dexFileNameList) {
        // 使用 apktool 加载 dex 文件
        DexBackedDexFile dexFile = DexFileFactory.loadDexFile(new File(inputFile, dexFileName), Opcodes.forApi(15));

        BaksmaliOptions options = new BaksmaliOptions();
        // class 类按自然排序
        List<? extends ClassDef> classDefs = Ordering.natural().sortedCopy(dexFile.getClasses());

        for (ClassDef classDef : classDefs) {
            // 从 ClassDef 中获取该 class 对应的 smali 代码
            String[] lines = ApkUtil.disassembleClass(classDef, options);
            if (lines != null) {
                // 分析 smali 代码
                readSmaliLines(lines);
            }
        }

    }
}
```

使用的是 APKTOOL 的 api，就不介绍了，我也不熟。

```java
private void readSmaliLines(String[] lines) {
    if (lines == null) {
        return;
    }
    for (String line : lines) {
        line = line.trim();
        // 找 常量字符串，因为使用 assets 中的资源文件，方式是很单一的，需要制定文件的名字
        // 所以遍历 smali 代码，找到所有使用常量字符串的地方
        // const-string 的语法：
        // const-string v0, "LOG"        # 将v0寄存器赋值为字符串常量"LOG"
        // 这算是一种简单的分析方式
        if (!Util.isNullOrNil(line) && line.startsWith("const-string")) {
            String[] columns = line.split(",");
            if (columns.length == 2) {
                // 获取 , 后面的
                String assetFileName = columns[1].trim();
                // 去除双引号
                assetFileName = assetFileName.substring(1, assetFileName.length() - 1);
                if (!Util.isNullOrNil(assetFileName)) {
                    // 遍历之前收集的所有资源文件的路径
                    for (String path : assetsPathSet) {
                        // 如果包含该文件
                        if (assetFileName.endsWith(path)) {
                            // 存放到 assetRefSet 里面
                            assetRefSet.add(path);
                        }
                    }
                }
            }
        }
    }
}
```

这里就是判断，**const-string 的字符串里面有没有与 asset 目录下资源名一样的，有的话就算这个资源文件被使用了。**

当然，如果恰好你的字符串与资源名相同，那就会误判。

如何分析 asset 资源文件有没有被使用，ApkChecker 提供了一个简单的判断方式，虽然不完全准确，但是对我们应该也有启发。

那么，可以思考一下，如何判断 libs 文件夹下的 jar 包有没有被引用？这个 ApkChecker 中并没有实现，如果有思路可以提pr。

下面再介绍一下，如何分析res中的资源有没有被引用？



### UnusedResourcesTask

分析 res 下无用资源与 assets 下无用资源是相似的，但是需要弄清楚一件事：

当我们在 layout 等xml资源中使用了 values 文件中的资源，这个在代码中是无法找到的，所以需要理清思路，分析各个资源的引用关系。

#### 思路

ApkChecker 里面提供的思路如下：

```
像分析无用对象一样，我们将 res 下的资源分为两部分：
一部分是 values 资源，就是 values 文件夹下的资源（包括各种衍生文件夹，values-v23 之类的）。
另一部分就是除了 values 资源下的其他 xml 资源文件。注意这里只有 .xml 文件。（那么仔细想一些，是不是还有部分资源没有包含进去）

然后，找到代码里面引用的所有资源，就可以开始分析资源引用关系了。

我们将 values 下的资源叫做 A，非 values 下的 xml 资源叫做 C，代码引用的资源叫做 B。
```

做下面的代码分析：

```java
private void readChildReference(String resource) throws IllegalStateException {
    if (nonValueReferences.containsKey(resource)) {
        visitPath.push(resource);
        // 获取该资源引用的其他资源
        Set<String> childReference = nonValueReferences.get(resource);
        // 资源有被引用则从 unusedResSet 里面移除
        // 验证一下，如果 a 引用 b，a没有用到，b会被发现吗？
        unusedResSet.removeAll(childReference);
        for (String reference : childReference) {
            if (!visitPath.contains(reference)) {
                readChildReference(reference);
            } else {
                visitPath.push(reference);
                throw new IllegalStateException("Found resource cycle! " + visitPath.toString());
            }
        }
        visitPath.pop();
    }
}
```

这是一个深度遍历。

resource参数是集合 A + B 中的元素。

我们假设，resource 是一个 layout 资源，按照上面的逻辑，该 layout 引用的所有 string，dimen， drawable 都会从 unusedResSet 集合中移除。这也就是说明它们被人引用过了，即判断它是有用资源。

这显然是一个有bug的逻辑，就像注释里面说的，当一个无用资源A引用了无用资源B，那无用资源B岂不是被标记为有用，确实！！！

所以，有必要的话可以删除一批无用资源后，再次重新运行该工具，直到资源无变化。

#### smali 引用方式分析

我们接下来，看看，如何从代码中找到资源的引用。有四种情况

```java
        1. const

        const v6, 0x7f0c0061

        2. sget

        // app 生成的是 static int 的，所以直接转换为了数值
        // 但是 lib 里面不是  final 的，所以会是引用的方式
        sget v6, Lcom/tencent/mm/R$string;->chatting_long_click_menu_revoke_msg:I
        sget v1, Lcom/tencent/mm/libmmui/R$id;->property_anim:I

        3. sput

        sput-object v0, Lcom/tencent/mm/plugin_welab_api/R$styleable;->ActionBar:[I   //define resource in R.java

        4. array-data

        :array_0
        .array-data 4
            0x7f0a0022
            0x7f0a0023
        .end array-data
```

第一种最简单，直接是以 R.layout.xxx 的方式。

第二种稍微绕一点，也是以 R.layout.xxx 的方式，但是不知在  app 中，而是在 modules 中，因为 modules 生成的资源不是 final 的（因为如果是 final 的，那资源id 就固定了，而 app 引用modules 的时候，是需要一起计算 id 的，id 是按顺序的）。

由于不是 final 的，所以就不是 const 语法，而是 sget 语法。

第三种，是 R.styleable，它的 R.java 是这样的：

```java
    public static final int[] SwitchCompat={
      0x01010124, 0x01010125, 0x01010142, 0x7f020101, 
      0x7f020107, 0x7f020112, 0x7f020113, 0x7f020115, 
      0x7f020123, 0x7f020124, 0x7f020125, 0x7f02013a, 
      0x7f02013b, 0x7f02013c
    };
```

对应的 smali 就是 sput 了。

第四种，就是 array 了。

```java
int[] arr = new int[]{
    R.layout.activity_main, R.string.app_name
};
```

注意 3 4 的区别。

以上，就是所有资源的引用分析了，具体代码就不贴了，无非就是如何根据 smali 语法分割出我们想要的资源id，然后从 build/intermediates/symbols/debug/R.txt 中，根据资源id找出对应的资源名（如果资源做了混淆，还要处理一下 resMapping）。

还有一个问题，就是使用 `android.content.res.Resources#getIdentifier`这种方式引用的资源是无法分析出来的。

我们看一下反编译的 smali 代码：

```
invoke-virtual {v0, v1, v2, v3}, Landroid/content/res/Resources;->getIdentifier(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)I
```

v1-v3是参数。

如果都是直接传递的字符串常量，那么ok，勉强是可以分析出来的，但是如果使用了局部变量，而且不是紧邻 getIdentifier 代码，那么可能连资源id都拿不到，就算拿到了资源id，也几乎分析不出来它的资源类型是啥。

而且，有时候还会使用动态拼接资源名的方式，这就更没法搞了。

所以，ApkChecker 里面是没有支持这种引用方式的，这个暂时无思路，可能的话，只能从侧面入手，比如写个插件，遇到 getIdentifier 就提醒一下，让开发者把资源添加到白名单里面去。

#### xml文件资源引用分析

这个 task 里面涉及到的东西还是比较多的，比如分析 .xml 文件的资源引用：

```java
String value = mParser.getAttributeValue(i);
String attributeName = mParser.getAttributeName(i);
if (!Util.isNullOrNil(value)) {
    if (value.startsWith("@")) {
        int index = value.indexOf('/');
        if (index > 1) {
            String type = value.substring(1, index);
            resourceRefSet.add(ApkConstants.R_PREFIX + type + "." + value.substring(index + 1).replace('.', '_'));
        }
    } else if (value.startsWith("?")) {
        int index = value.indexOf('/');
        if (index > 1) {
            resourceRefSet.add(ApkConstants.R_ATTR_PREFIX + "." + value.substring(index + 1).replace('.', '_'));
        }
    }
}
```

我们在 xml 中引用资源，都会使用 @ 方式，而 xml 编译之后，@还是存在的，所以以 @ 为标识来添加资源引用关系。

这里有个疑问，不知道是不是 bug。我们引用属性，通常会使用 `?attr/xxx`的方式，但是 xml 编译之后，会变成 `?xxx` ，所以 else if 里面的 if 条件是进不去的。这里我没搞懂，else if 是干啥的。

如下：

```
<vector android:tint="?colorControlNormal" 

<size android:height="@dimen/abc_progress_bar_height_material"
```



