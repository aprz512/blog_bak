---
title: 0002-自定义LayoutHelper
date: 2019-9-17
tags: vLayout
categories: Android源码分析-vLayout
---

在实现自己的 LayoutHelper 之前，我们先看看官方是如何实现的。如果你之前了解过自定义 LayoutManager，那么这个就是小意思了。

我们先来看一个比较简单的 `com.alibaba.android.vlayout.layout.FixLayoutHelper`。

> com.alibaba.android.vlayout.layout.FixLayoutHelper

```java
    @Override
    public void setItemCount(int itemCount) {
        if (itemCount > 0) {
            super.setItemCount(1);
        } else {
            super.setItemCount(0);
        }
    }
```

可以看到，固定的 item 只管理单个 view，就算我们设置了多个，也只取第一个。比如：

```java
FixLayoutHelper layoutHelper = new FixLayoutHelper(10, 10);
// itemCount 设置为 0，不显示，LayoutHelper 认为没有 view 需要排列
adapters.add(new SubAdapter(this, layoutHelper, 0));

layoutHelper = new FixLayoutHelper(FixLayoutHelper.TOP_RIGHT, 20, 20);

// 这里即使将 itemCount 设置为 10，效果也是一样的
adapters.add(new SubAdapter(this, layoutHelper, 1) {
    @Override
    public void onBindViewHolder(MainViewHolder holder, int position) {
        super.onBindViewHolder(holder, position);
        LayoutParams layoutParams = new LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 200);
        holder.itemView.setLayoutParams(layoutParams);
    }
});
```

看看别的方法，有三个比较重要，我们先看第一个：

> com.alibaba.android.vlayout.layout.FixLayoutHelper#layoutViews

```java
    @Override
    public void layoutViews(RecyclerView.Recycler recycler, RecyclerView.State state,
            VirtualLayoutManager.LayoutStateWrapper layoutState, LayoutChunkResult result,
            final LayoutManagerHelper helper) {
        // reach the end of this layout
        if (isOutOfRange(layoutState.getCurrentPosition())) {
            return;
        }

        if (!mShouldDrawn) {
            layoutState.skipCurrentPosition();
            return;
        }

        // find view in currentPosition
        View view = mFixView;
        if (view == null) {
            view = layoutState.next(recycler);
        } else {
            layoutState.skipCurrentPosition();
        }

        if (view == null) {
            result.mFinished = true;
            return;
        }

        mDoNormalHandle = state.isPreLayout();

        if (mDoNormalHandle) {
            // in PreLayout do normal layout
            helper.addChildView(layoutState, view);
        }

        mFixView = view;

        doMeasureAndLayout(view, helper);

        result.mConsumed = 0;
        result.mIgnoreConsumed = true;

        handleStateOnResult(result, view);

    }
```

我们看第 16-21 行的代码，这是很重要的一个部分，这里就是获取是否有可以复用的 view。往下看，到 37 行，这里就是处理了 view 的测量与布局。里面的逻辑还是很简单的，这里截取两段看一看：

```java
final int widthSpec = helper.getChildMeasureSpec(xxx);
heightSpec = helper.getChildMeasureSpec(xxx);

// do measurement
helper.measureChildWithMargins(view, widthSpec, heightSpec);
```

测量都是一样的，先创建 MeasureSpec，然后调用 measureChildWithMargins 方法，view 的大小就测量好了。

```java
top = helper.getPaddingTop() + mY + mAdjuster.top;
right = helper.getContentWidth() - helper.getPaddingRight() - mX - mAdjuster.right;
left = right - params.leftMargin - params.rightMargin - view.getMeasuredWidth();
bottom = top + params.topMargin + params.bottomMargin + view.getMeasuredHeight();
```

布局就更简单了，因为是固定位置，所以只需要算一下绝对位置就好了，没啥好说的。

我们再看另外两个方法：

> com.alibaba.android.vlayout.layout.FixLayoutHelper#beforeLayout

```java
@Override
public void beforeLayout(RecyclerView.Recycler recycler, RecyclerView.State state,
                         LayoutManagerHelper helper) {
    super.beforeLayout(recycler, state, helper);

    if (mFixView != null && helper.isViewHolderUpdated(mFixView)) {
        // recycle view for later usage
        helper.removeChildView(mFixView);
        recycler.recycleView(mFixView);
        mFixView = null;
        isAddFixViewImmediately = true;
    }

    mDoNormalHandle = false;
}
```

这个方法的作用，是在 layout 之前做一些事情，这里做的事情是如果检测到 viewHolder 变化了，就将 view 放入复用池中，便于接下来layout时复用。

> com.alibaba.android.vlayout.layout.FixLayoutHelper#afterLayout

```java
@Override
public void afterLayout(final RecyclerView.Recycler recycler, RecyclerView.State state,
                        int startPosition, int endPosition, int scrolled,
                        final LayoutManagerHelper helper) {
    super.afterLayout(recycler, state, startPosition, endPosition, scrolled, helper);

    // disabled if mPos is negative number
    if (mPos < 0) {
        return;
    }

    if (mDoNormalHandle && state.isPreLayout()) {
        if (mFixView != null) {
            //                Log.d(TAG, "after layout doNormal removeView");
            helper.removeChildView(mFixView);
            recycler.recycleView(mFixView);
            isAddFixViewImmediately = false;
        }

        mFixView = null;
        return;
    }

    // Not in normal flow
    if (shouldBeDraw(helper, startPosition, endPosition, scrolled)) {
        mShouldDrawn = true;
        if (mFixView != null) {
            // already capture in layoutViews phase
            // if it's not shown on screen
            if (mFixView.getParent() == null) {
                addFixViewWithAnimator(helper, mFixView);
            } else {
                // helper.removeChildView(mFixView);
                helper.addFixedView(mFixView);
                isRemoveFixViewImmediately = false;
            }
        } else {
            Runnable action = new Runnable() {
                @Override
                public void run() {
                    mFixView = recycler.getViewForPosition(mPos);
                    doMeasureAndLayout(mFixView, helper);
                    if (isAddFixViewImmediately) {
                        helper.addFixedView(mFixView);
                        isRemoveFixViewImmediately = false;
                    } else {
                        addFixViewWithAnimator(helper, mFixView);
                    }
                }
            };
            if (mFixViewDisappearAnimatorListener.isAnimating()) {
                mFixViewDisappearAnimatorListener.withEndAction(action);
            } else {
                action.run();
            }
        }
    } else {
        mShouldDrawn = false;
        if (mFixView != null) {
            removeFixViewWithAnimator(recycler, helper, mFixView);
            mFixView = null;
        }
    }

}
```

这个方法的作用，是在 layout 之后做一些事情。不知道你看到这里时有没有产生疑问：我们 RecyclerView 时从上往下，或者从下往上布局的，那么如果我们的屏幕上只能显示 10 个 item，而在第 100 个位置有一个 FixView，那么问题就来了，为什么我们一打开这个界面 FixView 就能立即显示出来，而不是滑动到了第 100 个位置才显示出来？

我们先看该方法时在什么时候被调用的？

> com.alibaba.android.vlayout.VirtualLayoutManager#runPostLayout

```java
    private void runPostLayout(RecyclerView.Recycler recycler, RecyclerView.State state, int scrolled) {
        mNested--;
        if (mNested <= 0) {
            mNested = 0;
            final int startPosition = findFirstVisibleItemPosition();
            final int endPosition = findLastVisibleItemPosition();
            List<LayoutHelper> layoutHelpers = mHelperFinder.getLayoutHelpers();
            Iterator<LayoutHelper> iterator = layoutHelpers.iterator();
            LayoutHelper layoutHelper = null;
            while (iterator.hasNext()) {
                layoutHelper = iterator.next();
                try {
                    layoutHelper.afterLayout(recycler, state, startPosition, endPosition, scrolled, this);
                } catch (Exception e) {
                    if (VirtualLayoutManager.sDebuggable) {
                        throw e;
                    }
                }
            }

            if (null != mViewLifeCycleHelper) {
                mViewLifeCycleHelper.checkViewStatusInScreen();
            }
        }
    }
```

这个方法在 `com.alibaba.android.vlayout.VirtualLayoutManager#onLayoutChildren` 里面被调用，而 onLayoutChildren 几乎是不断的在被调用，RecyclerView 刚显示时，滚动时都需要调用该方法，所以 runPostLayout 也是不断的会被调用。

看 runPostLayout 的内部逻辑，可以发现它调用了所有的 LayoutHelper 的 afterLayout 方法。即，与 layoutViews 不一样，layoutViews 只会在滚动到相应位置才会调用，afterLayout 与 beforeLayout 都会不断的被调用。

所以，FixView 一开始就可以显示，显然是在 afterLayout 中做了什么。

说实话，这段逻辑我没看懂，写这个代码的人太喜欢用bool变量来控制逻辑了，所以只好使出断点调试大法。界面（VLayoutActivity）刚展示的时候，只会走 afterLayout 与 beforeLayout，beforeLayout 没什么影响，afterLayout 走到了第 47 行：

> com.alibaba.android.vlayout.layout.FixLayoutHelper#addFixViewWithAnimator

```java
    private void addFixViewWithAnimator(LayoutManagerHelper layoutManagerHelper, View fixView) {
        if (mFixViewAnimatorHelper != null) {
            ViewPropertyAnimator animator = mFixViewAnimatorHelper
                    .onGetFixViewAppearAnimator(fixView);
            if (animator != null) {
                fixView.setVisibility(View.INVISIBLE);
                layoutManagerHelper.addFixedView(fixView);
                mFixViewAppearAnimatorListener.bindAction(layoutManagerHelper, fixView);
                animator.setListener(mFixViewAppearAnimatorListener).start();
            } else {
                layoutManagerHelper.addFixedView(fixView);
            }
        } else {
            layoutManagerHelper.addFixedView(fixView);
        }
        isRemoveFixViewImmediately = false;
    }
```

然后调用了第 14 行，所以，是 addFixedView 方法起了作用。该方法是将 FixView add 到了 LayoutManager 里面。这里需要 add 是因为本来 fixView 不应该在 LayoutManager 管理的集合里面，但是它又需要显示出来，所以需要 add。

addFixedView 这个方法里面的逻辑也很简单：

```java
    @Override
    public void addFixedView(View view) {
        addOffFlowView(view, false);
    }

    @Override
    public void addOffFlowView(View view, boolean head) {
        showView(view);
        addHiddenView(view, head);
    }

    protected void addHiddenView(View view, boolean head) {
        int index = head ? 0 : -1;
        addView(view, index);
        mChildHelperWrapper.hide(view);
    }
```

showView 是调用了 mChildHelperWrapper 的 show 方法，最终会调用到 `android.support.v7.widget.ChildHelper.Bucket#clear` 里面。这个类具体可以看看这篇文章：https://blog.csdn.net/fyfcauc/article/details/54175072 ，它的内部逻辑就是将该 view 标记为可见。

我刚开始以为这个 showView 方法没啥作用，因为后面不是又重新隐藏了么，直到我将这行代码注掉之后，发现刚开始是正常的吗，但是上下滑动之后， FixView 的层次变化了，但是没找到原因。

查看 LayoutManagerHelper 的注释与 layer 有点关系，下篇再讲。