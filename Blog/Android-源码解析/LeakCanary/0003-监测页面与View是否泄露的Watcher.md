---
title: 0003-监测页面与View是否泄露的Watcher
index_img: /cover/11.jpg
banner_img: /cover/top.jpg
date: 2021-3-11
tags: Android源码解析-LeakCanary
categories: LeakCanary
---



> 想要监测一个对象是否泄露，需要知道的是这个对象什么时候应该被销毁，然后再做相应的处理，所以第一个难点是如何找准这个销毁的时机。
>
> 下面我们具体分析源码，看它是如何做的。



所有的 Watcher 都在这个方法里面：

> leakcanary.AppWatcher#appDefaultWatchers

```java
  fun appDefaultWatchers(
    application: Application,
    reachabilityWatcher: ReachabilityWatcher = objectWatcher
  ): List<InstallableWatcher> {
    return listOf(
      ActivityWatcher(application, reachabilityWatcher),
      FragmentAndViewModelWatcher(application, reachabilityWatcher),
      RootViewWatcher(reachabilityWatcher),
      ServiceWatcher(reachabilityWatcher)
    )
  }
```

我们先分析 `InstallableWatcher` 这个接口：

> leakcanary.InstallableWatcher

```kotlin
interface InstallableWatcher {

  fun install()

  fun uninstall()
}
```

前面我们说过，想要监测内存泄露，需要调用 Watcher 的 install 方法，对应的，要停止监测，就调用 uninstall 方法，我们下面看看它的子类。



### 监测 activity 的泄露

> leakcanary.ActivityWatcher
>
> ActivityLifecycleCallbacks 的执行链：
>
> 1. application 中注册回调
> 2. activity 的 onDestroy 方法执行
> 3. activity 利用 getApplication 获取 application 对象，调用它的 dispatch 方法
> 4. dispatch 方法调用回调的对应方法

```kotlin
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}
```

这里的逻辑比较简单，就是在每个 activity 的 onDestroy 方法执行的时候去调用 `leakcanary.ReachabilityWatcher#expectWeaklyReachable` 这个方法。

`ReachabilityWatcher`它的实现类是 `leakcanary.ObjectWatcher`，我们**后面再具体分析它是如何监测某一个对象是否泄露了的**！



### 监测 fragment 的泄露

看 install 方法：

> leakcanary.FragmentAndViewModelWatcher#install

```kotlin
  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }
```

欸，这里它也注册了一个 activity 的生命周期的回调。然后在这里回调里面做了这样的事情：

> leakcanary.FragmentAndViewModelWatcher#lifecycleCallbacks

```kotlin
  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityCreated(
        activity: Activity,
        savedInstanceState: Bundle?
      ) {
        for (watcher in fragmentDestroyWatchers) {
          watcher(activity)
        }
      }
    }
```

> 这里的生命周期是 onActivityCreated

嗯，这里有许多个 fragmentDestroyWatcher：

- AndroidOFragmentDestroyWatcher
- AndroidXFragmentDestroyWatcher
- AndroidSupportFragmentDestroyWatcher

根据不同的库会有不同的实现，因为不同的库 fragment 的实现也不一样。这里我们只分析 androidx 的实现。

for 循环里面调用了 watcher 的 invoke 方法，我们看对应的代码：

> leakcanary.internal.AndroidXFragmentDestroyWatcher#invoke

```kotlin
  override fun invoke(activity: Activity) {
    if (activity is FragmentActivity) {
      val supportFragmentManager = activity.supportFragmentManager
      supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
      ViewModelClearedWatcher.install(activity, reachabilityWatcher)
    }
  }
```

这里是针对每一个 activity 都做了同样事情，在 activity 中注册了一个 FragmentLifecycleCallbacks，所以就能够监听到该 activity  中所有的 fragment 的生命周期。那么，如果监测 fragment 中的 fragment 的生命周期呢？看到上面的  `registerFragmentLifecycleCallbacks`这个方法中的第 2 个参数，它就是表示可以递归的注册的，所以，它可以监测 fragment 中的 fragment 的生命周期。

拿到 fragment 的生命周期之后，与 activity 一样，调用 ReachabilityWatcher 的方法：

> androidx.fragment.app.FragmentManager.FragmentLifecycleCallbacks#onFragmentDestroyed

```kotlin
    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      reachabilityWatcher.expectWeaklyReachable(
        fragment, "${fragment::class.java.name} received Fragment#onDestroy() callback"
      )
    }
  }
```

但是，fragment 与 activity 还不一样，它有两个 onDestroy 方法，一个是它自身的，一个是它的 view 的。当 fragment 的 onDestroyView 方法执行后，这个 View 应该被销毁，所以我们也要监测这个 view 是否泄露。因为很多情况下，fragment 的生命周期都只会走到 onDestroyView 这里，但是如果我们在内部类里面更新该 fragment 的界面的时候，就会出现内存泄露问题。

这样，fragment 的处理就差不多了，但是 invoke 方法里还有一行代码：

```kotlin
ViewModelClearedWatcher.install(activity, reachabilityWatcher)
```

看起来是监测 ViewModel 的。



### 监测 ViewModel 的泄露

看 install 方法：

> leakcanary.internal.ViewModelClearedWatcher.Companion#install

```kotlin
  companion object {
    fun install(
      storeOwner: ViewModelStoreOwner,
      reachabilityWatcher: ReachabilityWatcher
    ) {
      val provider = ViewModelProvider(storeOwner, object : Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel?> create(modelClass: Class<T>): T =
          ViewModelClearedWatcher(storeOwner, reachabilityWatcher) as T
      })
      provider.get(ViewModelClearedWatcher::class.java)
    }
  }
}
```

这里其实就是调用了 ViewModelClearedWatcher 的构造方法。可以理解为在 activity 的 viewModelStore 里面添加了一个 `ViewModelClearedWatcher` 对象。

当 activity 执行 onDestroy 方法的时候，viewModel 的 onClear 方法会执行。所以，我们可以在 `ViewModelClearedWatcher`  这个对象的 onClear 方法里面去判断其他的 viewModel 是否内存泄露了。

首先，使用反射拿到 activity 的 viewModeStore 的 `mMap`字段：

> leakcanary/internal/ViewModelClearedWatcher.kt:24

```kotlin
  init {
    // We could call ViewModelStore#keys with a package spy in androidx.lifecycle instead,
    // however that was added in 2.1.0 and we support AndroidX first stable release. viewmodel-2.0.0
    // does not have ViewModelStore#keys. All versions currently have the mMap field.
    viewModelMap = try {
      val mMapField = ViewModelStore::class.java.getDeclaredField("mMap")
      mMapField.isAccessible = true
      @Suppress("UNCHECKED_CAST")
      mMapField[storeOwner.viewModelStore] as Map<String, ViewModel>
    } catch (ignored: Exception) {
      null
    }
  }
```

然后，在 onClear 方法里面去遍历里面所有的对象，监测它们：

> leakcanary.internal.ViewModelClearedWatcher#onCleared

```kotlin
  override fun onCleared() {
    viewModelMap?.values?.forEach { viewModel ->
      reachabilityWatcher.expectWeaklyReachable(
        viewModel, "${viewModel::class.java.name} received ViewModel#onCleared() callback"
      )
    }
  }
```

上面只是说了监测 activity 中的 viewModel 的泄露，那么 fragment 中的 viewModel 没有监测吗？？

是有的，上面监测 activity 中的 viewModel 的泄露是在  onActivityCreated 的回调中处理的，而监测 fragment 中的 viewModel 的泄露是在下面的代码中做的：

> androidx.fragment.app.FragmentManager.FragmentLifecycleCallbacks#onFragmentCreated

```kotlin
    override fun onFragmentCreated(
      fm: FragmentManager,
      fragment: Fragment,
      savedInstanceState: Bundle?
    ) {
      ViewModelClearedWatcher.install(fragment, reachabilityWatcher)
    }
```



### 监测 window 的根 view 的泄露

