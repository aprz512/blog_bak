
---
title: Parcelable与Serializable
date: 2019-08-18 11：11：11
tags: Android-思考
---


通常，在Android里面传递数据都是使用的 Intent，而 Intent 里面想要放一个 Java 对象，那这个对象要么实现 Parcelable 要么实现 Serializable 。

一般我们都是选择实现 Parcelable，因为它的**效率高**，而且不会产生很多的临时对象。

当然这些都是网上说的，并没有证明，现在我们来测试一下，从传输效率开始。当然我也懒得做测试的，所以我找了一篇[文章](http://www.developerphil.com/parcelable-vs-serializable/)，个人觉得还是可以的，有好奇心的小伙伴可以自己测试一下。

测试过程如下：

- 模拟将对象传递给Activity的过程：直接调用 [Bundle#writeToParcel(Parcel, int)](https://developer.android.com/reference/android/os/Bundle.html#writeToParcel(android.os.Parcel, int))，然后再取出来。
- 循环 1000 次
- 做10次测试，取平均值
- 在多个设备上测试

下面给出该作者测试的结果：

![](http://www.developerphil.com/assets/parcelable-vs-serializable-e1366334109758.png)

> ###### Nexus 10
>
> Serializable: 1.0004ms,  Parcelable: 0.0850ms - 10.16x improvement.
>
> ###### Nexus 4
>
> Serializable: 1.8539ms - Parcelable: 0.1824ms - 11.80x improvement.
>
> ###### Desire Z
>
> Serializable: 5.1224ms - Parcelable: 0.2938ms - 17.36x improvement.

可以看出，区别还是很明显的。可能几ms对我们人类来说，感觉不到什么，但是对于CPU就像是过了几个月了。特别是Android需要16ms来绘制一帧，你传递一个对象就花了几ms，就没剩多少时间了。

好了说完了效率，我们再说说**为啥 Serializable 会产生很多临时对象**。

看一个简单的对象：

```java
public class Phone implements Serializable{

    public String name;
    public String address;

    public Phone() {
    }

    public Phone(String name, String address) {
        this.name = name;
        this.address = address;
    }
}
```

当我们把这个对象序列化到文件中的时候，它的内容如下：

```
aced 0005 7372 0011 636f 6d2e 6578 616d
706c 652e 5068 6f6e 6551 4868 16d4 8afd
8702 0002 4c00 0761 6464 7265 7373 7400
124c 6a61 7661 2f6c 616e 672f 5374 7269
6e67 3b4c 0004 6e61 6d65 7100 7e00 0178
7074 0007 6265 696a 696e 6774 0008 7a68
616e 6773 616e
```

我就不解释一个个解释它们代表什么了，这些数据包含了如下内容：

- 序列化协议，固定值
- 序列化协议版本
- 对象的开始标记，类的开始标记
- 类名长度，类名
- 有多少个字段
- 每个字段长度，字段名，字段值

嗯，还有一些东西，特别是有父类的，更蛋疼。可以看到，本来我们只需要传递 name 与 address 的值，它给我们搞了一大堆东西，非常的浪费。还有反序列的时候，由于它使用了反射机制，所以会产生很多的临时变量，可能会增加GC的频率。

Parcelable 的序列化与反序列化的过程就不一样了，它不像 Serializable 那么严格，它只存放了变量的值，其他的都没有存放。我们看一下实现 Parcelable 的过程。

```java
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeString(type);
        dest.writeString(accessId);
    }
```

可以看到，我们将对象的属性值写入到了 parcel 中。

而 parcel 里面可以理解为有一块内存空间，用来储存这些属性值，还有一个指针指向准备写入的位置，写入一个属性后就往后偏移一段距离。所以读取的时候，一定要按照写入的顺序，否则会出错。

然后在使用 parcel 读取出来：

```java
public Account createFromParcel(Parcel source) {
    return new Account(source);
}

public Account(Parcel in) {
    this.name = in.readString();
    this.type = in.readString();
    this.accessId = in.readString();
}
```

序列化与反序列化的核心都是 Parcel。

Parcel 的注释上说了，它是消息的容器，可以通过 IBinder 来传输。



我们再来思考一下，Bundle 中的 Serializable 数据是如何传输的。

> android.os.Parcel#writeValue

```java
 			else if (v instanceof Serializable) {
                // Must be last
                writeInt(VAL_SERIALIZABLE);
                writeSerializable((Serializable) v);
            }
```

writeSerializable 里面仍然是调用了 `java.io.ObjectOutputStream#writeObject` 方法来写对象。



> android.os.Parcel#readSerializable(java.lang.ClassLoader)

同样的，读取一个 Serializable 对象的时候，也是通过 `java.io.ObjectInputStream#readObject`  来获取的，走的是 Java 的序列化与反序列化逻辑。