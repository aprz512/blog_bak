## Dart简要笔记2



### 路由管理

路由(Route)在移动开发中通常指页面（Page） 。

Route在Android中通常指一个Activity，在iOS中指一个ViewController。 

所谓路由管理，就是**管理页面之间如何跳转**，通常也可被称为导航管理。Flutter中的路由管理和原生开发类似，无论是Android还是iOS，**导航管理都会维护一个路由栈，路由入栈(push)操作对应打开一个新页面，路由出栈(pop)操作对应页面关闭操作**，而路由管理主要是指如何来管理路由栈。 



#### 一个例子

```dart
// 它继承了 StatelessWidget类，这也就意味着Route本身也是一个widget
// 其实，应用本身也是一个 widget
class NewRoute extends StatelessWidget {
    
    // Flutter在构建页面时，会调用组件的build方法，widget的主要工作是提供一个build()方法来描述如何构建UI界面（通常是通过组合、拼装其它基础widget）。
  @override
  Widget build(BuildContext context) {
      
      // Scaffold 是一个实现基本的material design 的布局容器
    return Scaffold(
        // 标题栏
      appBar: AppBar(
        title: Text("New route"),
      ),
        // 内容
      body: Center(
        child: Text("This is new route"),
      ),
    );
  }
}
```



跳转到新的 Route，需要使用Navigator。下面的例子，给按钮加了一个跳转的点击事件：

```dart
FlatButton(
    child: Text("open new route"),
    textColor: Colors.blue,
    onPressed: () {
        Navigator.push(context, MaterialPageRoute(builder: (context) {
            return NewRoute();
        }));
    });
```



#### MaterialPageRoute

 `MaterialPageRoute`继承自`PageRoute`类，`PageRoute`类是一个抽象类，表示占有整个屏幕空间的一个模态路由页面，它还定义了**路由构建及切换时过渡动画**的相关接口及属性。 



#### Navigator

 `Navigator`是一个路由管理的组件，它提供了打开和退出路由页方法。`Navigator`通过一个栈来管理活动路由集合。**通常当前屏幕显示的页面就是栈顶的路由** 。

> Future push(BuildContext context, Route route)

将给定的路由入栈（即打开新的页面），返回值是一个`Future`对象，用以接收新路由出栈（即关闭）时的返回数据。

需要注意它的 pop 方法：

> bool pop(BuildContext context, [ result ])

将栈顶路由出栈，**`result`为页面关闭时返回给上一个页面的数据**。 只有显示调用了 pop 才有返回值，如果用户直接按了返回键，导致没有走 pop 调用逻辑，那么是不会有返回值的。



### 命名路由

 所谓“命名路由”（Named Route）即有名字的路由，我们可以先给路由起一个名字，然后就可以通过路由名字直接打开新的路由了，这为路由管理带来了一种直观、简单的方式。 



#### 路由表

 要想使用命名路由，我们必须先提供并注册一个路由表（routing table） ，如下

```dart
MaterialApp(
  title: 'Flutter Demo',
  theme: ThemeData(
    primarySwatch: Colors.blue,
  ),
  //注册路由表
  routes:{
   "new_page":(context)=>NewRoute(),
    ... // 省略其它路由注册信息
  } ,
  home: MyHomePage(title: 'Flutter Demo Home Page'),
);
```



routes 是一个 Map<String, WidgetBuilder> routes; 类型。

> 你可能注意到，上面的示例代码中，我们传递的是一个 Function 类型，怎么map 里面是 WidgetBuilder 类型呢？这对不上啊。
>
> 其实是这样的：
>
> typedef WidgetBuilder = Widget Function(BuildContext context);
>
> typedef  这个玩意真是让人又爱又恨啊



我们可以将 home 也注册到路由表里面，不过还需要配置一下 initialRoute，声明初始路由页面：

```dart
MaterialApp(
  title: 'Flutter Demo',
  initialRoute:"/", //名为"/"的路由作为应用的home(首页)
  theme: ThemeData(
    primarySwatch: Colors.blue,
  ),
  //注册路由表
  routes:{
   "new_page":(context)=>NewRoute(),
   "/":(context)=> MyHomePage(title: 'Flutter Demo Home Page'), //注册首页路由
  } 
);
```



#### 通过路由名打开新路由页

可以使用`Navigator` 的`pushNamed`方法 。



#### 命名路由参数传递

在Flutter最初的版本中，命名路由是不能传递参数的，后来才支持了参数； 

在打开路由时传递参数 ：

```dart
Navigator.of(context).pushNamed("new_page", arguments: "hi");
```

在路由页通过`RouteSetting`对象获取路由参数： 

```dart
class EchoRoute extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    //获取路由参数  
    var args=ModalRoute.of(context).settings.arguments
    //...省略无关代码
  }
}
```



#### 路由生成钩子

 假设我们要开发一个电商APP，当用户没有登录时可以看店铺、商品等信息，但交易记录、购物车、用户个人信息等页面需要登录后才能看。为了实现上述功能，我们需要在打开每一个路由页前判断用户登录状态！如果每次打开路由前我们都需要去判断一下将会非常麻烦，那有什么更好的办法吗？答案是有！ 

 `MaterialApp`有一个`onGenerateRoute`属性，它在打开命名路由时**可能会被调用**，之所以说可能，是因为当调用`Navigator.pushNamed(...)`打开命名路由时，如果指定的路由名在路由表中已注册，则会调用路由表中的`builder`函数来生成路由组件；如果路由表中没有注册，才会调用`onGenerateRoute`来生成路由。 

>  注意，`onGenerateRoute`只会对命名路由生效。 

有了`onGenerateRoute`回调，要实现上面控制页面权限的功能就非常容易：**我们放弃使用路由表**，取而代之的是提供一个`onGenerateRoute`回调，然后在该回调中进行统一的权限控制，如： 

```dart
MaterialApp(
  ... //省略无关代码
  onGenerateRoute:(RouteSettings settings){
      return MaterialPageRoute(builder: (context){
           String routeName = settings.name;
       // 如果访问的路由页需要登录，但当前未登录，则直接返回登录页路由，
       // 引导用户登录；其它情况则正常打开路由。
     }
   );
  }
);
```

既然放弃使用路由表，那么还是需要一个map将路由给存起来，然后在 onGenerateRoute 中根据 name 来获取对应的 WidgetBuilder.



### 包管理

 flutter使用配置文件`pubspec.yaml`（位于项目根目录）来管理第三方依赖包。 

 https://www.dartlang.org/tools/pub/dependencies 。 



### 资源管理



#### assets

 和包管理一样，Flutter也使用[`pubspec.yaml`](https://www.dartlang.org/tools/pub/pubspec)文件来管理应用程序所需的资源 



#### 加载 assets