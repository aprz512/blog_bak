---
title: Bitmap
index_img: /cover/18.jpg
banner_img: /cover/top.jpg
date: 2019-08-18
categories: Android
---


记录一下以前看到过的知识点。

### Bitmap 像素数据的储存位置

先看[官方文档](https://developer.android.com/topic/performance/graphics/manage-memory)：

> 在Android 3.0之前，Bitmap对象放在Java堆，⽽像素数据是放在Native内存中。如果不⼿动调⽤recycle，Bitmap Native内存的回收完全依赖finalize函数回调，熟悉Java的同学应该知道，这个时机不太可控。 
>
> Android 3.0～Android 7.0将Bitmap对象和像素数据统⼀放到Java堆中，这样就算我们不调⽤recycle，Bitmap内存也会随 着对象⼀起被回收。不过Bitmap是内存消耗的⼤户，把它的内存放到Java堆中似乎不是那么美妙。即使是最新的华为Mate 20，最⼤的Java堆限制也才到512MB，可能我的物理内存还有5GB，但是应⽤还是会因为Java堆内存不⾜导致OOM。 
>
> Bitmap放到Java堆的另外⼀个问题会引起⼤量的GC，对系统内存也没有完全利⽤起来。 有没有⼀种实现，可以将Bitmap内存放到Native中，也可以做到和对象⼀起快速释放，同时GC的时候也能考虑这些内存防 ⽌被滥⽤？NativeAllocationRegistry可以⼀次满⾜你这三个要求，Android 8.0正是使⽤这个辅助回收Native内存的机制，来实现像素数据放到Native内存中。Android 8.0还新增了硬件位图Hardware Bitmap，它可以减少图⽚内存并提升绘制效率。

假设我们有这样的一个手机，它的 system/build.prop 配置如下：

```java
// 表示应用程序启动后为其分配的初始大小为8m
dalvik.vm.heapstartsize=8m

// 每个应用程序最大内存可分配到64m
dalvik.vm.heapgrowthlimit=192m

// 单个虚拟机可分配的最大内存256m
// 使用大堆时，极限堆大小。一旦dalvik heap size超过这个值，直接引发oom。
// 在android开发中，如果要使用大堆，需要在manifest中指定android:largeHeap为true。这样dvm heap最大可达dalvik.vm.heapsize。
dalvik.vm.heapsize=512m

//  设定内存利用率的百分比，当实际的利用率偏离这个百分比的时候，虚拟机会在GC的时候调整堆内存大小，让实际占用率向个百分比靠拢。
dalvik.vm.heaptargetutilization=0.75

dalvik.vm.heapminfree=512k

dalvik.vm.heapmaxfree=8m
```

我们不断的解析图片并持有所有图片的引用：

```java
void test{
	Map<String, Bitmap> map = new HashMap<>();
	for(int i=0 ; i<10;i++) {
		Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.green);
		map.put("" + System.currentTimeMillis(), bitmap);
	}
}
```

在 6.0 的手机中，实际测试应用出现OOM的时候，是在 200M 左右。对于现在动辄6G内存的手机而言，存在严重的资源浪费。所以8.0之后，Android也向这个方向靠拢，最好的下手对象就是Bitmap，因为它是耗内存大户。图片内存被转移到native之后，一个APP的图片处理不仅能使用系统绝大多数内存，还能降低Java层内存使用，减少OOM风险。

在 8.0 的手机中，实际测试应用出现OOM的时候，是在 2G 左右，所以，内存无限增长的情况下，也会导致APP崩溃，但是这种崩溃已经不是OOM崩溃了，Java虚拟机也不会捕获。

注意，**上面的测试都是针对图片而言**。

### Bitmap 的复用

- 一是使用缓存：使用LruCache对Bitmap进行缓存，当再次使用到这个Bitmap的时候直接获取，而不用重走编码流程。

- 二是使用 inBitmap 字段：
  Android3.0(API 11之后)引入了BitmapFactory.Options.inBitmap字段，设置此字段之后解码方法会尝试复用一张存在的Bitmap。这意味着Bitmap的内存被复用，避免了内存的回收及申请过程，显然性能表现更佳。不过，使用这个字段有几点限制：

  ```java
    声明可被复用的Bitmap必须设置inMutable为true；
    Android4.4(API 19)之前只有格式为jpg、png，同等宽高（要求苛刻），inSampleSize为1的Bitmap才可以复用；
    Android4.4(API 19)之前被复用的Bitmap的inPreferredConfig会覆盖待分配内存的Bitmap设置的inPreferredConfig；
    Android4.4(API 19)之前待加载Bitmap的Options.inSampleSize必须明确指定为1。
    Android4.4(API 19)之后被复用的Bitmap的内存必须大于需要申请内存的Bitmap的内存；
  ```

  **使用 inBitmap 之前**：

  

  ![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E7%9F%A5%E8%AF%86%E7%82%B9/Bitmap/20190708114053537.png?raw=true)



**使用 inBitmap 之后**：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E7%9F%A5%E8%AF%86%E7%82%B9/Bitmap/20190708114228791.png?raw=true)

我刚开始看到这两张图的时候是很蛋疼的，因为它看起来就像是3张图片同时使用了一块内存，这特么怎么可能呢。但是实际上它是这样工作的：

> 假设我们需要在Android应用程序中加载一些图。 当我们加载bitmap1时，它将为bitmap1分配内存。 然后，如果我们不再需要bitmap1，请不要回收位图（因为回收涉及调用GC）。相反，使用此bitmap1作为bitmap2的inBitmap。这样，bitmap2可以复用bitmap1的内存位置。

### Bitmap 占用的内存大小

- getByteCount()：代表存储Bitmap的像素需要的最少内存。
- getAllocationByteCount()：代表在内存中为Bitmap分配的内存大小。

一般情况下两者是相等的。但是通过复用Bitmap来解码图片，如果被复用的Bitmap的内存比待分配内存的Bitmap大,那么getByteCount()表示新解码图片占用内存的大小（并非实际内存大小,实际大小是复用的那个Bitmap的大小），getAllocationByteCount()表示被复用Bitmap真实占用的内存大小（即mBuffer的长度）。

### 如何计算Bitmap占用的内存大小

公式：**占用的内存 = width \* height \* 一个像素所占的内存**。

一般情况下是正确的，但是有时候还需要考虑屏幕密度问题。比如，我们从资源文件中加载一张图片（BitmapFactory.decodeResource）：

> 它占用的内存 = width * height * nTargetDensity/inDensity * nTargetDensity/inDensity * 一个像素所占的内存

nTargetDensity/inDensity 实际上就是图片被缩放了，因为屏幕密度与图片资源文件夹密度不一致时，系统就会缩放图片，所以这里就会影响到计算结果。

除了加载本地资源文件的解码方法会默认使用资源所处文件夹对应密度和手机系统密度进行缩放之外，别的解码方法默认都不会。

### Bitmap 的压缩

#### inSampleSize

不多说了，注意会将所设置的值自动更正为2的幂次方（接近并且小于所设置的值），有人说不是所有版本都这样，但也无从考究了。

#### compress

压缩了文件的质量。这个玩意要了解还需要一定的储备知识，让我们从头说起。

首先我们需要了解的是图片各种相关的大小，文件大小、占用硬盘大小、占用内存大小。对于文件的大小实际就是它本身的包含的信息的大小，即实际具有的字节数，它以Byte为衡量单位，只要文件内容和格式不发生变化，文件大小就不会发生变化。我们可以直接拿File的length()方法就能获得文件的大小。那么硬盘的大小是什么呢？不管在什么系统中，我们经常会看到文件大小和占用磁盘空间的大小不一致，这是为什么呢？

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E7%9F%A5%E8%AF%86%E7%82%B9/Bitmap/20190708140031172.png?raw=true)

文件在磁盘上的所占空间不是以Byte为衡量单位的，它最小的计量单位是“簇(Cluster)”。 扇区是磁盘最小的物理存储单元，但由于操作系统无法对数目众多的扇区进行寻址，所以操作系统就将相邻的扇区组合在一起，形成一个簇，然后再对簇进行管理。每个簇可以包括2、4、8、16、32或64个扇区。显然，簇是操作系统所使用的逻辑概念，而非磁盘的物理特性。 为了更好地管理磁盘空间和更高效地从硬盘读取数据，**操作系统规定一个簇中只能放置一个文件的内容，因此文件所占用的空间，只能是簇的整数倍；而如果文件实际大小小于一簇，它也要占一簇的空间。**从上面的图可以看出，Windows 的一簇是4KB，所以，一般情况下文件所占空间要略大于文件的实际大小，只有在少数情况下，即文件的实际大小恰好是簇的整数倍时，文件的实际大小才会与所占空间完全一致。

这样我们就理解了为什么文件大小和实际占用硬盘大小不一样了。那么现在就剩内存的大小了。
图片在内存里占用的大小和它本身的大小没有关系。我们知道图像是由一个一个的像素组成的，我们的图片的尺寸就是像素的多少，例如宽高是1024*1024的图像就是横竖都有1024个像素的图片，根据图片的编码格式不同，每个像素所占的内存大小是不一样的：

- ALPHA_8：表示8位Alpha位图,即A=8,一个像素点占用1个字节,它没有颜色,只有透明度

- ARGB_4444：表示16位ARGB位图，即A=4,R=4,G=4,B=4,一个像素点占4+4+4+4=16位，2个字节

- ARGB_8888：表示32位ARGB位图，即A=8,R=8,G=8,B=8,一个像素点占8+8+8+8=32位，4个字节

- RGB_565：表示16位RGB位图,即R=5,G=6,B=5,它没有透明度,一个像素点占5+6+5=16位，2个字节

其中A代表透明度；R代表红色；G代表绿色；B代表蓝色。

最终，图片在内存所占空间的大小是：图片长度 x 图片宽度 x 一个像素点占用的字节数。即，例如我们有一个1024\*1024大小的ARGB的图，那么它在内存里的大小为1024\*1024\*4=4M，但是在硬盘上的大小甚至可以小到6.4k。

这是为什么，为什么差别这么大？因为图片在内存中时是完整的图片信息，例如即使一个图是全白不透明或全黑全透明也会全部在内存中 (FFFFFFFF/00000000) 占用空间。但是在硬盘上却是被压缩的状态，例如平时我们常见的jpg和png，都是将图片信息进行了压缩，然后存储在了硬盘上。所以说一个图片在内存中占用的空间要远大于在硬盘的空间。

以上说了这么多，最后的引出的结论很关键：**jpg和png都是对图片信息进行压缩然后存储到硬盘上的。它们有什么区别呢？jpg实际是有损压缩，而png是无损压缩。**现在来看最上面的图片压缩的方法，可以看到Bitmap的压缩格式是JPEG（即jpg）。为什么是jpg，因为只有jpg才支持压缩，png是无损的，根本就不能进行再压缩。所以说Bitmap的compress方法只能对jpg起作用，当然，局限不仅仅是这一点。我们不禁要问，jpg能再压缩，那到底压缩了什么呢？

首先jpg与png不同，png支持透明度，但是jpg不支持，所以jpg本身就比png小了四分之一的空间。其次jpg是有损压缩，除了透明度被干掉外，本身的RGB颜色也被压缩了，当然了，**压缩的算法非常复杂，不在本文的研究范围内。压缩本身也是有等级的，压缩的越厉害，图像失真也越厉害，但是最终压缩都是有上限的，就是说从算法上来说就不支持任意一个图片压缩到任意小**。我们可以**参考PS工具**最后保存jpg图片的质量（品质）那个选项，可以选择从0到100，0当然代表质量最差了，我们选择0那么就会出来最小的图片，这个应该就是这张jpg图片能够压缩到的最小值。有人说可以把压缩到0后的图片再去重复一遍这个步骤就好了嘛。实际这样是不行的，这里要说明的是0到100是一个绝对值，就是说一个图片的质量（品质）就是0到100，**不能循环压缩**，你可以试一下，用一个已经是0得图片再次压缩到50，那么它的大小不但不会小，反而会增大！！！

最后，compress方法是质量压缩，压缩后改变大小的是jpg文件的大小，图片在内存中的大小还是不变的！因为宽高和每个像素占的空间都是没有变化的！

#### inDensity 与 inTargetDensity

这个是我在《Android权威编程指南》上看到的。

上面说的，inSampleSize 只能缩放 2 的幂次方。这个在某些情况下可能不好满足需求，比如：一张1200 * 1200 的图，想要缩放到 500 * 500。这个时候使用 inSampleSize 就搞不定了！

所以可以借助于 inDensity 和 inTargetDensity，用 inDensity 与 inTargetDensity 就能做到任意等比缩放。

可以先使用 inSampleSize 缩小到 600 * 600（不直接使用 inDensity 与 inTargetDensity的原因书上没有说，但是我猜想应该是它的效率高），然后再使用 inDensity 与 inTargetDensity ，将 inDensity 设置成 600，将 inTargetDensity 设置成 500，就可以了。

实际上原理，就是与 android.graphics.BitmapFactory#decodeResourceStream 差不多啦。

给出项目代码：

```java
    public static int calInSampleSize(BitmapFactory.Options options,
                                      int reqWidth, int reqHeight) {
        final int height = options.outHeight;
        final int width = options.outWidth;
        int inSampleSize = 1;
        if (height > reqHeight || width > reqWidth) {
            final int halfHeight = height / 2;
            final int halfWidth = width / 2;
            while ((halfHeight / inSampleSize) > reqHeight
                    && (halfWidth / inSampleSize) > reqWidth) {
                inSampleSize *= 2;
            }
        }
        return inSampleSize;
    }
```

```java
    public static BitmapFactory.Options getOptions(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
        options.inSampleSize = calInSampleSize(options, reqWidth, reqHeight);
        options.inScaled = true;
        // 这里使用设置要缩放的宽高比，与 decodeResource 的缩放一样
        options.inDensity = options.outHeight;
        options.inTargetDensity = reqHeight * options.inSampleSize;
        options.inJustDecodeBounds = false;
        return options;
    }

```

使用这个方法就可以做出任意的等比例缩放了。
