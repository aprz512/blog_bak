---
title: 013-Matrix源码分析：检测Activity内存泄漏
index_img: /cover/17.jpg
banner_img: /cover/top.jpg
date: 2020-8-17
categories: Matrix
---

Matrix的框架抽象了一个 Plugin 类。我们可以继承这个类来实现我们想做的功能。使用的时候只需要：

```java
        TracePlugin tracePlugin = (new TracePlugin(traceConfig));
        builder.plugin(tracePlugin);
```

matrix-resource-canary-android这个module里面就提供了一个 ResourcePlugin。其主要作用就是检测 APP 中的 activity 泄漏，然后 dump 出 hprof 文件，再裁剪 hprof 文件，最后上传。

检测 activity 泄漏，通常会想到 leakcanary，这个module 也是在 leakcanary 的基础上进行了二次开发，我们看看源码。

核心代码在 ActivityRefWatcher 里面。

### start

plugin 里面分别调用了 watcher 的 start stop等方法，这些方法属于 plugin 的生命周期，从这里入手逻辑会更清晰。

首先既然要检测泄漏，那么必然要监听 activity 的 destroy 方法，老一套，使用了 ActivityLifecycleCallbacks：

```java
        @Override
        public void onActivityDestroyed(Activity activity) {
            // 该方法里面的逻辑也很简单，就是将activity的信息添加到一个集合中
            // 后面会开启扫描任务扫描这个集合
            pushDestroyedActivityInfo(activity);
       /*     synchronized (mDestroyedActivityInfos) {
                mDestroyedActivityInfos.notifyAll();
            }*/
        }
```

在 LeakCanary 里面，我们判断是否泄漏是延迟了 5S，而且还加上了 gc，再去判断对象是否还存在。

Matrix的判断方式有点不一样，往下看。

### RetryableTask

在start的时候，也开启了一个task，这个task是运行在 HandlerThread 中，也就是一个线程了。

我们分析看这个 task 做了啥：

```java
            // 创建一个 weak 引用，触发 gc，看有没有被回收，触发 gc，系统不一定会叼你
            final WeakReference<Object> sentinelRef = new WeakReference<>(new Object());
            triggerGc();
            if (sentinelRef.get() != null) {
                // System ignored our gc request, we will retry later.
                MatrixLog.d(TAG, "system ignore our gc request, wait for next detection.");
                return Status.RETRY;
            }
```

这个 task 做了很多处理，减少泄漏误判情况，比如，这里它创建了一个弱引用，用来判断 gc 是否真的执行了。需要注意一下，这个 task 是有返回值的，Status.RETRY 表示这个 task 会再次执行。

接下来就是扫描上面说的那个集合：

```java
                // activity 已经被回收了
                if (destroyedActivityInfo.mActivityRef.get() == null) {
                    // The activity was recycled by a gc triggered outside.
                    MatrixLog.v(TAG, "activity with key [%s] was already recycled.", destroyedActivityInfo.mKey);
                    infoIt.remove();
                    continue;
                }
                ++destroyedActivityInfo.mDetectedCount;

                // 泄露检测测试超过一定次数才认为是真的泄露了
                if (destroyedActivityInfo.mDetectedCount < mMaxRedetectTimes
                        || !mResourcePlugin.getConfig().getDetectDebugger()) {
                    // Although the sentinel tell us the activity should have been recycled,
                    // system may still ignore it, so try again until we reach max retry times.
                    MatrixLog.i(TAG, "activity with key [%s] should be recycled but actually still \n"
                                    + "exists in %s times, wait for next detection to confirm.",
                            destroyedActivityInfo.mKey, destroyedActivityInfo.mDetectedCount);
                    continue;
                }
```

这里也是减少误判，只有当一个 activity 泄漏到了一定次数后，才会认为它是真的泄漏了。

后面的逻辑，就是根据不同的模式来进行不同的逻辑处理，这个插件支持的模式如下：

```java
    public enum DumpMode {
        NO_DUMP, AUTO_DUMP, MANUAL_DUMP, SILENCE_DUMP
    }
```

这里只介绍 AUTO_DUMP 模式。

```java
// 如果是 AUTO_DUMP 模式，那么就去自动分析 heap 文件了，与 LeakCanary 类似
final File hprofFile = mHeapDumper.dumpHeap();
if (hprofFile != null) {
    markPublished(destroyedActivityInfo.mActivityName);
    // dump hprof 文件
    final HeapDump heapDump = new HeapDump(hprofFile, destroyedActivityInfo.mKey, destroyedActivityInfo.mActivityName);
    // 处理 dump 出来的 hprof 文件
    mHeapDumpHandler.process(heapDump);
    infoIt.remove();
} else {
    MatrixLog.i(TAG, "heap dump for further analyzing activity with key [%s] was failed, just ignore.",
                destroyedActivityInfo.mKey);
    infoIt.remove();
}
```

我们看看它是怎么处理 hprof 文件的，我之前以为这个模式会与 LeakCanary一样会使用 haha 库来分析泄漏引用链，但是看了源码后，发现没有，这里只是做了 hprof 文件的裁剪。

```java
    public static void shrinkHprofAndReport(Context context, HeapDump heapDump) {
        final Intent intent = new Intent(context, CanaryWorkerService.class);
        intent.setAction(ACTION_SHRINK_HPROF);
        intent.putExtra(EXTRA_PARAM_HEAPDUMP, heapDump);
        enqueueWork(context, CanaryWorkerService.class, JOB_ID, intent);
    }
```

将裁剪任务交给了 JobIntentService。虽然里面有个 MatrixJobIntentService，但是基本上是拷贝的 JobIntentService，在里面做了一些 try catch 操作。

接下来，分析它具体的裁剪代码：

```java
            is = new FileInputStream(hprofIn);
            os = new BufferedOutputStream(new FileOutputStream(hprofOut));
            final HprofReader reader = new HprofReader(new BufferedInputStream(is));
            // 这里是做了一个访问者模式，所以，核心代码都在 visitor 里面
            // 不了解 hprof 的结构。里面的代码没法看
            reader.accept(new HprofInfoCollectVisitor());
            // Reset.
            is.getChannel().position(0);
            reader.accept(new HprofKeptBufferCollectVisitor());
            // Reset.
            is.getChannel().position(0);
            reader.accept(new HprofBufferShrinkVisitor(new HprofWriter(os)));
```

由于我对 hprof 结构也不熟悉，查了些文档也收获不大，所以就不细说了。

里面的逻辑大致分为几步：

- 收集 hprof 文件的 bitmap 与 string 对象（索引id）

  ```java
  @Override
  public void visitStringRecord(ID id, String text, int timestamp, long length) {
      // 主要是处理了 Bitmap 与 String 这两个类
  
      // Bitmap 有个 mBuffer 字段与 mRecycled 字段
      // Bitmap在android sdk < 26之前（> 2.3），存儲像素的byte數組是放在Java層的，26之後是放在native層的。
  
      // String 有个 value 字段
      // String在android sdk < 23之前，存儲字符的byte數組是放在Java層的，23之後是放在native層的。
  
      if (mBitmapClassNameStringId == null && "android.graphics.Bitmap".equals(text)) {
          mBitmapClassNameStringId = id;
      } else if (mMBufferFieldNameStringId == null && "mBuffer".equals(text)) {
          mMBufferFieldNameStringId = id;
      } else if (mMRecycledFieldNameStringId == null && "mRecycled".equals(text)) {
          mMRecycledFieldNameStringId = id;
      } else if (mStringClassNameStringId == null && "java.lang.String".equals(text)) {
          mStringClassNameStringId = id;
      } else if (mValueFieldNameStringId == null && "value".equals(text)) {
          mValueFieldNameStringId = id;
      }
  }
  ```

- 收集 string 与 bitmap 的字段（索引id），重要的是 bitmap 的 mBuffer 的索引

  ```java
  // 找到Bitmap實例
  if (mBmpClassId != null && mBmpClassId.equals(typeId)) {
      ID bufferId = null;
      Boolean isRecycled = null;
      final ByteArrayInputStream bais = new ByteArrayInputStream(instanceData);
      for (Field field : mBmpClassInstanceFields) {
          final ID fieldNameStringId = field.nameId;
          final Type fieldType = Type.getType(field.typeId);
          if (fieldType == null) {
              throw new IllegalStateException("visit bmp instance failed, lost type def of typeId: " + field.typeId);
          }
          // 找到這個實例mBuffer字段的索引id
          if (mMBufferFieldNameStringId.equals(fieldNameStringId)) {
              bufferId = (ID) IOUtil.readValue(bais, fieldType, mIdSize);
          }
          // 找到這個實例mRecycled的boolean值(基礎數據類型，沒有引用關係)
          else if (mMRecycledFieldNameStringId.equals(fieldNameStringId)) {
              isRecycled = (Boolean) IOUtil.readValue(bais, fieldType, mIdSize);
          } else if (bufferId == null || isRecycled == null) {
              IOUtil.skipValue(bais, fieldType, mIdSize);
          } else {
              break;
          }
      }
      bais.close();
      // 確認Bitmap沒有被回收
      final boolean reguardAsNotRecycledBmp = (isRecycled == null || !isRecycled);
      if (bufferId != null && reguardAsNotRecycledBmp && !bufferId.equals(mNullBufferId)) {
          // 將mBuffer對應的byte數組索引id加入集合
          mBmpBufferIds.add(bufferId);
      }
  }
  ```

- 去除**非 bitmap 的 buffer**与**重复的bitmap的buffer**（String 的除外）

  ```java
  if (deduplicatedId != null && !bufferId.equals(deduplicatedId) && !bufferId.equals(mNullBufferId)) {
      // 让重复的 buf 指向同一个
      modifyIdInBuffer(instanceData, bufferIdPos, deduplicatedId);
  }
  ```

  ```java
  // 重複的byte數組索引 重定向之後的 索引id
  final ID deduplicatedID = mBmpBufferIdToDeduplicatedIdMap.get(id);
  // Discard non-bitmap or duplicated bitmap buffer but keep reference key.
  // 将非 bitmap 数组也给裁了
  if (deduplicatedID == null || !id.equals(deduplicatedID)) {
      // 这里判断了 string
      // 也就是说，Hprof文件裁剪的過程主要是裁剪了重複Bitmap的byte[]數據，String 是用来判断的
      if (!mStringValueIds.contains(id)) {
          // 这里直接 return，没有调用 super 方法，
          // 即没有调用 `com.tencent.matrix.resource.hproflib.HprofWriter.HprofHeapDumpWriter.visitHeapDumpPrimitiveArray` 的方法，
          // 所以不会写入数组信息
          return;
      }
  }
  super.visitHeapDumpPrimitiveArray(tag, id, stackId, numElements, typeId, elements);
  ```

  经过这些步骤之后，hprof 文件就裁剪完成了。

裁剪完成之后，会执行 CanaryResultService，回调到：

```java
    private void doReportHprofResult(String resultPath, String activityName) {
        try {
            final JSONObject resultJson = new JSONObject();
//            resultJson = DeviceUtil.getDeviceInfo(resultJson, getApplication());
			// resultPath 裁剪后的hprof压缩文件路径
            resultJson.put(SharePluginInfo.ISSUE_RESULT_PATH, resultPath);
            resultJson.put(SharePluginInfo.ISSUE_ACTIVITY_NAME, activityName);
            Plugin plugin =  Matrix.with().getPluginByClass(ResourcePlugin.class);

            if (plugin != null) {
                plugin.onDetectIssue(new Issue(resultJson));
            }
        } catch (Throwable thr) {
            MatrixLog.printErrStackTrace(TAG, thr, "unexpected exception, skip reporting.");
        }
    }
```

最终会调用，plugin 的 onDetectIssue 方法，这里我们可以进行对应的处理。
