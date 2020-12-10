---
title: Apk 打包流程及扩展
date: 2019-08-18 11：11：11
categories: Android
---



我们开发一个应用，大概会有这些东西：

- 源代码

- 三方库

- 图片，so等一些资源

  

那我们从源文件说起，一个项目的源文件一般都是java文件，有需要还可能会有AIDL 文件。

> Java文件会毫无疑问的被 javac 编译成 class 文件。
>
> 而 AIDL 文件经过 aidl 工具编译之后也会生成相应的 Java 代码，被编译成 class 文件。
>
> 还有一个不要忘记了，资源文件经过编译之后会生成 R.java 文件，它也会被编译成 class 文件。

这里扩展一下，我们知道 app 的 R.java 文件里面的变量都是 final 的，如下：

```java
public static final int main=0x7f030004;
```

而 lib 工程的 R.java 文件的变量却不是 final 的（从 ADT14 开始），如下：

```java
public static int main=0x7f030004;
```

那么这个又是什么原因呢？

其实很简单，如果lib中 R.java 文件里面的变量是 final 的，那么会有两个问题：

- 编译速度：每次编译都需要将所有的资源与代码都编译一次，以免变量值产生碰撞
- lib无法复用：如果生成的值是 final 的，那么这个lib给别人用的时候，值很可能与其他 lib 一样，导致问题

所以，为了避免上面的问题，就将变量改为非 final 的，从而 lib 中使用 id 的时候，就只能使用 if-else，而不是使用 switch。



好的，源代码我们处理完了，接下来就需要处理 class 文件了，class 文件除了上面生成的之外，还有我们引入的三方库，他们也是 class 文件。

> dx工具会将这些 class 文件打包成 dex 文件。
>
> 主要工作是将Java字节码转成成Dalvik字节码、压缩常量池、消除冗余信息等。



我们回头在说说资源文件的处理。

Android中的资源文件有那些呢?

res 文件夹下的资源，以及 assets 目录下的资源。

res 资源经过 aapt 的编译之后，会编译为二进制文件。

> 会为每个文件赋予一个resource id。**对于该类资源的访问，应用层代码则是通过resource id进行访问的**。
>
> 生成一个resource.arsc文件，resource.arsc文件相当于一个文件索引表，记录了很多跟资源相关的信息。

而 assets 的资源保持不动，所以我们只能通过名字来获取它。



资源处理完了，class文件也达成dex包了，接下来就要使用 apkBuilder 将**编译后的资源、dex文件、so文件**等等打成 apk 包了。



打好apk包了之后，需要签名，因为一旦APK文件生成，它必须被签名才能被安装在设备上。



签好名之后，还需要对**apk文件进行对齐处理**。那么对齐的作用是什么呢？让我们细细道来。

在开发人员的眼中，CPU是这样访问内存的：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%AF%8F%E6%97%A5%E4%B8%80%E9%97%AE/apk%E6%89%93%E5%8C%85/howProgrammersSeeMemory.jpg?raw=true)

CPU 读取内存中的数据是一个一个读取的。

然而，实际上CPU是这样读取内存数据的：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%AF%8F%E6%97%A5%E4%B8%80%E9%97%AE/apk%E6%89%93%E5%8C%85/howProcessorsSeeMemory.jpg?raw=true)

它是一块一块的读取的，CPU把内存当成是一块一块的，块的大小可以是2，4，8，16字节大小，因此CPU在读取内存时是一块一块进行读取的，可以当成是内存读取粒度。

那么这样会导致啥问题呢？假设我们需要读取一个 int 值到寄存器，分两种情况讨论：

- 数据从地址0开始
- 数据从地址1开始

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%AF%8F%E6%97%A5%E4%B8%80%E9%97%AE/apk%E6%89%93%E5%8C%85/singleByteAccess.jpg?raw=true)

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%AF%8F%E6%97%A5%E4%B8%80%E9%97%AE/apk%E6%89%93%E5%8C%85/doubleByteAccess.jpg?raw=true)

当该数据是从0字节开始时，很CPU只需读取内存一次即可把这4字节的数据完全读取到寄存器中。

当该数据是从1字节开始时，问题变的有些复杂，此时该int型数据不是位于内存读取边界上，这就是一类内存未对齐的数据。此时CPU先访问一次内存，读取0—3字节的数据进寄存器，并再次读取4—5字节的数据进寄存器，接着把0字节和6，7，8字节的数据剔除，最后合并1，2，3，4字节的数据进寄存器。对一个内存未对齐的数据进行了这么多额外的操作，大大降低了CPU性能。



最后贴一张Google官方为我们提供的详细的构建过程图：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%AF%8F%E6%97%A5%E4%B8%80%E9%97%AE/apk%E6%89%93%E5%8C%85/1612d19485723038?raw=true)



PS：

AnnotationProcessor 发生在 java -> class 之前，因为它需要生成 java 文件。

Transform 发生在 class -> dex 之前，它需要修改class。

Proguard 发生在 Transform 之后，因为混淆后就找不到方法名了。

流程是这样的：

`annotationProcessor` ->` javac `-> `proguard` -> `Transform `

