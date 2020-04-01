---
title: 003_单元测试中如何使用Android相关的类?
date: 2020-3-18 10：12：29
tags: Android-单元测试
---

习惯写单元测试的人都会这样说：android unit testing不好做！！！

我们知道安卓的app需要运行在delvik上面，我们开发Android app是在JVM上面，在开发之前我们需要下载各个API-level的SDK的，下载的每个SDK都有一个android.jar的包，这些可以在你的*android_sdk_home*/platforms/下面看到。

当我们开发一个项目的时候，我们需要指定一个API-level，其实就是将对应的android.jar 加到这个项目的build path里面去。这样我们的项目就可以编译打包了。然而现在的问题是，**我们的代码必须运行在emulator或者是device上面**，说白了，就是我们的IDE和SDK只提供了开发和编译一个项目的环境，并没有提供运行这个项目的环境，原因是因为android.jar里面的class实现是不完整的，它们只是一些stub，如果你打开android.jar下面的代码去看看，你会发现所有的方法都只有一行实现： 
`throw RuntimeException("stub!!");` 

而运行unit test，说白了还是个运行的过程，**所以如果你的unit test代码里面有android相关的代码的话，那运行的时候将会抛出RuntimeException(“stub!!”)**。

为了解决这个问题，现在业界提出了很多不同的程序架构，比如MVP、MVVM等等，这些架构的优势之一，就是将其中一层抽出来，变成pure Java实现，这样做unit testing就不会遇到上面这个问题了，因为其中没有android相关的代码。 

但是 MVP、MVVM这些架构模式虽然解决了部分问题，可以测试项目中不含android相关的类的代码，然而一个项目中还是有很大部分是android相关的代码的，所以上面那种解决方案，其实是放弃了其中一大块代码的unit test。 

当然，话说回来，android还是提供了他自己的testing framework，叫instrumentation，但是这套框架还是绕不开刚刚提到的问题，他们必须跑在emulator或者是device上面。这是个很慢的过程，因为要打包、dexing、上传到机器、运行起来界面。。。这个相信大家都有体会，尤其是项目大了以后，运行一次甚至需要一两分钟，项目小的话至少也要十几秒或几十秒。

那么怎么样即可以给android相关的代码做测试，又可以很快的运行这些测试呢？

### Robolectric 来也

解决的办法就是使用一个开源的framework，叫[robolectric](http://robolectric.org/)，他们的做法是通过实现一套JVM能运行的Android代码。

这样的话，调用 android 类就和调用普通类是一样的了。

### 看一个例子

> com.aprz.daggerexamples.ex03.TipsHelper

```kotlin
class TipsHelper constructor(private val status: Int) {

    fun buildTips(context: Context): String {
        return doSomething() + LicenseStatusResouceResolver.getString(context, status)
    }

    private fun doSomething(): String = "doSomething"

}
```

>  com.aprz.daggerexamples.ex03.LicenseStatusResourceResolver

```kotlin
class LicenseStatusResourceResolver {

    companion object {
        fun getString(context: Context, @LicenseStatus licenseStatus: Int): String {
            when (licenseStatus) {
                WILL_EXPIRE -> {
                    return context.getString(R.string.license_will_expire)
                }
                EXPIRED -> {
                    return context.getString(R.string.license_expired)
                }
            }
            return context.getString(R.string.unknown_error)
        }
    }


}

const val WILL_EXPIRE = 1
const val EXPIRED = 2

@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.SOURCE)
@IntDef(WILL_EXPIRE, EXPIRED)
annotation class LicenseStatus
```

假如，让你测试一下`com.aprz.daggerexamples.ex03.TipsHelper#buildTips` 这个方法是否正常，你准备如何写做呢？

如果我们按照单元测试的写法，做测试的话，会发现一个问题，就是我们**无法获取一个 Context 对象**。就算我们强行创建一个对象的话，传递进去，也会报出各种错误。

但是使用了Robolectric，就可以很容易的获取 Context，甚至 Activity，还可以控制 Activity 的生命周期。

看具体代码：

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [28])
class TipsHelperTest {

    private lateinit var context: Context

    @Before
    fun setupContext() {
        val activity = Robolectric.buildActivity(MainActivity::class.java)
        context = activity.get()
    }

    @Test
    fun testBuildTips() {
        val willExpireTips = TipsHelper(WILL_EXPIRE)
        assertEquals("doSomething您的证件即将过期！！！", willExpireTips.buildTips(context))

        val expiredTips = TipsHelper(EXPIRED)
        assertEquals("doSomething您的证件已过期！！！", expiredTips.buildTips(context))

        val unknownTips = TipsHelper(-1)
        assertEquals("doSomething未知错误！！！", unknownTips.buildTips(context))
    }

}
```

可以看到，我们在类上加了些注解。

@RunWith 表示该测试是运行在 Robolectric 的 testRunner 上的，写过 androidTest 的都知道`@RunWith(AndroidJUnit4::class)`，表示运行在 AndroidJUnit4 上。

@Config 是对 RobolectricTestRunner 的运行环境的一些配置，可以配置 sdk 等等，具体可以戳类里面进去看看。

想要获取一个 Context，有几种方法：

```kotlin
val activity = Robolectric.buildActivity(MainActivity::class.java)
```

这可以创建出一个 MainActivity 的对象，而且它是有生命周期的。比如，我们可以调用 `activity.pause()` 方法让 activity 进入暂停状态。

当然，不要忘记Gradle配置：

```groovy
android {
    testOptions {
        unitTests {
            includeAndroidResources = true
        }
    }
}

dependencies {
    testImplementation 'org.robolectric:robolectric:4.3.1'
}
```

想了解 Robolectric 更多的API信息，只能去[官网](http://robolectric.org/writing-a-test/)了。就先简单的介绍到这。



最后说一下，Robolectric 到底是个啥？

**Robolectric就是一个能够让我们在JVM上跑测试时够调用安卓的类的框架**。

在没有Robolectric的pure JUnit世界，我们是很难对一整个流程进行测试的，因为上层的界面是安卓的类，底层的数据库和SharedPreference等等是安卓的类。

然而有了robolectric以后，我们就可以这么做了：启动activity，向网络或数据库请求数据，更新界面。。。基本上覆盖的所有的逻辑。

我更倾向于使用 robolectric 做单元测试。





