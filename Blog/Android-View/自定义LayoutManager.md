---
title: 自定义LayoutManager
date: 2019-08-18 11：11：11
tags: Android-View
---


在做了这么长时间的Android开发，还没有遇到过这个需求，不过看了别人的很多效果，感觉很棒，所以找了时间就研究了一下，现在做一些记录，等以后有了相关需要可以快速回顾。



在我学习的过程中，不可避免的遇到了很多问题，有的已经解决，有的还未解决，所以这个 Demo 就是看个乐呵吧。



自定义一个 LayoutManager 整体给我的感觉是与实现自定义 ViewGroup 的 onLayout 比较像。其他的测量绘制方法都不需要我们实现，测量方法还有很多可以直接使用的：

```java
androidx.recyclerview.widget.RecyclerView.LayoutManager#measureChild
androidx.recyclerview.widget.RecyclerView.LayoutManager#measureChildWithMargins
```

这两个方法可以测量 child 的大小，一个不计算 child.layoutParams，一个计算。



测量完成之后，我们就可以获取 child 的大小了：

```java
androidx.recyclerview.widget.RecyclerView.LayoutManager#getDecoratedMeasuredWidth
androidx.recyclerview.widget.RecyclerView.LayoutManager#getDecoratedMeasuredHeight
```

这里不直接使用 child.getMeasureWidth 显然是因为 RecyclerView 是有一个 ItemDecorate 可以设置，不设置就是获取的 getMeasureWidth 的值。



更完美的是，layout child 的时候，也有相应的方法：

```java
androidx.recyclerview.widget.RecyclerView.LayoutManager#layoutDecoratedWithMargins
```

使用这个方法，我们就可以将 child 摆到我们想要摆放的位置了，我们只需要传递四大金刚的位置：l,t,r,b。网上看到一个效果就是，它计算出某个 path 的所有点，然后存起来，根据滚动的距离，来取相应的点，然后根据这个点以及child的大小，就可以将这个 child 摆到 path 的路径上，实现一个 item 跟随 path 的效果。只要想清楚了还是不难的。



了解这些，我们还需要了解一下 RecyclerView 的回收机制，因为 RecyclerView 最吊的地方就是回收复用，如果你搞了一个 LayoutManager 但是却无法回收复用，那岂不是很沙雕，关于回收这里就不仔细讲了，看[我的另一篇文章](https://github.com/aprz512/blog4aprz512/blob/master/Blog/Android-View/RecyclerView 的缓存机制.md) 吧。



有了上面这些基础，我们就可以开始动手写了。

首先，我们需要确定我们想要的效果，我们先看一下这个效果图：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-View/layoutmanager/Gif_20190805_140935.gif?raw=true)

可以看到：

- 最左边，是有几个 item 堆在一起了的，那么是怎么实现的呢，其实就是 layout 的时候将 item 之间摆进一点就好了。比如：item 的宽度是 100，高度是 150，第一个item的位置为 【（0，0），（100， 150）】，第二个 item 的位置（假设 item 的 divider 宽度为 0）为 【（100, 0），（200, 150）】。但是我在摆第二个 item 的时候，我偏不从 100 开始摆，我从 20 开始摆，那么第二个 item 就叠在第一个 item 上面了。
- 右边的就简单了，按照通常的摆法就好了，不搞啥幺蛾子。



对效果了然于胸，我们就可以开始敲代码了，首先自然是继承父类：

> androidx.recyclerview.widget.RecyclerView.LayoutManager#LayoutManager

它只有一个抽象方法：

```kotlin
    override fun generateDefaultLayoutParams(): LayoutParams {
        return LayoutParams(
            LayoutParams.WRAP_CONTENT,
            LayoutParams.WRAP_CONTENT
        )
    }
```

首先，网上一致都是这样实现的，我就很蛋疼了，都不说为什么。

然后，我就去查看了注释，它是这样说的：为 RecyclerView 的 child 生成一个默认的 LayoutParams。那么为何要生成一个默认的 LayoutParams 呢？比如，有的同学在 Adapter 里面加载布局的时候，parent 会传 null，这个时候 child 的就是没有 LayoutParams 的，所以需要生成一个。同样的，PopupWindow 也会遇到。这里我就清楚了，传递 WRAP_CONTENT 是一个保险行为，有最好，没有就使用这个。

注释还说了，这个返回的是 RecyclerView.LayoutParams，所以如果你还想带一些自带的信息在 LayoutParams 里面，你可以继承这个 LayoutParams，然后实现下面3个方法：

```java
androidx.recyclerview.widget.RecyclerView.LayoutManager#checkLayoutParams
androidx.recyclerview.widget.RecyclerView.LayoutManager#generateLayoutParams(android.view.ViewGroup.LayoutParams)
androidx.recyclerview.widget.RecyclerView.LayoutManager#generateLayoutParams(android.content.Context, android.util.AttributeSet)
```

就 OK 了。



搞定了唯一的一个抽象方法，我们就可以正常运行了，但是没啥效果，我们还需要实现  child 的摆放与回收。首先我们搞定 child 的摆放。child 的摆放是在 

> androidx.recyclerview.widget.RecyclerView.LayoutManager#onLayoutChildren 

这个方法里面。我们复写一下：

```kotlin
    override fun onLayoutChildren(recycler: Recycler?, state: State?) {
        super.onLayoutChildren(recycler, state)
    }
```

好奇的点进去看看父类做了什么：

```kotlin
        public void onLayoutChildren(Recycler recycler, State state) {
            Log.e(TAG, "You must override onLayoutChildren(Recycler recycler, State state) ");
        }
```

嗯，他要我们必须重写这个方法，但是又不是抽象的方法，嗯...有趣的女人...

一开始，我是完全不知道应该怎么搞了，所以我看了别人的 demo，发现他们都在最开头搞了这样的一个开头：

```kotlin
        // 这个方法看不出来有啥意义啊，应该是根据
        // androidx.recyclerview.widget.LinearLayoutManager.onLayoutChildren
        // copy 出来的
        if (state?.itemCount == 0) {
            recycler?.apply {
                removeAndRecycleAllViews(this)
            }
            return
        }
```

嗯，我是很奇怪的，为啥要写这个玩意啊，后来我戳到 LinearLayoutManager 学习了一下，发现它有一个类似的代码，但是它是嵌套在 if 里面的：

```java
        if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
            if (state.getItemCount() == 0) {
                removeAndRecycleAllViews(recycler);
                return;
            }
        }
```

这里，我是真的没有搞懂哦。除非是 itemCount 突然变成 0 了，那么需要将所有 child 都移除并且放到 recycler 里面，但是为啥 LinearLayoutManager 是有条件的呢？？？嗯，猜一下与动画相关吧...

好的，我们继续往下，这里只不过是一个前置处理，处理某些特殊情况，下面我们开始摆放 child，为了方便我们写一个方法：

```kotlin
    override fun onLayoutChildren(recycler: Recycler?, state: State?) {
        super.onLayoutChildren(recycler, state)

        // 这个方法看不出来有啥意义啊，应该是根据
        // androidx.recyclerview.widget.LinearLayoutManager.onLayoutChildren
        // copy 出来的
        if (state?.itemCount == 0) {
            recycler?.apply {
                removeAndRecycleAllViews(this)
            }
            return
        }

        layoutChildren(recycler, state)
    }
```

这样看着会舒服一点，一般我不知道咋下手的时候，就会抽一个方法出来，把能写的都写了，至少思路清晰。

开始摆放 child，我们需要计算出第一个 item 的 left：

```kotlin
var left = paddingLeft
```

嗯，很简单。

这个时候，就有一个问题浮现出来了，我们的每个 item 是一样大吗？？？

如果是的话，好说，但是 RecyclerView 就成了岳不群了。如果不是的话，那就复杂了。这里为了简单，我们要求 item 是一样的，毕竟，不一样大也没法堆叠。所以其实仔细想想，实现一个看起来特别酷的效果，限制也很多。

我们定义两个索引值，一个指向屏幕上的第一个 item，一个指向屏幕上的最后一个 item 的后一个位置，就像一个半开区间。

```kotlin
    private var firstVisiblePos = 0
    private var lastVisiblePos = 0
```

然后，我们随便取一个 item，测量一下它的大小：

```	kotlin
        // 需要每个child一样大小
        val firstView: View = recycler.getViewForPosition(firstVisiblePos)
        measureChildWithMargins(firstView, 0, 0)
        unitDistance = getDecoratedMeasuredWidth(firstView) + gap
        // 这个时候，还没有开始摆放，所以用完了再放回去，为了后面的逻辑统一处理
        recycler.recycleView(firstView)
```

这里的逻辑很简单，算出 child 的大小，加上 item 之间的距离，算作一个单元距离，就是一个item从一个位置移动到相邻位置的绝对距离。

这里要说的一个重要的点就是，如果我们需要摆一个 child，只能像 recycler 要，并且，用完了还给 recycler，这样我们神不知鬼不觉的就达成了复用的效果。

接着，我们根据滚动的距离来计算出第一个item的索引：

```kotlin
        // 根据 scroll 的距离来计算 firstPos 的位置
        firstVisiblePos = floor(abs(scrollX).toDouble() / unitDistance).toInt()
```

这里，我们也可以利用 RecyclerView 的宽度算出 lastVisiblePos 的值，但是就有点重复了，我们在摆放 child 的时候去动态的计算会更好一点，所以这里，我们将 lastVisiblePos 赋值为 itemCount。

```kotlin
        // 该值会动态更新
        lastVisiblePos = state.itemCount
```

计算好了两个索引值，我们还需要处理一个问题，就是根据滚动的距离来计算出 View 的偏移距离。

```kotlin
        val frac: Float = (abs(scrollX) % unitDistance) / (unitDistance * 1f)
        val stackOffset = (frac * stackGap).toInt()
        val viewOffset = (frac * unitDistance).toInt()
```

unitDistance 是相邻item的单位距离，frac就表示移动到的百分比。利用这个百分比换算出堆叠区域和普通区域布局起始位置的偏移量，然后可以开始布局了。

```kotlin
// 属于堆叠区域
if (i - firstVisiblePos < MAX_STACK_COUNT) {
    // 手指向左滑动，则 scrollX 的值会越来越大，frac 也会慢慢变大（0 -> 1 为一个周期）
    // item 会向右移动
    // 这里需要减去，item 才会向左移动
    left -= stackOffset
}
```

这里又有几个问题：

- 这个 layoutManager 一初始化就会堆叠起来，导致前面几个的内容看不到了， 解决办法就是做出一个无限循环的效果，这样就会对数目有所限制，至少是知道有多少数据，或者是做成动态的，一开始不会堆叠， 滑动的时候再考虑如何堆叠。
- stackOffset 只需要减去一次，后面的 item 不用重复减去该值，这里我使用了笨办法，搞一个变量标识一下

```kotlin
if (i - firstVisiblePos < MAX_STACK_COUNT) {
    if (!stackOffsetDone) {
        left -= stackOffset
        stackOffsetDone = true
    }
    left += stackGap
}
```

 这个 left 就是 child 布局时用到的值了。对于非堆叠区域同样处理。

后面的代码就简单了：

```kotlin
val view = recycler.getViewForPosition(i)
addView(view)
measureChildWithMargins(view, 0, 0)

val l = left
val t = paddingTop
val r = l + getDecoratedMeasuredWidth(view)
val b = t + getDecoratedMeasuredHeight(view)
layoutDecoratedWithMargins(view, l, t, r, b)
```

把上面的这些逻辑放入循环就搞定了。



这里效果基本就实现了，但是实际上测试的时候会发现，回收复用会有问题。这个有两个方面的问题：

- 在 layout 的时候需要将所有 view 全部 detach 再重新布局
- detach 的 view 被放入了 scrap 中，我们需要将 scrap 中残留的 item 全部放入 pool 中。

所以，我们可以不用自己一个一个的手动回收，而是可以这样：

在 layout 的前面调用 detachAndScrapAttachedViews ，然后在最后回收。

```kotlin
    private fun layoutChildren(
        recycler: Recycler?,
        state: State?
    ) {

        if (recycler == null || state == null) {
            return
        }

        detachAndScrapAttachedViews(recycler)

        ...
			
        val scrapList = recycler.scrapList
        for (i in scrapList.indices) {
            val holder = scrapList[i]
            removeAndRecycleView(holder.itemView, recycler)
        }

    }
```

实际上我测试的效果比之前好了很多，但是我还是不太满意，因为有的时候还是会出现 createViewHolder 的调用，虽然次数极少，但是肯定是哪里出了问题，不然是不会这样的。



所以建议还是手动的一个个标记回收。我也对回收这个问题还有疑问，所以这里就不说下去了。



除了回收还有一个问题，就是关于 fling 效果。我测试这个demo 的时候，发现我从最后一个一下滑动到第一个的时候，item停留的位置总是不对，我一直以为是我的计算有问题：

```kotlin
    /**
     * dx(dy) 表示本次较于上一次的偏移量，<0为 向右(下) 滚动，>0为向左(上) 滚动；
     * 这个算法还是无法满足 fling 的要求，fling 的时候 停留的位置不对
     * 查了一些资料，可能还需要自定义一个 SnapHelper -> https://www.jianshu.com/p/0e4a93d8e2de
     */
    private fun consume(dx: Int): Int {
        val consumed: Int
        // dx < 0 表示向右滚动，需要显示左边的内容
        if (dx < 0) {
            // 到了最左边
            if (scrollX + dx < 0) {
                consumed = if (scrollX > 0) {
                    dx - scrollX
                } else {
                    0
                }
                scrollX = 0
                return consumed
            }
        }

        // dx > 0 表示向左滚动，右边的内容需要显示出来
        if (dx > 0) {
            if (scrollX + dx > maxScrollX) {
                consumed = if (scrollX < maxScrollX) {
                    maxScrollX - scrollX
                } else {
                    0
                }
                scrollX = maxScrollX
                return consumed
            }
        }

        scrollX += dx

        return dx
    }
```

但是，经过我的打印，发现，这个计算是没有问题了。

顺便说一下，要想滑动，需要重写 LayoutManager 的方法：

```kotlin
    override fun canScrollHorizontally(): Boolean {
        return true
    }
```

然后，在 scrollHorizontallyBy 返回消耗的值：

```kotlin
    override fun scrollHorizontallyBy(dx: Int, recycler: Recycler?, state: State?): Int {

        if (dx == 0 || state?.itemCount == 0) {
            return 0
        }

        layoutChildren(recycler, state)

        return consume(dx)
    }
```

我就感觉这个与 NestedScrolling 接口很相似。

跑题了，关于 fling，查了一些资料，可能还需要自定义一个 SnapHelper -> https://www.jianshu.com/p/0e4a93d8e2de。

当然你也可以自己监控滑动状态，然后自动的调整滑动位置，就像画廊一样。

项目地址：

https://github.com/aprz512/LayoutManagerDemo