---
title: 006-Flutter的状态管理
index_img: /cover/10.jpg
banner_img: /cover/top.jpg
date: 2020-8-10
categories: Flutter
---

### InheritedWidget

可以实现跨组件数据的传递。需要 Widget 之间有同一个 Parent。

```dart
class HYDataWidget extends InheritedWidget {
  finalint counter;

  HYDataWidget({this.counter, Widget child}): super(child: child);

  // 该方法通过context开始去查找祖先的HYDataWidget
  static HYDataWidget of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType();
  }

  // 对比新旧HYDataWidget，是否需要对更新相关依赖的Widget
  @override
  bool updateShouldNotify(HYDataWidget oldWidget) {
    returnthis.counter != oldWidget.counter;
  }
}
```

### Provider

Provider是目前官方推荐的全局状态管理工具。

在使用Provider的时候，我们主要关心三个概念：

- ChangeNotifier：真正数据（状态）存放的地方
- ChangeNotifierProvider：Widget树中提供数据（状态）的地方，会在其中创建对应的ChangeNotifier
- Consumer：Widget树中需要使用数据（状态）的地方

**第一步：创建自己的ChangeNotifier**

```dart
class CounterProvider extends ChangeNotifier {
  int _counter = 100;
  int get counter {
    return _counter;
  }
  set counter(int value) {
    _counter = value;
    notifyListeners();
  }
}
```

**第二步：在Widget Tree中插入ChangeNotifierProvider**

我们需要在Widget Tree中插入ChangeNotifierProvider，以便Consumer可以获取到数据：

```
void main() {
  runApp(ChangeNotifierProvider(
    create: (context) => CounterProvider(),
    child: MyApp(),
  ));
}
```

**第三步：在接收处处理事件**

```dart
      body: Center(
        child: Consumer<CounterProvider>(
          builder: (ctx, counterPro, child) {
            return Text("当前计数:${counterPro.counter}", style: TextStyle(fontSize: 20, color: Colors.red),);
          }
        ),
      ),
```

**第四步：在按钮处，发送事件**

```dart
floatingActionButton: Selector<CounterProvider, CounterProvider>(
  selector: (ctx, provider) => provider.count,
  shouldRebuild: (pre, next) => false,
  builder: (ctx, counterPro, child) {
    print("floatingActionButton展示的位置builder被调用");
    return FloatingActionButton(
      child: child,
      onPressed: () {
        // 调用 set counter 方法，发送通知
        counterPro.counter += 1;
      },
    );
  },
  child: Icon(Icons.add),
),
```

### MultiProvider

在开发中，我们需要共享的数据肯定不止一个，并且数据之间我们需要组织到一起，所以一个Provider必然是不够的。

```dart
runApp(MultiProvider(
  providers: [
    ChangeNotifierProvider(create: (ctx) => CounterProvider()),
    ChangeNotifierProvider(create: (ctx) => UserProvider()),
  ],
  child: MyApp(),
));
```



### Selector 与 Consumer

**Selector控制的粒度比Consumer更细，Consumer是监听一个Provider中所有数据的变化，Selector则是监听某一个/多个值的变化**。

- Selector相当于Cosumer，但是可以在某些值不变的情况下，防止rebuild。

- selector方法：Selector使用 Provider.of获取共享的数据。数据作为selector方法的入参A，执行selector方法，返回build需要的数据S，返回的数据要尽可能少，能满足build就好。

- shouldRebuild：默认判断前后两次S相等性，来决定是否rebuild。并且也提供了自定义的shouldRebuild方法来判断,参数是前后两次S。

- S：selector的数据，必须是immutable(不可变)的.因此 selector通常返回集合或覆盖了"=="的类。

### 参考文档

https://blog.csdn.net/u013894711/article/details/102785532
