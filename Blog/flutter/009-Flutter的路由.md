---
title: 009-Flutter的路由
date: 2020-8-10
categories: Flutter
---

### Route

Route：一个页面要想被路由统一管理，必须包装为一个Route

### Navigator

Navigator：管理所有的Route的Widget，通过一个Stack来进行管理的

基本的跳转还是比较简单的，看如下例子。

#### demo1

点击按钮跳转到详情页面

```dart
_onPushTap(BuildContext context) {
  Navigator.of(context).push(MaterialPageRoute(
    builder: (ctx) {
      return DetailPage();
    }
  ));
}
```

#### demo2

详情页返回

```dart
_onBackTap(BuildContext context) {
  Navigator.of(context).pop("a detail message");
}
```

#### demo3

跳跳详细页，传递一个参数，参数在 page 的构造函数中接收。

```dart
_onPushTap(BuildContext context) {
  // 1.跳转代码
  final future = Navigator.of(context).push(MaterialPageRoute(
    builder: (ctx) {
      return DetailPage("a home message");
    }
  ));

  // 2.获取结果
  future.then((res) {
    setState(() {
      _message = res;
    });
  });
}
```

### 返回细节

监听返回按钮的点击（给Scaffold包裹一个WillPopScope）

- WillPopScope有一个onWillPop的回调函数，当我们点击返回按钮时会执行

- 这个函数要求有一个Future的返回值：

- - true：那么系统会自动帮我们执行pop操作
  - false：系统不再执行pop操作，需要我们自己来执行

### 命名路由

- 命名路由是将名字和路由的映射关系，在一个地方进行统一的管理
- 有了命名路由，我们可以通过`Navigator.pushNamed()` 方法来跳转到新的页面
- 命名路由在哪里管理呢？可以放在MaterialApp的 `initialRoute` 和 `routes` 中
  - `initialRoute`：设置应用程序从哪一个路由开始启动，设置了该属性，就不需要再设置`home`属性了
  - `routes`：定义名称和路由之间的映射关系，类型为Map<String, WidgetBuilder>

看如下例子：

```dart
return MaterialApp(
  title: 'Flutter Demo',
  theme: ThemeData(
    primarySwatch: Colors.blue, splashColor: Colors.transparent
  ),
  initialRoute: "/",
  routes: {
    "/home": (ctx) => HYHomePage(),
    "/detail": (ctx) => HYDetailPage()
  },
);
```

现在跳转可以使用如下方法：

```dart
_onPushTap(BuildContext context) {
  Navigator.of(context).pushNamed("/detail");
}
```

### 命名路由参数传递

pushNamed时，如何传递参数：

```dart
_onPushTap(BuildContext context) {
  Navigator.of(context).pushNamed(HYDetailPage.routeName, arguments: "a home message of naned route");
}
```

在HYDetailPage中，如何获取到参数呢？在**build**方法中ModalRoute.of(context)可以获取到传递的参数，由于只能在 build 方法中获取参数，所以一般不这样传递参数。

```
  Widget build(BuildContext context) {
    // 1.获取数据
    final message = ModalRoute.of(context).settings.arguments;
  }
```

#### onGenerateRoute

onGenerateRoute的钩子函数：

- 当我们通过pushNamed进行跳转，但是对应的name没有在routes中有映射关系，那么就会执行onGenerateRoute钩子函数；

- 我们可以在该函数中，手动创建对应的Route进行返回；

- 该函数有一个参数RouteSettings，该类有两个常用的属性：

- - name: 跳转的路径名称
  - arguments：跳转时携带的参数

```dart
onGenerateRoute: (settings) {
  if (settings.name == "/about") {
    return MaterialPageRoute(
      builder: (ctx) {
        return HYAboutPage(settings.arguments);
      }
    );
  }
  return null;
},
```

#### onUnknownRoute

如果我们打开的一个路由名称是根本不存在，这个时候我们希望跳转到一个统一的错误页面。

```dart
onUnknownRoute: (settings) {
  return MaterialPageRoute(
    builder: (ctx) {
      return UnknownPage();
    }
  );
},
```

