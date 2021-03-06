---
title: 001-做单元测试为何要使用Dagger2？
index_img: /cover/17.jpg
banner_img: /cover/top.jpg
date: 2020-3-17
tags: Android-单元测试
---

首先，我们说一下依赖注入（Dependency Injection，以下简称DI）。

为了解释以下概念，我将使用“权力的游戏”作为类比。如果你没有听说过，可以将（Targaryens和Dragons）替换为A和B。

举个例子，我们有一个叫做Targaryens的类。这个类里面用到了另一个叫做 Dragons 的类。我们称 Targaryens 依赖 Dragons 。当我们要想使用 Targaryens 类，就必须也要一个 Dragons 类，Targaryens 不能单独存在。看下面的代码：

```java
class Targaryens {

    public Targaryens() {
        //Each  time we use Targaryens, we need to create Dragons instance
        Dragons dragons = new Dragons();
        dragons.callForWar();
    }
    
}
```

这样写有什么不好的地方呢？其实会有很多问题，比如，可重用行，可维护性，但是这些问题都很模糊，没有刻骨经历过的人都是体会不到的。看了也会转头忘掉。

由于我是基于学习单元测试的原因学习的dagger，所以我就只说一点原因--可测试性。

继续上面的例子，我们想测试一下 Targaryens 的构造函数中，callForWar 方法是否被调用了一次 ：

那么我们就需要 mock 出一个 Dragons 对象来测试（这里不明白的需要先去补补 [mock](https://en.wikipedia.org/wiki/Mock_object) 相关的东西再往下看）。现在问题就来了，就算我们 mock 出了一个 Dragons 对象出来后，怎么才能让这个 mock 对象替换构造函数里面的那个对象呢？？？这里是没法替换的！！！

由此可见，这里这样写代码的话，我们的测试很难进行下去。

那么应该如果做呢？我们可以使用**构造函数注入**。看如下代码：

```java
class Targaryens {
    
    Dragons dragons;

    public Targaryens(Dragons dragons) {
        this.dragons = dragons;
        dragons.callForWar();
    }
    
}
```

这样改写之后，我们 mock 出来的对象就可以传递到 Targaryens 中，接着就可以测试 callForWar 方法是否被正确的调用了，而且可以查看调用次数。

> 假如你的代码里面，一个类用到了另外一个类，那么前者叫Client，后者叫Dependency。结合上面的例子，`Targaryens`用到了`Dragons`，那么`Targaryens`叫Client，`Dragons`叫Dependency。
>
> 当然，这是个相对的概念，一个类可以是某个类的Dependency，却是另外一个类的Client。
>
> DI的基本思想就是，对于Dependency的创建过程，并不在Client里面进行，而是由外部创建好，然后通过某种方式set到Client里面。这种模式，就叫做依赖注入。
>
> 是的，依赖注入就是这么简单的一个概念，这边需要澄清的一点是，这个概念本身跟dagger2啊，RoboGuice这些框架并没有什么关系。现在很多介绍DI的文章往往跟dagger2是在一起的，因为dagger2的使用相对来说不是很直观，所以导致很多人认为DI是多么复杂的东西，甚至认为只能用dagger等框架来实现依赖注入，其实不是这样的。实现依赖注入很简单，dagger这些框架只是让这种实现变得**更加**简单，简洁，优雅而已。

好了，了解了依赖注入之后，下面我们接着说，为何要使用 dagger 2！

依赖注入可以让我们更轻松的写单元测试（当然还有其它好处），那么有的人就会说，这很简单的，我只要遵循这个规则，永远把依赖在外部实例化，不就好了吗？为啥我要用 dagger2 呢？？

请看下面的一个例子：

假设有一个登录界面，`LoginActivity`，他有一个`LoginPresenter`，`LoginPresenter`用到了`UserManager`和`PasswordValidator`，为了让问题变得更明显一点，我们假设`UserManager`用到`SharedPreference`（用来存储一些用户的基本设置等）和`UserApiService`，而`UserApiService`又需要由`Retrofit`创建，而`Retrofit`又用到`OkHttpClient`（比如说你要自己控制timeout、cache等东西）。
应用DI模式，UserManager的设计如下：

```java
public class UserManager {
    private final SharedPreferences mPref;
    private final UserApiService mRestAdapter;

    public UserManager(SharedPreferences preferences, UserApiService userApiService) {
        this.mPref = preferences;
        this.mRestAdapter = userApiService;
    }

    /**Other code*/
}
```

LoginPresenter的设计如下：

```java
public class LoginPresenter {
    private final UserManager mUserManager;
    private final PasswordValidator mPasswordValidator;

    public LoginPresenter(UserManager userManager, PasswordValidator passwordValidator) {
        this.mUserManager = userManager;
        this.mPasswordValidator = passwordValidator;
    }

    /**Other code*/
}
```

在这种情况下，最终的client LoginActivity里面要new一个presenter，需要做的事情如下：

```java
public class LoginActivity extends AppCompatActivity {
    private LoginPresenter mLoginPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        OkHttpClient okhttpClient = new OkHttpClient.Builder()
                .connectTimeout(30, TimeUnit.SECONDS)
                .build();
        Retrofit retrofit = new Retrofit.Builder()
                .client(okhttpClient)
                .baseUrl("https://api.github.com")
                .build();
        UserApiService userApiService = retrofit.create(UserApiService.class);
        SharedPreferences preferences = PreferenceManager.getDefaultSharedPreferences(this);
        UserManager userManager = new UserManager(preferences, userApiService);

        PasswordValidator passwordValidator = new PasswordValidator();
        mLoginPresenter = new LoginPresenter(userManager, passwordValidator);
    }
}
```

**可以看到。我们想使用一个简单的 LoginPresenter ，却要实例化一大堆我们根本不关心的东西。**我们遵循了依赖注入的规则，却陷入了另一个漩涡。

dagger2 就是帮助我们逃离这个漩涡的工具。大致过程是：我们提供依赖的实例化方法，dagger2会管理这些方法，当某个对象需要一个依赖时，对应的实例化方法就会被调用。这样我们就不用手动的去实例化一大堆的对象了。
