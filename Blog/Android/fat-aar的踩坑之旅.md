---
title: fat-aar的踩坑之旅
date: 2020-12-8
categories: Android
---

因为某些原因（比如来自业务的不可抗力），要求将所有模块打包成一个sdk提供给别人使用，所以才有了这篇文章。虽然在实践的过程中学到了很多东西，但是我还是想吐槽一下这个需求。

我们使用的是 [fat-aar-android](https://github.com/kezong/fat-aar-android) 这个开源插件。

在集成的过程中遇到了很多小问题，暂且不聊，说一些比较蛋疼的问题。

###主题问题

第一个是 activity 的主题问题，因为 activity 使用的是 application 的主题，现在打成 sdk 了，肯定不能让别人使用我们的 application。所以需要解决 activity 的主题设置问题。

给每个 activity 加上 theme，这是我们的目标。但是肯定不能手动加，几百个 activity，加上去就太累了，想到的方案是处理合并之后的 manifest 文件，这样可以利用 gradle 脚本来完成，具体如下：

```groovy
    // 处理合并之后的 manifest 文件，给没有设置主题的 activity 加上主题
    project.afterEvaluate {
        libraryVariants.all { variant ->
            String variantName = variant.name.capitalize()
            def processManifestTask = project.tasks.getByName("process${variantName}Manifest")
            processManifestTask.doLast { pmt ->
                println(">>> " + "处理合并之后的 manifest 文件" + " <<<")
//                String manifestPath = "$pmt.manifestOutputDirectory/AndroidManifest.xml"
                String manifestPath = "${project.buildDir.path}/intermediates/library_manifest/${variant.name}/AndroidManifest.xml"

                def manifestFile = file(manifestPath)
                // 忽略命名空间，否则语法比较蛋疼
                def xml = new XmlSlurper(false, false)
                        .parse(manifestFile)

                xml.'application'.'activity'.each { activity ->
                    def themeExist = activity['@android:theme'].toString()
                    if (!themeExist) {
                        println(">>>" + "为${activity['@android:name']}添加XXXTheme" + "<<<")
                        activity['@android:theme'] = '@style/XXXTheme'
                    }
                }

                def outputBuilder = new StreamingMarkupBuilder()
                String updatedXml = outputBuilder.bind { mkp.yield xml }
                file(manifestPath).write(updatedXml)
            }
        }
    }
```

其实，就是在 processXXXManifest 这个任务后面加上自己的一段逻辑。

不过，这里有个问题需要注意，因为 library 不像 app，它不会将依赖工程的 manifest 给合进来，所以 fat-aar 自己创建了一个 “task”（不算一个真正的task） 来做这件事，所以你需要先弄清楚 task 之间的先后顺序，再来看在这里处理 manifest 是否合适。

看它的代码，它创建的 合并task 也是在 processXXXManifest 之后，在我们的逻辑之前执行，所以没有问题。

### 资源问题

第二个问题是，R 文件导致的问题。

我们知道，aar 里面是不能有 R.java 这个东西的，因为 R.java 如果存在的话，它的字段值就被固定了，这样，app 集成 aar 的时候，就很容易导致字段值的冲突问题。所以，aar 里面是采用 R.txt 这个文件来代替 R.java，app集成的时候，会有task根据这个问题，重新计算所有资源的id，为每个module生成 R.java 文件。

那么，问题就来了，当我们将多个 aar 合并成一个的时候，app集成时，只会生成一个R.java 文件，就是主工程的。主工程的依赖工程就会报错，因为它写代码的时候，用的是自己的 R 文件，但是 app 集成的时候，却没有生成对应的 R 文件。

fat aar 提供的解决方案是这样的，它创建了一个 task，生成了依赖工程的 R 文件。但是 R 文件里面的值是无法确定的（它的值应该是 app 集成时确定），所以，fat-aar 是这样做的：

```java
public final class R {
  public static final class string {
    // 将依赖工程的 R 字段，指向主工程对应的字段
    public static int res1 = com.main.library.R.string.res1;
    }
}
```

主工程的R.txt 合并所有依赖工程的 R.txt，这样就解决了R文件的问题。

但是，实际上，我们在集成的时候，遇到了这样的问题：

`NoSuchFieldError`

这是因为，假如模块使用的A版本的Support库中有资源文件res1，而接入方App使用了更高版本的Support库中没有res1这个资源，那么App构建生成的R文件中将没有res1这个资源索引，可是我们打入AAR中的R文件还保留着对res1的符号引用，结果就是在运行时类初始化失败。

那么，应该如何处理呢？

一种解决方案是：

将这种“桥接”的方式改成使用字节码修改的方式！
比如，某个依赖中是这样使用R 的

```
import com.lib.R.layout;

public class A {
        static {
            sKeys.put("layout/slbase_activity_camera_web_0", layout.slbase_activity_camera_web);
        }
}
```

经过 transform 后，会变成下面的样子：

```
// 这里包名变了
import com.aar.R.layout;

public class A {
        static {
            sKeys.put("layout/slbase_activity_camera_web_0", layout.slbase_activity_camera_web);
        }
}
```

就是使用 transform 将所有依赖的 R 文件的包名改成 aar 的R包名。这样可以避免字段失效问题，就不用为每个依赖生成 R 文件，以及打对应的 jar 包了。

这个已经向 fat-aar-andorid 提了一个 pr。

还有一种更为简单的方式，就是将 gradle plugin 升级到 3.6.4（往上应该也可以）。这又是为啥呢?

在App的构建流程中，会根据最终的Support库版本（App依赖的Support库版本不可知）对各个R.txt里的内容做过滤，只保留App工程中确实存在的资源文件，生成最终的R文件。所以我们才遇到了上面的字段问题，那么如果打 aar 的时候也进行过滤呢？不就没有问题了吗！

事实上，在 3.6.4 里面，在打 aar 的时候，生成的 R.java 文件，确实进行了过滤。下面贴一段 fat-aar的代码：

```groovy
    File getLocalSymbolFile() {
        // > 3.6.0, R.txt contains remote resources, so we use R-def.txt
        if (Utils.compareVersion(mGradlePluginVersion, "3.6.0") >= 0) {
            return mProject.file(mProject.buildDir.path + '/intermediates/local_only_symbol_list/' + mVariant.name + "/R-def.txt")
        } else if (Utils.compareVersion(mGradlePluginVersion, "3.1.0") >= 0) {
            return mProject.file(mProject.buildDir.path + '/intermediates/symbols/' + mVariant.dirName + "/R.txt")
        } else {
            return mProject.file(mProject.buildDir.path + '/intermediates/bundles/' + mVariant.name + "/R.txt")
        }
    }
```

可以看到，这里面过滤了远程依赖的资源(使用R-def.txt文件)，只保留了本地的资源。

### databinding问题

最后还有一个问题需要说一下，就是  databinding 是在 gradle 里面的，版本升级之后，databinding 生成的类的签名有些发生了改变，若你你的sdk采用的是 3.4.1 编译的，那么在 3.2.1 工程里面是无法使用的。请看：https://stackoverflow.com/questions/54221707/databinding-nosuchmethoderror-with-buildtools-3-4-0



### 参考文档