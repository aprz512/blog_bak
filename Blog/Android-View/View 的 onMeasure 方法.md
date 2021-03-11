---
title: View 的 onMeasure 方法
index_img: /cover/18.jpg
banner_img: /cover/top.jpg
date: 2019-08-18
categories: View
---




View 的测量过程中，有一个比较重要的类需要掌握：MeasureSpec。我们在阅读源码的时候会发现，在 View 的测量过程中，MeasureSpec 是一个会经常出现的类，如果不先掌握这个类的话，是没法阅读下去的。

MeasureSpec 会在很大程度上决定一个 View 的尺寸规格，之所以是很大程度上是因为这个过程还受父容器的影响，因为父容器影响 View 的 MesaureSpec 的创建过程。

在测量过程中，系统会将 View 的 LayoutParams 根据父容器所施加的规则转换成对应的 MeasureSpec，然后再根据这个 MeasureSpec 来测量出 View 的宽/高。

### MeasureSpec

MeasureSpec 是一个int值，但是这个int值被分为了两部分，一部分表示 SpecMode （测量模式），一部分表示 SpecSize（在某种测量模式下的大小）。

可能有很多人想不通，一个int型整数怎么可以表示两个东西（大小模式和大小的值），一个int类型我们知道有32位。而**模式有三种**，要表示三种状态，至少得2位二进制位。于是系统采用了最高的2位表示模式。如图：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-View/View/1346374636_1935.png?raw=true)

最高两位是00的时候表示"未指定模式"，即MeasureSpec.UNSPECIFIED。

最高两位是01的时候表示"'精确模式"，即MeasureSpec.EXACTLY。

最高两位是11的时候表示"最大模式"，即MeasureSpec.AT_MOST。

- 精确模式（MeasureSpec.EXACTLY）

  在这种模式下，尺寸的值是多少，那么这个组件的长或宽就是多少。

- 最大模式（MeasureSpec.AT_MOST）

  这个也就是父组件，能够给出的最大的空间，当前组件的长或宽最大只能为这么大，当然也可以比这个小。

- 未指定模式（MeasureSpec.UNSPECIFIED）

  这个就是说，当前组件，可以随便用空间，不受限制。

MeasureSpec 通过将 SpecMode 与 SpecSize 打包成一个 int 值来避免过多的对象内存分配。为了方便操作，它还提供了对应的打包与解包方法。

```java
// 将 size 与 mode 组合成一个 MeasureSpec 对象
android.view.View.MeasureSpec#makeMeasureSpec

// 从 MeasureSpec 中获取 mode
android.view.View.MeasureSpec#getMode

// 从 MeasureSpec 中获取 size
android.view.View.MeasureSpec#getSize
```



在 View 测量的时候，系统会将 LayoutParams 在父容器的约束下转换成对应的 MeasureSpec，然后再根据这个 MeasureSpec 来确定 View 测量后的大小。（**这里需要注意，MeasureSpec 不是由 LayoutParams 唯一决定的，而是由 LayoutParams 与父布局一起决定的**）



### 各种 View 测量的区别

#### 顶层 View

我们知道一般的 View 都会有父布局，但是最顶层的 View 是没有的，那么它是如何测量的呢？

首先它会获取 LayoutParams，再判断 LayoutParams 宽高的值：

- 如果为 LayoutParams.MATCH_PARENT，这表示精确模式，大小就是窗口大小。
- 如果为 LayoutParams.WRAP_CONTENT，这表示最大模式，大小未定，但是不能超过窗口大小。

这就比较简单了，顶层View的测量，一般宽高都是 LayoutParams.MATCH_PARENT，大小为窗口大小。

#### 子 View

对于普通的 View 来说，它的测量与父布局有关，而每个父布局的特性又不同，无法每个都涉及到，所以这里采取一个“管中窥豹，可见一斑”的方法。

这里介绍一下 ViewGroup 的 measureChildWithMargins 方法。

> android.view.ViewGroup#measureChildWithMargins

```java
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

方法中的 getChildMeasureSpec 方法比较长，就不贴代码了，下面会有文字说明。

getChildMeasureSpec 其实最后就是生成了一个 MeasureSpec 对象。

它的 size 由**两部分决定**，一个是 `parentWidthMeasureSpec`，一个是 `lp.width / lp.height`。

具体的规则用表说明：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-View/View/捕获.PNG?raw=true)

从这个表中我们可以看到，在构造 child 的 MeasureSpec 还是优先考虑了 child 自身的 size 的，特别是 child 直接要求一个固定的值的时候。

我们深入思考一下，比如我们经常使用到的 LinearLayout（竖向），它在决定 child 的大小的时，肯定不能让 child 的高度与自己的高度一样大，那么它是如何处理的呢？我们看看源码：

> android.widget.LinearLayout#measureVertical

```java
final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(Math.max(0, childHeight), MeasureSpec.EXACTLY);
final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,lp.width);
child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
```

可以看到，LinearLayout 并没有使用 ViewGroup 提供的 measureChild 方法，因为它并不符合 LinearLayout 的特性。那么哪一个布局符合呢？？？FrameLayout！！！

> android.widget.FrameLayout#onMeasure

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();
			
        ...

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
				...
            }
        }
        ...
    }
```

可以看到，它就是直接使用了 measureChildWithMargins 方法，因为它的特性只是做了一个层叠，并没有其他对 child 有其他限制，所以可以直接使用。



### 例子

说了这么多，我们来看看一个实际的应用场景吧。

我们知道，TextView 是有自己的 onMeasure 方法的。系统提供的 TextView，文字是从做到右的，那么我们现在想做这样的一个效果，将 TextView 的显示旋转一下，从上到下。如下图：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-View/View/捕获1.PNG?raw=true)

变成这样：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-View/View/捕获2.PNG?raw=true)

那么，有的同学就说话了，这旋转一下不就不可以了吗！是这样吗？在我们的屏幕中，不可能只会有一个 TextView，所以这个 TextView 很可能会与其他控件一起排列，当我们在旋转之前，假设它的宽与高是 100\*300，那么旋转之后，它的高度变成了300，这个时候，由于父布局的限制，我们很可能看不到整个 TextView。而且我们在 xml 中也不好去预览它的效果。

那么下面，我们就来实现一下这个效果。

首先我们需要处理该控件的大小。

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.measure(heightMeasureSpec, widthMeasureSpec);
        setMeasuredDimension(getMeasuredHeight(), getMeasuredWidth());
    }
```

我们首先来看这个方法：`super.measure(heightMeasureSpec, widthMeasureSpec);`。

这行代码非常容易引起误解，**有的人就以为，我们将 widthMeasureSpec 与 heightMeasureSpec 换了一下，那么它测量出来的宽高自然就会互换。这是错误的理解！！！**

比如，我们在 xml 中是这样使用这个自定义控件的：

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res/com.yoog.widget"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity" >
    
    <com.yoog.widget.VerticalTextView
        android:text="20:59"                              
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
    
</RelativeLayout>

```

我们假设`20：59`这串文字的长度为 300，高度为 100。父布局的宽高均大于 300。

所以，TextView 测量出来的就是 300\*100，VerticalTextView 测量出来的值应该是 100 \* 300。

但是，如果父布局的高度只有 250 的时候，横着测量的时候，是完全没有问题的，仍然测量出 300 \* 100，但是竖着测量的时候，高度只能有 250，它需要换行，所以，结果是 100 \* 250。

所以说，**将 widthMeasureSpec 与 heightMeasureSpec 互换，只是为了正确的将父布局对 child 的影响正确的传递进去**。

由于，正确的传递了父容器的宽与高，走 TextView 的方法自然就会测量出正确的值，然后我们调用 `setMeasuredDimension(getMeasuredHeight(), getMeasuredWidth());` 方法，就可以将宽与高换过来了。

至于 onDraw 方法，我们就不深入了，只需要在 view 的左上角旋转一下画布就好了。

