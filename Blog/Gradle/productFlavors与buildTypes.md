---
title: productFlavors与buildTypes
date: 2019-08-18 11：11：11
tags: Gradle
---

> modify at 2019-8-26
>
> 节操社要祸祸“食戟之灵”了，我TM...口吐芬芳




> 之所以会想到要写这个，是发现了很多以前没注意过的东西，记录一下。



起因是由于一个紧急版本，然后我需要配置一下一个新增的H5地址，发现在配置的过程中非常的麻烦。

1. config.gradle 中添加地址
2. app / build.gradle 添加 manifestPlaceholders ，为不同的 productFlavors 配置不同的地址
3. AndroidManifest.xml 中添加 meta-data
4. Java 中添加Key

这是在是太麻烦了，很容易哪一步就忘记了。于是问了一下，为啥要写在 AndroidManifest.xml 里面？然后给出的答案是为了安全！

嗯，无稽之谈，于是我就收集了一些证据：

- 将 debug，release，加固后的release包都拖到 AS 中，打开 AndroidManifest.xml，显然里面的每一根毛都看的清清楚楚
- 使用压缩文件解压apk包，用notepad++等工具打开，显示是二进制

显然用文本编辑器打开，看不出什么，就是所谓的“安全”了。当是这种东西就相当于序列化一个 Java 对象，你看不出什么来，不代表别人看不出来，写一个工具就好了，里面的二进制数据都是有规则的，不然怎么读取里面的东西。

好了，有了这个作为前提，由于我们的包是加固的，所以放在 Java 里面比 AndroidManifest.xml 要安全很多，至少你要懂脱壳（这里不讨论别的了）。于是我就想将地址放在 BuildConfig 里面，减少配置的步骤，我使用一个单独的 gradle 文件配置一下，然后每个 variant 取相应的地址就好了。（虽然后来才发现，需要在base库里面生成才能有用，但是再学习的过程中还是很有收获的。）

比如，我在 urls.gradle 中这样写：

```groovy
ext {

    urls = [
            HOME: [
                    DEBUG_HOST    : "https://www.debug.com",
                    RELEASE_HOST: "https://www.release.com",
                    PATHS       : [
                            a: '/1',
                            b: '/2',
                            c: '/3',
                            d: '/4'
                    ]
            ]

    ]

}
```

然后在BuildConfig里面生成这样的东西：

```java
  public static final String HOME_a = "https://www.debug.com/1";
  public static final String HOME_b = "https://www.debug.com/2";
  public static final String HOME_c = "https://www.debug.com/3";
  public static final String HOME_d = "https://www.debug.com/4";
```

当我将 Build Variant 切换为 release 的时候，BuildConfig 会变为：

```java
  public static final String HOME_a = "https://www.release.com/1";
  public static final String HOME_b = "https://www.release.com/2";
  public static final String HOME_c = "https://www.release.com/3";
  public static final String HOME_d = "https://www.release.com/4";
```



那么，如何才能做到上面的效果呢？我们先从 build type 说起。

## BuildType

首先，我们需要知道的是，当我们创建一个 module 的时候，AS 会自动的为我们创建 debug 与 release 这两个构建类型，不管我们写没写，就算你把它给删了，仍然会有。从源码里面没有找到对应的地方，可能是在插件的代码里面，不管这个，我们继续。

我们点击 release 进入源码，发现它是一个 `com.android.build.gradle.internal.dsl.BuildType` 对象。

里面会有很多我们熟悉的方法，比如：

```groovy
    @NonNull
    public BuildType proguardFiles(@NonNull Object... files) {
        checkPostProcessingConfiguration(PostProcessingConfiguration.OLD_DSL, "proguardFiles");
        for (Object file : files) {
            proguardFile(file);
        }
        return this;
    
```

这个货，就对应着我们配置的混淆文件。如下：

```groovy
proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
```

由于上面的方法是一个可变参数作为参数，所以我们可以在 `proguardFiles`后面，添加多个 proguard 文件。

除了 proguardFiles，还有其他的很多属性可以配置：

```groovy
minifyEnabled false
applicationIdSuffix ".debug"
debuggable true
buildConfigField "boolean", "DEBUG", "true"
// release版的签名
signingConfig signingConfigs.app

// 还有一个 initWith 属性
initWith debug
```

说一下这个 initWith 属性，比如上面我们指定了 debug，那么就是说将 debug 里面的所有属性都继承过来，算是一种复用。

再说 product flavors。

## Product Flavors

build type，我们比较好理解，debug 用于开发，release 用于发布，一般这两个就够用了。那么 product flavors 又是什么鬼呢？可以这样想，幸平创真在创造一道新菜的时候，在创造的过程中，会慢慢的调配各种配料的比例，这中间烧的菜就是 debug，当创造完成之后可以端给客人吃了，烧的菜就是 release。但是经过观察后，幸平创真发现，不同的人口味不同，然后他就又为不同的人调配了不同口味的蘸料，这些不同口味的蘸料就是 product flavors。

我们看看配置代码：

```groovy
    productFlavors {
        demo {
            // productFlavors 需要指定 dimension
            // 如果你的 flavorDimensions 只有一个值，
            // 那么可以不用指定，会自动赋值，但是一定要声明 flavorDimensions
            dimension "version"
            applicationIdSuffix ".demo"
            versionNameSuffix "-demo"
        }
        full {
            dimension "version"
            applicationIdSuffix ".full"
            versionNameSuffix "-full"
        }
    }
```

配置好了 product flavors 之后，你就可以打多个不同的包了，包的个数为 【build type 的个数】x 【product flavors  的个数】，相当于交叉的集合：

> - *demoDebug*
> - *demoRelease*
> - *fullDebug*
> - *fullRelease*

这个很好理解，拿煎饼举例：

> 开发版鸡蛋煎饼
>
> 完成版鸡蛋煎饼
>
> 开发版培根煎饼
>
> 完成版培根煎饼



下面再说说 `flavorDimensions` 的作用，不知道从哪个版本开始，`productFlavors` 需要配合 `flavorDimensions` 使用才行。一般的，没有特殊需求的话，`buildTypes`加上 `productFlavors` 已经完全够用了，`flavorDimensions` 是用于这样的需求的：

> 还是拿煎饼举例，有的人喜欢鸡蛋煎饼，有的人喜欢培根煎饼，但是有的人都喜欢，那怎么办呢？总不能买两个吧，所以好的办法是做一个加鸡蛋和培根的煎饼。这个就是 flavorDimensions 的作用了。

```groovy
android {
  ...
  buildTypes {
    debug {...}
    release {...}
  }

  // Specifies the flavor dimensions you want to use. The order in which you
  // list each dimension determines its priority, from highest to lowest,
  // when Gradle merges variant sources and configurations. You must assign
  // each product flavor you configure to one of the flavor dimensions.
  flavorDimensions "api", "mode"

  productFlavors {
    demo {
      // Assigns this product flavor to the "mode" flavor dimension.
      dimension "mode"
      ...
    }

    full {
      dimension "mode"
      ...
    }

    // Configurations in the "api" product flavors override those in "mode"
    // flavors and the defaultConfig block. Gradle determines the priority
    // between flavor dimensions based on the order in which they appear next
    // to the flavorDimensions property above--the first dimension has a higher
    // priority than the second, and so on.
    minApi24 {
      dimension "api"
      minSdkVersion 24
      // To ensure the target device receives the version of the app with
      // the highest compatible API level, assign version codes in increasing
      // value with API level. To learn more about assigning version codes to
      // support app updates and uploading to Google Play, read Multiple APK Support
      versionCode 30000 + android.defaultConfig.versionCode
      versionNameSuffix "-minApi24"
      ...
    }

    minApi23 {
      dimension "api"
      minSdkVersion 23
      versionCode 20000  + android.defaultConfig.versionCode
      versionNameSuffix "-minApi23"
      ...
    }

    minApi21 {
      dimension "api"
      minSdkVersion 21
      versionCode 10000  + android.defaultConfig.versionCode
      versionNameSuffix "-minApi21"
      ...
    }
  }
}
...
```

这样我们就可以打更多的包了：

> Build variant: `[minApi24, minApi23, minApi21][Demo, Full][Debug, Release]`
>
> Corresponding APK: `app-[minApi24, minApi23, minApi21]-[demo, full]-[debug, release].apk`

flavorDimensions 有两个值，一个值有 3 个对应的 `productFlavor`，一个值有 2 个对应的 `productFlavor`。`buildTypes` 有 2 个 ，所以共有 3 x 2 x 2 = 12 个构建变体。



我们知道，`productFlavors` 相当于蘸料，作为一个蘸料，它肯定要与主食材配合，互相激发各自的味道才行，那么 `productFlavors`是如何影响打包内容的呢？我们介绍一下 sourceSet。

SourceSets中的属性，可以**指定哪些文件夹下的源文件、资源要被编译，哪些源文件要被排除**。

从我们的工程目录可以看出来，`com.android.application`插件默认就给我们定了3个 sourceSet：

- main
- androidTest
- test

默认情况下，main里面的东西大概是这样的：

```groovy
main {
    manifest.srcFile 'src/main/AndroidManifest.xml'
    java.srcDirs = ['src/main/java']
    res.srcDirs = ['src/main/res']
    assets.srcDirs = ['src/main/assets']
}
```

可以看到这都是我们熟悉的目录，既然 sourceSet 是可以配置的，就说明只要我们有需求，我们的 Java 文件就可以写到其他地方，而不是只能在 `src/main/java` 中。比如我们再添加一个 java 目录：

```groovy
main {
    manifest.srcFile 'src/main/AndroidManifest.xml'
    java.srcDirs = ['src/main/java', 'src/main/java2']
    res.srcDirs = ['src/main/res']
    assets.srcDirs = ['src/main/assets']
}
```

这样，我们就有了两个 java 目录，用于存放不同作用的文件集合。这个功能配合 `productFlavors` 管理不同作用版本的源代码就很方便了。比如，我出一个 free 版的应用，功能简单，代码都放在 java 里面，我再出一个 pro 版的，高级功能，代码放在 java2 里面，打包的时候将 java2 与 java 都打进去，就是一个完整版的。

我们的 src 是存放所有源代码与资源的目录，默认它有3个子目录，但是我们也可以添加自己的目录。比如：

- src/demoDebug/ 这个就是 `productFlavors` 为 demo，`buildTypes`为 debug 的构建变体的目录。
- src/debug/ 这个是 `buildTypes`为 debug 的构建变体的目录。

当我们构建的时候，会根据选择的构建类型来合并某些目录，比如构建 demoDemo，会合并 main + debug + debugDemo，合并的时候会有优先级的，构建变体越具体的优先级越高，demoDebug 就比 debug 要具体，所以 demoDebug  的优先级高，而 debug 又比 mian 高。资源会替换，但是 Java 类会报类重复的错。

当然 sourceSet 还有更多的用法，这里就不深入了。



还有一个很重要的知识需要补充，因为我最早是认为为不同 build type 产生不同的 BuildConfig 文件内容是不可行的，因为我们的 BuildConfig 需要在APP结构的最底层生成，这样才能被其他组件所引用到。而我是一直认为 library 工程只能打 release 包的，所以它只能生成  release 版的 BuilConfig 文件。直到我看到了官方文档的描述：

> Android plugin 3.0.0 and higher include a new dependency mechanism that automatically matches variants when consuming a library. 

也就是说从 3.0.0 的插件开始，就可以自动匹配了，而不是一直打 release 版的了，Nice~~~

另外说一句，如果 library 没有匹配的，可以使用 matchingFallbacks 指定备用匹配的。

```groovy
// In the app's build.gradle file.
android {
    buildTypes {
        debug {}
        release {}
        staging {
            // Specifies a sorted list of fallback build types that the
            // plugin should try to use when a dependency does not include a
            // "staging" build type. You may specify as many fallbacks as you
            // like, and the plugin selects the first build type that's
            // available in the dependency.
            matchingFallbacks = ['debug', 'qa', 'release']
        }
    }
}
```



下面我们正式进入主题，实现我们所说的功能，有了上面的基础，我们缺的就只是 gradle api 的熟悉程度了，这个没什么办法，只能自己查资料了。

首先，根据官方文档的例子，我们知道，android 为我们提供了一个叫做 `variantFilter`的东西，就是一个过滤器，显然是用来过滤哪些我们不想打的包：

```groovy
android {
  ...
  buildTypes {...}

  flavorDimensions "api", "mode"
  productFlavors {
    demo {...}
    full {...}
    minApi24 {...}
    minApi23 {...}
    minApi21 {...}
  }

  variantFilter { variant ->
      def names = variant.flavors*.name
      // To check for a certain build type, use variant.buildType.name == "<buildType>"
      if (names.contains("minApi21") && names.contains("demo")) {
          // Gradle ignores any variants that satisfy the conditions above.
          setIgnore(true)
      }
  }
}
...
```

嗯，不符合我们的要求，但是它给了我们一个提示，既然有过滤器，那么应该也有一个遍历所有构建变体的方法，我们在遍历的时候，去读取 urls.gradle 里面的 map，然后动态的添加进去不就好了吗！！

经过搜索之后，我就发现了这个东西：

```groovy
    applicationVariants.all {
        variant ->
            def buildType = getBuildType()
            rootProject.ext.urls.each {
                entry ->
                    entry.value['PATHS'].each {
                        path ->
                            def key = "${entry.key}_${path.key}"
                            def value
                            if (getFlavorName() == 'full' && buildType.name == 'release') {
                                value = "${entry.value['PRODUCT_HOST']}${path.value}"
                            } else {
                                value = "${entry.value['SIT_HOST']}${path.value}"
                            }
                            variant.buildConfigField "String", key, "\"${value}\""
                    }
            }
    }
```

applicationVariants 里面有所有需要打包的变体，我们遍历一下，既可以获取变体对象，然后动态的添加我们想要添加的 buildConfigField。

需要注意的是，我最开始以为 `com.android.build.gradle.api.BaseVariant#getBuildType`返回的是`com.android.build.gradle.internal.dsl.BuildType` ，但是没想到返回的是`com.android.builder.model.BuildType`，不注意就搞错了，`com.android.builder.model.BuildType`里面是无法添加 buildConfigField 的。