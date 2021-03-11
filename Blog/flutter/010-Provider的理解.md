---
title: Provider的理解
date: 2021-03-04
index_img: /cover/4.jpg
banner_img: /cover/top.jpg
categories: Flutter
---

Provider 是官方推荐的状态管理框架。那么状态又是什么东西？在 Flutter 中，万物都是 Widget，Widget 又分为两种：

- StatelessWidget（无状态）
- StatefulWidget  (有状态）

```
有些widgets是有状态的, 有些是无状态的
如果用户与widget交互，widget会发生变化，那么它就是有状态的.
widget的状态（state）是一些可以更改的值, 如一个slider滑动条的当前值或checkbox是否被选中.
widget的状态保存在一个State对象中, 它和widget的布局显示分离。
当widget状态改变时, State 对象调用setState(), 告诉框架去重绘widget.
```



### 管理状态

```
有多种方法可以管理状态.
如果不是很清楚时, 那就在父widget中管理状态吧.
```

以下是管理状态的最常见的方法：

- [widget管理自己的state](https://flutterchina.club/tutorials/interactive/#self-managed)
- [父widget管理 widget状态](https://flutterchina.club/tutorials/interactive/#parent-managed)
- [混搭管理（父widget和widget自身都管理状态））](https://flutterchina.club/tutorials/interactive/#mix-and-match)

如何决定使用哪种管理方法？以下原则可以帮助您决定：

- 如果状态是用户数据，如复选框的选中状态、滑块的位置，则该状态最好由父widget管理
- 如果所讨论的状态是有关界面外观效果的，例如动画，那么状态最好由widget本身来管理.



### Provider

通常，我们的一个页面会有许多控件，对应着许多状态。在看法Android的时候，每个页面对应着一个 ViewModel，这个 ViewModel 里面的数据是变化的，它的每个快照，就是页面该时刻的状态。

上面，我们说到，Widget 可以将状态的管理交给父类，实际上，通常使用 Provider 的时候，页面上所有的 widget 会将状态交给顶层 Widget 去管理，由于它是对 [InheritedWidget](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html) 进行了封装，所以页面上所有的子 Widget 可以进行数据共享与监听。



### InheritedWidget

下面介绍一下，InheritedWidget 是如何做到子 Widget 可以获取父 Widget 的数据的，以及一些特殊的效果。

我们使用 InheritedWidget 时涉及到的工作量主要有 2 部分：

- 创建一个继承自 InheritedWidget 的类，并将其插入 Widget 树
- 通过 BuildContext 对象提供的 `inheritFromWidgetOfExactType` 方法查找 Widget 树中最近的一个特定类型的 InheritedWidget 类的实例

这里还暗含了一个逻辑，那就是当我们通过 `inheritFromWidgetOfExactType` 查找特定类型 InheritedWidget 的时候，这个 InheritedWidget 的信息是由父元素层层向子元素传递下来的呢？还是 `inheritFromWidgetOfExactType` 方法自己层层向上查找的呢？

#### InheritedWidget 相关信息的传递机制

每个 Element 实例上都有一个 `_inheritedWidgets` 属性。该属性的类型为：

```dart
HashMap<Type, InheritedElement>
```

其中保存了祖先节点中出现的 InheritedWidget 与其对应 element 的映射关系。在 element 的 [`mount` 阶段](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L2754)和 [`active` 阶段](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L3019)，会执行 `_updateInheritance()` 方法更新这个映射关系。

对于普通 Element 实例，`_updateInheritance()` 只是[单纯把父 element 的 `_inheritedWidgets` 属性保存在自身 `_inheritedWidgets` 里](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L3251-L3254)。从而实现映射关系的层层向下传递。

```dart
  void _updateInheritance() {
    assert(_active);
    _inheritedWidgets = _parent?._inheritedWidgets;
  }
```

由 InheritedWidget 创建的 InheritedElement [重写了该方法](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L4039-L4047)：

```dart
  void _updateInheritance() {
    assert(_active);
    final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      _inheritedWidgets = new HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      _inheritedWidgets = new HashMap<Type, InheritedElement>();
    _inheritedWidgets[widget.runtimeType] = this;
  }
```

可以看出 InheritedElement 实例会把自身的信息添加到 `_inheritedWidgets` 属性中，这样其子孙 element 就可以通过前面提到的 `_inheritedWidgets` 的传递机制获取到此 InheritedElement 的引用。

> 总结一下：Widget 树中的依赖关系，是一层一层的往下传递的，如果遇到了 InheritedWidget ，还需要将这个 Widget 的依赖关系（这个 Widget 以及它向子Widget 提供的数据类型）添加进去。

#### InheritedWidget 的更新通知机制

首先，说一个小知识点。

前文提到 `_inheritedWidgets` 属性存在于 Element 实例上，而我们代码中调用的 `inheritFromWidgetOfExactType` 方法则存在于 BuildContext 实例之上。那么 BuildContext 是如何获取 Element 实例上的信息的呢？答案是**不需要获取**。因为**每一个 Element 实例也都是一个 BuildContext 实例**。

既然可以拿到 InheritedWidget 的信息了，那接下让我们通过源码看看更新通知机制的具体实现。

首先看一下 [`inheritFromWidgetOfExactType` 的实现](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L3230-L3242)：

```dart
  InheritedWidget inheritFromWidgetOfExactType(Type targetType) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[targetType];
    if (ancestor != null) {
      assert(ancestor is InheritedElement);
      _dependencies ??= new HashSet<InheritedElement>();
      _dependencies.add(ancestor);
      ancestor._dependents.add(this);
      return ancestor.widget;
    }
    _hadUnsatisfiedDependencies = true;
    return null;
  }
```

首先在 `_inheritedWidget` 映射中查找是否有特定类型 InheritedWidget 的实例。如果有则将该实例添加到自身的依赖列表中，同时将自身添加到对应的依赖项列表中。这样该 InheritedWidget 在更新后就可以通过其 `_dependents` 属性知道需要通知哪些依赖了它的 widget。

每当 [InheritedElement 实例更新](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L3914-L3923)时，会执行实例上的 [`notifyClients` 方法](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L4070-L4086)通知依赖了它的子 element 同步更新。`notifyClients` 实现如下：

```dart
  void notifyClients(InheritedWidget oldWidget) {
    if (!widget.updateShouldNotify(oldWidget))
      return;
    assert(_debugCheckOwnerBuildTargetExists('notifyClients'));
    for (Element dependent in _dependents) {
      assert(() {
        // check that it really is our descendant
        Element ancestor = dependent._parent;
        while (ancestor != this && ancestor != null)
          ancestor = ancestor._parent;
        return ancestor == this;
      }());
      // check that it really depends on us
      assert(dependent._dependencies.contains(this));
      dependent.didChangeDependencies();
    }
  }
```

首先执行相应 InheritedWidget 上的 `updateShouldNotify` 方法判断是否需要通知，如果该方法返回 `true` 则遍历 `_dependents` 列表中的 element 并执行他们的 `didChangeDependencies()` 方法。这样 InheritedWidget 中的更新就通知到依赖它的子 widget 中了。

> 总结一下：inheritFromWidgetOfExactType 先根据 type 找到 InheritedElement，然后储存依赖 element ，当 [InheritedElement 实例更新](https://github.com/flutter/flutter/blob/v0.5.7/packages/flutter/lib/src/widgets/framework.dart#L3914-L3923)时，通知储存的 element 去更新。

#### 好处

看一个例子：

```dart
class MyInheritedWidget extends InheritedWidget {
  final int accountId;
  final int scopeId;

  MyInheritedWidget(accountId, scopeId, child): super(child);
  
  @override
  bool updateShouldNotify(MyInheritedWidget old) =>
    accountId != old.accountId || scopeId != old.scopeId;
}

class MyPage extends StatelessWidget {
  final int accountId;
  final int scopeId;
  
  MyPage(this.accountId, this.scopeId);
  
  Widget build(BuildContext context) {
    return new MyInheritedWidget(
      accountId,
      scopeId,
      const MyWidget(),
     );
  }
}

class MyWidget extends StatelessWidget {

  const MyWidget();
  
  Widget build(BuildContext context) {
    // somewhere down the line
    const MyOtherWidget();
    ...
  }
}

class MyOtherWidget extends StatelessWidget {

  const MyOtherWidget();
  
  Widget build(BuildContext context) {
    final myInheritedWidget = MyInheritedWidget.of(context);
    print(myInheritedWidget.scopeId);
    print(myInheritedWidget.accountId);
    ...
```

1. `accountId`或者`scopeId`值更新的时候，MyInheritedWidget会被重新创建，但是它的child不一定会被创建，取决于是否用到了`accountId`或者`scopeId`，上面的这个例子中`MyOtherWidget`会被rebuild，**但是`MyWidget`不会被rebuild**
2. 如果tree被其他事件触发rebuild，例如orientation changes，`InheritedWidget`会被rebuild，但是child同样不一定被rebuild，因为`accountId`和`scopeId`没变。