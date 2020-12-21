---
title: 009-Matrix源码分析：ApkChecker解压apk.md
index_img: /cover/2.jpg
banner_img: /cover/top.jpg
date: 2020-8-2
categories: Matrix
---

这篇本来无甚难度，之所以单独一篇，是因为它输出的一些结果，后面的 task 都会用上。



#### init 函数

```java
    @Override
    public void init() throws TaskInitException {
        super.init();
        inputFile = new File(config.getApkPath());

        outputFile = new File(config.getUnzipPath());

        mappingTxt = new File(config.getMappingFilePath());

        resMappingTxt = new File(config.getResMappingFilePath());
    }
```

inputFile 是 apk 的路径。

outputFile 是 apk 解压后的输出路径。ApkChecker运行完成之后，会删除该目录。

mappingTxt 是 mapping 文件，因为 R.string 等类可能会被混淆。

resMappingTxt 是使用了 AndResGuard 的工程会有这个，是因为资源名会被混淆，比如 res/layout/activity_main 混淆成了 r/l/a，输出混淆后的名称出来，根本看不懂。



#### call 函数

```java
    @Override
    public TaskResult call() throws TaskExecuteException {

        try {
            ...

            // 读 mappping 文件
            readMappingTxtFile();
            config.setProguardClassMap(proguardClassMap);
            // 读 resource_mapping 文件
            readResMappingTxtFile();
            config.setResguardMap(resguardMap);

            Enumeration entries = zipFile.entries();
            JsonArray jsonArray = new JsonArray();
            String outEntryName = "";
            while (entries.hasMoreElements()) {
                ZipEntry entry = (ZipEntry) entries.nextElement();
                // writeEntry 里面只是将文件解压出来了
                // 返回的 outEntryName 是资源未混淆之前的名字
                outEntryName = writeEntry(zipFile, entry);
                if (!Util.isNullOrNil(outEntryName)) {
                    JsonObject fileItem = new JsonObject();
                    fileItem.addProperty("entry-name", outEntryName);
                    fileItem.addProperty("entry-size", entry.getCompressedSize());
                    jsonArray.add(fileItem);
                    // key：文件未混淆时名字
                    // value： <文件大小，文件压缩后大小>
                    entrySizeMap.put(outEntryName, Pair.of(entry.getSize(), entry.getCompressedSize()));
                    // key： 文件混淆时名字
                    // value：文件未混淆时名字
                    entryNameMap.put(entry.getName(), outEntryName);
                }
            }

            config.setEntrySizeMap(entrySizeMap);
            config.setEntryNameMap(entryNameMap);
            ...
            return taskResult;
        } catch (Exception e) {
            throw new TaskExecuteException(e.getMessage(), e);
        }
    }
```

核心代码还是比较简单的。

先读取 mapping 文件，好还原 R$xxx.class 是啥。读取的代码就不分析了，对着 mapping 文件的格式来看还是很好理解的。

`config.setProguardClassMap(proguardClassMap);` 这行代码比较重要，因为 proguardClassMap 这个 map 后面的 task 会常常用到，可以用于还原类名。

在读取，resMapping 文件，好还原资源名。读取代码也不分析了，生成 resMapping 需要接入 AndResGuard，可以试试，会长不少见识。也是对着生成的文件看代码，没啥好说的。

`config.setResguardMap(resguardMap);`，这个resguardMap后面的task也是会用到的。

entrySizeMap 与 entryNameMap ，后面的 task 也会用到。

总结，这个 task 里面做了如下动作：

- mapping
- resMapping
- 文件名字（混淆前后，主要针对资源文件），文件大小（压缩前后）

更详细的注释，查看 github 的 note 分支。
