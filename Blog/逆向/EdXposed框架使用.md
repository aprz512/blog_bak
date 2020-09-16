---
title: EdXposed框架使用
date: 2020-9-8
categories: 逆向
---

### 环境搭建

#### 刷机

手机：Pixel

镜像：8.1.0 (OPM4.171019.021.P1, Jul 2018)，使用 10.0.0 失败了，EdXposed 不完善。

下载刷机包：[Link](https://dl.google.com/dl/android/aosp/sailfish-opm4.171019.021.p1-factory-0bcf4315.zip)

下载完成后，直接解压，运行 flash-all 即可（需要解锁 bootloader，并进入 bootloader 模式）。

进入 bootloader 模式，使用命令：

> adb reboot bootloader



#### twrp

刷机完成之后（如果之前root过的话，最好重新刷机一下，不然可能会无限重启...），再 root。

先安装 twrp，具体过程参考 https://twrp.me/google/googlepixel.html，**每个手机都不一样**。

**实践**：

下载 twrp，https://dl.twrp.me/sailfish/

然后进入 bootloader：

> **adb reboot bootloader**

![img](https://www.online-tech-tips.com/wp-content/uploads/2020/01/adb-reboot-bootloader.png.webp)

然后，进入 twrp，**将 twrp.img 改成你下载的文件的全路径，最好直接拖到cmd里面**

> fastboot boot twrp.img

![img](https://www.online-tech-tips.com/wp-content/uploads/2020/01/boot-twrp.png.webp)

然后，手机会出现如下画面，**如果手机有密码，记得在弹出的窗口输入密码，不然文件乱码**：

![img](https://www.online-tech-tips.com/wp-content/uploads/2020/01/install-zip-from-recovery-1.png.webp)



#### magisk

下载 magiskmanager，https://magiskmanager.com/downloading-magisk-manager

安装完成后，发现里面**一直检查更新**，点击**菜单->设置->更新通道->自定义**，输入：https://gitee.com/QingFeiDeiYi/Magisk/raw/master/stable.json 这个地址，然后返回刷新界面，就好了。

下载 magisk，https://github.com/topjohnwu/Magisk/releases

然后将文件 push 到手机，进入 twrp，安装，重启，即可，手机就 root 了。



#### EdXposed 安装

下载3个文件：

- Riru-Core – [Download](https://github.com/RikkaApps/Riru/releases)
- EDXposed Magisk Module – [Download](https://github.com/ElderDrivers/EdXposed/releases) (YAHFA and SandHook are two variants available – try both and adopt stable variant which is best for your device)
- EDXP Manager APK – [Download](https://github.com/ElderDrivers/EdXposedManager/releases)

将前面两个压缩文件 push 到手机上，打开  magisk manager，点击 菜单 模块 ，将上面两个 zip 文件刷入就好了。

安装 EDXP Manager APK，重启，这样 EdXposed 框架就安装好了，打开 EdXposed Manager 会提示你是否安装好了。








### EdXposed 使用

这个玩意的使用有点蛋疼，首先需要安装一个 app：https://github.com/PAGalaxyLab/VirtualHook

安装好之后，编写 hook 代码，例子：https://github.com/PAGalaxyLab/YAHFA

编写好 hook 代码之后，打包成 apk，然后 push 到手机 sdcard 根目录。

打开安装的 VirtualHook app，里面有个 **添加** 按钮，点击它，出现一个页面，这个页面有**两个 tab**，切换到 **APPS IN SDCARD** 这个 tab（第二个）。

然后，选择 hook apk，即可。

**测试的 Android 10 ，会报错**：https://github.com/PAGalaxyLab/YAHFA/issues/129

**测试的 Android 8，会报错**：https://github.com/PAGalaxyLab/VirtualHook/issues/100

暂时放弃，搞 Frida 去了...



### wifi 叉号问题

https://support.google.com/pixelphone/forum/AAAAb4-OgUsK_o4E4r67dY

直接运行

> adb shell "settings put global captive_portal_https_url https://developers.google.cn/generate_204"