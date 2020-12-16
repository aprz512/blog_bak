---
title: 压缩、混淆、优化你的App
index_img: /cover/18.jpg
banner_img: /cover/top.jpg
date: 2019-08-18
tags: Android-汉化
---




为了让App的体积尽可能的小，我们在打 release 包的时候，应该开启 shrinking 选项来移除无用代码和资源。当开启 shrinking 后，还会带来一些额外好处，比如混淆，它会缩短App里面类与成员的名字，比如优化，它会采取一些激进的策略来进一步减小App的大小。

当我们的项目使用的是 [Android Gradle plugin 3.4.0](https://developer.android.com/studio/releases/gradle-plugin#3-4-0) 及以上的时候，gradle 插件不再使用 ProGuard 来执行编译时的代码优化，而是使用 R8 编译器来处理下面的编译时任务：

- **代码压缩**：检测 App 与依赖的 library 中的无用类，字段，方法，属性并移除它们（可以缓解一下 64K 问题）。比如：如果我们只使用了一个依赖库中的少量方法，代码压缩就可以识别这些使用的代码，并且移除哪些未使用的代码。
- **资源压缩**：移除打包App中的未使用的资源，包括library中未使用的资源。它最好与代码压缩一起使用，因为代码被移除之后，这些代码引用的资源也就可以安全移除了。
- **混淆**：缩短类与成员的名字的长度，也就可以减少 dex 的大小。
- **优化**：检查并重写代码，来进一步减小 dex 的大小。比如：R8发现 `else{}` 是一段无法到达的代码（永远不会走这个分支），那么它会移除这个 `else{}`分支。

当构建 release 版本的时候，R8 会默认执行上面说的任务。你也可以通过 ProGuard 规则文件禁止这些任务的执行。实际上，R8 编译器与 ProGuard 一样，都会受到 ProGuard 规则文件的影响。



## 开启压缩，混淆与优化

当使用Android Studio 3.4或Android Gradle插件3.4.0及更高版本时，将项目的Java字节码转换为在Android平台上运行的DEX文件的默认编译器是 R8。但是，**使用Android Studio创建新项目时**，默认情况下不会启用这些优化。这是因为这些编译时优化会增加项目的构建时间，而且如果你不知道如何去自定义的保留代码，可能会导致运行时错误。

所以，最好是在准备发布，打包App的最终版本的时候，开启这些选项（module下的 build.gradle）：

```groovy
android {
    buildTypes {
        release {
            // Enables code shrinking, obfuscation, and optimization for only
            // your project's release build type.
            minifyEnabled true

            // Enables resource shrinking, which is performed by the
            // Android Gradle plugin.
            shrinkResources true

            // Includes the default ProGuard rules files that are packaged with
            // the Android Gradle plugin. To learn more, go to the section about
            // R8 configuration files.
            proguardFiles getDefaultProguardFile(
                    'proguard-android-optimize.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```



## R8 配置文件

R8使用 ProGuard 规则文件来修改其默认行为，还可以更好地了解应用程序的结构，例如那些作为应用程序代码入口点的类。尽管这些规则文件可以修改，但是有些规则文件是编译工具自动生成的，比如 AAPT2，还有些是从 library 中继承过来的。

下面这个表描述了 R8 使用的规则文件的来源：

| Source                               | Location                                                     | Description                                                  |
| ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Android Studio                       | \<module-dir>/proguard-rules.pro                             | 创建一个新的工程的时候，IDE会在 module 目录下生成一个文件，里面默认是没有应用任何规则，但是我们可以添加自定义规则。 |
| Android Gradle plugin                | Gradle 插件在编译时生成文件                                  | Android Gradle插件会生成proguard-android-optimize.txt，其中包含对大多数Android项目有用的基本规则，并且还启用了 `@Keep` 注解。 |
| Library dependencies                 | AAR libraries: `<library-dir>/proguard.txt` <br/> JAR libraries: `<library-dir>/META-INF/proguard/` | 如果AAR库在发布时使用自己的ProGuard规则文件，当我们的项目依赖该 AAR 时，则R8会在编译项目时自动应用其规则。<br />为AAR库配置打包的规则文件非常有用，因为它意味着——库开发人员已经为我们执行了故障排除步骤。<br />但是，我们应该知道，因为ProGuard规则是附加的，所以AAR库依赖项包含的某些规则无法删除，并且可能会影响应用程序其他部分的编译。例如，如果库包含禁用代码优化的规则，则该规则将禁用整个项目的优化。 |
| Android Asset Package Tool 2 (AAPT2) | 设置 minifyEnabled 为 true，构建之后，会生成文件 \<module-dir>/build/intermediates/proguard-rules/debug/aapt_rules.txt | AAPT2根据应用程序manifest，布局和其他应用程序资源中的类的引用生成保留规则。例如，AAPT2为应用程序manifest中作为入口点注册的每个Activity都有保留规则。 |
| 自定义配置文件                       | 与第一个一样，我们可以额外的创建一些文件用来指定规则，创建之后需要在 build.gradle 中配置 | 可以包含其他配置，R8会在编译时应用它们。                     |

将minifyEnabled属性设置为true时，R8会合并上面列出的所有可用来源的规则。当使用R8进行故障排除时，你一定要了解这些，以免出现一些你无法理解的行为，比如 library 中的规则影响到整个工程。

你可以使用下面的命令来输出一份详细的关于 R8 编译的报告文件：

```shell
// You can specify any path and filename.
-printconfiguration ~/tmp/full-r8-config.txt
```



## 包含其他配置

使用Android Studio创建新项目或模块时，IDE会创建|<module-dir> /proguard-rules.pro文件，以便我们包含自己的规则。我们还可以通过将其他规则文件添加到模块的build.gradle文件中的proguardFiles属性中来包含其他的规则。

例如，可以通过在相应的productFlavor块中添加另一个proguardFiles属性来为指定的构建变体添加额外的规则。下面的示例将`flavor2-rules.pro`添加到`flavor2`的 `product flavor`中。现在，`flavor2` 就有了3个规则文件，因为 release 块中配置的文件也包含在里面。

```groovy
android {
    ...
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile(
              'proguard-android-optimize.txt'),
              // List additional ProGuard rules for the given build type here. By default,
              // Android Studio creates and includes an empty rules file for you (located
              // at the root directory of each module).
              'proguard-rules.pro'
        }
    }
    flavorDimensions "version"
    productFlavors {
        flavor1 {
          ...
        }
        flavor2 {
            proguardFile 'flavor2-rules.pro'
        }
    }
}
```



## 压缩代码

将minifyEnabled属性设置为true时，默认情况下会启用使用R8的代码压缩。

代码压缩是 R8 删除在运行时不需要的代码的过程。整个过程可以极大的减少App的体积，特别是你引入很多库，但是每个库都只使用了少量的功能。

要压缩应用程序的代码，R8首先根据组合的配置文件集确定应用程序代码中的所有入口点。这些入口点包括Android平台可用于打开应用程序的所有Activity或Service。R8会从每个入口点开始检查应用程序的代码，检测应用程序可能在运行时访问的所有方法，成员变量和其他类，然后构建一个图。未连接到该图的代码被视为无法访问，可能会从应用中删除。如下图所示：

![](https://developer.android.com/studio/images/build/r8/tree-shaking.png)

上图中建立的图结构中，OkayApi 类没有在其中，所以它会被删除。



R8 通过工程的配置文件集来决定所有入口点，也就是说，keep规则指定R8在缩小应用程序时不应丢弃的类，R8将这些类视为应用程序的可能入口点。Android Gradle插件和AAPT2会自动生成大多数应用项目所需的保留规则，例如应用的activities，views和services。当然，如果又需要，你也可以添加自己的规则。

或者，您可以将@Keep注释添加到要保留的代码中。在类上添加@Keep会使整个类保持原样。在方法或字段上添加它将保持方法/字段（及其名称）以及类名完整。请注意，此注释仅在使用`AndroidX`注释库时以及包含随Android Gradle插件打包的ProGuard规则文件时才可用。



## 压缩资源

资源压缩只有配合代码压缩才会起作用。在代码缩减器删除所有未使用的代码之后，资源缩减器可以识别应用程序仍在使用哪些资源。添加包含资源的代码库时尤其如此 - 必须删除未使用的库代码，以便库的资源不会被引用，从而这些资源可以被资源缩减器移除。

想要开启资源缩减，只需要配置一下：

```groovy
android {
    ...
    buildTypes {
        release {
            shrinkResources true
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
}
```



## 自定义需要保留的资源

通过配置一个 xml 文件，你可以自定义需要保留或者丢弃的资源。

```groovy
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@layout/l_used*_c,@layout/l_used_a,@layout/l_used_b*"
    tools:discard="@layout/unused2" />
```

`tools:keep` 与 `tools:discard`都可以指定多个资源，使用分号隔开。还可以使用 * 号作为通配符。

将该文件放置到工程的资源目中，比如：`res/raw/keep.xml`。 该文件不会被打包到 APK 中。

指定哪些资源需要被删除，看起来没啥屌用，因为我们可以直接的手动删除这个资源。但是如果我们在构建多个 `variants` 的时候就非常有用了，比如，我们可能将所有资源放在了一个 common 工程中，然后创建了 keep.xml 来为每个 `variant` 来指定需要保留与丢弃的资源。因为这个时候，资源都在代码里面被引用了，所以资源缩减器不会移除这些资源。还有另外一种情况，构建工具可能无法区别资源id的引用与 int 值的区别，如果刚好某个资源的id与我们代码中int值相等，它也不会移除这个资源。



## 开启严格引用检查

通常情况下，资源缩减器可以正确的识别哪些资源被引用了。但是，如果我们这样引用资源：

```java
Resources.getIdentifier()
```

或者，我们的依赖库中有这样使用的（AppCompat就是）。遇到这种情况，资源缩减器默认情况下会采取**防御措施**，并将所有具有匹配名称格式的资源标记为可能已使用且无法删除。

举个例子：

```kotlin
val name = String.format("img_%1d", angle + 1)
val res = resources.getIdentifier(name, "drawable", packageName)
```

这会将具有img_前缀的所有资源标记为已使用。

资源缩减器还会查看代码中的所有字符串常量以及各种`res/ raw/ `资源，以及类似于`file:///android_res/drawable//ic_plus_anim_016.png`的格式的资源URL。如果它发现像这样的字符串或者 url，它就不会移除这些资源。

这个行为默认是开启的，但是如果想要关闭它，就可以使用 `strict` 模式：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:shrinkMode="strict" />
```

开启了严格模式之后，使用上面的方式引用的资源也会被删除，所以我们需要在 `keep.xml` 里面保留这些资源。



## 移除未使用的可选资源

Gradle资源缩减器仅删除应用程序代码未引用的资源，这意味着它不会删除不同设备配置的备用资源。

如有必要，您可以使用Android Gradle插件的resConfigs属性来删除您的应用不需要的备用资源文件。

例如，如果您使用的是包含语言资源的库（例如AppCompat或Google Play服务），那么您的APK会包含这些库中所有翻译语言的字符串。如果您只想保留应用正式支持的语言，可以使用resConfig属性指定这些语言，删除未指定语言的资源。

下面的代码就展示了如何只保留两种语言资源：

```groovy
android {
    defaultConfig {
        ...
        resConfigs "en", "fr"
    }
}
```

只保留了英语与法语。同样的，我们还可以只保留某一屏幕密度下的资源。



## 合并重复资源

默认情况下，Gradle还会合并具有相同名称的资源，例如可能位于不同资源文件夹中的同名的drawable（这里应该指的是不同library的资源）。**此行为不受shrinkResources属性控制，并且无法禁用**，因为当多个资源与您的代码查找的名称匹配时，必须避免这种情况。

仅当两个或多个文件共享相同的资源名称，类型和限定符时，才会发生资源合并。 Gradle选择它认为最合适的一个并保留。

Gradle 从下面的路径来寻找重复的资源：

- src/main/res
- build flavor 与 build type
- 依赖库

资源优先级从低到高：

依赖库 → Main → Build flavor → Build type

举个例子：如果`Main`和`build flavor`中出现重复的资源，Gradle将选择`build flavor`中的资源。

如果重复资源在同一层次出现，比如`src/main/res/` 和 `src/main/res2/`，则 gradle 无法完成资源合并，这时会报资源合并错误。





## 代码混淆

混淆的目的是通过缩短APP的类，方法和字段的名称来减少应用程序的大小。以下是使用R8进行混淆处理的示例：

```java
androidx.appcompat.app.ActionBarDrawerToggle$DelegateProvider -> a.a.a.b:
androidx.appcompat.app.AlertController -> androidx.appcompat.app.AlertController:
    android.content.Context mContext -> a
    int mListItemLayout -> O
    int mViewSpacingRight -> l
    android.widget.Button mButtonNeutral -> w
    int mMultiChoiceItemLayout -> M
    boolean mShowTitle -> P
    int mViewSpacingLeft -> j
    int mButtonPanelSideLayout -> K
```

虽然混淆不会从程序中删除代码，但是如果你的程序中有许多类，方法和字段，那么节省的大小还是很可观的。但是需要注意是，由于代码进行了混淆，那么运行时发生了错误，打印出来的堆栈，我们就看不懂了，需要一些额外的工具来帮助我们进行还原代码信息。

**另外，如果你写的代码需要依赖原本类或者方法的名字（比如，反射等），那么你就需要在 ProGuard 文件中 keep 你使用到的类或者方法。**



## 解码混淆后的堆栈信息

在 R8 混淆了代码之后，阅读堆栈信息几乎是不可能的了，因为方法名类名都换成了很简单的英文字母了。除了重命名，R8 还会改变堆栈信息代码所在的行号。不过还好，我们可以通过 mapping.txt 文件来还原堆栈信息，我们每次构建App的时候，都会生成一个 mapping.txt 文件，里面会包含所有类，方法，字段的混淆前与混淆后的映射关系，当然，里面也有行号的对应关系。该文件一般在 `<module- name>/build/outputs/mapping/<build-type>/` 目录下面。

> R8每次构建项目时都会覆盖生成的mapping.txt文件，因此每次发布新版本时都必须小心保存副本。通过为每个发布版本保留mapping.txt文件的副本，如果用户从旧版本的应用程序提交混淆后的堆栈信息，您将能够追踪调试问题。

在Google Play上发布您的应用时，您可以为每个版本的APK上传mapping.txt文件。然后，Google Play会根据用户报告的问题对传入的堆栈跟踪进行反混淆处理，以便您可以在Google Play控制台中查看这些堆栈信息。有关详细信息，请参阅帮助中心的文章，了解如何对崩溃堆栈跟踪进行反混淆处理。

 要将混淆的堆栈跟踪转换为可读的堆栈跟踪，请使用 [ReTrace](https://www.guardsquare.com/en/products/proguard/manual/retrace) 脚本，在`SDK/tools/proguard/bin` 目录下。



## 代码压缩

为了进一步的减小 App 的体积，R8 会在更深的层次上来检查你的代码，然后移除无用代码，甚至有时候会重写我们的代码，让其更简洁。下面举几个例子：

- 如果你写了一个 `else{}` 块，但是任何情况下都不会走这个分支，那么R8可能会将它删除。
- 如果某个方法只在唯一一个地方被调用，那么 R8 可能会将方法内联到调用的地方，并将这个方法移除。
- 如果某个类只有一个子类，并且该类没有被实例化（举个例子，一个抽象类，只有一个子类），那么 R8 会将这两个类合成一个类。
- 更多内容请看大神的博客 [R8 optimization blog posts](https://jakewharton.com/blog/)

R8不允许您禁用或启用离散优化，或修改优化的行为。实际上，R8忽略了任何试图修改默认优化的ProGuard规则，例如-optimizations和 - optimizepasses。此限制很重要，因为随着R8的不断改进，维护标准的优化行为有助于Android Studio团队轻松排除故障并解决您可能遇到的任何问题。



## 启动更加激进的优化策略

R8 里面还包含一些默认没有开启的优化行为。你可以开启它们，自需要在 gradle.properties 里面添加：

```properties
android.enableR8.fullMode=true
```

假设您的代码通过Java Reflection API引用了一个类。默认情况下，R8假定您打算在运行时检查和操作该类的对象 - 即使您的代码实际上没有 - 并且它会自动保留该类及其静态初始化程序。

但是，当使用“**完整模式**”时，R8不会做出这种假设，如果R8断言你的代码在运行时从不使用该类，它会从你应用程序的最终DEX中删除该类。也就是说，如果要保留类及其静态初始化程序，则需要在规则文件中包含保留规则才能执行此操作。

有问题，查看文档，并报告问题。

> If you encounter any issues while using R8’s “full mode”, refer to the [R8 FAQ page](https://r8.googlesource.com/r8/+/refs/heads/master/compatibility-faq.md) for a possible solution. If you are unable to resolve the issue, please [report a bug](https://issuetracker.google.com/issues/new?component=326788&template=1025938).



##  使用R8进行故障排除

当我们开启了代码压缩，资源压缩等选项的时候，可能会遇到一些问题，可以查看帮助文档。

> If you do not find a solution to your issue below, also read the [R8 FAQ page](https://r8.googlesource.com/r8/+/refs/heads/master/compatibility-faq.md) and [ProGuard’s troubleshooting guide](https://www.guardsquare.com/en/products/proguard/manual/troubleshooting).



### 生成被移除代码报告记录

为了帮助您解决某些R8带来问题，查看R8从您的应用中删除的所有代码的报告可能会很有用。将`-printusage <output-dir> /usage.txt`添加到自定义规则文件中，就可以为模块生成报告记录。当您启用R8并构建应用程序时，R8会输出一个包含您指定的路径和文件名的报告。已删除代码的报告类似于以下内容：

```java
androidx.drawerlayout.R$attr
androidx.vectordrawable.R
androidx.appcompat.app.AppCompatDelegateImpl
    public void setSupportActionBar(androidx.appcompat.widget.Toolbar)
    public boolean hasWindowFeature(int)
    public void setHandleNativeActionModesEnabled(boolean)
    android.view.ViewGroup getSubDecor()
    public void setLocalNightMode(int)
    final androidx.appcompat.app.AppCompatDelegateImpl$AutoNightModeManager getAutoNightModeManager()
    public final androidx.appcompat.app.ActionBarDrawerToggle$Delegate getDrawerToggleDelegate()
    private static final boolean DEBUG
    private static final java.lang.String KEY_LOCAL_NIGHT_MODE
    static final java.lang.String EXCEPTION_HANDLER_MESSAGE_SUFFIX
...
```

如果你想查看，R8保留了哪些类，可以添加 `-printseeds <output-dir>/seeds.txt` 到自定义规则文件中。文件内容大致如下：

```java
com.example.myapplication.MainActivity
androidx.appcompat.R$layout: int abc_action_menu_item_layout
androidx.appcompat.R$attr: int activityChooserViewStyle
androidx.appcompat.R$styleable: int MenuItem_android_id
androidx.appcompat.R$styleable: int[] CoordinatorLayout_Layout
androidx.lifecycle.FullLifecycleObserverAdapter
...
```



### 资源故障排除

压缩资源时，“构建”窗口 ![](https://developer.android.com/studio/images/buttons/window-build-2x.png) 会显示从APK中删除的资源的摘要。

比如：

```shell
:android:shrinkDebugResources
Removed unused resources: Binary resource data reduced from 2570KB to 1711KB: Removed 33%
:android:validateDebugSigning
```

Gradle还在`<module-name> / build / outputs / mapping / release /`（与ProGuard的输出文件相同的文件夹）中创建名为`resources.txt`的诊断文件。此文件包含详细信息，例如哪些资源引用其他资源以及使用或删除了哪些资源。

比如，你想知道为啥 `@drawable/ic_plus_anim_016` 这个文件被打包到了 APK 中，你可以打开 resource.txt 文件，然后搜索这个文件名。你就可以找到谁引用了这个资源：

```shell
16:25:48.005 [QUIET] [system.out] &#64;drawable/add_schedule_fab_icon_anim : reachable=true
16:25:48.009 [QUIET] [system.out]     &#64;drawable/ic_plus_anim_016
```

这个说明了，add_schedule_fab_icon_anim 引用了 ic_plus_anim_016，现在你要找出为什么 add_schedule_fab_icon_anim  是可达的！继续向上搜索，你会发现这个资源在 `The root reachable resources are:` 下面的列表中。这表示有代码引用了 add_schedule_fab_icon_anim 。

如果我们没有使用 `strict` 检查，还有一种情况下资源id也可能被标记为可达的：

```java
10:32:50.590 [QUIET] [system.out] Marking drawable:ic_plus_anim_016:2130837506
    used because it format-string matches string pool constant ic_plus_anim_%1$d.
```

这个显然就是使用资源名来动态加载资源的情况。遇到这种情况，你需要手动的指定该资源需要被移除。
