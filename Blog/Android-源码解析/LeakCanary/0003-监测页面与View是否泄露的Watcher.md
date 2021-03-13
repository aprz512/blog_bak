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



### 监测 rootView 的泄露

RootView 有多种：

- PHONE_WINDOW

  - Activity（我们已经监测了 activity，所以这个没必要）
  - Dialog

- POPUP_WINDOW

  由于 popup_window 通常会被缓存，所以没必要监测，当它 dismiss 后，还可能再显示出来

- TOOLTIP

  Tooltips可以实现类似pc端网页鼠标悬停时出现描述信息的功能，而到安卓中，如果给一个控件使用了Tooltips，那么当用户长按这个控件时，我们预设的描述信息就会悬浮出现在控件附件某个位置。

- TOAST

所以，最后，Dialog，TOOLTIP，TOAST 是需要监测的对象。当然，leakCanary 里面还做了 `else` 逻辑处理，略。

我们看它是在什么时候监测 rootView 的：

> curtains.internal.WindowManagerSpy#swapWindowManagerGlobalMViews

在分析这个方法之前，我们看一下这个方法的名字，因为后面的许多逻辑都是这个套路，为了不重复，这里只分析一遍。

`swapWindowManagerGlobalMViews` 

swap  是交换

WindowManagerGlobal 是一个类，Framework 中的

mViews 是 WindowManagerGlobal 的一个字段。

很明显，这个方法的作用，是使用反射来替换掉 WindowManagerGlobal  的 mViews 字段。

```kotlin
  // You can discourage me all you want I'll still do it.
  @SuppressLint("PrivateApi", "ObsoleteSdkInt", "DiscouragedPrivateApi")
  fun swapWindowManagerGlobalMViews(swap: (ArrayList<View>) -> ArrayList<View>) {
    if (SDK_INT < 19) {
      return
    }
    try {
      windowManagerInstance?.let { windowManagerInstance ->
        mViewsField?.let { mViewsField ->
          @Suppress("UNCHECKED_CAST")
          val mViews = mViewsField[windowManagerInstance] as ArrayList<View>
          mViewsField[windowManagerInstance] = swap(mViews)
        }
      }
    } catch (ignored: Throwable) {
      Log.w("WindowManagerSpy", ignored)
    }
  }
```

这个方法的注释也很有意思，你不让我搞，我还是要搞，我就是要出狂战斧！！！里面的逻辑就不细说了，就是反射。说个额外话题，就是为啥要 hook 这个字段呢？

因为，view想要显示出来，需要将自己添加到 window上，而 window 是由 WMS 管理的，app 与 WMS 是通过 Binder 通信，app 端的代理就是 WindowManagerGlobal 这个对象。所以，每个 activity，dialog，toast 显示的时候，都需要经过 WindowManagerGlobal 的 addView 方法，而所有的添加的 view 都存放在了 WindowManagerGlobal  的 mViews 字段里面。

这个方法的作用说清楚了，再说一下这个方法的构成，从它的名字，我们可以知道这个方法的作用，但是还需要注意的时，**这个方法的参数是一个 lambda 表达式。这个 lambda 表达式的参数，是该方法要替换的那个字段。**

有点绕，但是好理解，因为有时候我们不仅仅是需要将字段替换一下，而是需要利用原来的字段值做一些处理之后再设置回去，这样，外部调用这个方法的时候，就可以直接使用 lambda 的参数即可。

我们看看使用这个方法的地方：

> curtains.internal.RootViewsSpy.Companion#install

```kotlin
    fun install(): RootViewsSpy {
      return RootViewsSpy().apply {
        WindowManagerSpy.swapWindowManagerGlobalMViews { mViews ->
          delegatingViewList.apply { addAll(mViews) }
        }
      }
    }
```

其实就是将 mViews 字段，替换为 delegatingViewList，delegatingViewList 将 mViews 里面的值 copy 过来了。这个方法里面的逻辑如此简单，相比奥妙必然在 delegatingViewList 里面。

> curtains/internal/RootViewsSpy.kt:23

```java
  private val delegatingViewList = object : ArrayList<View>() {
    override fun add(element: View): Boolean {
      listeners.forEach { it.onRootViewsChanged(element, true) }
      return super.add(element)
    }

    override fun removeAt(index: Int): View {
      val removedView = super.removeAt(index)
      listeners.forEach { it.onRootViewsChanged(removedView, false) }
      return removedView
    }
  }
```

它重写了 list 的 add 与 removeAt 方法，在这两个方法执行的时候，调用了 listener 的对应方法。这个  listener 是在下面的方法处添加的：

> leakcanary.RootViewWatcher#install

```kotlin
  override fun install() {
    Curtains.onRootViewsChangedListeners += listener
  }
```

看这个 listener 的实现：

> leakcanary.RootViewWatcher#listener

```kotlin
  private val listener = OnRootViewAddedListener { rootView ->
    val trackDetached = when(rootView.windowType) {
      PHONE_WINDOW -> {
        when (rootView.phoneWindow?.callback?.wrappedCallback) {
          // Activities are already tracked by ActivityWatcher
          is Activity -> false
          is Dialog -> rootView.resources.getBoolean(R.bool.leak_canary_watcher_watch_dismissed_dialogs)
          // Probably a DreamService
          else -> true
        }
      }
      // Android widgets keep detached popup window instances around.
      POPUP_WINDOW -> false
      TOOLTIP, TOAST, UNKNOWN -> true
    }
    if (trackDetached) {
      rootView.addOnAttachStateChangeListener(object : OnAttachStateChangeListener {

        val watchDetachedView = Runnable {
          reachabilityWatcher.expectWeaklyReachable(
            rootView, "${rootView::class.java.name} received View#onDetachedFromWindow() callback"
          )
        }

        override fun onViewAttachedToWindow(v: View) {
          mainHandler.removeCallbacks(watchDetachedView)
        }

        override fun onViewDetachedFromWindow(v: View) {
          mainHandler.post(watchDetachedView)
        }
      })
    }
  }
```

其实就是，为每个 rootView 都添加了一个 addOnAttachStateChangeListener 监听。在 onViewDetachedFromWindow 的时候，去监听这个对象是否泄露。



### 监测 Service 的泄露

service 的泄露监测点要稍微复杂一点，涉及到两个方面：

- mH
- ActivityManager

> leakcanary.ServiceWatcher#install

```kotlin
// 替换 ActivityThread 里面 mH 的 callback 字段
swapActivityThreadHandlerCallback { mCallback ->
  // uninstall 的时候，需要将替换的字段替换回来
  uninstallActivityThreadHandlerCallback = {
    swapActivityThreadHandlerCallback {
      mCallback
    }
  }
  // 创建新的 callback 对象，替换原来的
  Handler.Callback { msg ->
    // 针对 STOP_SERVICE 消息进行处理
    if (msg.what == STOP_SERVICE) {
      val key = msg.obj as IBinder
      // 从 activityThread 的 mServices 字段里面，拿到 Service 对象
      activityThreadServices[key]?.let {
        // 将 service 用弱引用包装后放入 map
        onServicePreDestroy(key, it)
      }
    }
    // 返回 false，继续走 handler 的 handleMessage 逻辑
    mCallback?.handleMessage(msg) ?: false
  }
}
```

以  swap 方法开头的前面已经说过这个套路了，就不重复了，上面的代码就是 替换 ActivityThread 里面 mH 的 callback 字段。

由于，我们监测的是 Service，hook 它的作用就是可以拿到 service 对应的 binder 对象，通过这个 binder 对象，我们就可以获取到对应的 Service，然后在监测这个 Service。

上面的代码中，有个 `onServicePreDestroy` 方法，它就是将 binder 与 service 储存起来了。

> leakcanary.ServiceWatcher#onServicePreDestroy

```kotlin
  private fun onServicePreDestroy(
    token: IBinder,
    service: Service
  ) {
    servicesToBeDestroyed[token] = WeakReference(service)
  }
```

然后，找到真正 service 执行 onDestroy 的地方：

> leakcanary.ServiceWatcher#install

```kotlin

      // 因为 handler 的 handleMessage 逻辑里面，会调用 ActivityManager.getService().serviceDoneExecuting() 方法
      // 所以需要 hook ActivityManager，在这个方法里去监测 service 的泄露情况
      swapActivityManager { activityManagerInterface, activityManagerInstance ->
        // 卸载函数，同上
        uninstallActivityManager = {
          swapActivityManager { _, _ ->
            activityManagerInstance
          }
        }
        // 动态代理，hook ActivityManager 的 serviceDoneExecuting 方法
        Proxy.newProxyInstance(
          activityManagerInterface.classLoader, arrayOf(activityManagerInterface)
        ) { _, method, args ->
          if (METHOD_SERVICE_DONE_EXECUTING == method.name) {
            val token = args!![0] as IBinder
            if (servicesToBeDestroyed.containsKey(token)) {
              // serviceDoneExecuting 方法的参数里面只有 token，所以在 onServicePreDestroy 里面保存了 token - service 的键值对（弱引用map）
              onServiceDestroyed(token)
            }
          }
          // 调用原来的逻辑
          try {
            if (args == null) {
              method.invoke(activityManagerInstance)
            } else {
              method.invoke(activityManagerInstance, *args)
            }
          } catch (invocationException: InvocationTargetException) {
            throw invocationException.targetException
          }
        }
      }
    } catch (ignored: Throwable) {
      SharkLog.d(ignored) { "Could not watch destroyed services" }
    }
```

service 的 onDestroy 方法执行后，接下来就执行 ActivityManager 的 serviceDoneExecuting 方法，所以hook这个方法也没问题。

`onServiceDestroyed`这个方法里面，就是监测代码了：

> leakcanary.ServiceWatcher#onServiceDestroyed

```kotlin
  private fun onServiceDestroyed(token: IBinder) {
    // 从 map 中取出弱引用来
    servicesToBeDestroyed.remove(token)?.also { serviceWeakReference ->
      // 拿到 service
      serviceWeakReference.get()?.let { service ->
        // 监测 service
        reachabilityWatcher.expectWeaklyReachable(
          service, "${service::class.java.name} received Service#onDestroy() callback"
        )
      }
    }
  }
```

