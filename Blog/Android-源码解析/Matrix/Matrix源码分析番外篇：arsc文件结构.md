---
title: Matrix源码分析番外篇：arsc文件结构
date: 2020-7-15
categories: Matrix
---

在 gradle 中，配置如下代码可以将无用的资源移除：

```groovy
android {
    ...
    buildTypes {
        release {
            shrinkResources true
            minifyEnabled true
        }
    }
}
```

“shrinkResources”资源压缩功能，它需要配合ProGurad的“minifyEnabled”功能同时使用。

如果ProGuard把部分无用代码移除，这些代码所引用的资源也会被标记为无用资源，然后通过资源压缩功能将它们移除。

这个看起来很不错，但是实际上却有些待改进的地方。

- 对于一些无用的 String、ID、Attr、Dimen 等资源，实际上还存在于 .arsc 文件中。
- 对于Drawable、Layout这些无用资源，shrinkResources也没有真正把它们删掉，而是仅仅替换为一个空文件。

还是用例子说话吧，我们写一个初始demo（为了减少资源，这里连一个依赖库都不引入，appcompat 也不要），里面有些无用的资源：

> strings.xml

```xml
<resources>
    ...
    <string name="unused_string">这是个无用的字符串</string>
</resources>
```

> activity_old_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:text="old main activity" />

</android.support.constraint.ConstraintLayout>
```

> build.gradle

```groovy
    buildTypes {
        debug {
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
        release {
            shrinkResources true
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
```

我们这里，有一个无用的字符串：unused_string，有一个无用布局文件：activity_old_main，可以打 debug 与 release 包，release 包配置了混淆，先使用 010Editor 分析一下 debug 包的 .arsc 文件，看看这两个文件的信息是否存在。

#### 字符串池

![image-20200715145733039](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/arsc1.png?raw=true)

可以看到，这个无用文件的路径是存在的。路径存在字符串池中，这里储存了**资源路径与strings.xml中定义的字符串**等等一些资源。

#### 资源名称字符串池

![image-20200715145935250](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/arsc2.png?raw=true)

在package中，也是有这两个资源的数据，它们也是在字符串池中，注意与上面的字符串池的区别。

#### 资源类型字符串池

这里的字符串池储存的是资源的 **类型**  与 **名称**。我们展开 typeStrings 可以看到类型：

![image-20200715150514262](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/arsc3.png?raw=true)

#### ResTable_type

注意到，string 是 strdata 数组第 6+1 个元素，所以，我们展开  `ResTable_typeSpec typeSpec[6]` 后面的 typeType，可以看到：

![image-20200715151305634](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/arsc4.png?raw=true)

这里可以看到，我们定义的 app_name，unused_string，但是他们的值却看不到，我刚开始以为是模板出错了，后来想了一下，可能是因为如果是图片等资源的话，根本就解析不出来，所以这里的 value 因该是指向的第一个字符串池，里面储存的是资源路径，拿到资源路径后再去加载资源。理论上值就是上面字符串池中的索引引用。

layout 也是一样的，这里就不贴图了。

有的 typeType 会有多个，是因为不同配置的原因：

![image-20200715151657788](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/arsc5.png?raw=true)

可以看到，这里 mipmap 有6个，是因为我们的 res 目录里面就有 6 个 mipmap 文件夹，其作用就是用作适配的。不同的手机会从不同的结构体中读取资源。

####  ResTable_entry 与 Res_value

还是要重点说一下， ResTable_entry 与 Res_value，最简单的理解就是：

```xml
<bool name="abc_action_bar_embed_tabs">false</bool>
```

ResTable_entry  里面存了 abc_action_bar_embed_tabs，但是不是直接存的，存的是上面字符串池的字符索引。

Res_value 里面存了当前资源的类型，我们需要先要搞清楚它的类型，然后，再去 data 里面去拿资源路径。

我们拿，app_name 来说明一下，arsc 是如何根据 resourceId 来获取资源的。

![image-20200715155705882](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/arsc6.png?raw=true)

我们可以看到，该资源的类型 id 为 7。其 entry 的偏移为 0（它是 entry 数组的第 1 项）。

所以它的 resourceId 为 0x7F070000。

那么这个是怎么的出来的呢？

```
首先，7F 是 packageId，包的命名空间，取值范围为[0x01, 0x7F]，一般第三方应用均为7F，系统的为 01。

07 是资源类型id，这里代表的是 string。

0000 就是在 entry 中的偏移。
```

它的data为0，指向的是第一个字符串池中的第1项，其内容是 Sample。

OK，这里我们对 arsc 应该有一定的了解了，再回到主题，看看 release 包有哪些变化呢？

这里，我就不贴图了，结果就是 .arsc 文件是一样的，连大小都一样。但是我们去 res 目录里面发现，activity_old_main.xml 文件变了。debug 包里面的文件，有 540 个字节，但是 release 包里面的文件只有 104 个字节，使用编辑器打开，发现 release 包里面的文件数据都是 null，与上面的提到的类似，变成了一个空文件。

所以，shrinkResources 并没有我们想的那么美好。

#### ResTable_map_entry

还有，对于 style 与 attr 来说，他们的储存方式又不一样，为啥呢？从写法就可以看出来：

```xml
<attr name="buttonTintMode">
	<enum name="src_over" value="3"/>
	<enum name="src_in" value="5"/>
	<enum name="src_atop" value="9"/>
	<enum name="multiply" value="14"/>
	<enum name="screen" value="15"/>
	<enum name="add" value="16"/>
</attr>

<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
</style>
```

这种写法，显然不能像 string drawable 一样，搞个键值对，只能使用 map：

![image-20200715164958314](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/arsc7.png?raw=true)

这个map 里面没啥东西，是因为我把 AppTheme 里面的东西都删了。

最后，再贴一张 .arsc 的结构图

![](https://user-gold-cdn.xitu.io/2019/8/10/16c7a2df8e9b7641?imageslim)

有了上面的经验，再看这张图应该会清晰一点吧。



其实，我这里讲的很粗，但是我研究了几天这个文件结构，发现比较重要的东西就是这些了，除非你想对 arsc 做自定义处理，否则的话了解上面的东西也就够了。再深入的话，需要去查看对应的 C 结构，然后搞懂每个字段的意义。

老罗的文章讲这个讲的很细，但是对于现在的我来说，看完就忘，因为用不到，所以我暂时只深入到这里吧...

有个额外的东西需要说一下，我特么还在 stackoverflow 上提了问题。

关于字符串常量头的结构：

```c

struct ResStringPool_header
{
    struct ResChunk_header header;
 
    // Number of strings in this pool (number of uint32_t indices that follow
    // in the data).
    uint32_t stringCount;
 
    // Number of style span arrays in the pool (number of uint32_t indices
    // follow the string indices).
    uint32_t styleCount;
 
    // Flags.
    enum {
        // If set, the string index is sorted by the string values (based
        // on strcmp16()).
        SORTED_FLAG = 1<<0,
 
        // String pool is encoded in UTF-8
        UTF8_FLAG = 1<<8
    };
    uint32_t flags;
 
    // Index from header of the string data.
    uint32_t stringsStart;
 
    // Index from header of the style data.
    uint32_t stylesStart;
}
```

string 与 stringStart 都好理解，style 是个啥，字符串还有样式？？？

老罗的文章中说，`<b>man</b><i>go</i>`  b 与 i 就是字符串样式。

```
注意到第四个字符串“<b>man</b><i>go</i>”，它实际表示的是一个字符串“mango”，不过它的前三个字符“man”通过b标签来描述为粗体的，而后两个字符通过i标签来描述为斜体的。字符串“mango”来有两个sytle，第一个style表示第1到第3个字符是粗体的，第二个style表示第4到第5个字符是斜体的。
```

我使用 010Editor 打开后，发现 styleCount 为 1。

这里我没有搞懂读取带标签的 string 的时候，是怎么一个流程。

#### 参考文档

[http://blog.islinjw.cn/2019/05/18/%E5%8F%AF%E8%83%BD%E6%98%AF%E5%85%A8%E7%BD%91%E8%AE%B2%E6%9C%80%E7%BB%86%E7%9A%84%E5%AE%89%E5%8D%93resources-arsc%E8%A7%A3%E6%9E%90%E6%95%99%E7%A8%8B-%E4%B8%80/](http://blog.islinjw.cn/2019/05/18/可能是全网讲最细的安卓resources-arsc解析教程-一/)

[http://blog.islinjw.cn/2019/05/21/%E5%8F%AF%E8%83%BD%E6%98%AF%E5%85%A8%E7%BD%91%E8%AE%B2%E6%9C%80%E7%BB%86%E7%9A%84%E5%AE%89%E5%8D%93resources-arsc%E8%A7%A3%E6%9E%90%E6%95%99%E7%A8%8B-%E4%BA%8C/](http://blog.islinjw.cn/2019/05/21/可能是全网讲最细的安卓resources-arsc解析教程-二/)

https://juejin.im/post/5d4e60c15188255d2a78b86d

https://blog.csdn.net/luoshengyang/article/details/8744683