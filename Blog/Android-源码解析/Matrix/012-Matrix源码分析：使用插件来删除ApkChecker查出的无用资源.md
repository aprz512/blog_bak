---
title: Matrix源码分析番外篇：Dex文件结构
date: 2020-8-12
categories: Matrix
---

在第 010 篇文章中，我们说过了 ApkChecker 是如何查找 apk 中的无用资源的，那么找到之后，如果你不想手动删除资源，然后重新打包的话，该怎么办？

Matrix 也提供了一个插件来做这个事情，也是在 matrix-gradle-plugin 中，我们介绍方法插桩的时候分析过其中一个插件，还有一个没有介绍，就留到现在说，串起来舒服些。

里面有个 RemoveUnusedResourcesTask，它就是用来删除无用资源的。

直接上源码部分：

> com.tencent.matrix.plugin.task.RemoveUnusedResourcesTask#removeResources

```java
// 获取未签名的 apk 路径，这些都是 gradle 的 api，就不介绍了，需要自己看文档
String unsignedApkPath = output.outputFile.getAbsolutePath();
Log.i(RemoveUnusedResourcesTask.TAG, "original apk file %s", unsignedApkPath);
long startTime = System.currentTimeMillis();
// 获取 R.txt 文件路径，已经签名信息
removeUnusedResources(unsignedApkPath, project.getBuildDir().getAbsolutePath() + "/intermediates/symbols/${variant.name}/R.txt", variant.variantData.variantConfiguration.signingConfig);
```

上面就是获取必要的信息。

> com.tencent.matrix.plugin.task.RemoveUnusedResourcesTask#removeUnusedResources

removeUnusedResources 还是比较长的，只截取部分代码，主要流程有就行，其他的可以自行拉我 fork 库的 note 分支查看。

```java
File inputFile = new File(originalApk);
Set<String> ignoreRes = project.extensions.matrix.removeUnusedResources.ignoreResources;
// 加载配置的应该忽略的资源
for (String res : ignoreRes) {
    // 通配符转转正则表达式
    // 配置的语法应该支持 * 啥的吧
    ignoreResources.add(Util.globToRegexp(res));
}
// 加载配置的 未使用的资源，这个就是 ApkChecker 分析出来的资源
Set<String> unusedResources = project.extensions.matrix.removeUnusedResources.unusedResources;
```

这里是从我们的 build.gradle 文件中，读取配置信息，配置信息长这样：

> build.gradle

```groovy
apply plugin: 'com.tencent.matrix-plugin'
matrix {
    trace {
        enable = true
        baseMethodMapFile = "${project.projectDir}/matrixTrace/methodMapping.txt"
        blackListFile = "${project.projectDir}/matrixTrace/blackMethodList.txt"
    }
    removeUnusedResources {
        enable true
        variant = "debug"
        needSign true
        shrinkArsc true
        //Notice: You need to modify the  value of $apksignerPath on different platform. the value below only suitable for Mac Platform,
        //if on Windows, you may have to  replace apksigner with apksigner.bat.
        apksignerPath = "${android.getSdkDirectory().getAbsolutePath()}/build-tools/${android.getBuildToolsVersion()}/apksigner.bat"
        unusedResources = project.ext.unusedResourcesSet
        ignoreResources = ["R.id.*", "R.bool.*"]
    }
}

```

看上面的removeUnusedResources部分，里面有个 unusedResources，这里就应该填上 ApkChecker 分析出来的资源，我们直到 ApkChecker 分析出来的结果是一个 json 文件，那么应该怎么与它关联起来呢？

官方的 Sample 里面，有一个用法是这样的。

首先，我们定义一个 ext 属性：

```groovy
ext.unusedResourcesSet = new HashSet<String>();
```

然后，在打包的时候，插入如下动作：

```groovy
applicationVariants.all { variant ->
    // 只对 debug 包做处理
    if (variant.name.equalsIgnoreCase("debug")) {
        // packageDebug 是一个内置属性
        packageDebug.doLast {
            // 打包完成之后，使用 apkchecker 分析这个包
            ProcessBuilder processBuilder = new ProcessBuilder();
            println configurations.apkCheckerDependency.getAt(0).getAbsolutePath()
            processBuilder.command("java",
                                   "-jar", configurations.apkCheckerDependency.getAt(0).getAbsolutePath(),
                                   "--apk", variant.outputs.first().outputFile.getAbsolutePath(),
                                   "--output", project.getProjectDir().getAbsolutePath() + "/unused_resources",
                                   "--format", "json",
                                   "-unusedResources", "--rTxt", project.getBuildDir().getAbsolutePath() + "/intermediates/symbols/${variant.name}/R.txt");
            Process process = processBuilder.start();
            // 等待程序执行完成
            process.waitFor();
            File outputFile = new File(project.getProjectDir().getAbsolutePath() + "/unused_resources.json");
            // 读取 json 文件到 unusedResourcesSet 里面
            if (outputFile.exists()) {
                Gson gson = new Gson();
                JsonArray jsonArray = gson.fromJson(outputFile.text, JsonArray.class);
                for (int i = 0; i < jsonArray.size(); i++) {
                    if (jsonArray.get(i).asJsonObject.get("taskType").asInt == 12) {
                        JsonArray resList = jsonArray.get(i).asJsonObject.get("unused-resources").asJsonArray;
                        for (int j = 0; j < resList.size(); j++) {
                            project.ext.unusedResourcesSet.add(resList.get(j).asString);
                        }
                        println "find unused resources:\n" + unusedResourcesSet
                        break;
                    }
                }
                outputFile.delete();
            }
        }
    }
}
```

这样，我们就拿到了 ApkChecker 里面分析出来的结果，而且还是一步到位。回到源码部分，接着是读取 rTxt 文件：

```java
readResourceTxtFile(resTxtFile, resourceMap, styleableMap);
```

就是将 R.txt 中的符号表内存读到map里面。

比如：

```
int attr layout_editor_absoluteY 0x7f0200c5 
就会变成 {"R.attr.layout_editor_absoluteY":0x7f0200c5}
```

```
int[] styleable ViewStubCompat { 0x010100d0, 0x010100f2, 0x010100f3 } 
会变成 {"R.styleable.ViewStubCompat" : [Pair("R.styleable.ViewStubCompat", 0x010100d0)]}
```

接下来是，拷贝apk里面的文件，针对 res 文件做如下处理：

```java
if (zipEntry.name.startsWith("res/")) {
    // zipEntry.name --> res/mipmap-hdpi-v4/ic_launcher_round.png
    // resourceName --> R.mipmap.ic_launcher_round.png
    String resourceName = entryToResouceName(zipEntry.name);
    if (!Util.isNullOrNil(resourceName)) {
        // 如果有配置了 unusedResources，这里就不拷贝这个资源到新的 apk 里面了
        if (removeResources.containsKey(resourceName)) {
            Log.i(TAG, "remove unused resource %s", resourceName);
            continue;
        } else {
            addZipEntry(zipOutputStream, zipEntry, zipInputFile);
        }
    } else {
        addZipEntry(zipOutputStream, zipEntry, zipInputFile);
    }
}
```

因为 unusedResources 里面的资源是需要移除的，所以这里只拷贝不在该集合中的资源。

拷贝文件，针对非 res 文件做如下处理：

```java
// 为啥 META-INF/ 下的文件也不拷贝
// 里面的几个签名文件可以不用管，因为后面会重新签名，但是还有别的文件呢
if (needSign && zipEntry.name.startsWith("META-INF/")) {
    continue;
} else {
    // shrinkArsc 需要精简 arsc 文件，这是大头
    if (shrinkArsc && zipEntry.name.equalsIgnoreCase("resources.arsc") && unusedResources.size() > 0) {
        File srcArscFile = new File(inputFile.getParentFile().getAbsolutePath() + "/resources.arsc");
        File destArscFile = new File(inputFile.getParentFile().getAbsolutePath() + "/resources_shrinked.arsc");
        if (srcArscFile.exists()) {
            srcArscFile.delete();
            srcArscFile.createNewFile();
        }
        // 将 zip 文件中的 .asrs 文件解压出来
        unzipEntry(zipInputFile, zipEntry, srcArscFile);

        // 分析 .arsc 文件，需要一张图配合看
        // https://user-gold-cdn.xitu.io/2019/5/24/16ae9b85b2f4e918?imageView2/0/w/1280/h/960/format/webp/ignore-error/1
        // 或者使用 010 打开看看（推荐）
        ArscReader reader = new ArscReader(srcArscFile.getAbsolutePath());
        ResTable resTable = reader.readResourceTable();
        for (String resName : removeResources.keySet()) {
            ArscUtil.removeResource(resTable, removeResources.get(resName), resName);
        }
        // 重新生成 .arsc 文件
        ArscWriter writer = new ArscWriter(destArscFile.getAbsolutePath());
        writer.writeResTable(resTable);
        Log.i(TAG, "shrink resources.arsc size %f KB", (srcArscFile.length() - destArscFile.length()) / 1024.0);
        addZipEntry(zipOutputStream, zipEntry, destArscFile);
    } else {
        addZipEntry(zipOutputStream, zipEntry, zipInputFile);
    }
}
```

里面主要是针对 arsc 文件做了处理，其他的文件原封不动的拷贝就好了。对于 arsc 文件结构，番外篇已经介绍了一部分，这里就只说说它处理了什么吧。

使用 010 Editor 打开 arsc 文件，会发现如下结构：

```
TablePackageType
	...
	--ResTable_typeSpec
	--ResTable_type
		--ResTable_entry
		--Res_value
```

当我们从apk中删除了一些资源后，比如，我们删除了一个 drawable 资源（因为 values 下面的资源都在一个文件中，比如 string.xml 等，所以拷贝时无法删除其中的某一项），那么它的 ResTable_type 这个结构就需要改一下，需要将这个资源对应的 ResTable_entry 与 Res_value 删除才行。这里还要考虑文件的格式，删除还是挺麻烦的，具体可以看代码。

需要注意的时，删除的时候，需要保证原来的索引不变。比如，有两个 String，A 与 B，假设他们生成的 id 为 （A）0x01 与 （B）0x02，A是个无用资源，当你删除之后，仍然要保证 arsc 文件中，B的id是0X02，而 id 是与该 entry 在数组中的index有关。

文件都拷贝完成之后，就需要进行签名：

```java
// 调用 apksigner 程序，进行签名
Log.i(TAG, "resign apk...");
ProcessBuilder processBuilder = new ProcessBuilder();
processBuilder.command(apksigner, "sign", "-v",
                       "--ks", signingConfig.storeFile.getAbsolutePath(),
                       "--ks-pass", "pass:" + signingConfig.storePassword,
                       "--key-pass", "pass:" + signingConfig.keyPassword,
                       "--ks-key-alias", signingConfig.keyAlias,
                       outputFile.getAbsolutePath());
//Log.i(TAG, "%s", processBuilder.command());
Process process = processBuilder.start();
process.waitFor();
```

直接调用了 apksigner 来做这件事。

然后是移除 styleable，上面的逻辑只处理了非 styleable 资源：

```java
Iterator<String> styleableItera =  styleableMap.keySet().iterator();
while (styleableItera.hasNext()) {
    String styleable = styleableItera.next();
    Pair<String, Integer>[] attrs = styleableMap.get(styleable);
    int i = 0;
    for (i = 0; i < attrs.length; i++) {
        if (!removeResources.containsValue(attrs[i].right)) {
            break
        }
    }
    if (attrs.length > 0 && i == attrs.length) {
        Log.i(TAG, "removed styleable " + styleable);
        styleableItera.remove();
    }
}
```

最后，压缩 R.txt，因为删除了些资源：

```java
shrinkResourceTxtFile(newResTxtFile, resourceMap, styleableMap);
```

这样，整个流程就完毕了。