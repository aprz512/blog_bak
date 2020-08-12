---
title: 0007-Flutter的主题
date: 2020-8-10
categories: Flutter
---

Theme分为：全局Theme和局部Theme

主题有两个作用：

- 设置了主题之后，某些Widget会自动使用主题的样式（比如AppBar的颜色）
- 将某些样式放到主题中统一管理，在应用程序的其它地方直接引用

###  全局Theme

全局Theme会影响整个app的颜色和字体样式。

使用起来非常简单，只需要向MaterialApp构造器传入 `ThemeData` 即可。

- 如果没有设置Theme，Flutter将会使用预设的样式。
- 当然，我们可以对它进行定制。

```
class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        // 1.亮度: light-dark
        brightness: Brightness.light,
        // 2.primarySwatch: primaryColor/accentColor的结合体
        primarySwatch: Colors.red,
        // 3.主要颜色: 导航/底部TabBar
        primaryColor: Colors.pink,
        // 4.次要颜色: FloatingActionButton/按钮颜色
        accentColor: Colors.orange,
        // 5.卡片主题
        cardTheme: CardTheme(
          color: Colors.greenAccent,
          elevation: 10,
          shape: Border.all(width: 3, color: Colors.red),
          margin: EdgeInsets.all(10)
        ),
        // 6.按钮主题
        buttonTheme: ButtonThemeData(
          minWidth: 0,
          height: 25
        ),
        // 7.文本主题
        textTheme: TextTheme(
          title: TextStyle(fontSize: 30, color: Colors.blue),
          display1: TextStyle(fontSize: 10),
        )
      ),
      home: HYHomePage(),
    );
  }
}
```

### 局部Theme

如果某个具体的Widget不希望直接使用全局的Theme，而希望自己来定义，应该如何做呢？

- 非常简单，只需要在Widget的父节点包裹一下Theme即可

创建另外一个新的页面，页面中使用新的主题：

- 在新的页面的Scaffold外，包裹了一个Theme，并且设置data为一个新的ThemeData

```
class HYSecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Theme(
      data: ThemeData(),
      child: Scaffold(
      ),
    );
  }
}
```

但是，我们很多时候并不是想完全使用一个新的主题，而且在之前的主题基础之上进行修改：

```
class HYSecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Theme(
      data: Theme.of(context).copyWith(
        primaryColor: Colors.greenAccent
      ),
      child: Scaffold(
      ),
    );
  }
}
```

### darkTheme

目前很多应用程序都需要适配暗黑模式，Flutter中如何做到暗黑模式的适配呢？

事实上，MaterialApp中有theme和dartTheme两个参数：

- 按照下面的写法，我们已经默认适配了暗黑主题

```
class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData.light(),
      darkTheme: ThemeData.dark(),
      home: HYHomePage(),
    );
  }
}
```