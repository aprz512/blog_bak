---
title: 003-vscode环境搭建
index_img: /cover/13.jpg
banner_img: /cover/top.jpg
date: 2021-6-13
categories: C++
---

> 距离上次写博客已经过了两个月了，第一个月在钓鱼，第二个月在适应新公司。
>
> 新公司确实比之前的要好，但是没有想象中的那么好，总之，还是朝着微软努力吧。
>
> 最近也升级了电脑的主板，由于操作的原因，导致了需要重装系统，所以各种环境还得再搭一遍。以前是按照别人的博客搭建的，现在自己按照官方文档搞了一下，发现超级简单。

### 下载 vscode

https://code.visualstudio.com/

### 安装c++插件

1. 打开 vscode
2. 点击左边的扩展按钮 （Ctrl+Shift+X）
3. 搜索 c++
4. 安装扩展

![](https://code.visualstudio.com/assets/docs/languages/cpp/search-cpp-extension.png)

这个插件安装之后，就可以支持 c/c++ 文件了。



### 安装编译器

我选择的是  mingw64，但是很遗憾的是，我的在线安装下载不下来，所以选择的是离线安装。

- 在线安装 [Mingw-w64 link](https://sourceforge.net/projects/mingw-w64/files/Toolchains targetting Win32/Personal Builds/mingw-builds/installer/mingw-w64-install.exe/download) 

- 离线安装 

  链接：[https://pan.baidu.com/s/13ukGn27JEseCpJ2PokzEPg](https://links.jianshu.com/go?to=https%3A%2F%2Fpan.baidu.com%2Fs%2F13ukGn27JEseCpJ2PokzEPg)
  提取码：x2ms

具体的一下选项，我是按照这个图(x86_64-8.1.0-posix-seh-rt_v6-rev0)：

![](https://code.visualstudio.com/assets/docs/languages/cpp/choose-x86-64.png)



### 编译器环境变量配置

这个没啥好说的，找到安装（解压）包的位置，在 path 里面添加一下就好了，添加完之后测试一下：

```shell
g++ --version
gdb --version
```



### 安装 code runner 插件

这个是为了在右上角有个点击运行的按钮，方便一些。安装完成之后，output 会是乱码，是因为我们的 cmd 是 gbk，而 coder runner 默认是 utf-8，所以改一下默认编码就好了，加一个配置项：

```
-fexec-charset=GBK
```

位置在下面的知乎链接里有。



### 参考文档

https://code.visualstudio.com/docs/languages/cpp

https://zhuanlan.zhihu.com/p/153252108

