---
title: 001-Dart语法捡漏
index_img: /cover/4.jpg
banner_img: /cover/top.jpg
date: 2020-8-4
categories: Flutter
---

#### Object 与 dynamic

Dart 中所有的类都继承于 Object，所以我们可以这样写：

```dart
Object x = '123';
```

dynamic 是动态类型，会自己做类型推断，所以也可以这样写：

```dart
dynamic x = '123';
```

他们之间的区别是：

```dart
void main(List<String> args) {
    dynamic x = '123';
    print(x.substring(1));
}
```

在这种情况下，由于 x 被推断为 String 类型，所以可以调用 String 的方法，但是换成 Object 就不行了。



#### final 与 const

final和const都是用于定义常量的, 也就是定义之后值都不可以修改

```
final name = 'coderwhy';
name = 'kobe'; // 错误做法
const age = 18;age = 20; // 错误做法
```

final和const有什么区别呢?

- const在赋值时, 赋值的内容必须是在编译期间就确定下来的
- final在赋值时, 可以动态获取, 比如赋值一个函数

 ```dart
String getName() {
  return 'coderwhy';
}

main(List<String> args) {
  const name = getName(); // 错误的做法, 因为要执行函数才能获取到值
  final name = getName(); // 正确的做法
}
 ```

另外，const 还可以修饰构造函数。可以让同样字段值的构造函数返回同样的对象。



#### **命名可选参数** 和 **位置可选参数**

定义方式:

```
命名可选参数: {param1, param2, ...}
位置可选参数: [param1, param2, ...]
```

位置可选参数是有顺序的，命名可选参数没有。



#### ??=赋值操作

- 当变量为null时，使用后面的内容进行赋值。
- 当变量有值时，使用自己原来的值。



#### 初始化列表

```dart
class Point {
  final num x;
  final num y;
  final num distance;

  // 错误写法
  // Point(this.x, this.y) {
  //   distance = sqrt(x * x + y * y);
  // }

  // 正确的写法
  Point(this.x, this.y) : distance = sqrt(x * x + y * y);
}
```

初始化列表相对于命令可选参数来说，它更灵活，可以使用表达式，而可选参数的值需要是 const。



#### const 构造方法

```dart
main(List<String> args) {
  var p1 = const Person('why');
  var p2 = const Person('why');
  print(identical(p1, p2)); // true
}

class Person {
  final String name;

  const Person(this.name);
}
```

将构造方法前加`const进行修饰`，那么可以保证同一个参数，创建出来的对象是相同的。

- **注意一：拥有常量构造方法的类中，所有的成员变量必须是final修饰的**.
- **注意二:** 为了可以通过常量构造方法，创建出相同的对象，不再使用 **new**关键字，而是使用const关键字



#### 工厂构造方法

```dart
main(List<String> args) {
  var p1 = Person('why');
  var p2 = Person('why');
  print(identical(p1, p2)); // true
}

class Person {
  String name;

  static final Map<String, Person> _cache = <String, Person>{};

  factory Person(String name) {
    if (_cache.containsKey(name)) {
      return _cache[name];
    } else {
      final p = Person._internal(name);
      _cache[name] = p;
      return p;
    }
  }

  Person._internal(this.name);
}
```

工厂构造函数就是需要自己手动的返回一个对象。



#### Mixin混入

```java
main(List<String> args) {
  var superMan = SuperMain();
  superMan.run();
  superMan.fly();
}

mixin Runner {
  run() {
    print('在奔跑');
  }
}

mixin Flyer {
  fly() {
    print('在飞翔');
  }
}

// implements的方式要求必须对其中的方法进行重新实现
// class SuperMan implements Runner, Flyer {}

class SuperMain with Runner, Flyer {

}
```

这种写法有点像是对接口的使用简化。用接口来实现一些行为，然后可以直接被使用。
