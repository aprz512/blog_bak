## Dart简要笔记1



### 变量



#### var

Dart 是强类型语言， 它可以接收任何类型的变量，但是var**变量一旦赋值，类型便会确定**，则不能再改变其类型 。

这实际上是属于自动类型推断：

> Dart在编译时会根据第一次赋值数据的类型来推断其类型，编译结束后其类型就已经被确定 

```
// 在java中
int x = 10;

// 在dart中
var x =10;

x 都是整形
```



#### dynamic 与 object

 `Object` 是Dart所有对象的根基类 。

`dynamic`与`Object`的特殊之处在于，他们声明的变量**可以在后期改变赋值类型**，可理解为弱类型，与 var 对应。 

```dart
 dynamic t;
 Object x;
 t = "hi world";
 x = 'Hello Object';
 //下面代码没有问题
 t = 1000;
 x = 1000;

// 可以暂时理解为父类接收了子类的类型
```



`dynamic`与`Object`不同的是，`dynamic`声明的对象编译器会提供所有可能的组合， 而`Object`声明的对象只能使用Object的属性与方法，否则编译器会报错（**只须记住dynamic相当于自动做了强制转换即可**）。如: 

```dart
 dynamic a;
 Object b;
 main() {
     a = "";
     b = "";
     printLengths();
 }   

 printLengths() {
     // no warning
     print(a.length);
     // warning:
     // The getter 'length' is not defined for the class 'Object'
     print(b.length);
 }

// 可以理解为编译器可以分辨出 dynamic （应该是程序流的最后一次赋值）变量的具体类型，然后编译器就可以支持调用该类型上的方法，IDE也可以提示（应该是提前编译了）
// 想象的出，这样不固定类型的话，在多线程下会很容易出问题，即使非多线程，如果暴露的引用过多，也很容易出问题
```



#### final与const

都是设置常量， `const` 变量是一个编译时常量，`final`变量在第一次使用时被初始化。被`final`或者`const`修饰的变量，变量类型可以省略 ：

```dart
//可以省略String这个类型声明
final str = "hi world";
//final String str = "hi world"; 
const str1 = "hi world";
//const String str1 = "hi world";
```



### 函数

Dart是一种真正的面向对象的语言，所以即使是**函数也是对象**，并且有一个类型**Function**。这意味着**函数可以赋值给变量或作为参数传递给其他函数**，这是函数式编程的典型特征。 



**函数定义与Java差不多**，不同之处有几个：

####不指定返回类型，此时默认为dynamic，不是bool

```dart
// 函数返回值没有类型推断
isNoble(int atomicNumber) {
  return _nobleGases[atomicNumber] != null;
}
```



####  对于只包含一个表达式的函数，可以使用简写语法 

```dart
// 能接受 lambda 的，这个应该也很好接受，无非就是将 {} 改成了 =>
bool isNoble (int atomicNumber)=> _nobleGases [ atomicNumber ] ！= null ;


void main() => runApp(MyApp());
```



#### 函数作为变量

```java
// 将一个函数赋值给 say 变量， = 右边是一个函数，这又是函数的另一种写法，省略函数名与返回值
// 这里的函数返回值似乎是 dynamic 类型，但是如果函数里面有分支，而分支里面返回了不同的类型，函数的返回类型似乎是 object，都只是测试出来的，没有查看详细的官方文档
var say = (str){
  print(str);
};
say("hi world");
```



####  可选的位置参数 

**可选的，可以不传。**

用[]标记为可选的位置参数，只能放在最后，否则无法编译

```dar
String say(String from, String msg, [String device]) {
  var result = '$from says $msg';
  if (device != null) {
    result = '$result with a $device';
  }
  return result;
}
```



####  可选的命名参数 

**可选的，可以不传。**

 定义函数时，使用{param1, param2, …}，用于指定命名参数。 

```dart
//设置[bold]和[hidden]标志
void enableFlags({bool bold, bool hidden}) {
    // ... 
}
```

调用的时候，可以指定参数名：

```dart
enableFlags(bold: true, hidden: false);
```

这个只是一个便于开发人员阅读代码的时候更容易理解的特性，不然的话，传递一个true，一个false，不戳一下源码，看方法参数与注解，鬼知道这两个表示啥意思。



### 异步支持

异步：在设置好一些耗时操作之后立即返回，比如像 IO操作。而不是等到这个操作完成。 



#### Future

表示一个异步操作的最终完成（或失败）及其**结果值**的表示。 

>  它就是用于处理异步操作的，异步处理成功了就执行成功的操作，异步处理失败了就捕获错误或者停止后续操作。一个Future只会对应一个结果，要么成功，要么失败。 

`Future` 的所有API的返回值仍然是一个`Future`对象，所以可以很方便的进行链式调用。 



##### API 介绍

`Future.delayed` ：创建了一个延时任务 

`Future.then` 创： 然后我们在`then`中接收异步结果 

`Future.catchError` ：如果异步任务发生错误，我们可以在`catchError`中捕获错误.

`Future.whenComplete`： 有些时候，我们会遇到无论异步任务执行成功或失败都需要做一些事的场景，比如在网络请求前弹出加载对话框，在请求结束后关闭对话框。 

`Future.wait`： 有些时候，我们需要等待多个异步任务都执行结束后才进行一些操作，比如我们有一个界面，需要先分别从两个网络接口获取数据，获取成功后，我们需要将两个接口数据进行特定的处理后再显示到UI界面上 。

一个完整的例子：

```dart
Future.wait([
  // 2秒后返回结果  
  Future.delayed(new Duration(seconds: 2), () {
    return "hello";
  }),
  // 4秒后返回结果  
  Future.delayed(new Duration(seconds: 4), () {
    return " world";
  })
]).then((results){
  print(results[0]+results[1]);
}).catchError((e){
  print(e);
}).whenComplete((){
   //无论成功或失败都会走到这里
});
```



#### Async/await



##### 什么是地狱回调

想象一下这样的业务逻辑：

比如现在有个需求场景是用户先登录，登录成功后会获得用户ID，然后通过用户ID，再去请求用户个人信息，获取到用户个人信息后，为了使用方便，我们需要将其缓存在本地文件系统 。

对于这样的需求，使用 Future 会怎么样呢？初学者可能都会这样写：

```dart
//先分别定义各个异步任务
Future<String> login(String userName, String pwd){
    ...
    //用户登录
};
Future<String> getUserInfo(String id){
    ...
    //获取用户信息 
};
Future saveUserInfo(String userInfo){
    ...
    // 保存用户信息 
};

login("alice","******").then((id){
 //登录成功后通过，id获取用户信息    
 getUserInfo(id).then((userInfo){
    //获取用户信息后保存 
    saveUserInfo(userInfo).then((){
       //保存用户信息，接下来执行其它操作
        ...
    });
  });
})
```

可以感受一下，如果业务逻辑中有大量异步依赖的情况，将会出现上面这种在**回调里面套回调**的情况，过多的嵌套会导致的代码可读性下降以及出错率提高，并且非常难维护，这个问题被形象的称为**回调地狱（Callback Hell）**。 

那么应该如何消除呢？？？出现回调里面嵌套回调，是因为在then里面，又调用了 then 方法。能不能不在then里面调用then呢？看下面的写法：

```dart
login("alice","******").then((id){
      return getUserInfo(id);
}).then((userInfo){
    return saveUserInfo(userInfo);
}).then((e){
   //执行接下来的操作 
}).catchError((e){
  //错误处理  
  print(e);
});
```

其实逻辑是一样的，但是then里面返回了一个 Future对象，这样就将嵌套的逻辑给拉平了。

  *“`Future` 的所有API的返回值仍然是一个`Future`对象，所以可以很方便的进行链式调用”* ，如果在then中返回的是一个`Future`的话，该`future`会执行，执行结束后会触发后面的`then`回调，这样依次向下，就避免了层层嵌套。 



##### 使用async/await消除callback hell

 通过`Future`回调中再返回`Future`的方式虽然能避免层层嵌套，但是还是有一层回调，有没有一种方式能够让我们可以**像写同步代码那样来执行异步任务**而不使用回调的方式？答案是肯定的，这就要使用`async/await`了 。

```dart
task() async {
   try{
    String id = await login("alice","******");
    String userInfo = await getUserInfo(id);
    await saveUserInfo(userInfo);
    //执行接下来的操作   
   } catch(e){
    //错误处理   
    print(e);   
   }  
}
```

- `async`用来表示函数是异步的，**是一个修饰符**，定义的函数会返回一个`Future`对象，可以使用then方法添加回调函数。
- `await` 后面是一个`Future`，**也是一个修饰符**，表示等待该异步任务完成，异步完成后才会往下走；`await`必须出现在 `async`函数内部。

这种写法，阅读起来会比较方便，就像面向过程编程一样，不过个人还是倾向于 Futrue 的链式写法，阅读起来逻辑也很清晰。



#### Stream

 `Stream` 也是用于接收异步事件数据，和`Future` 不同的是，它可以接收多个异步操作的结果（成功或失败）。 也就是说，在执行异步任务时，可以通过多次触发成功或失败事件来传递结果数据或错误异常。 `Stream` 常用于会多次读取数据的异步任务场景，如网络内容下载、文件读写等。举个例子： 

```java
  Stream.fromFutures([
    // 1秒后返回结果
    Future.delayed(new Duration(seconds: 1), () {
      return "hello 1";
    }),
    // 抛出一个异常
    Future.delayed(new Duration(seconds: 2), () {
      throw AssertionError("Error");
    }),
    // 3秒后返回结果
    Future.delayed(new Duration(seconds: 3), () {
      return "hello 3";
    })
  ]).listen((data) {
    print(data);
  }, onError: (e) {
    print(e.message);
  }, onDone: () {
    print("done");
  });
```

会输出：

```
hello 1
Error
hello 3
done
```

