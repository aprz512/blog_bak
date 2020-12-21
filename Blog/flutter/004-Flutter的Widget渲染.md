---
title: 004-Flutter的Widget渲染
index_img: /cover/11.jpg
banner_img: /cover/top.jpg
date: 2020-8-11
categories: Flutter
---

事情要从[Flutter-你还在滥用StatefulWidget吗](https://lizhaoxuan.github.io/2019/01/02/Flutter-你还在滥用StatefulWidget吗/) 说起。看了这篇文章之后，我还是以为很有道理的，直到我看了 fish-redux 的大致结构，发现根本就没必要。而且这样做会带来很多麻烦，比如：组件的抽取，又要考虑不能多次 build，又要考虑复用，这两者几乎是冲突的，写出来的代码很难看。但是话说回来，能减少 build 的控件当然最好了，这样创建销毁的对象也少些。

fish-redux 里面将 page 作为一个整体来 build，每次 state 发生变化的时候，重新 build 一次就好了。即使只有一个 text 发生了变化，也要重新 build 一次。虽然看起来，build 了很多无用的东西，实际上渲染的时候，只渲染了变化的 text 部分。

为啥呢？这就要从 3 棵树说起。

#### Widget tree

第一个是 Widget 树，它的里面没有渲染相关的的东西，只负责**描述** Widget 的大小与位置。我们把它当作普通类理解就好了。

#### Element  tree

第二个是 Element 树，Element 是啥呢？每个 Widget 都有 build 方法（之类会在这个基础上再改变一下）：

```dart
// 这个是 StatefulWidget 的 build 方法
Widget build(BuildContext context);
```

这个 context 实际上就是一个 Element。

Element 可以理解为一个中间人，每个 Widget 都有一个对应的 Element 对象。我们对 Widget 树的更改会反应到 Element 树上。但是 Element 有一个 diff 机制，就是说它会复用之前的 Element 对象。

##### demo1

比如，屏幕上有3个 widget：

```java
  @override
  Widget build(BuildContext context) {
    if (map.containsKey(widget.name)) {
      print('isSame = ${context == map[widget.name]}, time = ${DateTime.now()}');
      print('old context = ${map[widget.name].toString()}, time = ${DateTime.now()}');
      print('context = ${context.toString()}, time = ${DateTime.now()}');
    }

    map[widget.name] = context;

    return Container(
      child: Text(
        widget.name,
        style: TextStyle(color: Colors.white, fontSize: 30),
      ),
      height: 80,
      color: randColor,
    );
  }
```

我们，每次 build 的时候，会生成新的 widget 对象，是肯定的，但是 context 却是复用的。打印下 log：

```
I/flutter ( 8694): isSame = true, time = 2020-08-06 23:08:50.033290
I/flutter ( 8694): old context = ListItemFul-[<'aaaa'>](dirty, state: _ListItemFulState#58614), time = 2020-08-06 23:08:50.033968
I/flutter ( 8694): context = ListItemFul-[<'aaaa'>](dirty, state: _ListItemFulState#58614), time = 2020-08-06 23:08:50.034196
I/flutter ( 8694): isSame = true, time = 2020-08-06 23:08:50.035217
I/flutter ( 8694): old context = ListItemFul-[<'bbbb'>](dirty, state: _ListItemFulState#7d32b), time = 2020-08-06 23:08:50.035636
I/flutter ( 8694): context = ListItemFul-[<'bbbb'>](dirty, state: _ListItemFulState#7d32b), time = 2020-08-06 23:08:50.035891
I/flutter ( 8694): isSame = true, time = 2020-08-06 23:08:50.037472
I/flutter ( 8694): old context = ListItemFul-[<'cccc'>](dirty, state: _ListItemFulState#cd85b), time = 2020-08-06 23:08:50.037819
I/flutter ( 8694): context = ListItemFul-[<'cccc'>](dirty, state: _ListItemFulState#cd85b), time = 2020-08-06 23:08:50.038083
```

无论，每次 build 多少次，发现都是复用的同一个 element 对象。这是因为它满足了复用规则：

```dart
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
```

##### demo2

我们接下来，改一下这个例子：

```java
  final List<String> names = ["aaaa", "bbbb", "cccc"]; 

  Widget buildPage() {
    return Scaffold(
      appBar: AppBar(
        title: Text("列表测试"),
      ),
      body: ListView(
        children: names.map((item) {
          return ListItemFul(
            item,
//            key: ValueKey(item),
          );
        }).toList(),
      ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.delete),
        onPressed: () {
          setState(() {
            names.removeAt(0);
            names.add('aaaa');
          });
        },
      ),
    );
  }
```

我们加了一个 button，点击这个 button，将第一个与第3个 widget 换个位置。看看打印的是什么信息：

```
I/flutter ( 8694): isSame = false, time = 2020-08-06 23:45:17.885164
I/flutter ( 8694): old context = ListItemFul(state: _ListItemFulState#93a82), time = 2020-08-06 23:45:17.886470
I/flutter ( 8694): context = ListItemFul(dirty, state: _ListItemFulState#65e5d), time = 2020-08-06 23:45:17.886706
I/flutter ( 8694): isSame = false, time = 2020-08-06 23:45:17.888019
I/flutter ( 8694): old context = ListItemFul(state: _ListItemFulState#5f8fd), time = 2020-08-06 23:45:17.888226
I/flutter ( 8694): context = ListItemFul(dirty, state: _ListItemFulState#93a82), time = 2020-08-06 23:45:17.888521
I/flutter ( 8694): isSame = false, time = 2020-08-06 23:45:17.889172
I/flutter ( 8694): old context = ListItemFul(state: _ListItemFulState#65e5d), time = 2020-08-06 23:45:17.889322
I/flutter ( 8694): context = ListItemFul(dirty, state: _ListItemFulState#5f8fd), time = 2020-08-06 23:45:17.889634
```

我们发现，element 不一样了，但是仔细看一下，会发现其实是，widget 1 复用了 widget 3 的 element，widget 2 复用了 widget 1 的 element，widget 3 复用了 widget 2 的 element。

这是因为，我们没传递 key，所以只要 runtimeType 一样，就可以复用。大致过程如下：

```
build 执行，
需要先 build 'bbbb' 控件
发现位置上有一个 'aaaa' 的 element
判断该 element 是否可用，发现可用，则直接使用。
重复上面的过程
```

##### demo3

再对该例子做点改动，将 `key: ValueKey(item),` 这行代码的注释去掉，再点击 button，打印结果如下：

```dart
I/flutter ( 8694): isSame = true, time = 2020-08-07 01:53:35.103764
I/flutter ( 8694): old context = ListItemFul-[<'aaaa'>](dirty, state: _ListItemFulState#28e81), time = 2020-08-07 01:53:35.104217
I/flutter ( 8694): context = ListItemFul-[<'aaaa'>](dirty, state: _ListItemFulState#28e81), time = 2020-08-07 01:53:35.104387
I/flutter ( 8694): isSame = true, time = 2020-08-07 01:53:35.104683
I/flutter ( 8694): old context = ListItemFul-[<'bbbb'>](dirty, state: _ListItemFulState#6e7c6), time = 2020-08-07 01:53:35.104839
I/flutter ( 8694): context = ListItemFul-[<'bbbb'>](dirty, state: _ListItemFulState#6e7c6), time = 2020-08-07 01:53:35.105229
I/flutter ( 8694): isSame = true, time = 2020-08-07 01:53:35.106242
I/flutter ( 8694): old context = ListItemFul-[<'cccc'>](dirty, state: _ListItemFulState#ab41f), time = 2020-08-07 01:53:35.106712
I/flutter ( 8694): context = ListItemFul-[<'cccc'>](dirty, state: _ListItemFulState#ab41f), time = 2020-08-07 01:53:35.106977
```

发现，每个 widget 都复用了自己原来的 element。这是为什么呢？是因为 ValueKey 是一个 LocalKey。它在源码里面有这样的一个注释：

```dart
Keys must be unique amongst the [Element]s with the same parent.
```

这句话就是说一个 parent 的所有 child 的 element 的 key 应该是唯一的，这样的话，方便复用。它会在 `RenderObjectElement.updateChildren` 里进行 diff 算法，计算 old 与 new 的可复用关系。因为，我们只是交换位置，所以 element 完全可以复用，设置一个 LocalKey就可以做到 parent 级别的复用。



#### RenderObject 

它才是真正负责 widget 渲染的树。每当 Widget 重新 build 后，element 会计算出哪些可复用的，拿到可复用的之后，就会开始更新 renderObject：

```dart
void updateRenderObject(BuildContext context, covariant RenderObject renderObject) { }
```

我们拿 RichText 举例：

```dart
  @override
  void updateRenderObject(BuildContext context, RenderParagraph renderObject) {
    assert(textDirection != null || debugCheckHasDirectionality(context));
    renderObject
      ..text = text
      ..textAlign = textAlign
      ..textDirection = textDirection ?? Directionality.of(context)
      ..softWrap = softWrap
      ..overflow = overflow
      ..textScaleFactor = textScaleFactor
      ..maxLines = maxLines
      ..strutStyle = strutStyle
      ..textWidthBasis = textWidthBasis
      ..textHeightBehavior = textHeightBehavior
      ..locale = locale ?? Localizations.localeOf(context, nullOk: true);
  }
```

可以看到，它就是重新设置了一下属性值而已。但是 RenderObject  在下一帧绘制的时候，就绘制的是改变之后的值。需要注意的是 RenderObject 数也正是重新渲染了变化的节点。



#### 参考文档

https://www.youtube.com/watch?v=996ZgFRENMs
