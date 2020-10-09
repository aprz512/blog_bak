---
title: Frida环境搭建
date: 2020-9-9
categories: 逆向
---

### Anaconda

Anaconda 是一个用于科学计算的 Python 发行版，支持 Linux, Mac, Windows, 包含了众多流行的科学计算、数据分析的 Python 包。

可以立即为它就是Python，但是它集成了很多东西，用起来很方便。

https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/

根据自己的系统，下载完对应的安装包之后，安装就好了。

我的是 Windows，记得安装的时候**勾上添加环境变量**，不然还得自己加。

安装完毕输入：

```
conda list
```

查看安装结果。



### 依赖包

Anaconda 安装好了之后，再安装两个依赖包：

- frida
- frida-tools

```
 pip install frida 			// 这个贼慢
 pip install frida-tools
```

frida 默认安装的是最新的版本，你可以指定版本号。这里假设你的版本是 12.3.6，那么 python 必须要是 3.7 的。这是 Python 唯一蛋疼的地方，版本不兼容。

安装完成后输入：

```
python
```

进入python环境，导入个包试试：

```
import frida
```

如果没报错，就ok了。



### frida-server

下载地址：https://github.com/frida/frida/releases

选择 frida-server 开头的，然后选择你自己的环境，我做 Android 逆向，就选 Android。然后选择手机版本，我的手机是 arm64 的。

查看自己手机CPU类型：

```
adb shell getprop ro.product.cpu.abi
```

![img](https://img-blog.csdnimg.cn/20190424154339833.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzE3NDQx,size_16,color_FFFFFF,t_70)

我下载的是：[frida-server-12.11.12-android-arm64.xz](https://github.com/frida/frida/releases/download/12.11.12/frida-server-12.11.12-android-arm64.xz)

模拟器，一般选择 x86 的。

下载完server程序后，**解压缩**，然后推送到手机内部存储路径：

```
adb push frida-server-12.11.12-android-arm64 /data/local/tmp/fs
```

这里顺便做了一个重命名，改成了 fs。

然后，修改文件执行权限：

```
chmod 777 fs
```

然后，运行文件，记得授予shell root 权限，否则可能会报权限拒绝错误：

```
./fs
```

然后，做一个端口转发，默认使用27042端口与frida-server通信：

```
adb forward tcp:27042 tcp:27042
```

上面命令的意思：即把PC电脑端TCP端口27042的数据转发到与电脑通过adb连接的Android设备的TCP端口27042上。

