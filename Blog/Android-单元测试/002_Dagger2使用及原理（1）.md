---
title: 002_Dagger2使用及原理（1）
index_img: /cover/17.jpg
banner_img: /cover/top.jpg
date: 2020-3-17
tags: Android-单元测试
---

我们将上篇文章的例子用 dagger2 实现一下。

dagger2 的引入就不说了，[github](https://github.com/google/dagger)文档很详细了。

首先我们实现 Dragons 类：

```kotlin
class Dragons {
    fun callForWar() {
        Log.e("EX01", "Dragons callForWar...")
    }
}
```



再看 Targaryens  类：

```kotlin
class Targaryens constructor(dragons: Dragons) {

    // 因为 dragons 是私有的，所以只能修饰 set 方法
    var dragons = dragons
        @Inject set

    init {
        DaggerEx01Component.create().inject(this)
    }

    fun war() {
        Log.e("EX01", "Targaryens call war...")
        dragons.callForWar()
    }

}
```

我们使用 @Inject 注解修饰了 dragons 变量的 set 方法，在Java里面是修饰的字段，kotlin的写法有点不一样。

`@Inject` 注解是用来标记依赖的。它可以修饰字段与方法。这里表示这个字段需要注入，有点像butterknife。但是 @Inject 在不同的地方会有不同的意义，我们后面再说。

init 方法里的代码我们后面会说到。



现在，我们有了接收依赖（需要注入）的地方，那么哪里是生产依赖的地方呢？答案就是 @Module 注解了。

```kotlin
@Module
class Ex01Module {

    @Provides
    fun provideDragons(): Dragons {
        return Dragons()
    }

    @Provides
    fun provideTargaryens(dragons: Dragons): Targaryens {
        return Targaryens(dragons)
    }

}
```

首先，我们使用 @Module 修饰了这个类。类中有方法，使用 @Provides 修饰。

我们先说这个 `provideDragons` 方法的作用，显然就是提供一个 Dragons 的实例。看到这里，我们就立刻明白这里是生产依赖的地方。

`provideTargaryens` 提供一个 Targaryens 实例，这个方法很重要。因为上一篇文章，我们说过，如果每次创建一个 Targaryens 实例的时候，还要自己去创建一个 Dragons 实例，那写起代码来就很痛苦了。那么怎么办呢？办法就是把这种麻烦事交给 Dagger2 。

这里我们提供一个创建 Targaryens 实例的方法，这个方法需要一个 Dragons 作为参数，神奇的是，Dagger2 会自动使用 ` provideDragons` 创建 Dragons 实例。所以这个 `provideTargaryens` 方法不是给我们使用的，而是给 Dagger2 用的。那么我们怎么从 Dagger2 获取 Targaryens 实例呢？后面再说。



可以想象一下，一个接收，一个生产，这两处地方，需要有一根线将它们连接起来，才能正常工作。

那么，怎么牵这根线呢？？？答案是使用 @Component 注解。

```kotlin
@Component(modules = [Ex01Module::class])
interface Ex01Component {

    fun inject(targaryens: Targaryens)

    fun newTargaryens(): Targaryens

}
```

需要注意的是，Ex01Component 是**一个接口**，不是一个类！！！

@Component 注解可以设置参数值，是一个数组，里面是 class 值。我们传递了 Ex01Module 的 class 进去，这样就将 Module 连起来了。

接口里面有一个方法，这个方法相当于一个定义，因为我们要给 Targaryens 类的字段赋值，所以必须要声明一个方法。实际上，写完这个接口之后，对应的生成类就会生成了：

> com.aprz.daggerexamples.ex01.DaggerEx01Component

```java
public final class DaggerEx01Component implements Ex01Component {
  private final Ex01Module ex01Module;

  private DaggerEx01Component(Ex01Module ex01ModuleParam) {
    this.ex01Module = ex01ModuleParam;
  }

  public static Builder builder() {
    return new Builder();
  }

  public static Ex01Component create() {
    return new Builder().build();
  }

  @Override
  public void inject(Targaryens targaryens) {
    injectTargaryens(targaryens);}

  @Override
  public Targaryens newTargaryens() {
    return Ex01Module_ProvideTargaryensFactory.provideTargaryens(ex01Module, Ex01Module_ProvideDragonsFactory.provideDragons(ex01Module));}

  private Targaryens injectTargaryens(Targaryens instance) {
    Targaryens_MembersInjector.injectSetDragons(instance, Ex01Module_ProvideDragonsFactory.provideDragons(ex01Module));
    return instance;
  }

  public static final class Builder {
    private Ex01Module ex01Module;

    private Builder() {
    }

    public Builder ex01Module(Ex01Module ex01Module) {
      this.ex01Module = Preconditions.checkNotNull(ex01Module);
      return this;
    }

    public Ex01Component build() {
      if (ex01Module == null) {
        this.ex01Module = new Ex01Module();
      }
      return new DaggerEx01Component(ex01Module);
    }
  }
}

```

这个类很简单，里面有一个 Builder 类。但是一般我们不需要使用，因为这个类又提供了一个 create 方法，将Builder 又简化了。我们可以直接使用这个方法。

DaggerEx01Component类实现了 Ex01Component，我们看它的 inject 方法是做了什么。

> com.aprz.daggerexamples.ex01.DaggerEx01Component#inject

```java
  @Override
  public void inject(Targaryens targaryens) {
    injectTargaryens(targaryens);}

  private Targaryens injectTargaryens(Targaryens instance) {
    Targaryens_MembersInjector.injectSetDragons(instance, Ex01Module_ProvideDragonsFactory.provideDragons(ex01Module));
    return instance;
  }
```

这两个方法都很短，一看就明白，这里调用了 MembersInjector 类，一看就是给字段注入值。

> com.aprz.daggerexamples.ex01.Targaryens_MembersInjector#injectSetDragons

```java
  public static void injectSetDragons(Targaryens instance, Dragons p0) {
    instance.setDragons(p0);
  }
```

方法很简单，就是直接调用 set 方法就好了。

这里我没有看 Java 生成类是不是一样的，因为 Java 要求待注入的字段不能是私有的，所以是通过在同包下直接调用类的字段赋值，和 butterKnife 一样。

还有一个方法 :

> com.aprz.daggerexamples.ex01.DaggerEx01Component#newTargaryens

```kotlin
  @Override
  public Targaryens newTargaryens() {
    return Ex01Module_ProvideTargaryensFactory.provideTargaryens(ex01Module, Ex01Module_ProvideDragonsFactory.provideDragons(ex01Module));}
```

这个就更简单了，其实就是调用了 Module 的 provide 方法创建了 Targaryens。可以看到，Dragons 也是由 Module 创建的。



到这里，基本就分析完了 dagger 最简单的运作原理。还是还有两个个地方。

第一个地方就是 `com.aprz.daggerexamples.ex01.DaggerEx01Component#inject` 这个方法应该在哪里调用呢？因为只有调用了这个方法，才算真正的赋值了。

因为 inject 方法需要一个 Targaryens 类的实例，所以，你可以在访问到该实例的任何地方调用。但是我们最好写在类里面，这样就不用没有实例都调用一遍，具体看上面的 Targaryens 类的 init 方法里面。

第二个地方是，如何获取 Targaryens 对象呢？显然是需要从 Component 里面获取。

```kotlin
val targaryens = DaggerEx01Component.create().newTargaryens()
targaryens.war()
```





Dagger2 其实还有另外一种使用方式，不使用 Module 的方式。

> com.aprz.daggerexamples.ex02.Dragons2

```kotlin
class Dragons2 @Inject constructor() {
    fun callForWar() {
        Log.e("EX02", "Dragons callForWar...")
    }
}
```

注意这里与上面的不同之处，我们使用 @Inject 修饰了它的构造函数。

> com.aprz.daggerexamples.ex02.Ex02Component

```kotlin
@Component(modules = [])
interface Ex02Component {

    fun inject(targaryens2: Targaryens2)

}
```

这里，我们没有使用Module，所以不用传值，

> com.aprz.daggerexamples.ex02.Targaryens2

```kotlin
class Targaryens2 {

    // 直接标注属性@Inject lateinit var car: Car，编译时会报错
    lateinit var dragons: Dragons2
        @Inject set

    init {
        DaggerEx02Component.create().inject(this)
    }

    fun war() {
        Log.e("EX02","Targaryens call war...")
        dragons.callForWar()
    }

}
```

这里与之前代码一样。

可以看到，实际上，我们使用 @Inject 修饰构造函数的方法，代替了 @Module，少写了不少东西。

我们再看看生成的类，其中的不同之处在于：

> com.aprz.daggerexamples.ex02.DaggerEx02Component#injectTargaryens2

```kotlin
  private Targaryens2 injectTargaryens2(Targaryens2 instance) {
    Targaryens2_MembersInjector.injectSetDragons(instance, new Dragons2());
    return instance;
  }
```

这里是直接调用了 Dragons2 的构造函数，而 Module 写法是调用了 Module 类的 provide 方法。



所以有两种写法，这两种写法是可以配合使用的，看需要选择使用。

说明一下，这里如果我们使用的是 @Inject 修饰构造函数的方式，那么我们想 mock Dragons2 对象还是比较蛋疼的。但是我们可以取巧，我们直接 new 一个 Targaryens2 对象，mock 一个 Dragons2 对象出来，然后调用set方法设置进去就行了，但是这也只能对 kotlin 起作用，因为 java 不会自动生成 set 方法。所以最好还是使用 Module。



最后再总结一下各个注解的作用：

@Inject 是用来标记依赖的（修饰字段是表示该字段要被注入依赖实例，修饰构造函数是）。

@Module 是用来给 Dagger2 自己使用的，创建依赖图的。

@Component 是用来给开发者使用的，它的生成类实现了我们定义的接口，用于我们的开发。

