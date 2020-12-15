---
title:  Android 无障碍服务
date: 2019-08-18 11：11：11
categories: Android
---



由于某种原因，我们需要屏蔽某些界面上的无障碍服务的使用，刚开始我以为是做不到的。因为我们的思路出了问题，就直接想着要去停止系统的这个服务。但是经过一番搜索之后，没有发现任何结果。官方文档也说了：

> The lifecycle of an accessibility service is managed exclusively by the system and follows the established service life cycle.

无障碍服务是只由系统管理的。

然后我就又开始搜索其他方面的资料，比如：无障碍的原理与使用。巧合的是，我搜索到了这篇文章：

[随手记Android无障碍实践](https://juejin.im/post/5af95b46f265da0ba2671c16)

看着看着，我发现这是一篇关系如何优化自己的App，让应用可以被障碍人士使用的文章。当我失望的准备关掉页面的时候，突然我想到了，既然他能优化使用，那我是不是能做一个反优化，让它不能使用呢！！！

于是，我仔细的看了文章，果然找到了一个令我感兴趣的东西。

在它们改造非标准组件的选中状态的时候，是这样做的：

> 给控件添加无障碍代理（AccessibilityDelegate），在onInitializeAccessibilityNodeInfo()方法中调用AccessibilityNodeInfo对象的setChecked方法设置选中状态。

```java
rootView.setAccessibilityDelegate(new View.AccessibilityDelegate() {
    @Override
    public void onInitializeAccessibilityNodeInfo(View host, AccessibilityNodeInfo info) {
        super.onInitializeAccessibilityNodeInfo(host, info);
        info.setCheckable(true);
        info.setChecked(itemData.isSelected());
    }
});
```

看到没有，View 是可以设置一个关于无障碍的代理的。

既然可以设置代理，那么思路就很清晰了，一个好的代理，可以省很多事，但是我也可以搞一个吊都不吊你的代理。

在设置了一个代理之后，在用我们自己写的无障碍app测试的时候，果然无效了，而且自动化测试也失效了，效果还是可以的，对原来的逻辑也基本没有影响。



后来，我又找了一些关系 AccessibilityService 的文章，也看官方文档的东西，总算对这个东西有了一个整体的了解。贴一个比较有趣的图：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-思考/20160804232139269?raw=true)

这个图讲的是 AccessibilityEvent 事件产生后是如何发送到 AccessibilityService 的。

刚开始，我有点无法理解这个图，因为我对无障碍的理解，最初是从抢红包开始的，总觉得它是一个可以用来在别的App里面搞一些操作的东西。所以我总搞不明白为啥事件是从 App 里面传到 Service 里面，而不是 Service 传递到 App 里面，如果不是这样，它是怎么点别的 App 里面的东西的呢？？？

在我看完了官方文档，我才知道能够点击别的App的方法是在后来才加进去的，原本是没有这些操作的。也就是说 AccessibilityService 最初涉及出来是用来做一些辅助操作的，比如我们有一个 App，我们可以在这个 App 里面自己实现一个 AccessibilityService，当用户点击了某个按钮的时候，我们就可以提示用户你点击了啥按钮，你点击的按钮是啥颜色的等等...，这样的话事件的传递才是有用的，正确的。

后面加入的按钮点击操作，应该与 AccessibilityEvent  无关。

