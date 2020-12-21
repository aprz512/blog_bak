---
title: 002-Flutter控件捡漏
index_img: /cover/5.jpg
banner_img: /cover/top.jpg
date: 2020-8-5
categories: Flutter
---

#### Text.rich

```dart
  @override
  Widget build(BuildContext context) {
    return Text.rich(
      TextSpan(
//        text: "Hello World",
//        style: TextStyle(color: Colors.red, fontSize: 20)
          children: [
            TextSpan(text: "Hello World", style: TextStyle(color: Colors.red)),
            TextSpan(text: "Hello flutter", style: TextStyle(color: Colors.green)),
            WidgetSpan(child: Icon(Icons.favorite, color: Colors.red,)),
            TextSpan(text: "Hello dart", style: TextStyle(color: Colors.blue)),
          ]
      )
    );
  }
```

富文本，可以进行图文混排，以 span 为单位。



#### FadeInImage

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final title = 'Fade in images';

    return new MaterialApp(
      title: title,
      home: new Scaffold(
        appBar: new AppBar(
          title: new Text(title),
        ),
        body: new Stack(
          children: <Widget>[
            new Center(child: new CircularProgressIndicator()),
            new Center(
              child: new FadeInImage.memoryNetwork(
                placeholder: kTransparentImage,
                image:
                    'https://github.com/flutter/website/blob/master/_includes/code/layout/lakes/images/lake.jpg?raw=true',
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

显示一个占位符，然后在图像加载完显示时淡入。





#### Align

组件可以调整子组件的位置，并且可以根据子组件的宽高来确定自身的的宽高。

```dart
Container(
  height: 120.0,
  width: 120.0,
  color: Colors.blue[50],
  child: Align(
    alignment: Alignment.topRight,
    child: FlutterLogo(
      size: 60,
    ),
  ),
)
```

Alignment 会以**矩形的中心点作为坐标原点**。上面的例子，就是以 Container 的中心为坐标原点 （0, 0），左上角是 （-1，-1），右下角是（1，1）。

Align 还有两个参数：`widthFactor`和`heightFactor`是用于确定`Align` 组件本身宽高的属性；它们是两个缩放因子，会分别**乘以子元素**的宽、高。比如，child 的高是 100，heightFactor 设置为 2，则 Align 的高度是 200。

**Center 继承自 Align**。



#### Container - BoxShadow

Container 有个 BoxDecoration 参数，可以用于实现一些背景装饰。

BoxDecoration 里面有个 boxShadow 参数，它可以用于实现阴影效果。

```dart
decoration: new BoxDecoration(
    border: new Border.all(color: Color(0xFFFF0000), width: 0.5), // 边色与边宽度
// 生成俩层阴影，一层绿，一层黄， 阴影位置由offset决定,阴影模糊层度由blurRadius大小决定（大就更透明更扩散），阴影模糊大小由spreadRadius决定
    boxShadow: [
        BoxShadow(color: Color(0x99FFFF00), offset: Offset(5.0, 5.0), blurRadius: 10.0, spreadRadius: 2.0), 
        BoxShadow(color: Color(0x9900FF00), offset: Offset(1.0, 1.0)), BoxShadow(color: Color(0xFF0000FF))
    ],

```



#### Widget的大小是如何约束的

熟记这些规则：

- **首先，上层 widget 向下层 widget 传递约束条件。**
- **然后，下层 widget 向上层 widget 传递大小信息。**
- **最后，上层 widget 决定下层 widget 的位置。**

**更多细节：**

- Widget 会通过它的 **父级** 获得自身的约束。 约束实际上就是 4 个浮点类型的集合： 最大/最小宽度，以及最大/最小高度。
- 然后，这个 widget 将会逐个遍历它的 **children** 列表。向子级传递 **约束**（子级之间的约束可能会有所不同），然后询问它的每一个子级需要用于布局的大小。
- 然后，这个 widget 就会对它子级的 **children** 逐个进行布局。 （水平方向是 `x` 轴，竖直是 `y` 轴）
- 最后，widget 将会把它的大小信息向上传递至父 widget（包括其原始约束条件）。

看链接：https://juejin.im/post/6846687593745088526



#### Container - Text 为何没居中？

我们在 Container 里面套一个 Text，你会发现，Text 的文字不是居中的。这是因为，Text 的布局被强制为与 Container 的大小一样大了，也就是说，上层 widget 向下层 widget 传递约束条件，其中 最大/最小宽度，以及最大/最小高度都是最大值。

而，我们设置了 alignment 之后，发现，Text 居中了，这是因为 alignment 内部创建了一个 Align 控件，将 Text 包裹起来了，而 Align 不会强制 Text 的大小与自己一样，Container 强制 Align 大小与自己一样，所以就有了 Text 居中的效果。



```dart
ConstrainedBox(
   constraints: BoxConstraints(
      minWidth: 70, 
      minHeight: 70,
      maxWidth: 150, 
      maxHeight: 150,
   ),
   child: Container(color: Colors.red, width: 10, height: 10),
)
```

但我们在 homePage 里面传递这个参数的时候，发现，结果如下：

![Example 9 layout](https://flutter.dev/assets/ui/layout/layout-9-0fdac80cd5d63906e0af3dc1574fb79f22577cf69cfc0a17b3823f5b2bfa999c.png)

这个我还没搞懂，官方文档说参数被忽略了，但是我没搞清楚为啥被忽略。

看了下源码，是因为这里的约束时附加约束，它还是要结合父布局传过来的约束进行处理。

比如，我们传递的 minWidth 是 70，那么它需要进行计算，这里假设父布局的高度约束是 [400, 400]。

那么，新计算出来的约束实际上是将 70 clamp 到 [400, 400] 这个范围，得出结果是 400，所以相当于忽略了。

如果 父布局的高度约束是 [0, 400]，那么 clamp 出来的结果就是 70，则使用了约束的高度。



#### Expand 的空间分配问题

```dart
        children: <Widget>[
          /**
           * Flexible中的属性:
           * - flex
           * Expanded(更多) -> Flexible(fit: FlexFit.tight)
           * 空间分配问题
           */
          Expanded(flex: 1, child: Container(height: 60, color: Colors.red)),
          Expanded(
              flex: 2,
              child: Container(width: 1000, height: 100, color: Colors.green)),
          Container(width: 90, height: 80, color: Colors.blue),
          Container(width: 50, height: 120, color: Colors.orange),
        ],
```

两个 expand 的宽度比例为 1：2，就是 flex 的比例。

还有一个 Flexible，它的空间分配可以参考：严格约束或宽松约束。

一个宽松约束换句话来说就是设置了最大宽度/高度， 但是让允许其子 widget 获得比它更小的任意大小。 换句话来说，宽松约束的最小宽度/高度为 **0**。

严格约束给你了一种获得确切大小的选择。 换句话来说就是，它的最大/最小宽度是一致的，高度也一样。



#### SafeArea 与 SliverSafeArea

现在的手机出现了刘海屏，当你的界面没有 appbar 的时候，那么刘海的位置可能会导致界面的部分内容不可见。使用安全区域可以避免这个问题。

SafeArea 就是说只让界面在刘海屏下面的位置展示，不使用刘海的位置。

而 SliverSafeArea 有点不一样，它是刚开始显示的时候，不使用刘海的位置，但是如果你的布局可以滚动，它还是会滚动到刘海位置。这样的话体验会稍微好点。



#### SliverPadding

SliverPadding 与 SliverSafeArea 的行为有点类似。

通常我们会遇到这样的ui：在GridView 的顶部需要与 AppBar 有一定距离，我们通常会给 GridView 设置一个 Padding，但是这样会导致 GridView 滑动起来，始终与 AppBar 都有一段距离，而 SliverPadding 就可以解决这个问题，它刚开始的时候，与 AppBar 都有一段距离，但是滑动后，与 AppBar 没有距离。



#### NotificationListener

这个是一个 Widget，可以监听可控件的一些通知，比如：可以监听可滚动 Widget 什么时候开始滚动，什么时候结束滚动等。

```dart
class _MyHomePageState extends State<MyHomePage> {


  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: NotificationListener<ScrollNotification>(
        onNotification: (ScrollNotification notification){
          ScrollMetrics metrics = notification.metrics;
          print(metrics.pixels);// 当前位置
          print(metrics.atEdge);//是否在顶部或底部
          print(metrics.axis);//垂直或水平滚动
          print(metrics.axisDirection);// 滚动方向是down还是up
          print(metrics.extentAfter);//视口底部距离列表底部有多大
          print(metrics.extentBefore);//视口顶部距离列表顶部有多大
          print(metrics.extentInside);//视口范围内的列表长度
          print(metrics.maxScrollExtent);//最大滚动距离，列表长度-视口长度
          print(metrics.minScrollExtent);//最小滚动距离
          print(metrics.viewportDimension);//视口长度
          print(metrics.outOfRange);//是否越过边界
          print('------------------------');
          return true;
        },
          child: ListView.builder(
            itemExtent: 50,
              itemCount: 50,
              itemBuilder: (BuildContext context,int index){
              return ListTile(title: Text(index.toString()),);
              },
          ),
      ),
    );
  }
}
```

ScrollNotification 有很多子类，我们可以在里面判断：

```dart
if (notification is ScrollUpdateNotification)

if (notification is ScrollStartNotification)

if (notification is ScrollEndNotification)
```



#### CustomClipper

可以裁剪区域。比如，要做一个评分控件，有时候需要显示半颗星，那么需要对星星裁剪。

```java
class StarPath extends CustomClipper<Path> {
  StarPath({this.scale = 2.5});

  final double scale;

  double perDegree = 36;

  /// 角度转弧度公式
  double degree2Radian(double degree) {
    return (pi * degree / 180);
  }

  @override
  Path getClip(Size size) {
    ...
    return path;
  }

  @override
  bool shouldReclip(StarPath oldClipper) {
    return oldClipper.scale != this.scale;
  }
}
```

getClip 是生成我们想要的裁剪的区域的 path。

shouldReclip 表示是否需要重新裁剪。

