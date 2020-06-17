---
title: Protobuf 通信协议详解
date: 2020-6-14
categories: 网络优化
---



## Protobuf 通信协议详解

在移动互联网时代，手机流量、电量是最为有限的资源，而移动端的即时通讯应用无疑必须得直面这两点。

解决流量过大的基本方法就是**使用高度压缩的通信协议**，而数据压缩后流量减小带来的自然结果也就是省电：因为大数据量的传输必然需要更久的网络操作、数据序列化及反序列化操作，这些都是电量消耗过快的根源。

当前即时通讯应用中最热门的通信协议无疑就是Google的Protobuf了。本文将详细介绍Protobuf的使用、原理等。

### protobuf 简介

github地址：https://github.com/protocolbuffers/protobuf

Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。目前提供了 C++、Java、Python 三种语言的 API。

看了这些介绍，可能你和我一样刚开始都是一脸懵逼，不过不要紧，我们写一个小例子来帮助我们理解。

### 一个简单的例子

#### 安装 Google Protocol Buffer

首先，我们去Github地址，里面有  Protobuf 的源码，你可以 clone 下来自己编译。当然，Google 也提供已经编译好了的程序供我们直接使用。这里就不说如何自己编译了，有兴趣的搜索一下。我们直接去 release 页面，下载对应平台的编译器，我的是  windows 系统，所以我选择下载 protoc-3.12.3-win64.zip 文件。

下载完成之后，打开，里面有如下文件：

```
-bin
-include
-readme.txt
```

bin 里面就是存放的编译器了，专门编译 .proto 文件，类似于 javac。

include 里面是 Google 提供的一些标准 proto 文件，你可以把它当作 sample 看待。

这里，我们只需要使用 bin 里面的 protoc.exe 程序。你可以把它放到一个特定的目录，然后添加到 path，因为之后需要在 terminal 中使用，或者你直接拷贝到你的工程目录下。

#### 关于例子的简单描述

这里我是用 java 来写这个简单的程序，程序分为两部份，一个 Writer，一个 Reader。

Writer 负责将一些结构化的数据写入一个磁盘文件，Reader 则负责从该磁盘文件中读取结构化数据并打印到屏幕上。

准备用于演示的结构化数据是 HelloWorld，它包含两个基本数据：

- ID，为一个整数类型的数据
- Str，这是一个字符串

#### 写 .proto 文件

```protobuf
syntax = "proto3"; //文件的第一行指定了你正在使用proto3语法：如果你没有指定这个，编译器会使用proto2。

option java_package = "com.aprz.proto"; // 生成的类的包名
option java_outer_classname = "HelloWorldProto"; // 生成的外部类的名字，该文件定义的消息体都是这个类的内部类

// 如果我们使用 java，这个消息体的类名就是：com.aprz.proto.HelloWorldProto.HelloWord
message HelloWorld {
    int32 number = 1; // proto3 不用写 required 修饰符了
    string text = 2;
}

message HelloWorld2 {
    float money = 1;
    string text = 2;
}
```

里面的 option 只针对 java 才生效。

在消息定义中，每个字段都有唯一的一个数字标识符。这些标识符是用来在消息的二进制格式中识别各个字段的，一旦开始使用就不能够再改变。注：[1,15]之内的标识号在编码的时候会占用一个字节。[16,2047]之内的标识号则占用2个字节。所以应该为那些频繁出现的消息元素保留 [1,15]之内的标识号。切记：要为将来有可能添加的、频繁出现的标识号预留一些标识号。

#### 编译 .proto 文件

这个就很简单了，会 javac 的肯定会这个。

假设您的 proto 文件存放在 $SRC_DIR 下面，您也想把生成的文件放在同一个目录下，则可以使用如下命令：

```
protoc -I=$SRC_DIR --java_out=$DST_DIR addressbook.proto

-I 后面是 proto 文件所在目录
--java_out 后面是生成的 java 文件存放的地址
最后是 proto 文件名称 
```

#### 引入依赖

执行编译命令之后，发现生成了 HelloWorldProto.java 文件，但是却有很多地方报错，这里因为该类里面引用到了很多 protobuf 库里面的类，所以我们需要在工程里面导入该依赖库。

这里，我导入的是 ：

```
com.google.protobuf:protobuf-java:3.12.2
```

导入完成后，发现类中没有了错误。有兴趣的可以自己看一下生成的消息类，由于生成的消息类的代码太多（一个600多行），所以这里就不贴代码了。

#### 编写 Writer 与 Reader

在编写 Writer 与 Reader 之前，我们先看看如何使用生成的消息体。

```java
HelloWorldProto.HelloWorld helloWorld = HelloWorldProto.HelloWorld.newBuilder()
    .setNumber(10)
    .setText("123")
    .build();
```

可以看到，protobuf 为我们定义的每一个消息体都生成了一个 builder 类，所以使用起来是很方便的。

编写 Writer：

```java
File dst = new File("protobuf-text");
try {
    BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(dst));
    helloWorld.writeTo(bos);
    bos.close();
} catch (IOException e) {
    e.printStackTrace();
}
```

生成的类提供了 writeTo 方法，可以直接写进输出流，当然你也可以自己做处理，里面有很多方法可以使用。

编写 Reader：

```java
File tar = new File("protobuf-text");
try {
    HelloWorldProto.HelloWorld helloWorld1 = HelloWorldProto.HelloWorld.parseFrom(new FileInputStream(tar));
    System.out.println(helloWorld1.getNumber());
    System.out.println(helloWorld1.getText());
} catch (IOException e) {
    e.printStackTrace();
}
```

直接从输入流中解析出 java 对象，也很方便。其实这就是一个反序列化的过程，将二进制流转成一个对象的字段。

```java
int tag = input.readTag();
switch (tag) {
    case 0:
        done = true;
        break;
    case 8: {

        number_ = input.readInt32();
        break;
    }
    case 18: {
        java.lang.String s = input.readStringRequireUtf8();

        text_ = s;
        break;
    }
    default: {
        if (!parseUnknownField(
            input, unknownFields, extensionRegistry, tag)) {
            done = true;
        }
        break;
    }
}
```

从这段代码应该可以看出一点东西，不过这里面涉及到 protobuf 序列化的规则，后面会介绍到。

这个例子本身并无意义，但只要您稍加修改就可以将它变成更加有用的程序。比如将磁盘替换为网络 socket，那么就可以实现基于网络的数据交换任务。而存储和交换正是 Protobuf 最有效的应用领域。

### Protobuf 的优点

Protobuf 有如 XML，不过它更小、更快、也更简单。你可以定义自己的数据结构，然后使用代码生成器生成的代码来读写这个数据结构。你甚至可以在无需重新部署程序的情况下更新数据结构。只需使用 Protobuf 对数据结构进行一次描述，即可利用各种不同语言或从各种不同数据流中对你的结构化数据轻松读写。

它有一个非常棒的特性，即“向后”兼容性好，人们不必破坏已部署的、依靠“老”数据格式的程序就可以对数据结构进行升级。这样您的程序就可以不必担心因为消息结构的改变而造成的大规模的代码重构或者迁移的问题。因为添加新的消息中的 field 并不会引起已经发布的程序的任何改变。

Protobuf 语义更清晰，无需类似 XML 解析器的东西（因为 Protobuf 编译器会将 .proto 文件编译生成对应的数据访问类以对 Protobuf 数据进行序列化、反序列化操作）。

使用 Protobuf 无需学习复杂的文档对象模型，Protobuf 的编程模式比较友好，简单易学，同时它拥有良好的文档和示例，对于喜欢简单事物的人们而言，Protobuf 比其他的技术更加有吸引力。

### Protobuf 的不足

Protbuf 与 XML 相比也有不足之处。它功能简单，无法用来表示复杂的概念。

XML 已经成为多种行业标准的编写工具，Protobuf 只是 Google 公司内部使用的工具，在通用性上还差很多。

由于文本并不适合用来描述数据结构，所以 Protobuf 也不适合用来对基于文本的标记文档（如 HTML）建模。另外，由于 XML 具有某种程度上的自解释性，它可以被人直接读取编辑，在这一点上 Protobuf 不行，它以二进制的方式存储，除非你有 .proto 定义，否则你没法直接读出 Protobuf 的任何内容。

### 高级应用

#### 导入

如果想要使用的消息类型已经在其他.proto文件中已经定义过了呢？
你可以通过导入（import）其他.proto文件中的定义来使用它们。要导入其他.proto文件的定义，你需要在你的文件中添加一个导入声明，如：

```
import "myproject/other_protos.proto";
```

#### 嵌套类型

你可以在其他消息类型中定义、使用消息类型，在下面的例子中，Result消息就定义在SearchResponse消息内，如：

```protobuf
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    // repeated 在一个格式良好的消息中，这种字段可以重复任意多次（包括0次）,一般对应数组或者List
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

### 细节问题

同 XML 相比， Protobuf 的主要优点在于性能高。它以高效的二进制方式存储，比 XML 小 3 到 10 倍，快 20 到 100 倍。

对于这样的说法，严格的程序员需要一个解释。因此在本文的最后，让我们稍微深入 Protobuf 的内部实现吧。

有两项技术保证了采用 Protobuf 的程序能获得相对于 XML 极大的性能提高。

第一点，我们可以考察 Protobuf 序列化后的信息内容。您可以看到 Protocol Buffer 信息的表示非常紧凑，这意味着消息的体积减少，自然需要更少的资源。比如网络上传输的字节数更少，需要的 IO 更少等，从而提高性能。

第二点我们需要理解 Protobuf 封解包的大致过程，从而理解为什么会比 XML 快很多。

#### Google Protocol Buffer 的 Encoding

Protobuf 序列化后所生成的二进制消息非常紧凑，这得益于 Protobuf 采用的非常巧妙的 Encoding 方法。

考察消息结构之前，让我首先要介绍一个叫做 Varint 的术语。

Varint 说的通俗一点，就是“**变长**”。它用一个或多个字节来表示一个数字，值越小的数字使用越少的字节数。这能减少用来表示数字的字节数。（那么问个问题，让你设计一个变长类型，你如何设计？？？）

比如对于 int32 类型的数字，一般需要 4 个 byte 来表示。但是采用 Varint，对于很小的 int32 类型的数字，则可以用 1 个 byte 来表示。当然凡事都有好的也有不好的一面，采用 Varint 表示法，大的数字则需要 5 个 byte 来表示。从统计的角度来说，一般不会所有的消息中的数字都是大数，因此大多数情况下，采用 Varint 后，可以用更少的字节数来表示数字信息。下面就详细介绍一下 Varint。

变长类型还是挺常见的，比如描述 dex 的结构里面，就会有很多类型都是变长的，为了节省空间。

Varint 中的**每个 byte 的最高位** bit 有特殊的含义，**如果该位为 1，表示后续的 byte 也是该数字的一部分，如果该位为 0，则结束**。其他的 7 个 bit 都用来表示数字。因此小于 128 的数字都可以用一个 byte 表示。大于 128 的数字，比如 300，会用两个字节来表示：

```
1010 1100 0000 0010
我们用计算器看一下 300 的二进制表示：‭
0001 0010 1100‬ -> 
0000010 0101100 -> 
00000010 10101100 -> 
10101100 00000010
```

![图 6. Varint 编码](https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/image006.jpg)



看这张图应该会比较好理解，它将300的二进制表示分成了两段（7 bit 为一段，不足的补0），由于Google Protocol Buffer 字节序采用 little-endian 的表示方式，所以需要将字节倒过来看。

消息经过序列化后会成为一个二进制数据流，该流中的数据为一系列的 Key-Value 对。如下图所示：

![图 7. Message Buffer](https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/image007.jpg)

采用这种 Key-Pair 结构**无需使用分隔符来分割不同的 Field**。对于可选的 Field，如果消息中不存在该 field，那么在最终的 Message Buffer 中就没有该 field，这些特性都有助于节约消息本身的大小。

假设我们的消息体中有两个字段：

```protobuf
message HelloWorld {
    int32 number = 1;
    string text = 2;
}
```

那么 number 的 key 为：

```java
(field_number << 3) | wire_type
```

field_number 就是我们定义的数字标识符了，这里 number 的为 1。 那么 wire_type 是什么呢？wire_type 需要查表才能得到：

| **Type** | **Meaning**   | **Used For**                                             |
| -------- | ------------- | -------------------------------------------------------- |
| 0        | Varint        | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1        | 64-bit        | fixed64, sfixed64, double                                |
| 2        | Length-delimi | string, bytes, embedded messages, packed repeated fields |
| 3        | Start group   | Groups (deprecated)                                      |
| 4        | End group     | Groups (deprecated)                                      |
| 5        | 32-bit        | fixed32, sfixed32, float                                 |

可以看到 int32 的 wire_type 为 0 。

所以 number 的 key 为 (1 << 3 | 0) = 8，text 的 key 为 （2 << 3 | 2） = 18，这两个数是不是有点熟悉，是的！就是上面生成的代码里面，我们见过的。那部分代码其实就是反序列话，根据 key 读取 value，然后给创建的对象赋值。

看到这里，你是不是觉得变长类型就已经很吊了呢？那么我有一个问题，表示正数 1 可以只需要一个字节，那么表示负数 -1，需要几个字节呢？

我们看一下 -1 的二进制代码：

```
‭11111111 11111111 11111111 11111111
```

一共有32位，这显然没法用一个字节表示，需要 5 个字节，那怎么办呢？

为此 Google Protocol Buffer 定义了 sint32 这种类型，采用 zigzag 编码。

Zigzag 编码用无符号数来表示有符号数字，正数和负数交错，这就是 zigzag 这个词的含义了。

![图 8. ZigZag 编码](https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/image008.jpg)

使用 zigzag 编码，绝对值小的数字，无论正负都可以采用较少的 byte 来表示，充分利用了 Varint 这种技术。

我们运行程序来看一下是不是这样吧：

```java
HelloWorldProto.HelloWorld helloWorld = HelloWorldProto.HelloWorld.newBuilder()
    .setNumber(10)
    .setText("1123").build();
for (byte b : helloWorld.toByteArray()) {
    System.out.println(b + " = " + Integer.toBinaryString((b & 0xFF) + 0x100).substring(1));
}
```

看一下输出的结果：

```
8 = 00001000 // number 的 key
10 = 00001010 // number 的 value
18 = 00010010 // text 的 key
4 = 00000100  // text 的 value 的长度
49 = 00110001 // 1
49 = 00110001 // 1
50 = 00110010 // 2
51 = 00110011 // 3
```

这里需要知道的一点是**：字符串等则采用类似数据库中的 varchar 的表示方法**，即用一个 varint 表示长度，然后将其余部分紧跟在这个长度部分之后即可。

text 的 value 是 4 个长度，每个字节可以查 ascii 码得到。这里的编码不太清楚，可能是 utf8。

我们设置一个负数来看看结果：

```java
HelloWorldProto.HelloWorld helloWorld = HelloWorldProto.HelloWorld.newBuilder()
    .setNumber(-1)
    .setText("李").build();
for (byte b : helloWorld.toByteArray()) {
    System.out.println(b + " = " + Integer.toBinaryString((b & 0xFF) + 0x100).substring(1));
}
```

输出结果：

```
8 = 00001000
-1 = 11111111
-1 = 11111111
-1 = 11111111
-1 = 11111111
-1 = 11111111
-1 = 11111111
-1 = 11111111
-1 = 11111111
-1 = 11111111
1 = 00000001
18 = 00010010
3 = 00000011
-26 = 11100110
-99 = 10011101
-114 = 10001110
```

可以看到，-1 的表示非常的长，用了 10 个字节。那么问题就来了，我们不是声明的 int32 吗，为啥实际上是 int64，还有王法吗，还有法律吗！！！

我查了些资料：

https://developers.google.com/protocol-buffers/docs/encoding

>  If you use int32 or int64 as the type for a negative number, the resulting varint is always ten bytes long – it is, effectively, treated like a very large unsigned integer.

这个我是没想到为啥要这样搞！！！

我们改一下 proto 中消息体的定义，使用 sint32 重新生成一下。

```
===== Byte 开始 =====
8 = 00001000
1 = 00000001
18 = 00010010
1 = 00000001
49 = 00110001
===== Byte 结束 =====
```

看到，生成的二进制是短了很多。

那么为啥这个就能生成的短些呢？难道 number 字段在源代码里面不是 int 吗？我们可以分析源码：

```java
private int number_; // 是 int 类型没变

// 但是它序列化与反序列化的时候使用的是不同的方法
output.writeSInt32(1, number_);
number_ = input.readSInt32();
```

更细节的部分，可以自行查看。

### 封解包的速度

首先我们来了解一下 XML 的封解包过程。XML 需要从文件中读取出字符串，再转换为 XML 文档对象结构模型。之后，再从 XML 文档对象结构模型中读取指定节点的字符串，最后再将这个字符串转换成指定类型的变量。这个过程非常复杂，其中将 XML 文件转换为文档对象结构模型的过程通常需要完成词法文法分析等大量消耗 CPU 的复杂计算。

反观 Protobuf，它只需要简单地将一个二进制序列，按照指定的格式读取到 Java 对应的结构类型中就可以了。从上一节的描述可以看到消息的 decoding 过程也可以通过几个位移操作组成的表达式计算即可完成。速度非常快。

述可以看到消息的 decoding 过程也可以通过几个位移操作组成的表达式计算即可完成。速度非常快。

为了说明这并不是我拍脑袋随意想出来的说法，下面让我们简单分析一下 Protobuf 解包的代码流程吧。

以代码清单 3 中的 Reader 为例，该程序首先调用 msg1 的 ParseFromIstream 方法，这个方法解析从文件读入的二进制数据流，并将解析出来的数据赋予 helloworld 类的相应数据成员。

该过程可以用下图表示：

![图 9. 解包流程图](https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/image009.jpg)

整个解析过程需要 Protobuf 本身的框架代码和由 Protobuf 编译器生成的代码共同完成。Protobuf 提供了基类 Message 以及 Message_lite 作为通用的 Framework，，CodedInputStream 类，WireFormatLite 类等提供了对二进制数据的 decode 功能，从 5.1 节的分析来看，Protobuf 的解码可以通过几个简单的数学运算完成，无需复杂的词法语法分析，因此 ReadTag() 等方法都非常快。 在这个调用路径上的其他类和方法都非常简单，感兴趣的读者可以自行阅读。 相对于 XML 的解析过程，以上的流程图实在是非常简单吧？这也就是 Protobuf 效率高的第二个原因了。

### 编解码性能测试

建议看看这篇文章，有从各个方面对比。

得出的结论是，protobuf **在整形与浮点型** 编解码方面具有很大的优势。但是在**字符串与对象**的编解码方面性能不如 [DSL-JSON](https://github.com/ngs-doo/dsl-json)。

http://www.52im.net/thread-772-1-1.html

### 参考文档

http://blog.csdn.net/u011518120/article/details/54604615

https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/index.html

https://developers.google.com/protocol-buffers/docs/overview

https://www.jianshu.com/p/bb3ac7e5834e