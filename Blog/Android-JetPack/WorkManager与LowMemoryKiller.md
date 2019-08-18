---
title: WorkManager 与 LowMemoryKiller 理解
date: 2019-08-18 11：11：11
tags: Jetpack
---



今天在查看bugly的时候，发现了如下错误：

```
android.app.RemoteServiceException
Context.startForegroundService() did not then call Service.startForeground()
```

发现是由于WorkManager引起的，原因是由于我们刚刚引入了WorkManager，不想对原来的代码改动太大，所以只是将AlarmManager替换成了WorkManager。

但是我突然想到，既然已经使用了WorkManager，它能保证任务的执行，那为啥还要启动Service呢？不是多次一举吗！

现在我们来从头理一下，**为啥我们需要在Service里面启动线程？**

Android是基于linux内核的系统，但是它与其他基于linux内核的系统有一个不同之处，就是它没有“交换空间”。

交换空间的作用：**当 RAM 满了之后，而系统还需要额外的内存空间，系统会将内存中的相对不经常使用的内存页放入到硬盘上，腾出位置给正在运行的应用程序。**

取而代之的，它使用 OOM Killer 来管理内存。

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-JetPack/WorkManager/1_92pIQu01ijeZ08BulvDm3Q.png?raw=true)

OOM Killer 的目标是通过基于其“可见性状态”和消耗的内存量来杀死进程来释放内存。

ActivityManager 会给每个进程一个 oom_adj 值，这个值越大，表示该进程的优先级越低。比如，前台进程的优先级就是0。

```c++
# Define the oom_adj values for the classes of processes that can be
# killed by the kernel.  These are used in ActivityManagerService.
    setprop ro.FOREGROUND_APP_ADJ 0
    setprop ro.VISIBLE_APP_ADJ 1
    setprop ro.SECONDARY_SERVER_ADJ 2
    setprop ro.BACKUP_APP_ADJ 2
    setprop ro.HOME_APP_ADJ 4
    setprop ro.HIDDEN_APP_MIN_ADJ 7
    setprop ro.CONTENT_PROVIDER_ADJ 14
    setprop ro.EMPTY_APP_ADJ 15
```

Android 中，存在着一张内存阈值表，这张阈值表是可以在 init.rc 中进行配置的，合理配置这张表，对于小内存设备有非常重要的作用。我们来看 lowmemorykiller.c 中这张默认的阈值表：

```java
static int lowmem_adj[6] = {
    0,
    1,
    6,
    12,
};
static int lowmem_adj_size = 4;
static int lowmem_minfree[6] = {
    3 * 512,/* 6MB */
    2 * 1024,/* 8MB */
    4 * 1024,/* 16MB */
    16 * 1024,/* 64MB */
};
static int lowmem_minfree_size = 4;

```

这个表的意思是：

> 当系统的可用内存小于6MB时，警戒级数为0；
> 当系统可用内存小于8M而大于6M时，警戒级数为1；
> 当可用内存小于64M大于16MB时，警戒级数为12。

比如，当 LMK 被触发的时候，发现系统的内存只剩下了 7M，那么查找上面的表会发现，警戒级数为 1。那么 LMK 会开始遍历系统所有进程，过程如下：

```java
for (遍历所有进程) {
	if (p is 内核进程) {
		continue;
	}
	
	if (p.oom_score_adj < 警戒值) {
		continue;
	}

	if (有被选中的进程) {
		if (当前进程的 oom_score_adj 比选中进程的小) {
			continue;
		}
		if (当前进程的 oom_score_adj 等于选中进程的小 && 当前进程占用的内存小) {
			continue;
		}
		
	}

	选中当前进程
}
```

可以看出如下结论：

> 内核线程不参加这个杀进程的游戏
> 进程的 oom_score_adj 越大越危险
> 进程的 oom_score_adj 相同的情况下，占用内存越大越危险



Low memory killer的规则就是根据当前系统的可用内存多少来获取当前的警戒级数，如果进程的oom_adj大于警戒级数并且最大，进程将会被杀死（**具有相同omm_adj的进程，则杀死占用内存较多的**）。omm_adj越小，代表进程越重要。一些前台的进程，oom_adj会比较小，而后台的服务，omm_adj会比较大，所以当内存不足的时候，Low memory killer必然先杀掉的是后台服务而不是前台的进程。

所以，我们要使用 Service 的原因：

- 我们需要执行一个长时间运行的操作，所以需要一个比较低的 oom_adj 值 （服务进程比后台进程值小）
- 可以单独开启一个进程

但是随着Android版本的升级，使用 Service 会带来一些其他的问题：

1. 电量消耗

   开发人员可以在后台做任何他们想做的事情，没有任何限制。

   所以Google搞了一个Doze模式：

   > 简而言之 - 在用户关闭设备屏幕后，Doze 模式启动并禁用网络，同步，GPS，警报和wifi扫描。直到用户打开屏幕或连接到充电器。这是为了 - 减少执行不重要工作的应用程序的数量，并且这样做 - 节省了用户的电量

2. 使用限制

   从 API 26 开始，如果应用的 targetSdkVersion 在 26 以上，在后台进程里面调用 startService 方法会抛出 [IllegalStateException](https://developer.android.com/reference/java/lang/IllegalStateException.html)。

说了这么多，得出一个结论：苍天已死，黄天当立。



![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-JetPack/WorkManager/1_ISeFOxwzOKMzsoz3SMm9Nw.png?raw=true)

既然Service已经不再能够实现它的主要目的（在后台长时间的运行任务），所以最好就不要在使用它了。



## WorkManager ： Just because work should be easy to do.

> WorkManager可以简化开发人员的工作，它提供了一流的api。
>
> 它适用于即使应用程序不再位于前台也应运行的后台作业。
>
> 在可能的情况下，它使用JobScheduler或Firebase JobDispatcher来完成工作。
>
> 如果你的应用程序在前台，它甚至会尝试直接在你的进程中完成工作。



**WorkManger 的使用可以查看最后面的官方文档，讲的非常详细，这里并不介绍。**



WorkManger 的体系结构如下：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-JetPack/WorkManager/1_VkznGM_XrSK9kmOujJCV6w.png?raw=true)

可以看到，WorkManger 在 enqueue work 的时候，将 work 保存到了数据库中（使用 room），用于满足条件之后再执行。所以，如果遇到报数据库相关的错误，而你的项目又没有相关代码，记得检查这里。



# # 参考文档

[Services. The life with/without. And WorkManager.](<https://medium.com/google-developer-experts/services-the-life-with-without-and-worker-6933111d62a6>)

[Android low memory killer 机制](<https://www.wolfcstech.com/2015/10/04/lowmemorykiller/>)

[Schedule tasks with WorkManager](<https://developer.android.com/topic/libraries/architecture/workmanager>)

[Location all the time with WorkManager!!](<https://medium.com/@prithvibhola08/location-all-the-time-with-workmanager-8f8b58ae4bbc>)