---
title: 依赖aar引起的问题
date: 2020-12-24
index_img: /cover/24.jpg
banner_img: /cover/top.jpg
categories: bugs
---

### gradle 里面依赖  aar 的两种方式

第一种是，将 aar 放入 libs 里面，然后添加下面的代码：

```groovy
implementation fileTree(dir: "libs", include: ["*.aar"])
```

这样，libs 下所有的 aar 文件都会被依赖。

第二种是，将 aar 放入 libs，然后将 libs 作为仓库：

```groovy
android {
    ...

    repositories {
        flatDir {
            dir 'libs'
        }
    }
}
```

然后，在依赖里面，指明要依赖的aar文件：

```groovy
implementation(name: 'mylibrary-debug', ext: 'aar')
```

这两种方式，看起来似乎没啥区别，都是依赖了我们指定的aar，但是在打包的过程中，这两种不同的依赖方式，会产生不同的效果。

### 引起的问题

首先看一下我们的打包环境：

```groovy
dependencies {
    classpath "com.android.tools.build:gradle:3.6.4"
}

distributionUrl=https\://services.gradle.org/distributions/gradle-5.6.4-all.zip
```

再看一下工程结构，很简单：

```
Project
-> app
-> mylibrary
```

当我将 mylibrary 打成 aar 后，放入 app 的 libs 里面，这里我们使用的是上面的**第二种aar依赖方式**，执行 assembleDebug 命令时，报如下错误：

```
Default method desugaring of `com.aprz.mylibrary.LibActivity` failed because its super class `androidx.appcompat.app.AppCompatActivity` is missing
```

我们加上 -s，看看详细信息，如果信息展示不全还可以输出到文件 log.txt：

```
gradlew assembleDebug -s >log.txt 2>&1
```

```
> Transform artifact mylibrary-debug.aar (:mylibrary-debug-d8:) with DexingWithClasspathTransform
D8: Default method desugaring of `com.aprz.mylibrary.LibActivity` failed because its super class `androidx.appcompat.app.AppCompatActivity` is missing

...

Caused by: com.android.tools.r8.utils.b: Error: Default method desugaring of `com.aprz.mylibrary.LibActivity` failed because its super class `androidx.appcompat.app.AppCompatActivity` is missing
        at com.android.tools.r8.utils.y0.a(:21)
        at com.android.tools.r8.utils.O.a(:51)
        ... 101 more
```

我们可以看到，详细信息里面可以看到出错的地方与 r8 有关系。

这个错误是以前没有遇到过的，暂时不知道如何解决，所以**我们使用了第一种aar依赖方式，居然就没有问题了，很神奇**。

后来我去 Google 了，经过一番搜索之后，大概猜测出了使用第二种aar依赖方式引发出错的原因：

> 我们的 AGP 版本太低，自带的 r8 版本低，有 bug。该版本的 r8 无法 desugar **接口默认方法**

因为，上面报错的 LibsActivity 实现了一个接口：

```java
public interface R8 {
    default void test() {
        System.out.println("test");
    }
}
```

这个接口里面有一个默认方法，当我**去掉这个默认方法的时候，再次打包，果然就好了**。

为了证实是该版本的 R8 有问题，可以做下面的验证：

- 升级 AGP，我使用了新建工程的打包环境（4.0.2 + 6.1.1） 没有问题

- 指定 r8 为最新版本，也没有问题

  ```groovy
  buildscript {
      repositories {
          maven {
              url 'http://storage.googleapis.com/r8-release/raw'
          }
      }
  }
  dependencies {
      classpath 'com.android.tools:r8:2.1.67'
  }
  ```

### 还有一个问题

经过一系列的骚操作，我都快忘记了一个问题，那就是就算找出了使用第二种aar依赖方式报错的原因，但是还有一个问题没有解决，那就是，**为啥使用第一种aar依赖方式的时候不会报错**？？？

从出错的日志里面可以看到是`DexingWithClasspathTransform`这个 task 出错了。为了验证一个猜想，我们输出所有 transform 相关信息， 查看两种依赖方式的不同之处。

```
在命令上加上 --info 既可输入 transform 的详细信息
```

搜索输出的日志（搜索 `> Transform artifact mylibrary`）：

```java
	Line 638: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with JetifyTransform
	Line 774: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarToClassTransform
	Line 1150: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with ExtractAarTransform
	Line 1293: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarTransform
	Line 1572: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarTransform
	Line 2223: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with LibrarySymbolTableTransform
	Line 2480: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarResourcesCompilerTransform
	Line 2782: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarTransform
	Line 3067: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarTransform
	Line 3792: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarTransform
	Line 4209: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarTransform
	Line 4873: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarToClassTransform
	Line 5188: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with AarTransform
	Line 5221: > Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with DexingWithClasspathTransform
```

在最后一个记录下，果然发现了错误信息：

```java
> Transform artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with DexingWithClasspathTransform
Transforming artifact mylibrary-debug-d8.aar (:mylibrary-debug-d8:) with DexingWithClasspathTransform
Caching disabled for DexingWithClasspathTransform: C:\Users\aprz512\.gradle\caches\transforms-2\files-2.1\fd035d5206d4a7a1458c993599791019\jetified-mylibrary-debug-d8-runtime.jar because:
  Build cache is disabled
DexingWithClasspathTransform: C:\Users\aprz512\.gradle\caches\transforms-2\files-2.1\fd035d5206d4a7a1458c993599791019\jetified-mylibrary-debug-d8-runtime.jar is not up-to-date because:
  Task has failed previously.
C:\Users\aprz512\.gradle\caches\transforms-2\files-2.1\fd035d5206d4a7a1458c993599791019\jetified-mylibrary-debug-d8-runtime.jar: D8: Type `androidx.appcompat.app.AppCompatActivity` was not found, it is required for default or static interface methods desugaring of `void com.aprz.mylibrary.LibActivity.onCreate(android.os.Bundle)`
D8: Default method desugaring of `com.aprz.mylibrary.LibActivity` failed because its super class `androidx.appcompat.app.AppCompatActivity` is missing
DexingWithClasspathTransform (Thread[Execution worker for ':' Thread 3,5,main]) completed. Took 0.885 secs.
```

那么，我们再看看**使用第一种aar依赖方式，打包的日志（搜索 `> Transform artifact mylibrary`），没有发现任何记录**！！！

这个结果还是符合我的猜想的，那就是使用第一种aar依赖方式的时候，libs 里面的 aar 包并不会经过 DexingWithClasspathTransform 的处理。

所以，这两种依赖 aar 的方式，还是有点区别的。