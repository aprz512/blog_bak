---
title: fish-redux使用说明
index_img: /cover/11.jpg
banner_img: /cover/top.jpg
date: 2020-9-11
categories: Flutter
---

一个页面大致可以分为这几个部分：

- 控件
- 数据
- 交互逻辑

fish-redux 针对这3个部分，抽象出了如下几个文件：

### 控件

view.dart	这个对应的页面上的控件，如同Android的layout.xml文件

还有一个 page.dart，这个就更简单了，它是一个脚手架，就是啥事也不干，只是把各个文件里面的函数组装起来。

### 交互逻辑

effect.dart 与 reducer.dart    这两个文件，对应的就是交互逻辑。

还有一个 action.dart ，描述的是每个交互的别称，就是给每个交互动作起个名字，好区分。

那交互逻辑为啥要分为两个文件呢？

这就涉及到数据的管理了。fish-redux是这样管理页面中的数据的。它认为页面中的数据始终是一个快照数据，每当数据变化的时候，就会生成一个新的数据对象出来。我们在进行多线程编程的时候，应该接触过这样的东西。

我们想一下 Flutter 的工作方式：在 Flutter 中，Widget 上展示的数据如果需要更新，那么就必须要调用 setState，将该 Widget 标记为 dirty。然后 widget 重新构建，构建的时候使用变化之后的数据。

那么有趣的地方就来了，为啥 fish-redux 一定要重新生成一份数据呢？其实这只是一种设计方式，当然也可以不用每次更新 state，每次更新 state，是因为里面的内部逻辑是判断前后两个 state 是否一样来决定是否更新 view 的。它也提供了自定义的更新逻辑：

```dart
typedef ShouldUpdate<T> = bool Function(T old, T now);
```

比如，你可以使用 T 里面的属性来决定是否更新，但是这样属性很多的话，页面更新逻辑就非常的麻烦了。

好的，数据管理说了这么多，回到 effect.dart 与 reducer.dart 这两个文件上，根据上面的知识，要更新页面，就得创建出新的数据出来（默认逻辑）。那么，在哪里创建新的数据呢？答案是 reducer.dart 里面。而 effect.dart 是用来处理除了数据改变的其他逻辑的。

举个例子：

一个feed流页面，下拉刷新后，去请求网络，数据拿到后，需要更新页面。

根据fish-redux的分类，effect.dart 需要处理网络请求，reducer.dart 需要处理更新页面。

所以，大致逻辑如下：

> effect.dart

```dart
void _init(Action action, Context<ListState> ctx) async {
  var datas = await fetchData();
  // 拿到网络数据后，发送给 reducer.dart
  ctx.dispatch(ActionCreator.onFetchDataSuccessAction(datas));
}
```

> reducer.dart

```dart
ListState _onFetchDataSuccess(ListState state, Action action) {
  // 这里clone了一份原来的数据，然后更改其属性，最后这个属性会传递到 view.dart 里面作为参数
  return state.clone()..items = action.payload;
}
```

可以看到，上面出现了 action，我们说过，action 是交互动作的名字，它也可以携带参数。

effect.dart 拿到网络数据后，我们就可以更新页面了，但是不能直接更新，因为框架限制了我们需要改变数据，才能更新，而 effect 里面是访问不到数据的，想要改变数据，就需要到 reducer.dart  里面去，所以我们就发送了一个 action，而reducer.dart里面可以接受这个 action，从而做对应的处理。

effect 与  reducer 其实就是两个 map，key 是 action，value 就是收到 action 后应该如何处理。

> effect.dart

```
Effect<ListState> buildEffect() {
  return combineEffects(<Object, Effect<ListState>>{
    ///进入页面就执行的初始化操作
    Lifecycle.initState: _init,
  });
}
```

combineEffects 是框架提供的方法，它的作用是从 map 里面取出对应 action 的 effect 方法。

> reducer.dart

```dart
Reducer<ListState> buildReducer() {
  return asReducer(
    <Object, Reducer<ListState>>{
      FourthAction.fetchDataSuccess: _onFetchDataSuccess
    },
  );
}
```

asReducer 的作用与上面的 combineEffects  是一样的。



### 数据

state.dart 这里里面存放的是页面所有的数据。需要实现 Cloneable，因为方便创建新的数据对象。



### 流程分析

```dart
Widget buildView(ListState state, Dispatch dispatch, ViewService viewService) {}
```

当我们在 view 里面，调用 dispatch(action) 的时候，我们看看整个流程是什么样的。其实这里最好的自己打个断点，流程就很清楚了。

还有一个知识点需要注意，dispatch 是一个函数，这个框架里面很多地方都是传递的函数，如果不习惯函数编程的，会很绕。

dispatch 是 helper 类创建的。

> lib/src/redux_component/helper.dart:100

```dart
Dispatch createDispatch<T>(Dispatch onEffect, Dispatch next, Context<T> ctx) => (Action action) {
      final Object result = onEffect?.call(action);
      if (result == null || result == false) {
        next(action);
      }

      return result == _SUB_EFFECT_RETURN_NULL ? null : result;
    };
```

从这里可以看到，这里先让 effect 处理了 action，如果没有设置effect，或者 effect 方法返回了 false，那么才执行 next 函数。也就是说，如果你的 action 想两个都走的话，就让 effect 返回 false 就好了。

官方文档也说了：

````
当前effect返回 true 的时候，就会停止后续的effect和reducer的操作

当前effect返回 false 的时候，后续effect和reducer继续执行
````

我们继续看看 next ：

> lib/src/redux_component/helper.dart:94

```dart
Dispatch createNextDispatch<T>(ContextSys<T> ctx) => (Action action) {
      ctx.broadcastEffect(action);
      ctx.store.dispatch(action);
    };
```

这里是兵分两路。bus先不管，看store。调用了 store 的 dispatch 方法：

```dart
   return Store<T>()
    ..getState = (() => _state)
    ..dispatch = (Action action) {
      ...

      try {
        _isDispatching = true;
        _state = _reducer(_state, action);
      } finally {
        _isDispatching = false;
      }

      ...
    }
```

可以看到，它调用了 reducer 。

所以，流程图如下：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xODgyMDg5MC0yMmJiNGRkZmMzNWJlMzM4?x-oss-process=image/format,png)

需要注意的是，我们在 effect，reducer 里面 dispatch 的 action，也会重新走整个派发流程，所以不要发送自己可以接受的 action，**避免造成死循环**。

我们继续往后面跟踪代码，会发现调用到了：

> lib/src/redux_component/context.dart:236

```dart
  @override
  void onNotify() {
    final T now = state;
    if (shouldUpdate(_latestState, now)) {
      _widgetCache = null;

      markNeedsBuild();

      _latestState = now;
    }
  }
```

将 widget 标记为 dirty 的。我们在 view 里面创建的 widget 最终也会添加到一个内部的 widget 上。当这个内部的 widget 更新的时候，它会重新build，然后就会调用到 view 里面的 buildView 方法了。

> lib/src/redux_component/context.dart:216

```dart
  @override
  Widget buildWidget() {
    Widget result = _widgetCache;
    if (result == null) {
      // 这里就是调用了 view 的  buildView 方法，因为 buildView 也是一个函数，所以这里是调用了一个函数
      result = _widgetCache = view(state, dispatch, this);

      dispatch(LifecycleCreator.build(name));
    }
    return result;
  }
```

里面做了缓存，那么，问题就来了，既然缓存起来了，那么怎么才能更新界面呢？很简单，将 _widgetCache 的值置为null就好了。上面的 onNotify 方法就做了这件事。所以 dispatch action 之后，如果 reducer 产生了新的数据对象，那么页面就会刷新。

### 其他概念

#### 组件

component 与 page 的用法差不多。因为 component 继承的 page，可以理解为 View 与 ViewGroup 的关系。component 创建出来后，可以在 page 里面使用。

如下，需要先在 page 里面的依赖申明一下，我有这个组件：

```dart
          dependencies: Dependencies<ThirdState>(
              slots: <String, Dependent<ThirdState>>{
                // 连接器理解为数据传递
                "card1": Card1Connector() + CardComponent(),
              }),
```

然后，在 view 里面使用：

```dart
viewService.buildComponent('card1')
```

buildComponent 的返回值就是一个 widget。

#### 连接器

它表达了如何从一个大数据中读取小数据，同时对小数据的修改如何同步给大数据，这样的数据连接关系。

在fish-redux的页面设计中，数据全部都给 page 管理，component 的数据也在 page 里面，component虽然是有自己的数据，但是是从 page 里面传递过来的。我们使用 connector 将page中的数据传递给 component使用，同时也将 component更改的数据同步回page。

> page 里面的数据

```
class ThirdState implements Cloneable<ThirdState> {
  HeaderState headerState;
  ...
}
```

> component 里面的数据

```dart
class HeaderState implements Cloneable<HeaderState> {
  int count;
  ...
}
```

> connector

```dart
// 两个泛型 T -> P，T 是大数据（页面），P是小数据（页面的组件）
// get 是用来从大数据中获取小数据，父类重写了 get 方法（里面做了缓存等），提供了一个 computed 方法
// set 是用来将小数据同步给大数据，保持数据同步
// 它支持 + 操作符号，可以看重写代码
class HeaderConnector extends ConnOp<ThirdState, HeaderState>
    with ReselectMixin<ThirdState, HeaderState> {
  @override
  HeaderState computed(ThirdState state) {
    return state.headerState.clone();
  }

  // 这个方法会将 HeaderState 里面的数同步到 ThirdState 里面去
  @override
  void set(ThirdState state, HeaderState subState) {
    state.headerState = subState;
  }
}
```

#### 中间件

可以理解为 AOP。做切面编程了，比如，我先打印一个页面的生命周期：

```dart
class EntryPage extends Page<EntryState, Map<String, dynamic>> {
  EntryPage()
      : super(
          ...
          // 这里还可以传一些 middleWare，用来做 AOP，比如打印生命周期
          effectMiddleware: <EffectMiddleware<EntryState>>[
            pageLifecycle(),
          ],
        );
}
```

这里我给 effectMiddleware 传递了一个列表，里面有一个元素。

```dart
EffectMiddleware<EntryState> pageLifecycle() {
  return (AbstractLogic<dynamic> logic, Store<EntryState> store,) {
    return (Effect<dynamic> effect) {
      return (Action action, Context<dynamic> ctx) {
        if (logic is Page<dynamic, dynamic> && action.type is Lifecycle) {
          print('${logic.runtimeType} ${action.type.toString()} ');
        }
        return effect?.call(action, ctx);
      };
    };
  };
}
```

EffectMiddleware 是一个函数，而这个函数的返回值也是函数，所以这里看起来相当蛋疼。它的作用就是打印了当前page的生命周期，当然也可以给所有page做切面，不过不再这里，在 PageRoutes 里面。

#### 全局状态

之前流程分析源码那里有个兵分两路的，它是调用了 store 的dispatch，所以 page 内部也是使用的 store。

全局状态需要我们自己创建一个 store。

```dart
/// 建立一个全局数据区
class GlobalStore {
  // Store 里面的泛型是储存的数据类型
  static Store<GlobalState> _globalStore;

  static Store<GlobalState> get store =>
      _globalStore ??= createStore<GlobalState>(GlobalState(), buildReducer());
}
```

store 里面储存全局数据。

```dart
abstract class GlobalBaseState {
  Color get themeColor;
  set themeColor(Color color);
}

class GlobalState implements GlobalBaseState, Cloneable<GlobalState> {
  @override
  Color themeColor;

  @override
  GlobalState clone() {
    return GlobalState();
  }
}
```

然后，在 PageRoutes 建立全局数据与页面数据的关系：

```dart
        visitor: (String path, Page<Object, dynamic> page) {
          /// 只有特定的范围的 Page 才需要建立和 AppStore 的连接关系
          /// 满足 Page<T> ，T 是 GlobalBaseState 的子类
          if (page.isTypeof<GlobalBaseState>()) {
            /// 建立 AppStore 驱动 PageStore 的单向数据连接
            /// 1. 参数1 AppStore
            /// 2. 参数2 当 AppStore.state 变化时, PageStore.state 该如何变化
            page.connectExtraStore<GlobalState>(GlobalStore.store, (Object pageState, GlobalState appState) {
              final GlobalBaseState p = pageState;
              if (p.themeColor != appState.themeColor) {
                if (pageState is Cloneable) {
                  final Object copy = pageState.clone();
                  final GlobalBaseState newState = copy;
                  newState.themeColor = appState.themeColor;
                  return newState;
                }
              }
              return pageState;
            });
          }
        }
```

当全局数据被改变的时候，会调用 visitor 函数，所以 page 中的 state 也会改变。可以猜出来，它其实就是一个 reducer。



### 代码例子

https://github.com/aprz512/first_flutter_demo
