---
title: 003-Flutter的异步操作
index_img: /cover/5.jpg
banner_img: /cover/top.jpg
date: 2020-8-5
categories: Flutter
---

#### 如何catch error？

```java
void testFuture(){
  Future future = new Future(() => null);
  future.then((_){
    print("then");
  }).catchError((_){
    print("catchError");
  });
}
```

这是比较常见的一种捕获异常的方式，但是 then 里面也有一个 onError 参数，是可选命名参数。

那么 它 与 catchError 有啥区别呢？

then 里面的 onError 只会捕获当前 future.then 创建的新的 Future 里面的异常。

catchError 会捕获所有异常，除非你设置的 test 条件。

```dart
// 代码捕获了异常
Future future = new Future(() => null);
future.then((_){
    throw Exception('1');
}).then((e){
    throw Exception('2');
}).catchError((_){
    print("catchError");
});

// 代码未完全捕获异常
Future future = new Future(() => null);
future.then((_){
    throw Exception('1');
}, onError: (e) {
    print("catchError");
}).then((e){
    throw Exception('2');
});
```



#### error 指定参数会出错

在 catchError 里面需要参入一个参数，是一个函数，函数也有一个参数。

一般的我们都这样写：

```dart
catchError((e){
    print("catchError");
});
```

如果，我们确定了 e 的类型是 Exception，那么可能想这样写：

```dart
catchError((Exception e){
    print("catchError");
});
```

但是运行会报错，是因为 catchError 指定了函数的参数必须要是 dynamic 的，所以我们不能强制指定类型，只能判断后处理。



#### 奇怪的阻塞行为

看如下代码：

```dart
Futurn<String> network() async {
    await sleep(Duration(seconds: 3));
    return "123";
}
```

在 main 里面直接调用这个函数，发现 main 还是阻塞了，是为什么呢？

是因为该函数的 return 语句没有立刻执行，它调用了 sleep，没有立刻返回一个 future，所以 main 就阻塞了。



#### async * / sync *

先看一个例子：

```dart
foo1 (){	
  print('foo1 start');	
  for(int i = 0; i < 3; i++){	
    print(i);	
  }	
  print('foo1 stop');	
}
```

运行这个函数，输出：

```
foo1 start

0 1 2

foo1 stop
```

我们对上面的例子使用 `sync *`，如下：

```dart

Iterable<int> foo2() sync*{	
  print('foo2 start');	
  for(int i = 0; i < 3; i++){	
    print('运行了foo2，当前index：${i}');	
    yield i;	
  }	
  print('foo2 stop');	
}
```

这回我们在 `main` 函数里运行 `foo2()`，会出现什么效果？

答案是**什么也不会发生**，print也没有打印。这是为什么呢？！！

当我们调用 `foo2()`的时候，这里会**马上返回**一个 `Iterable`，就像网络请求会马上返回一个 `Future`一样。在我们没有调用 `Iterable` 的 `moveNext` 的时候，当前函数体是不会执行的。而当我们调用了 `moveNext` 方法后，代码会执行到 `yield` 关键字的位置，并且在这里停住。当我们再一次调用 `moveNext` 后，会再恢复执行，然后再次停到 `yield` 关键字的位置，依次循环，当没有下一个值得时候，函数会隐式的调用 return方法来终止函数。

看看运行输出：

```dart
var b = foo2().iterator;	
print('还没开始调用 moveNext');	
b.moveNext();	
print('第${b.current}次moveNext');	
b.moveNext();	
print('第${b.current}次moveNext');	
b.moveNext();	
print('第${b.current}次moveNext');
```

结果为：

```dart
还没开始调用 moveNext	
foo2 start	
运行了foo2，当前index：0	
第0次moveNext	
运行了foo2，当前index：1	
第1次moveNext	
运行了foo2，当前index：2	
第2次moveNext
```



说异步生成器之前，先来说一下普通的异步调用。

现在有一个这样的需求，我想每隔一秒钟请求一下数据，一共请求10次，看看有没有人关注我等等，

如果使用原始的 async，该怎么做？

```javascript
getData() async {	
  for (int i = 0; i < 10; i++){	
    await Future.delayed(Duration(seconds: 1), ()async {	
        Data data = await getXXX();	
      setState(){	
        //业务逻辑	
      };	
    });	
  }	
}
```

这里使用循环，然后每一秒钟请求依次接口，返回数据后 setState();

这样肯定不行，因为你不可能一两秒钟就 setState()一次，

这个时候 async* 就派上用场了：

```javascript
Stream<Data> getData() async* {	
  for (int i = 0; i < 10; i++){	
    await Future.delayed(Duration(seconds: 1));	
    yield await getXXX();	
  }	
}
```

在页面上，我们可以用 `StreamBuilder` 来包住，这样每次返回数据就不用 setState() 了。
