---
title: flutter的BLOC模式
date: 2019-12-2
tags: flutter
categories: flutter
---

学习 Flutter 已经差不多两周，使用 BLOC 模式写了一个简单的 App，回想起来，居然与 Android 开发还挺相似的。可能是因为我们的应用本来就是使用 Google 推荐的框架，所以发现 BLOC 模式与我们应用的模式居然是一模一样，这可真的没有夸张，就是将 ViewModel 的名字改成了 BLOC 而已。

下面，从几个方面来说说自己对 Flutter 的感觉吧。



### Dart语言

现在的语言越来越多了，但是真正学起来会发现，其实上手都很简单，觉得语言由百花齐放慢慢的趋向于大一统。

不是说语言会变得单一，而是说它们的用法开始越来越像了。像我在大学接触 C++ 的时候，简直痛苦的不行。花了2个月，看了半本 C++ Primer，书上的习题都自己敲过，然后过了半年，全忘光了，一点没剩下。

我自己觉得，如果你对 Java 比较精通，不用看 dart 的语法，强行阅读，也是可以读的通的。你再有点 kotlin 的知识，简直如虎添翼。其他的细微末节，平时多看看项目，查查资料就好了。

但是，如果想深入了解，我似乎还没有找到什么好的资料。比如，Dart 的类加载机制，它的全局变量是如何工作的，是与 kotlin 一样吗？



### 界面编写

Flutter 编写界面，是真的难受。尤其是从 Android 开发转过来的，界面写起来的感觉与用 Java 代码写 Android 的界面一样。不过使用 Dart 写起来，还是方便不少的，而且 Flutter 对 Widget 封装的比较好，所以基本只需要填填属性值就好了，与 XML 一样。

因为是使用的 Dart 来编写界面，也没有提供界面预览的方式，所以如果你想要改一个界面，还得先运行起来看一下它长什么样。然后再慢慢的看代码，慢慢改。

里面的一些控件的用法与 CSS 很像，熟悉 html 开发的，应该会比较有亲切感。

在 Flutter 中，刚开始接触很容易产生万物皆为 Widget 的感觉，在编写界面的时候，也是一个 Widget “嵌套” 一个 Widget，会导致代码层次嵌套的特别深，所以需要把握控件的粒度。



### 界面绘制

flutter 有自己的绘制引擎。我们将开发者模式打开，想查看过度绘制与布局边界，都是看不了的，因为它走的不是 Android 的绘制逻辑。

Flutter 与 Android 还有一个更大的不同之处在于，Flutter 中的 widget 可以理解为是不可变的。举个例子：

在 Android 中，我们想要显示一个文字，使用 TextView 控件就好了，当我们想要将文字改变的时候，自需要调用这个控件的 setText 方法即可。

但是，在 Flutter 中，我们想要显示一串文字，可以使用 Text 控件，但是当我们想要改变显示的文字的时候，只能重新创建一个新的 Text 控件，替换原来的 Text 控件。

虽然感觉不可思议，但是实际上却非常简单，Flutter 里面会自动重新走一遍创建 Text 的逻辑，所以会自动生成新的对象。

所以，我们在 build 方法中写的东西，相当于是一个蓝图，而 flutter 会依照这个蓝图创建出多个对象出来。同时需要好好思考，如何传递数据，以便创建出来的对象都不相同。就拿 Text 来说，每次 build 会创建不同的 Text 出来，但是如果你的 Text 的属性都是固定的，虽然每次都会创建出不同的 Text 对象，但是它们显示的却是同样的文字，所以需要将显示的文字传递进来。



当然，还有更多可以探索的东西，就不细说了，给两个链接：

 https://www.youtube.com/watch?v=wE7khGHVkYY&feature=youtu.be 

https://www.youtube.com/watch?v=AqCMFXEmf3w

这两个视频都讲的非常的不错，看懂了之后，至少一般的界面逻辑，你都看得懂，知道为什么。



### BLOC 模式

界面的东西了解了之后，其他的就是如何搭建一个结构清晰的应用框架了。

其实，像我们做一些简单的 App，似乎很难涉及到复杂的逻辑。一般都是请求后台，然后展示数据，很少会遇到比较复杂的功能。更加高级一点的，再写一个自定义控件，这也不过是对于绘制api的使用。

说了这许多，还是回到 BLOC。这个模式与 MVVM 等都没有啥重大的区别，只要掌握了思想就好了。写出来的东西都是一样的。由于我们一般都是做的“简单App”，所以掌握了这个模式之后，写起来很是非常顺畅的，接手别人的代码也会变得很容易。

我们先来看一张图：

![](https://miro.medium.com/max/1305/1*MqYPYKdNBiID0mZ-zyE-mA.png)



看过 Google jetpack 教程的，应该就很熟悉这张图了。

界面相关的东西，我们就放到 ui 层，逻辑相关的放在 bloc 层，数据相关的放在 repository 层。

由于我们的应用一般都是从网络请求数据，所以我将 Repository 与 Network Provider 合并层了一层。如果有用到缓存的话，还是不要合并，然后再实现 一个 Cache Provider 即可。

应用结构目录如下：

![](1.png)

每个模块下，都有这样的一个结构，这样看起来就会比较清晰。

下面再说说每个目录里面应该放什么。

####  bloc 目录

对应于 android 的 viewModel 目录：

```dart
class MoviesBloc {
  final _repository = MovieRepository();
  final _moviesFetcher = PublishSubject<ItemModel>();

  Observable<ItemModel> get allMovies => _moviesFetcher.stream;

  fetchAllMovies() async {
    ItemModel itemModel = await _repository.fetchMovieList();
    _moviesFetcher.sink.add(itemModel);
  }

  dispose() {
    _moviesFetcher.close();
  }
}
```

可以看的到，与 viewModel 的写法是一样的。只不过在 Android 中，我们现在使用了 LiveData 替换掉了 Rx，但是用法是一样的。

Bloc 需要数据会从 Repository 里面取，取到之后放入到 stream 中，然后我们只需要监听 stream 返回的数据就可以更新界面了。

#### repository目录

我这里是将 repository 与 data provider 合并了。

```dart
class MovieRepository {

  Future<ItemModel> fetchMovieList() async {
    final response = await client.get("$_baseUrl/popular?api_key=$_apiKey");
    if (response.statusCode == 200) {
      // If the call to the server was successful, parse the JSON
      return ItemModel.fromJson(json.decode(response.body));
    } else {
      // If that call was not successful, throw an error.
      throw Exception('Failed to load post');
    }
  }

}
```

client 是网络请求对象。

这里才是真正发送请求的地方，只需要写请求逻辑即可。



#### models目录

这个不用多说，就是存放 json 对应的对象的。有一个蛋疼的地方，dart 似乎没有类似 Gson 的库，因为它不让使用反射，所以只能自己手动解析 Json。不过还好有人写了对应的Json解析插件，自动生成代码。



#### ui目录

这里就是存放app的页面的地方，对应的 android 的 activity/fragment 目录。



bloc 模式差不多就是这些东西了，从 Jetpack 转过来还是非常容易入手的，都是 Google 的一套东西，用起来也很舒服。



我的进度似乎就到这里了，其余的就是慢慢熟悉里面的 api，熟悉各种控件的特殊用法了。