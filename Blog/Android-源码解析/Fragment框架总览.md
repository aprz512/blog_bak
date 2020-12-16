---
title: Fragment框架总览
index_img: /cover/1.jpg
banner_img: /cover/top.jpg
date: 2020-1-1
categories: Android-源码解析
---



在做一个图片尺寸检查框架的时候，需要研究Activity与Fragment相关的源码，之前每次都分析过，但是却没有一个总体的影响，每次想要找一个什么东西的时候，都不得不从头找起，很蛋疼。

这篇文章的作用就是整理出一个Fragment的轮廓，Fragment大概会涉及一些什么东西。



首先，我们使用Fragment的方式如下：

```java
    getSupportFragmentManager()
        .beginTransaction()
        .add(R.id.container, new MyFragment())
        .commit();
```

上面的代码中，getSupportFragmentManager 其实是调用的 FragmentController 类的方法。



### FragmentController

这里就出现了第一个需要了解的类：**FragmentController**.

了解一个类最好的方法就是看它的结构：

![](fragment_01.png)

我们顺便点个方法进去看：

>android.app.FragmentController#onCreateView

```java
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    return mHost.mFragmentManager.onCreateView(parent, name, context, attrs);
}
```

> android.app.FragmentController#dispatchCreate

```java
public void dispatchCreate() {
    mHost.mFragmentManager.dispatchCreate();
}
```

可以看到，这些方法都是转发给了 mFragmentManager。mFragmentManager 是一个 FragmentManagerImpl 对象。

除了这个 **FragmentManagerImpl** ，还有一个需要注意的地方，就是那个 **mHost** 变量。mFragmentManager 也是通过它调用的。

我们看一下构造函数：

> android.app.FragmentController#FragmentController

```java
private FragmentController(FragmentHostCallback<?> callbacks) {
    mHost = callbacks;
}

public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
    return new FragmentController(callbacks);
}
```

mHost 是传递进来的。搜索一下调用的地方，发现只有Activity调用。Activity 有个成员变量：

> android.app.Activity#mFragments

```java
final FragmentController mFragments = FragmentController.createController(new HostCallbacks());
```

所以，FragmentController 的 mHost 变量就是这里创建并传递进去的 HostCallbacks 了。



### HostCallbacks

HostCallbacks 是 Activity 的内部类，它的作用就和名字一样，是宿主的回调接口。

对于 Fragment 来说，Activity 就是宿主，这个类的作用，就是 Fragment 回调 Activity 方法的一个类。比如这个方法：

> android.app.Activity.HostCallbacks#onStartActivityFromFragment

```java
@Override
public void onStartActivityFromFragment(Fragment fragment, Intent intent, int requestCode, Bundle options) {
    Activity.this.startActivityFromFragment(fragment, intent, requestCode, options);
}
```

可以看到，就是调用了 activity 的方法。



Activity，FragmentController，HostCallbacks 3个类的关系如下：

![](fragment_02.png)



### FragmentManager

我们平常操作 `fragment` 所调用的 `getSupportFragmentManager()` 返回的就是 `FragmentManager`对象。`FragmentManagerImpl` 是 `FragmentManager` 的一个实现类。

`FragmentManager`是 fragment 的管理者，负责添加、删除、替换 fragment 等一些操作。

FragmentManager 做的一些操作，都是模仿数据库的，它会开启一个事务。实现这个事务的类叫做`BackStackRecord`。这个类不难，其实里面就是一个 ArrayList，然后将各种操作封装为 Op。事务就是将各种操作集合起来，然后一起执行。所以这个类的作用，就是将各种Op放到ArrayList里面，然后循环取出来执行。

最后，commit的时候，会调用到下面的这个方法：

> androidx.fragment.app.BackStackRecord#executeOps

```java
/**
 * Executes the operations contained within this transaction. The Fragment states will only
 * be modified if optimizations are not allowed.
 */
void executeOps() {
    final int numOps = mOps.size();
    for (int opNum = 0; opNum < numOps; opNum++) {
        final Op op = mOps.get(opNum);
        final Fragment f = op.fragment;
        if (f != null) {
            f.setNextTransition(mTransition, mTransitionStyle);
        }
        switch (op.cmd) {
            case OP_ADD:
                f.setNextAnim(op.enterAnim);
                mManager.addFragment(f, false);
                break;
            case OP_REMOVE:
                f.setNextAnim(op.exitAnim);
                mManager.removeFragment(f);
                break;
            case OP_HIDE:
                f.setNextAnim(op.exitAnim);
                mManager.hideFragment(f);
                break;
            case OP_SHOW:
                f.setNextAnim(op.enterAnim);
                mManager.showFragment(f);
                break;
            case OP_DETACH:
                f.setNextAnim(op.exitAnim);
                mManager.detachFragment(f);
                break;
            case OP_ATTACH:
                f.setNextAnim(op.enterAnim);
                mManager.attachFragment(f);
                break;
            case OP_SET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(f);
                break;
            case OP_UNSET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(null);
                break;
            default:
                throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
        }
        if (!mReorderingAllowed && op.cmd != OP_ADD && f != null) {
            mManager.moveFragmentToExpectedState(f);
        }
    }
    if (!mReorderingAllowed) {
        // Added fragments are added at the end to comply with prior behavior.
        mManager.moveToState(mManager.mCurState, true);
    }
}
```

这个方法里面就可以看到操纵 Fragment 的一些方法。然后这个方法里面又调用了另外一个很重要的方法：

> androidx.fragment.app.FragmentManagerImpl#moveToState(androidx.fragment.app.Fragment, int, int, int, boolean)

在这个方法里面，就有涉及到触发 fragment 生命周期的方法。比如说：

```java
case Fragment.CREATED:
if (newState > Fragment.CREATED) {
    if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
    if (!f.mFromLayout) {
        ViewGroup container = null;
        ...
        f.mContainer = container;
        f.performCreateView(f.performGetLayoutInflater(
            f.mSavedFragmentState), container, f.mSavedFragmentState);

```

这里就是调用 Fragment 的 onCreateView 方法的地方了，还有就是可以看到 inflater 的由来。



整体类图关系图如下：

![](fragment_03.png)

