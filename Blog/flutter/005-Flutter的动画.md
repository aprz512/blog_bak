---
title: 005-Flutter的动画
index_img: /cover/10.jpg
banner_img: /cover/top.jpg
date: 2020-8-10
categories: Flutter
---

###  Animation

- addListener方法

  每当动画的状态值发生变化时，动画都会通知所有通过 `addListener` 添加的监听器。（我们可以调用 setState 来更新动画效果）

- addStatusListener

  当动画的状态发生变化时，会通知所有通过 `addStatusListener` 添加的监听器。（我们可以在这里做些逻辑处理，比如：反转，重新开始等 ）

###  AnimationController

Animation是一个抽象类，并不能用来直接创建对象实现动画的使用。

AnimationController是Animation的一个子类，实现动画通常我们需要创建AnimationController对象。

- AnimationController会生成一系列的值，**默认情况下值是0.0到1.0区间的值**

AnimationController有一个必传的参数vsync（开发中比较常见的是将SingleTickerProviderStateMixin混入到State的定义中）。

### CurvedAnimation

它的目的是为了给AnimationController增加动画曲线。

### Tween

默认情况下，AnimationController动画生成的值所在区间是0.0到1.0，如果希望使用这个以外的值，或者其他的数据类型，就需要使用Tween。一个 AnimationController 可以使用多个 Tween，类似 Android的动画集合，一个动画集合可以包含透明度变化、大小变化、颜色变化、旋转动画等；

### AnimatedWidget

我们必须监听动画值的改变，并且改变后需要调用setState，这会带来两个问题：

- 执行动画必须包含这部分代码，代码比较冗余
- 调用setState意味着整个State类中的build方法就会被重新build

AnimatedWidget可以优化上面的操作。

### AnimatedBuilder

AnimatedWidget 是不是最佳的解决方案呢？

- 我们每次都要新建一个类来继承自AnimatedWidget
- 如果我们的动画Widget有子Widget，那么意味着它的子Widget也会重新build

AnimatedBuilder可以优化上面的操作。

```dart
class _AnimationDemo01State extends State<AnimationDemo01> with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> animation;

  @override
  void initState() {
    super.initState();

    // 1.创建AnimationController
    controller = AnimationController(duration: Duration(seconds: 1), vsync: this);
    // 2.动画添加Curve效果
    animation = CurvedAnimation(parent: controller, curve: Curves.elasticInOut, reverseCurve: Curves.easeOut);
    // 3.监听动画
    // 4.控制动画的翻转
    animation.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        controller.reverse();
      } elseif (status == AnimationStatus.dismissed) {
        controller.forward();
      }
    });
    // 5.设置值的范围
    animation = Tween(begin: 50.0, end: 120.0).animate(controller);
  }

  @override
  Widget build(BuildContext context) {
    return Center(
      child: AnimatedBuilder(
        animation: animation,
        builder: (ctx, child) {
          return Icon(Icons.favorite, color: Colors.red, size: animation.value,);
        },
      ),
    );
  }

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }
}
```



### 享元动画

在Flutter中，有一个专门的Widget可以来实现这种动画效果：Hero

实现Hero动画，需要如下步骤：

- 1.在第一个Page1中，定义一个起始的Hero Widget，被称之为source hero，并且绑定一个tag；
- 2.在第二个Page2中，定义一个终点的Hero Widget，被称之为 destination hero，并且绑定相同的tag；
- 3.可以通过Navigator来实现第一个页面Page1到第二个页面Page2的跳转过程；

Flutter会设置Tween来界定Hero从起点到终端的大小和位置，并且在图层上执行动画效果。

首页Page代码：

```dart
import'dart:math';

import'package:flutter/material.dart';
import'package:testflutter001/animation/image_detail.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
          primarySwatch: Colors.blue, splashColor: Colors.transparent),
      home: HYHomePage(),
    );
  }
}

class HYHomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Hero动画"),
      ),
      body: HYHomeContent(),
    );
  }
}

class HYHomeContent extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return GridView(
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        crossAxisSpacing: 8,
        mainAxisSpacing: 8,
        childAspectRatio: 2
      ),
      children: List.generate(20, (index) {
        String imageURL = "https://picsum.photos/id/$index/400/200";
        return GestureDetector(
          onTap: () {
            Navigator.of(context).push(PageRouteBuilder(
              pageBuilder: (ctx, animation, animation2) {
                return FadeTransition(
                  opacity: animation,
                  child: HYImageDetail(imageURL),
                );
              }
            ));
          },
          child: Hero(
            tag: imageURL,
            child: Image.network(imageURL)
          ),
        );
      }),
    );
  }
}

-------------------------------------------------
    
import'package:flutter/material.dart';

class HYImageDetail extends StatelessWidget {
  finalString imageURL;

  HYImageDetail(this.imageURL);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      body: Center(
        child: GestureDetector(
          onTap: () {
            Navigator.of(context).pop();
          },
          child: Hero(
            tag: imageURL,
            child: Image.network(
              this.imageURL,
              width: double.infinity,
              fit: BoxFit.cover,
            ),
          )),
      ),
    );
  }
}
```



