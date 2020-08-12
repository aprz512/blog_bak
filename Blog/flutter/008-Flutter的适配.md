---
title: 008-Flutter的适配
date: 2020-8-10
categories: Flutter
---

这里只说一个适配，rpx 适配。rpx可以理解为自适应的 px。啥意思呢？就是我们以 UI 给出的图为基准，将屏幕宽度分成固定的份数。

假如，UI是按照 500*800 出的设计图，那么我们将屏幕的宽度分为 500 份，那么每一份的宽度就是一个 rpx。

这样适配的话，就相当于是按照比例在适配，比如，界面上有一个 250 * 300 的正方形，那么它在 400 * 600 的屏幕上，应该显示的宽度是 ：

```
250 / 500 = x / 400
```

所以说，rpx 是自适应的，它在不同的屏幕上会有不同的大小。

其实，在Android中，这样的适配方式还是很常见的，我们拿一个举例：

```java
public static float (int unit, float value, DisplayMetrics metrics){
    switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
    }
    return 0;
}
```

Android中的控件都是使用的 dp，dp 会让所有设备的表现尽量一致，但是显然这就会引出很多问题，一个400dp的控件，在小屏手机就显示不下，所以使用比例适配是一种更加好的方案。

但是如果控件全部改成使用 px，那改动太大，然而我们分析系统源码，就可以发现使用dp也可以做到按照比例适配，其原因就是dp最终仍然被转成了px，所以我们需要做的就是干涉 dp 转 px 的这个过程。

上面的代码，已经将 dp 转 px 的过程贴出来了，我们只需要将 density 的值改一下就好了：

```java
public float density;
```

是 public 的，还不是 final 的，太好了！！！

比如，UI 出的图是按照 320 * 480 的，一个 32 * 32 的控件，那么在 1080 * 1920 的屏幕上就是 108 * 108，那么我们需要将 density 的值改成：

```
32 * density = 32 / 320 * 1080  => density = 1080 / 320
```

可以看出，其实就是让 density 维持了这个比例。



下面，给出一些常用的代码：

```dart
import 'dart:ui';

class SizeFit {
  // 1.基本信息
  static double physicalWidth;
  static double physicalHeight;
  static double screenWidth;
  static double screenHeight;
  static double dpr;
  static double statusHeight;

  static double rpx;
  static double px;

  static void initialize({double standardSize = 750}) {
    // 1.手机的物理分辨率
    physicalWidth = window.physicalSize.width;
    physicalHeight = window.physicalSize.height;

    // 2.获取dpr
    dpr = window.devicePixelRatio;

    // 3.宽度和高度
    screenWidth = physicalWidth / dpr;
    screenHeight = physicalHeight / dpr;

    // 4.状态栏高度
    statusHeight = window.padding.top / dpr;

    // 5.计算rpx的大小
    rpx = screenWidth / standardSize;
    px = screenWidth / standardSize * 2;
  }

  static double setRpx(double size) {
    return rpx * size;
  }

  static double setPx(double size) {
    return px * size;
  }
}
```

做适配的时候，可以使用 setPx 方法。setRpx用于 web 端。

当然，这样写很麻烦，可以使用新出的扩展函数语法，与 kotlin 的扩展函数一样。

