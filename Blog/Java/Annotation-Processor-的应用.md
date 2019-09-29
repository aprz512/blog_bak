---
title: Annotation Processor 的应用
date: 2019-09-29
tags: Java-AnnotationProcessor
categories: AnnotationProcessor
---

作为一个程序员，每天的生活都是平淡而且枯燥。先开需求会，再来画界面。画着画着，突然感觉有点不对劲，以前画个界面不应该这么麻烦的，不知道是不是错觉，感觉使用了 ConstraintLayout 之后，画个界面越来越慢了。

说到这里不得不吐槽一下，ConstraintLayout 虽然减少 了层级，但是阅读单独上升了不少，特比是对于刚接触的人来说，一个那么复杂的布局，就一层，里面十几个控件，位置乱放，毫无顺序，你要是不点一下右边的预览视图来看看它们之间的关系，根本看不懂。真希望谷歌出一个容器，将子控件可以包一下，但是却不参加编译。

画完了界面，就要开始写逻辑，有网络请求的页面，比如详情页面，应该还是要保存一下请求详情接口的 id 才行。



## DetailActivity

于是就有了下面的代码：

```java
    // handleIntent 是在 Base 里面稍微封装了一下的，可以忽略，当作是在 onCreate 里面就好了。
	@Override
    protected void handleIntent(Intent intent, Bundle savedInstanceState) {
        super.handleIntent(intent, savedInstanceState);
        if (savedInstanceState != null) {
            mId = savedInstanceState.getLong(KEY_ID);
        } else if (intent != null) {
            mId = intent.getLongExtra(KEY_ID, -1L);
        }
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putLong(KEY_ID, mId);
    }
```

这都不知道是我多少次写这样的代码了，写完之后，终于受不了了，先将需求（不多）抛在一遍，想办法将这些重复的东西搞一搞。



## BaseActivity

首先，将 if - else if 封装一下再说，于是 Base 里面就多了一些下面的方法：

```java
    protected int getSavedInt(Intent intent, Bundle savedInstanceState, String key) {
        return getSavedInt(intent, savedInstanceState, key, 0);
    }

    protected int getSavedInt(Intent intent, Bundle savedInstanceState, String key, int defaultValue) {
        if (savedInstanceState != null) {
            return savedInstanceState.getInt(key, defaultValue);
        } else if (intent != null) {
            return intent.getIntExtra(key, defaultValue);
        }
        return defaultValue;
    }
```

这里只列出了针对 int 的，当然还有其他类型的，按需添加。



## SavedFieldHandler

但是再一想，这个玩意虽然只在 Activity 里面用到，但是抽成一个工具类会不会更好一点，于是就有了一个工具类：

> SavedFieldHandler
>
> 将 Base 里面添加的方法，改为 public static 的，放入工具类：

```java
    public static int getSavedInt(Intent intent, Bundle savedInstanceState, String key) {
        return getSavedInt(intent, savedInstanceState, key, 0);
    }

    public static int getSavedInt(Intent intent, Bundle savedInstanceState, String key, int defaultValue) {
        if (savedInstanceState != null) {
            return savedInstanceState.getInt(key, defaultValue);
        } else if (intent != null) {
            return intent.getIntExtra(key, defaultValue);
        }
        return defaultValue;
    }
```



## DetailActivity

于是，我们的 Activity 里面的代码，就变成了这样：

```java
    // handleIntent 是在 Base 里面稍微封装了一下的，可以忽略，当作是在 onCreate 里面就好了。
	@Override
    protected void handleIntent(Intent intent, Bundle savedInstanceState) {
        super.handleIntent(intent, savedInstanceState);
        SavedFieldHandler.getSavedLong(intent, savedInstanceState, KEY_ID, 0L);
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putLong(KEY_ID, mId);
    }
```

看起来舒服了一点，但是每次复写这两个方法也很烦。再仔细观察一下这两个方法，其实就是 key 不一样，然后由于储存的类型不一样，所以调用的 get put 方法名也不一样。

那么，如果将这两个方法往上浮到 BaseActivity 里面，需要做一些什么呢？

我们需要知道 **字段的值**，与**字段对应的 KEY**。那么怎么才能获取这两个东西呢？想一下，似乎直接获取有点难度，那么加点辅助信息呢，比如说注解，我们定义一个这样的注解：

> SaveField

```java
public @interface SaveField {
    String key() default "";

    String defaultValue() default "";
}
```

key() 方法表示 **字段对应的 KEY**。defaultValue() 方法表示 **字段的值**。然后我们使用反射可以获取被该注解修饰的字段的值，嗯，完美。

当我准备开始写代码的时候，突然感觉哪里不对劲，值有问题，如果我是一个 long 型的变量的话，那岂不是还需要调用 Long.parseLong 方法转一下，而且，我们写 long 型值的时候，都习惯添加一个 L 在后面的，比如：long x = 3L; ，为了兼容这样的情况，我特么不是要做的判断更多了。

想到长痛不如短痛，多做几个判断就多做几个吧。我灵光一现，既然一个注解搞不定，那多搞几个注解不就好了，于是下面的注解就出现了：

> FloatSavedField

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface FloatSavedField {
    String key() default "";

    float defaultValue() default 0f;
}
```

Retention 一定要是 RetentionPolicy.RUNTIME，因为要在反射的时候获取。

类似的还有 IntSavedField 等等，都是一样的代码，就不说了。

定义好了注解之后，我们就可以开始写 BaseActivity 的逻辑了。



## BaseActivity

> BaseActivity

```java
    @Override
    protected void handleIntent(Intent intent, Bundle savedInstanceState) {
        super.handleIntent(intent, savedInstanceState);

        // 收集注解标识的字段
        Field[] declaredFields = this.getClass().getDeclaredFields();
        for (Field declaredField : declaredFields) {
            declaredField.setAccessible(true);
            Annotation[] annotations = declaredField.getAnnotations();
            injectField(annotations, declaredField, intent, savedInstanceState);
        }
    }
```

这里就是遍历所有的字段。

```java
    private void injectField(Annotation[] annotations, Field field, Intent intent, Bundle savedInstanceState) {
        try {
            for (Annotation annotation : annotations) {
                Class<?> type = field.getType();
                if (annotation instanceof IntSavedField) {
                    if (type != int.class && type != Integer.class) {
                        throw new IllegalArgumentException("字段类型与注解类型不匹配");
                    }
                    IntSavedField savedField = (IntSavedField) annotation;
                    int defaultValue = savedField.defaultValue();
                    String key = savedField.key();
                    field.set(this, SavedFieldHandler.getSavedInt(intent, savedInstanceState, key, defaultValue));
                    mSavedFieldMap.put(field, savedField);
                } else if (...) {...}
            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
```

看看字段上是否有我们定义的注解，如果有的话，将注解里面定义的 KEY 取出来，然后使用 SavedFieldHandler 去获取传递过来的值，最后设置到该字段里面。

mSavedFieldMap 是将有指定注解修饰的字段保存一下，以免在 onSaveInstanceState 又要重新遍历一下所有字段。



在 onSaveInstanceState  里面：

```java
    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        try {
            Set<Map.Entry<Field, Annotation>> entrySet = mSavedFieldMap.entrySet();
            for (Map.Entry<Field, Annotation> entry : entrySet) {
                Field key = entry.getKey();
                Annotation value = entry.getValue();
                if (value instanceof IntSavedField) {
                    outState.putInt(((IntSavedField) value).key(), (Integer) key.get(this));
                } else if(...) {...}
            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
```

这里的逻辑，也很简单，从字段里面取出值，然后储存到  outState  里面就好了。

运行一下demo，可以正常运行。但是这个 BaseActivity 里面的代码就不太好看了，是因为这两个方法里面有太长的 if - else if 了，那怎么解决呢？回想一下《重构》这本书，抽一个接口就好了：

> SavedFieldHandler

```java
public interface SavedFieldHandler<T extends Annotation> {

    void injectField(T annotation, Object target, Field field, Intent intent, Bundle savedInstanceState);

    void saveField(Bundle outState, Object target, Field key, T annotation);

}
```

然后分别为各种类型写一个实现：

> DoubleHandler

```java
public class DoubleHandler implements SavedFieldHandler<DoubleSavedField> {
    @Override
    public void injectField(DoubleSavedField annotation, Object target, Field field, Intent intent, Bundle savedInstanceState) {
        double defaultValue = annotation.defaultValue();
        String key = annotation.key();
        try {
            field.set(target, SavedFieldHandler.getSavedDouble(intent, savedInstanceState, key, defaultValue));
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void saveField(Bundle outState, Object target, Field key, DoubleSavedField annotation) {
        try {
            outState.putDouble(annotation.key(), (Double) key.get(target));
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}

```



再次重构 BaseActivity 的代码：

```kotlin
    private fun handleIntent(intent: Intent, savedInstanceState: Bundle?) {
        val s = System.nanoTime()

        val declaredFields = this.javaClass.declaredFields
        for (declaredField in declaredFields) {
            declaredField.isAccessible = true
            val annotations = declaredField.annotations
            for (annotation in annotations) {
                val handler = HandlerFactory.get(annotation)
                if (handler != null) {
                    handler.injectField(annotation, this, declaredField, intent, savedInstanceState)
                    mSavedFieldMap[declaredField] = annotation
                }
            }
        }

        Log.e(TAG, "获取耗时" + ((System.nanoTime() - s) / 1000000.0).toString() + "毫秒")
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)

        val s = System.nanoTime()

        val entrySet = mSavedFieldMap.entries
        for (entry in entrySet) {
            val key = entry.key
            val value = entry.value
            HandlerFactory.get(value)?.saveField(outState, this, key, value)
        }

        Log.e(TAG, "保存耗时" + ((System.nanoTime() - s) / 1000000.0).toString() + "毫秒")
    }
```

嗯，这次看起来就舒服多了。运行一下，看看耗时情况，结果如下：

```shell
获取耗时第一次为 2-3 毫秒，再次运行在 0.5 毫秒左右
保存耗时 为 0.2 ~ 0.3 毫秒左右
```

结果还是可以接收的。



在写完之后，又想了想，如果我不使用反射，那么耗时情况是怎么样的呢？既然不能使用反射，那么还要能给变量赋值与获取值，这咋办呢？给变量赋值...，ButterKnife 不就是给变量赋值，ButterKnife 的工作原理这里还是简单描述一下哈：

```
我们有一个目标类：SbActivity，它在 com.sb 包下。

我们使用注解处理器生成一个类 SbActivity_ViewBinding，也让它生成到 com.sb 包下面

给 SbActivity_ViewBinding 类搞一个构造函数，构造函数有两个参数：
第一个参数是 SbActivity，第二个参数是 SbActivity 的根 view

这样，我们可以在 SbActivity_ViewBinding类中拿到 SbActivity 的所有非私有变量，就可以给这个变量设置值，获取它的值
```

了解了它的工作原理，这完全和我们的需求一摸一样啊，所以，我们直接按照 ButterKnife 来设计我们的结构。



## SaveHelper

首先，我们需要一个工具类，它的作用与 ButterKnife 类一样，提供一个 bind 方法，返回一个 UnBinder 对象，这里我们另起一个方法名：

> SaveHelper

```kotlin
@UiThread
fun get(target: Activity, intent: Intent?, savedInstanceState: Bundle?): SaveUnbinder {
    return createBinding(target, intent, savedInstanceState)
}
```

这个方法的逻辑，都不用我们自己想，直接从 ButterKnife 里面 copy 出来用就好了。

```kotlin
fun createBinding(target: Activity, intent: Intent?, savedInstanceState: Bundle?): SaveUnbinder {
    val targetClass = target.javaClass
    Log.d(TAG, "Looking up binding for " + targetClass.name)
    val constructor = findBindingConstructorForClass(targetClass) ?: return EMPTY_UNBINDER

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
        return constructor.newInstance(target, intent, savedInstanceState)
    } catch (e: IllegalAccessException) {
        throw RuntimeException("Unable to invoke $constructor", e)
    } catch (e: InstantiationException) {
        throw RuntimeException("Unable to invoke $constructor", e)
    } catch (e: InvocationTargetException) {
        val cause = e.cause
        if (cause is RuntimeException) {
            throw cause
        }
        if (cause is Error) {
            throw cause
        }
        throw RuntimeException("Unable to create binding instance.", cause)
    }

}
```

这个方法，就是根据我们的 Activity，创建 Activity_ViewBinding 的一个对象，因为我们使用注解处理器创建的类的构造函数是有两个参数的，这里使用反射创建 Activity_ViewBinding  类的实例。

```kotlin
@Nullable
@CheckResult
@UiThread
fun findBindingConstructorForClass(cls: Class<*>): Constructor<out SaveUnbinder>? {

    var bindingCtor: Constructor<out SaveUnbinder>? = BINDINGS[cls]
    if (bindingCtor != null) {
        Log.d(TAG, "HIT: Cached in binding map.")
        return bindingCtor
    }

    val clsName = cls.name
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
        Log.d(TAG, "MISS: Reached framework class. Abandoning search.")
        return null
    }

    try {
        val bindingClass = cls.classLoader?.loadClass(clsName + "_FieldSaving")
        bindingCtor =
            bindingClass?.getConstructor(
                cls,
                Intent::class.java,
                Bundle::class.java
            ) as Constructor<out SaveUnbinder>
        Log.d(TAG, "HIT: Loaded binding class and constructor.")
    } catch (e: ClassNotFoundException) {
        Log.d(TAG, "Not found. Trying superclass " + cls.superclass?.name)
        cls.superclass?.apply {
            bindingCtor = findBindingConstructorForClass(this)
        }
    } catch (e: NoSuchMethodException) {
        throw RuntimeException("Unable to find binding constructor for $clsName", e)
    }

    bindingCtor?.apply {
        BINDINGS[cls] = this
    }

    return bindingCtor
}
```

这里是寻找 Activity_ViewBinding 这个类，然后加载这个类，获取它的 Constructor 并返回。

bindingCtor 缓存了对应的 Constructor 。

这里为了加以区分，我们注解处理器生成的类，后缀叫 _FieldSaving。



## SaveUnbinder

SaveUnbinder 对象的生成解决了，那么这个接口应该有哪些方法呢？

```kotlin
interface SaveUnbinder {

    @UiThread
    fun save(outState: Bundle)

    @UiThread
    fun unbind()

}
```

这个接口的定义还是很简单的。



## BaseActivity

那么我们的 BaseActivity 就可以这样写了：

```kotlin
abstract class BaseActivity2 : AppCompatActivity() {

    companion object {
        const val TAG = "BaseActivity2"
    }

    private lateinit var saveUnbinder: SaveUnbinder

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val s = System.nanoTime()
        saveUnbinder = get(this, intent, savedInstanceState)
        Log.e(TAG, "获取花费了 ${(System.nanoTime() - s) / 1000000.0} 毫秒")
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        val s = System.nanoTime()
        saveUnbinder.save(outState)
        Log.e(TAG, "保存花费了 ${(System.nanoTime() - s) / 1000000.0} 毫秒")
    }

    override fun onDestroy() {
        super.onDestroy()
        saveUnbinder.unbind()
    }


}
```

这个看起来就更舒服了。



接下来，就只需要搞定注解生成对应的类就好了，注解处理器，就不说了，主要是 JavaPoet 的使用，它生成的类应该如下：

```java
public final class MainActivity_FieldSaving implements SaveUnbinder {
  MainActivity target;

  public MainActivity_FieldSaving(MainActivity activity, Intent intent, Bundle bundle) {
    this.target = activity;
    this.target.testL = IntentHandlerKt.getSavedLong(intent, bundle, "testL", 71L);
    this.target.testS = IntentHandlerKt.getSavedString(intent, bundle, "testS", "74");
    this.target.testD = IntentHandlerKt.getSavedDouble(intent, bundle, "testD", 7.3D);
    this.target.testI = IntentHandlerKt.getSavedInt(intent, bundle, "testI", 70);
    this.target.testF = IntentHandlerKt.getSavedFloat(intent, bundle, "testF", 7.2F);
  }

  @Override
  public void save(Bundle outState) {
    outState.putLong("testL", this.target.testL);
    outState.putString("testS", this.target.testS);
    outState.putDouble("testD", this.target.testD);
    outState.putInt("testI", this.target.testI);
    outState.putFloat("testF", this.target.testF);
  }

  @Override
  public void unbind() {
    this.target = null;
  }
}

```

嗯，这样就搞定了。



最后将这些类分开，我们新建 3 个不同作用的 Module：

```
save-api 用于存放给外部使用的类
save-annotation 用于存放需要处理的注解
save-processor 用于处理注解
```



运行 demo，却报了一个错，报的是字段不能是私有的，这个是我在注解处理器里面输出的错误，可是这就奇怪了啊，我的字段不是私有的啊，经旁边同事的提醒，查看一下它的 byteCode，果然是私有的，原来是 Kotlin 搞的鬼，它为字段生成了公有的 get set 方法，所以变量是私有的了。那这可咋办呢？我去翻了一下 ButterKnife 的注解处理器，它也没有处理这种情况。看来 Java 的注解处理器来兼容 Kotlin，是用前朝的剑来斩本朝的官啊。不过听说又有一个 KotlinPoet，但是还是得分开处理，很麻烦。

本来想着，参考一下 Gson 的代码，看它是怎么处理有 get set 方法的。但是想了想还是算了，感觉这样兼容很脆弱，而且我对 Kotlin 还不太熟，不知道它没有什么注解处理器来兼容 Java。



最后，使用 Java 来写 Activity，测试性能结果：

```
获取耗时第一次为 1.7-2.3 毫秒，再次运行在 0.2 毫秒左右
保存耗时 为 0.06 毫秒左右
```

最后额外说一下，由于使用了注解处理器，注解的 Retention 可以改为了 SOURCE。

一个比较令人满意的依赖库就做好了。

项目代码并没有经过严格测试，只是写的有意思，所以分享一下，项目源代码地址如下：

https://github.com/aprz512/SaveHelper

