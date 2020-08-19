---
title: 015-Matrix源码分析：检测内存中重复的Bitmap
date: 2020-8-19
categories: Matrix
---

在分析activity的引用链的时候，我们还可以顺便分析一下，内存中有没有重复的Bitmap，如果有的话，就可以看看是什么原因导致的，这样对优化内存有帮助。

分析该任务的类是 `DuplicatedBitmapAnalyzer`。用过 as 的都知道它有一个 profiler功能，里面可以对CPU/网络/内存做分析。而内存分析的话，它也提供了 dump 与分析 hprof 文件的功能。

所以，我们查看这个模块的相关源码的话，就可以找到这个类：

> https://android.googlesource.com/platform/tools/base/+/studio-master-dev/perflib/src/main/java/com/android/tools/perflib/heap/memoryanalyzer/DuplicatedBitmapAnalyzerTask.java

里面是使用了一个算法来分析内存中的相同的bitmap。

先看看上面链接的这个类，然后再看 matrix 的这个类，会更好一点，因为 matrix 的这个类加了不少东西，导致方法很长，看起来就不好懂。

那我们就先分析 google 的那个类，它的逻辑很纯粹，好理解这个算法。

先说一下这个算法是如何做的：

首先，要判断两个 Bitmap 是否是一样的，只需要判断它的 mBuffer 字段是不是一样的，所以就归结为**比较两个 byte 数组是否一样**。

内存中的 Bitmap 对象那么多，怎么才能列举出所有相等的 mBuffer 呢？

第一步，将 mBuffer 按照 mBuffer[0] 的值进行分组。假设，得到了 8 组。

第二部，遍历上面得到的 8 组内容，对每组按照 mBuffer[1] 的值再进行分组。

重复上面的过程。一直到有 mBuffer 已经被遍历完了，就检查一下被遍历完成的集合大小，是否大于1。如果大于1，则说明这个集合里面的 mBuffer 是一样的。所以就找到了所有的相同的 mBuffer。下面是具体的实现代码：

> DuplicatedBitmapAnalyzerTask#analyze

```java
        int columnIndex = 0;
        while (!commonPrefixSets.isEmpty()) {
            for (Set<ArrayInstance> commonPrefixArrays : commonPrefixSets) {
                Map<Object, Set<ArrayInstance>> entryClassifier = new HashMap<>(
                        commonPrefixArrays.size());
                for (ArrayInstance arrayInstance : commonPrefixArrays) {
                    Object element = cachedValues.get(arrayInstance)[columnIndex];
                    if (entryClassifier.containsKey(element)) {
                        entryClassifier.get(element).add(arrayInstance);
                    } else {
                        Set<ArrayInstance> instanceSet = new HashSet<>();
                        instanceSet.add(arrayInstance);
                        entryClassifier.put(element, instanceSet);
                    }
                }
                for (Set<ArrayInstance> branch : entryClassifier.values()) {
                    if (branch.size() <= 1) {
                        // Unique branch, ignore it and it won't be counted towards duplication.
                        continue;
                    }
                    Set<ArrayInstance> terminatedArrays = new HashSet<>();
                    // Move all ArrayInstance that we have hit the end of to the candidate result list.
                    for (ArrayInstance instance : branch) {
                        if (instance.getLength() == columnIndex + 1) {
                            terminatedArrays.add(instance);
                        }
                    }
                    branch.removeAll(terminatedArrays);
                    // Exact duplicated arrays found.
                    if (terminatedArrays.size() > 1) {
                        int byteArraySize = -1;
                        ArrayList<Instance> duplicateBitmaps = new ArrayList<>();
                        for (ArrayInstance terminatedArray : terminatedArrays) {
                            duplicateBitmaps.add(byteArrayToBitmapMap.get(terminatedArray));
                            byteArraySize = terminatedArray.getLength();
                        }
                        results.add(
                                new DuplicatedBitmapEntry(new ArrayList<>(duplicateBitmaps),
                                        byteArraySize));
                    }
                    // If there are ArrayInstances that have identical prefixes and haven't hit the
                    // end, add it back for the next iteration.
                    if (branch.size() > 1) {
                        reducedPrefixSets.add(branch);
                    }
                }
            }
            commonPrefixSets.clear();
            commonPrefixSets.addAll(reducedPrefixSets);
            reducedPrefixSets.clear();
            columnIndex++;
        }
```

其实还有一种找出重复 mBuffer  的方式，比较简单，就是对 mBuffer 进行 md5或者hash。比如下面的代码：

```java
    /**
     * 处理 heap，找出有相同 buffer 的 bitmap 的 instance，存放在 map 中。
     * @param objMap 存放相同 buffer 的 instance 的map
     * @param heap 待处理的 heap
     * @param classObj 这里是  bitmap 的封装对象
     */
    private static void analyzeHeapForSameBuffer(Map<String, ArrayList<ObjNode>> objMap,
                                    Heap heap, ClassObj classObj){
        List<Instance> instances = classObj.getHeapInstances(heap.getId());
        for (Instance instance : instances){
            ArrayInstance buffer = HahaHelper.fieldValue(HahaHelper.classInstanceValues(instance), "mBuffer");
            byte[] bytes = HahaHelper.getByteArray(buffer);
            try {
                String md5String = Md5Helper.getMd5(bytes);
                if(objMap.containsKey(md5String)){
                    objMap.get(md5String).add(getObjNode(instance));
                }else {
                    ArrayList<ObjNode> objNodes = new ArrayList<>();
                    objNodes.add(getObjNode(instance));
                    objMap.put(md5String, objNodes);
                }
            }catch (Exception e){
                e.printStackTrace();
            }
        }

    }
```

重复的图片找到之后，再找到引用链（与分析 activity 的引用链一样），然后将这些信息保存起来。这里还保存了 mBuffer，用于还原图片，便于开发者寻找问题。

> com.tencent.matrix.resource.analyzer.CLIMain#analyzeAndStoreResult

```java
                // 使用 bitmap 的 buffer 还原成图片
                // Store bitmap buffer.
                final List<DuplicatedBitmapEntry> duplicatedBmpEntries = duplicatedBmpResult.getDuplicatedBitmapEntries();
                final int duplicatedBmpEntryCount = duplicatedBmpEntries.size();
                for (int i = 0; i < duplicatedBmpEntryCount; ++i) {
                    final DuplicatedBitmapEntry entry = duplicatedBmpEntries.get(i);
                    final BufferedImage img = BitmapDecoder.getBitmap(
                            new HprofBitmapProvider(entry.getBuffer(), entry.getWidth(), entry.getHeight()));
                    // Since bmp format is not compatible with alpha channel, we export buffer as png instead.
                    final String pngName = bufferContentsRootDirName + "/" + entry.getBufferHash() + ".png";
                    try {
                        zos.putNextEntry(new ZipEntry(pngName));
                        ImageIO.write(img, "png", zos);
                        zos.flush();
                    } finally {
                        try {
                            zos.closeEntry();
                        } catch (Throwable ignored) {
                            // Ignored.
                        }
                    }
                }
```

最后就是输出 json 信息：

```java
                    final JSONObject duplicatedBmpResultJson = new JSONObject();
                    duplicatedBmpResult.encodeToJSON(duplicatedBmpResultJson);
```

