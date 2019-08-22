---
title: Android V1 V2 签名机制
date: 2019-08-18 11：11：11
tags: Android-高级
---



> 很早之前就想写这个，直到现在才有时间。理解了这个发现对 HTTPS 也有了进一步的理解。



## 为什么需要签名 ？

了解 HTTPS 通信的同学应该知道，在消息通信时，必须至少解决两个问题：

- 一是确保消息来源的真实性

- 二是确保消息不会被第三方篡改

我们先来看 HTTPS 签名以及校验的过程：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/%E7%AD%BE%E5%90%8D.png?raw=true)

这里只是简单的理了一下核心思路，有不懂的还是应该查看相关文献。

在安装 APK 时，同样需要确保 APK 来源的真实性，以及 APK 没有被第三方篡改。如何解决这两个问题呢？方法就是开发者对 APK 进行签名：在 APK 中写入一个“指纹”。指纹写入以后，APK 中有任何修改，都会导致这个指纹无效，Android 系统在安装 APK 进行签名校验时就会不通过，从而保证了安全性。



## V1 签名过程

首先我们任意选取一个签名后的 APK（Sample-release.APK）解压：

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/02.png?raw=true)

在 `META-INF` 文件夹下有三个文件：`MANIFEST.MF`、`CERT.SF`、`CERT.RSA`。它们就是签名过程中生成的文件，它们就是签名三贱客。



###MANIFEST.MF

该文件中保存的内容其实就是**逐一遍历 APK 中的所有条目**，如果是目录就跳过，如果是一个文件，就用 SHA1（或者 SHA256）消息摘要算法**提取出该文件的摘要然后进行 BASE64 编码**后，作为“SHA1-Digest”属性的值写入到 MANIFEST.MF 文件中的一个块中。该块有一个“Name”属性， 其值就是该文件在 APK 包中的路径。

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/03.png?raw=true)

在这个文件里面，我们也可以搜索到我们的dex文件的摘要，资源的摘要，有兴趣的可以自己动手试试，将apk拖到AS里面就搞定了。需要注意的是，**这个文件中存放了未压缩之前的所有文件的摘要**。



### CERT.SF

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/04.png?raw=true)

发现这里里面的内容与 MANIFEST.MF 的文件差不多。MANIFEST.MF是对APK中的每个文件进行摘要，那么这个文件里面的条目是什么东西的摘要呢？

- SHA1-Digest-Manifest：对整个 MANIFEST.MF 文件做 SHA1（或者 SHA256）后再用 Base64 编码
- SHA1-Digest：对 MANIFEST.MF 的各个条目做 SHA1（或者 SHA256）后再用 Base64 编码

所以，CERT.SF 做了这些东西：

1. 计算这个MANIFEST.MF文件的整体SHA1值，再经过BASE64编码后，记录在CERT.SF主属性块（在文件头上）的“SHA1-Digest-Manifest”属性值值下
2. 逐条计算MANIFEST.MF文件中每一个块的SHA1，并经过BASE64编码后，记录在CERT.SF中的同名块中，属性的名字是“SHA1-Digest



### CERT.RSA

这里会把之前生成的 CERT.SF 文件，用私钥计算出签名, 然后将签名以及包含公钥信息的数字证书一同写入 CERT.RSA 中保存。这里要注意的是，Android APK 中的 CERT.RSA 证书是自签名的，并不需要这个证书是第三方权威机构发布或者认证的，用户可以在本地机器自行生成这个自签名证书。Android 目前不对应用证书进行 CA 认证。

我们在 gradle 文件中配置的签名文件：

```groovy
    signingConfigs {
        release {
            storeFile file('..\\release.jks')
            storePassword 'release'
            keyAlias = 'key0'
            keyPassword 'release'
        }
    }
```

.jks 文件里面就包含了这些东西（通过 keytool 命令可以查看，但是看不到私钥）：

- 私钥
- 证书

这里会把之前生成的 CERT.SF文件， 用私钥计算出签名, 然后将签名以及包含公钥信息的数字证书一同写入  CERT.RSA  中保存。

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/09.png?raw=true)



### 签名校验

签名验证是发生在APK的安装过程中，一共分为三步：

- 检查 APK 中包含的所有文件，对应的摘要值与 MANIFEST.MF 文件中记录的值一致。

- 使用证书文件（RSA 文件）检验签名文件（SF 文件）没有被修改过。

- 使用签名文件（SF 文件）检验 MF 文件没有被修改过。

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/10.png?raw=true)



由于使用了自签名证书，没有CA的参与，所以公钥有被替换的可能，但是由于是第三方重新签名，所以无法覆盖已安装的应用。



## 基于V1的多渠道打包方案

最早的多渠道打包方案是这样的，由于以前都是使用的友盟统计，按照友盟官方文档说明，渠道信息通常需要在AndroidManifest.xml中配置如下值：

```xml
<meta-data android:value="Channel ID" android:name="UMENG_CHANNEL"/>
```

然后，在build.gradle设置productFlavors：

```groovy
android {  
    productFlavors {
        kuan {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "kuan"]
        }
        xiaomi {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "xiaomi"]
        }
        qh360 {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "qh360"]
        }
        baidu {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "baidu"]
        }
        wandoujia {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "wandoujia"]
        }
    }  
}
```

这样打包虽然可以工作，但是只是为了替换一个 AndroidManifest.xml 里面的 meta-data 就需要将所有的 apk 文件重新打进一个新包里面，非常的浪费时间。

那么，有没有快速打包方法呢？显然是有的，下面介绍一下美团的打包方案。



### 美团V1签名打包方案

我们上面分析过APK签名的校验，但是仔细想想，它有个漏洞，它校验了所有的 APK 里面的文件，以及签名3剑客，但是却没有对 MATE-INF 这个文件夹做校验。那么我们就可以这样做：

> 在 META-INF 目录下添加空文件，用空文件的名称来作为渠道的唯一标识。

这样我们的渠道信息就写入apk中的，而且不会影响签名。然后在app运行的时候，从 apk 文件里面读取出来就好了。

具体过程如下：

```python
# 创建渠道名的空文件
f_empty_channel = open(channel_name, 'w')
f_empty_channel.close()
    
# 往渠道apk中添加空的渠道文件
dest_channel_path = "./META-INF/" + channel_name
f = zipfile.ZipFile(dest_apk, 'a')
f.write(channel_name, dest_channel_path)
f.close()
```

这样就搞定了，是不是很简单呢？这种方式的特点是：生成一个渠道包，需要经过解压缩、创建空文件、压缩这些步骤。

### 一种更快速的打包

继美团多渠道打包方案之后，万能的网友又想出了一种更快速的打包方式。

由于apk文件实质上就是个zip包，因此可以利用zip包的文件结构，将渠道信息带进去即可。这种方式的特点：没有解压缩、压缩、重签名等步骤，比美团的打包效率还要高。

有兴趣的可以找找代码看看。



## V2签名过程

APK 签名方案 v2 是一种全文件签名方案，该方案能够发现对 APK 的受保护部分进行的所有更改，从而有助于加快验证速度并增强完整性保证。

从 Android 7.0 开始，Android 支持了一套全新的 V2 签名机制，为什么要推出新的签名机制呢？通过前面的分析，可以发现 v1 签名有两个地方可以改进：

- 签名校验速度慢
  校验过程中需要对apk中所有文件进行摘要计算，在 APK 资源很多、性能较差的机器上签名校验会花费较长时间，导致安装速度慢。

- 完整性保障不够
  META-INF 目录用来存放签名，自然此目录本身是不计入签名校验过程的，可以随意在这个目录中添加文件，比如一些快速批量打包方案就选择在这个目录中添加渠道文件。

为了解决这两个问题，在 Android 7.0 Nougat 中引入了全新的 APK Signature Scheme v2。



### V2 带来的影响

由于在 v1 仅针对单个 ZIP 条目进行验证，因此，在 APK 签署后可进行许多修改 — 可以移动甚至重新压缩文件。事实上，编译过程中要用到的 ZIPalign 工具就是这么做的，它用于根据正确的字节限制调整 ZIP 条目，以改进运行时性能。而且我们也可以利用这个东西，在打包之后修改 META-INF 目录下面的内容，或者修改 ZIP 的注释来实现多渠道的打包，在 v1 签名中都可以校验通过。

v2 签名将验证归档中的所有字节，而不是单个 ZIP 条目，因此，在签署后无法再运行 ZIPalign（必须在签名之前执行）。正因如此，现在，在编译过程中，Google 将压缩、调整和签署合并成一步完成。



### V2签名过程

v2 签名模式在原先 APK 块中增加了一个新的块（签名块），新的块存储了签名，摘要，签名算法，证书链，额外属性等信息，这个块有特定的格式，具体格式分析见后文，先看下现在 APK 成什么样子了。

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/11.png?raw=true)

为了保护 APK 内容，整个 APK（ZIP文件格式）被分为以下 4 个区块：

- ZIP 条目的内容（从偏移量 0 处开始一直到“APK 签名分块”的起始位置）
- APK 签名分块
- ZIP 中央目录
- ZIP 中央目录结尾

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/12.png?raw=true)

其中，应用签名方案的签名信息会被保存在 区块 2（APK Signing Block）中，而区块 1（Contents of ZIP entries）、区块 3（ZIP Central Directory）、区块 4（ZIP End of Central Directory）是受保护的，在签名后任何对区块 1、3、4 的修改都逃不过新的应用签名方案的检查。



### ZIP 文件结构

需要了解一下，不然不明白 ZIP 中央目录子类的东西。

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/14.png?raw=true)

zip文件分为3部分：

1. **数据区**

   此区块包含了zip中所有文件的记录，是一个列表，每条记录包含：文件名、压缩前后size、压缩后的数据等；

2. **中央目录**

   存放目录信息，也是一个列表，每条记录包含：文件名、压缩前后size、本地文件头的起始偏移量等。通过本地文件头的起始偏移量即可找到压缩后的数据；

3. **中央目录结尾记录**

   标识中央目录结尾，包含：中央目录条目数、size、起始偏移量、zip文件注释内容等。



继续回到正题。



### V2 签名摘要计算

![](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E9%AB%98%E7%BA%A7/%E7%AD%BE%E5%90%8D/13.png?raw=true)

说一下摘要计算规则：

- 将每个部分拆分成多个大小为 1 MB大小的chunk，最后一个chunk可能小于1M。之所以分块，是为了可以通过并行计算摘要以加快计算速度；
- 计算chunk摘要：字节 `0xa5` + 块的长度（字节数） + 块的内容 进行计算；
- 计算整体摘要：字节 `0x5a` + chunk数 + 块的摘要的连接（按块在 APK 中的顺序）进行计算。

最后，将 APK 的摘要 + 数字证书 + 其他属性生成签名数据写入到 APK Signing Block 区块。



### V2 签名多渠道打包方案

这里就不细说 APK Signing Block 区块里面的结构了，有兴趣的可以查查资料。

V2 签名这种方案，只保证了第1、3、4部分和第 2 部分（APK签名分块）包含的APK 签名方案 v2分块中的 `signed data` 分块的完整性。

APK签名分块包含了4部分：分块长度、ID-VALUE序列、分块长度、固定magic值。其中`APK 签名方案 v2分块`存放在ID为0x7109871a的键值对中。

所以，我们可以定义一个新的ID-VALUE，将渠道信息写入`APK签名分块`中。



## V2 V1 签名校验

2 签名机制是在 Android 7.0 以及以上版本才支持。因此对于 Android 7.0 以及以上版本，在安装过程中，如果发现有 v2 签名块，则必须走 v2 签名机制，不能绕过。否则降级走 v1 签名机制。

v1 和 v2 签名机制是可以同时存在的，其中对于 v1 和 v2 版本同时存在的时候，v1 版本的 META_INF 的 .SF 文件属性当中有一个 X-Android-APK-Signed 属性：

```
X-Android-APK-Signed: 2
```

因此如果想绕过 v2 走 v1 校验是不行的。



下一篇讲 V2 机制下的多渠道打包。