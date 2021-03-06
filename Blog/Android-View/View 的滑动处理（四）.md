---
title: View 的滑动处理（四）
index_img: /cover/21.jpg
banner_img: /cover/top.jpg
date: 2020-12-21
categories: View
---

> 这个例子可能举得不太恰当，应该用 NestedScrollView距离，因为在不同的版本上，下面的例子有不同的滑动效果。
>
> 例子基于 API 28

看下面这个布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">

            <Button
                android:layout_width="match_parent"
                android:layout_height="120dp"
                android:gravity="center"
                android:text="btn_01" />


            <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/recycler_view"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:background="@color/teal_200" />

            <Button
                android:layout_width="match_parent"
                android:layout_height="120dp"
                android:gravity="center"
                android:text="btn_01" />

            <Button
                android:layout_width="match_parent"
                android:layout_height="120dp"
                android:gravity="center"
                android:text="btn_01" />
        </LinearLayout>
    </ScrollView>
</androidx.constraintlayout.widget.ConstraintLayout>
```

忽略，其他无关的部分，其实就是一个 ScrollView 嵌套了一个 RecyclerView。

按照事件拦截机制来看，当 RecyclerView 滑动到底部的时候，应该就不能滑动了，而且 ScrollView 也无法滑动才对。

但是实际行为却不是这样的，实际上它们的滑动效果可以无缝链接，当 RecyclerView 滑动到底部的时候，继续滑动时，ScrollView 会滑动。惊了，决定看看源码，看看到底是怎么回事，这单纯用事件拦截机制是无法解释的。除非这两个控件强耦合。



那么，现在有几个问题是无法解释的：

- 滑动 RecyclerView 的时候，为什么不是 ScrollView 拦截这个滑动事件？它是怎么做到的？
- RecyclerView 滑动到底部的时候，ScrollView 是如何接着滑动的？



### 第一个问题

我们看 ScrollView 的 `android.widget.ScrollView#onInterceptTouchEvent` 方法：

> android.widget.ScrollView#onInterceptTouchEvent

```java
        if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
            return true;
        }
```

它决定是否拦截事件，主要取决于 `mIsBeingDragged` 这个变量。由于在这个滑动事件中，ScrollView 只会走 onInterceptTouchEvent 这个方法，所以我们只用看这个方法里面的逻辑，它里面 mIsBeingDragged 的赋值逻辑如下：

```java
            case MotionEvent.ACTION_MOVE: {
                final int y = (int) ev.getY(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                if (yDiff > mTouchSlop && (getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0) {
                    mIsBeingDragged = true;
```

可以看到，mIsBeingDragged 的值取决于 getNestedScrollAxes() 这个方法的值。

那么，这个`getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0` 条件啥时候为 true 呢？答案是当 ``getNestedScrollAxes()`不等于 SCROLL_AXIS_VERTICAL 的时候，也就是**嵌套滑动方向不是竖直滑动的时候**。

那么，我们看这个方法的逻辑：

> android.view.ViewGroup#getNestedScrollAxes

```java
    public int getNestedScrollAxes() {
        return mNestedScrollAxes;
    }
```

简单的返回了一个成员变量，看他在哪里赋值的：

```
android.view.ViewGroup#onNestedScrollAccepted
-androidx.core.view.ViewParentCompat#onNestedScrollAccepted(android.view.ViewParent, android.view.View, android.view.View, int, int)
--androidx.core.view.NestedScrollingChildHelper#startNestedScroll(int, int)
---androidx.recyclerview.widget.RecyclerView#startNestedScroll(int, int)
----androidx.recyclerview.widget.RecyclerView#onInterceptTouchEvent

----androidx.recyclerview.widget.RecyclerView#onTouchEvent
```

跟着这个调用链，可以知道，当 RecyclerView 的 onTouchEvent 方法执行的时候，就赋值为 `SCROLL_AXIS_VERTICAL`。

```java
                int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
                if (canScrollHorizontally) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
                }
                if (canScrollVertically) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
                }
                startNestedScroll(nestedScrollAxis, TYPE_TOUCH);
```

所以，ScrollView 的 mIsBeingDragged 一直为 false，它就不拦截事件。

整个流程就是：

- down 事件产生，传递到 recyclerView，recyclerView 的 child 没有能消费这个事件的，它自己处理了，调用到它自己的  onTouchEvent 方法
- 通过 NestedScrolling 的一些接口，通知父控件嵌套滑动方向为 SCROLL_AXIS_VERTICAL
- move 事件到来时，由于嵌套滑动方向是竖直的，ScrollView 就不拦截

### 第二个问题

在第一篇里面，我们是介绍了嵌套滑动的相关概念，这里就很像嵌套滑动。

然后看到了这样的调用链：

```
androidx.recyclerview.widget.RecyclerView#onTouchEvent
->androidx.recyclerview.widget.RecyclerView#dispatchNestedPreScroll(int, int, int[], int[], int)
-->androidx.core.view.NestedScrollingChildHelper#dispatchNestedPreScroll(int, int, int[], int[], int)
```

这个方法里面，就有一个很有意思的东西：

> androidx.core.view.NestedScrollingChildHelper#dispatchNestedPreScroll(int, int, int[], int[], int)

```java
    public boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed,
            @Nullable int[] offsetInWindow, @NestedScrollType int type) {
        if (isNestedScrollingEnabled()) {
            // ① 这里获取了 parent，在上面的例子中，这个parent 就是 ScrollView
            final ViewParent parent = getNestedScrollingParentForType(type);
            if (parent == null) {
                return false;
            }

            if (dx != 0 || dy != 0) {
                ...
                // ② 这里处理的嵌套滑动的逻辑    
                ViewParentCompat.onNestedPreScroll(parent, mView, dx, dy, consumed, type);

                ...
                return consumed[0] != 0 || consumed[1] != 0;
            } else if (offsetInWindow != null) {
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
```

我们先看注释①：

问题一：RecyclerView 是如何知道 ScrollView 是 parent 的？

> androidx.core.view.NestedScrollingChildHelper#getNestedScrollingParentForType

```java
    private ViewParent getNestedScrollingParentForType(@NestedScrollType int type) {
        switch (type) {
            case TYPE_TOUCH:
                return mNestedScrollingParentTouch;
            case TYPE_NON_TOUCH:
                return mNestedScrollingParentNonTouch;
        }
        return null;
    }
```

追踪链如下：

```
androidx.core.view.NestedScrollingChildHelper#mNestedScrollingParentTouch
->androidx.core.view.NestedScrollingChildHelper#setNestedScrollingParentForType
-->androidx.core.view.NestedScrollingChildHelper#startNestedScroll(int, int)
--->androidx.core.view.NestedScrollingChildHelper#setNestedScrollingParentForType
```

所以，parent 的赋值逻辑如下：

> androidx.core.view.NestedScrollingChildHelper#startNestedScroll(int, int)

```java
    public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
        if (hasNestedScrollingParent(type)) {
            // Already in progress
            return true;
        }
        if (isNestedScrollingEnabled()) {
            ViewParent p = mView.getParent();
            View child = mView;
            while (p != null) {
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                    setNestedScrollingParentForType(type, p);
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
```

这里的逻辑，就是不断的向上寻找 parent，直到 `androidx.core.view.ViewParentCompat#onStartNestedScroll(android.view.ViewParent, android.view.View, android.view.View, int, int)` 这个方法返回 true。看看这个方法的逻辑：

> androidx.core.view.ViewParentCompat#onStartNestedScroll(android.view.ViewParent, android.view.View, android.view.View, int, int)

```java
    public static boolean onStartNestedScroll(ViewParent parent, View child, View target,
            int nestedScrollAxes, int type) {
        if (parent instanceof NestedScrollingParent2) {
            // First try the NestedScrollingParent2 API
            return ((NestedScrollingParent2) parent).onStartNestedScroll(child, target,
                    nestedScrollAxes, type);
        } else if (type == ViewCompat.TYPE_TOUCH) {
            // Else if the type is the default (touch), try the NestedScrollingParent API
            if (Build.VERSION.SDK_INT >= 21) {
                try {
                    // 看这里，看这里
                    return parent.onStartNestedScroll(child, target, nestedScrollAxes);
                } catch (AbstractMethodError e) {
                    Log.e(TAG, "ViewParent " + parent + " does not implement interface "
                            + "method onStartNestedScroll", e);
                }
            } else if (parent instanceof NestedScrollingParent) {
                return ((NestedScrollingParent) parent).onStartNestedScroll(child, target,
                        nestedScrollAxes);
            }
        }
        return false;
    }
```

因为，ScrollView 是没有实现 NestedScrolling 相关接口的，所以，这里我们看 else if 中的逻辑，最后调用到了 ViewParent 的 onStartNestedScroll 方法。

看看这个方法有哪些类实现了，结果就有 ScrollView （ViewGroup 也实现了，只不过返回的是 false）。

> android.widget.ScrollView#onStartNestedScroll

```java
    @Override
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        return (nestedScrollAxes & SCROLL_AXIS_VERTICAL) != 0;
    }
```

到了这里，答案就开始浮现出来了，虽然，ScollView 没有实现 NestedScrolling  相关接口，但是它的接口（ViewParent）里面也有相关的方法，所以，最终它也可以做到嵌套滑动。

我们回忆一下嵌套滑动的核心逻辑：

```
在一个move产生后：

1. child 先询问 parent，能够消耗多少，没有消耗完
2. child 自己消耗，没有消耗完
3. child 再次询问 parent，我这还有没消耗完的，你能消耗多少，如果 parent 还是没有消耗完
4. child 自己处理

```

根据这个逻辑，我们追踪`androidx.core.view.ViewParentCompat#onNestedPreScroll(android.view.ViewParent, android.view.View, int, int, int[], int)`这个方法，调用链如下：

```
androidx.recyclerview.widget.RecyclerView#onTouchEvent
->androidx.recyclerview.widget.RecyclerView#scrollByInternal
-->androidx.recyclerview.widget.RecyclerView#dispatchNestedScroll(int, int, int, int, int[], int)
--->androidx.core.view.NestedScrollingChildHelper#dispatchNestedScroll(int, int, int, int, int[], int, int[])
---->androidx.core.view.NestedScrollingChildHelper#dispatchNestedScrollInternal
----->androidx.core.view.ViewParentCompat#onNestedScroll(android.view.ViewParent, android.view.View, int, int, int, int, int, int[])
------>android.view.ViewParent#onNestedScroll
------->android.widget.ScrollView#onNestedScroll
```

这样，一个嵌套滑动的流程就完成了。

> android.widget.ScrollView#onNestedScroll

```java
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed) {
        final int oldScrollY = mScrollY;
        // 这里是让 ScrollView 滚动
        scrollBy(0, dyUnconsumed);
        final int myConsumed = mScrollY - oldScrollY;
        final int myUnconsumed = dyUnconsumed - myConsumed;
        dispatchNestedScroll(0, myConsumed, 0, myUnconsumed, null);
    }	
```

最后总结一下：

首先是，事件拦截事件分发事件到 RecycleView，RecycleView先消耗move事件，拉到底部后，RecycleView无法消耗move事件了，然后通过 NestedScrolling 机制，将move距离往上分发到 ScrollView，ScrollView 即可滑动。

所以，如果我们想要实现一个支持嵌套滑动的控件，现在有两种方式了，一种是实现 NestedScrolling 接口，一种是直接复写 ViewGroup 里面的相关方法。不过这两种方式的接口都是一样的。



