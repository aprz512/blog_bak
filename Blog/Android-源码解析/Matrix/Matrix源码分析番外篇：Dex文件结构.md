---
title: Matrix源码分析番外篇：Dex文件结构
index_img: /cover/12.jpg
banner_img: /cover/top.jpg
date: 2020-8-12
categories: Matrix
---

### 构造DEX文件

首先，我们编写一个简单的程序，如下：

```java
public class HelloWorld {  
    int a = 0;  
    static String b = "HelloDalvik";  
  
    public int getNumber(int i, int j) {  
        int e = 3;  
        return e + i + j;  
    }  
  
    public static void main(String[] args) {  
        int c = 1;  
        int d = 2;  
        HelloWorld helloWorld = new HelloWorld();  
        String sayNumber = String.valueOf(helloWorld.getNumber(c, d));  
        System.out.println("HelloDex!" + sayNumber);  
    }  
}  
```

使用命令行编译成 dex 文件。不想使用命令的直接拖到 Android studio 里面，打个apk也行，不过后面的 dex 文件内容分析就对不上了。

拿到 dex 文件后，我们使用 010 editor 打开它，可以看到如下内容：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex1.png?raw=true)

下面的表格就是 dex 的大致结构。点开各个entry，里面又有很多东西，我们慢慢道来，其实这个与 class 文件结构很像，如果你读过 《深入理解Java虚拟机》就很容易上手。

### dex_header

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex2.png?raw=true)

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex3.png?raw=true)

1.  **magic[8]**；它代表dex中的文件标识，一般被称为魔数。是用来识别dex这种文件的，它可以判断当前的dex文件是否有效，可以看到它用了8个1字节的无符号数来表示，我们在010Editor中可以看到也就是“64 65 78 0A 30 33 35 00 ”这8个字节，这些字节都是用16进制表示的，用16进制表示的话，两个数代表一个字节（一个字节等于8位，一个16进制的数能表示4位）。这8个字节用ASCII码表转化一下可以转化为：dex 035。

2. **checksum**;  它是dex文件的校验和，通过它可以判断dex文件是否被损坏或者被篡改。它占用4个字节，也就是“5D 9D F9 59”。这里提醒一下，在010Editor中，其实可以分别识别我们在DexHeader中看到的这些字段的，你可以点一下这里：

   ![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex4.png?raw=true)

   你可以看到这个header列表展开了，其实我们分析下来就和它这个结构是一样的，你可以先看下，我们现在分析到了checksum中了，你可以看到后面对应的值是“59 F9 9D 5D”。咦？这好像和上面的字节不是一一对应的啊。对的，你可以发现它是反着写的。这是由于dex文件中采用的是**小字节序的编码方式**，也就是低位上存储的就是低字节内容，所以它们应该要反一下。

3. **signature[kSHA1DigestLen]**，signature字段用于检验dex文件，其实就是把整个dex文件用SHA-1签名得到的一个值。这里占用20个字节，你可以自己点010Editor看一看。

4. **fileSize**;表示整个文件的大小，占用4个字节。

5. **headerSize**;表示DexHeader头结构的大小，占用4个字节。这里可以看到它一共占用了112个字节，112对应的16进制数为70h，你可以选中头文件看看010Editor是不是真的占用了这么多：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex5.png?raw=true)

6. **endianTag**;代表 字节序标记，用于指定dex运行环境的cpu，预设值为0x12345678，对应在101Editor中为“78 56 34 12”（小字节序）。

7. 接下来两个分别是**linkSize**;和u4  **linkOff**;这两个字段，它们分别指定了链接段的大小和文件偏移，通常情况下它们都为0。linkSize为0的话表示静态链接。

8. 再下来就是**mapOff**字段了，它指定了DexMapList的文件偏移，这里我们先不过多介绍它，你可以看一下它的值为“14 04 00 00”，它其实对应的16进制数就是414h（别忘了小字节序），我们可以在414h的位置看一下它在哪里：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex6.png?raw=true)

​		其实就是dex文件最后一部分内容。关于这部分内容里面是什么，我们先不说，继续往下看。



9. **stringIdsSize** 和 **stringIdsOff**字段：这两个字段指定了dex文件中所有用到的字符串的个数和位置偏移，我们先看stringIdsSize，它的值为：“1C 00 00 00”，16进制的1C也就是十进制的28，也就是说我们这个dex文件中一共有28个字符串，然后stringIdsOff为：“70 00 00 00”，代表字符串的偏移位置为70h。
10. **typeIdsSize**和**typeIdsOff**。它们代表什么呢？它们代表的是类的类型的数量和位置偏移，也是都占4个字节。
11. 这下到了**protoIdsSize**和**protoIdsOff**了，它们代表的是dex文件中方法原型的个数和位置偏移。
12. **fieldIdsSize**和**fieldIdsOff**字段。这两个字段指向的是dex文件中字段名的信息。
13. **methodIdsSize**和**methodIdsOff**字段。这俩字段指明了方法所在的类、方法的声明以及方法名。
14. **classDefsSize**和**classDefsOff**字段。这两个字段指明的是dex文件中类的定义的相关信息。

下面，详细的解释一下上面 9-14的内容。

### dex_string_ids

这个里面描述的是字符串。

我们就先介绍一下DexStringId这个结构，图中从70h开始，

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex7.png?raw=true)

所有被选中的都是DexStringId这种数据结构的内容，DexStringId代表的是字符串的位置偏移，每个DexStringId占用4个字节，也就是说**它里面存的还不是真正的字符串，它们只是存储了真正字符串的偏移位置**（偏移位置从0开始算起）。

下面我们先分析几个看看：

取第一个“**B2 02 00 00**”，它代表的位置偏移是2B2h，我们先找到这个位置：   

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex8.png?raw=true)

可以发现我一共选中了10个字节，这10个字节就表示了一个字符串。下面我们看一下dex文件中的字符串是如何表示的。dex中的字符串采用了一种叫做**MUTF-8这样的编码**，它是经过传统的UTF-8编码修改的。在MTUF-8中，它的头部存放的是由uleb128编码的字符的个数。

也就是说在“08 3C 63 6C 69 6E 69 74 3E 00”这些字节中，**第一个08指定的是后面需要用到的编码的个数，也就是8个**，即“ 3C 63 6C 69 6E 69 74 3E”这8个，但是我们为什么一共选中了10个字节呢，**因为最后一个空字符“0”表示的是字符串的结尾**，字符个数没有把它算进去。下面我们来看看“ 3C 63 6C 69 6E 69 74 3E”这8个字符代表了什么字符串：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex9.png?raw=true)

（要说明的一点是，这里凑巧这几个uleb128编码的字符都用了1个字节，所以我们可以这样进行查询，uleb128编码标准用的是1~5个字节， 这里只是恰好都是一个字节）。也就是说上面的70h开始的第一个DexStringId指向的其实是字符串“<clinit>”（但是貌似我们的代码中没有用到这个字符串啊，先不用管，我们接着分析）。再看到这里：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex10.png?raw=true)

刚刚我们分析到“B2 02 00 00”所指向的真实字符串了，下面我们接着再分析一个，我们直接分析第三个，不分析第二个了。第三个为“**C4 02 00 00**”，对应的位置也就是2C4h，我们找到它：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex11.png?raw=true)

看这里，这就是2C4h的位置了。我们首先看第一个字符，它的值为0Bh，也就是十进制的11，也就是说接下来的11个字符代表了它的字符串，我们依旧是查看接下来11个字符代表的是什么，经过查询整理：  

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex12.png?raw=true)

上面就是“HelloDalvik”这个字符串，可以看看我们的代码，我们确实用了一个这样的字符串，bingo。

下面剩下的字符串就不分析了。其实直接使用 010 editor 会更直观一些。

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex13.png?raw=true)

我们这半天分析的stringIdsSize 和 stringIdsOff字段指向的位置就是上面那个箭头指向的位置，它们里面存储的是真实字符串的位置偏移，它们都存储在data区域。（先透露一下，后面我们要分析的几个也和stringIdsSize 与stringIdsOff字段类似，它们里面存储的基本都是位置偏移，并不是真正的数据，真正的数据都在data区域）

### dex_type_ids

这个里面描述的是类型（基本类型，类类型）。

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex14.png?raw=true)

typeIdsSize的值为9h，也就是我们dex文件中用到的类的类型一共有9个，位置偏移在E0h位置，下面我们找到这个位置

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex15.png?raw=true)

这里我们又得介绍一种数据结构了，因为这里的数据也是一种数据结构的数据组成的。

```c++
struct DexTypeId{
	u4 descriptorIdx;	/*指向DexStringId列表的索引*/
}
```

它里面只有一个数据descriptorIdx，它的值的内容是DexStringId列表的索引。

我们直接去 010 中的  dex_string_ids 中展开这个索引，就可以知道这个字符串是啥了。

先看第一个“05 00 00 00”，也就是05h，即十进位的5。然后我们在上面所有整理出的字符串看看5索引的是什么？翻上去可以看到是“I”。

### dex_proto_ids

这个里面描述的是方法签名。

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex16.png?raw=true)

protoIdsSize的值为十进制的7，说明有7个方法原型，然后位置偏移为104h，我们找到这个位置

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex17.png?raw=true)

下面又有新的数据结构了。

```c++
struct DexProtoId{
	u4 shortyIdx;			/*指向DexStringId列表的索引*/
	u4 returnTypeIdx;		/*指向DexTypeId列表的索引*/
	u4 parametersOff;		/*指向DexTypeList的位置偏移*/
}
```

这个数据结构由三个变量组成。

第一个shortyIdx它指向的是我们上面分析的DexStringId列表的索引，代表的是方法声明**字符串**。

第二个returnTypeIdx它指向的是 我们上边分析的DexTypeId列表的索引，代表的是方法返回类型**字符串**。

第三个parametersOff指向的是DexTypeList的位置索引，**这又是一个新的数据结构了**，先说一下这里面 存储的是方法的参数列表。可以看到这三个参数，有方法声明字符串，有返回类型，有方法的参数列表，这基本上就确定了我们一个方法的大体内容。

我们接着看看DexTypeList这个数据结构，看看参数列表是如何存储的。

```c++
struct DexTypeList{
	u4 size;				/*DexTypeItem的个数*/
	DexTypeItem list[1];	/*DexTypeItem结构*/
}
```

它有两个参数，其中第一个size说的是DexTypeItem的个数，那DexTypeItem又是啥咧？它又是一种数据结构。我们继续看看

```c++
struct DexTypeItem{
	u2 typeIdx;				/*指向DexTypeId列表的索引*/
}
```

就是一个指向DexTypeId列表的索引，也就是代表参数列表中某一个具体的参数的位置。

下面我们具体地分析一个类吧。

一个DexProtoId一共占用12个字节。所以，我们取前12个字节进行分析。“06 00 00 00，00 00 00 00，94 02 00 00”，这就是那12个字节了。

首先“06 00 00 00”代表的是shortyIdx，它的值是指向DexStringId列表的索引，我们找到DexStringId列表中第6个对应的值，也就是III，说明这个方法中声明字符串为三个int。

接着，“00 00 00 00”代表的是returnTypeIdx，它的值指向的是DexTypeId列表的索引，我们找到对应的值，也就是I，说明这个方法的返回值是int类型的。

最后，我们看“94 02 00 00”，它代表的是DexTypeList的位置偏移，它的值为294h，我们找到这个位置：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex18.png?raw=true)

这里是DexTypeList结构，首先看前4个字节，代表的是DexTypeItem的个数，“02 00 00 00 ”也就是2，说明接下来有2个DexTypeItem的数据，每个DexTypeItem占用2个字节，也就是两个都是“00 00”，它们的值是DexTypeId列表的索引，我们去找一下，发现0对应的是I，也就是说它的两个参数都是int型的。因此这个方法的声明我们也就确定了，也就是int(int,int)。

### dex_field_ids

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex19.png?raw=true)

fieldIdsSize为3h，说明共有3个字段。fieldIdsOff为158h，说明偏移为158h，我们继续看到158h这里：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex20.png?raw=true)

接下来的数据结构是DexFieldId，我们看下

```c++
struct DexFieldId{
	u2 classIdx;		/*类的类型，指向DexTypeId列表的索引*/
	u2 typeIdx;		/*字段类型，指向DexTypeId列表的索引*/
	u4 nameIdx;		/*字段名，指向DexStringId列表的索引*/
}
```

我们依旧是分析一下第一个字段，“01 00 ，00 00，13 00 00 00”，类的类型为DexTypeId列表的索引1，也就是HelloWorld。

字段的类型为DexTypeId列表中的索引0，也就是int。

字段名为DexStringId列表中的索引13h，即十进制的19，找一下，是a，也就是说我们这个字段就确认了，即int HelloWorld.a。这不就是我们在HelloWorld.java文件里定义的变量a嘛。



### dex_method_ids

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex21.png?raw=true)

methodIdsSize，为Ah，即十进制的10，说明共有10个方法。methodIdsOff，为170h，说明它们的位置偏移在170h。我们看到这里

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex22.png?raw=true)

请看DexMethodId

```c++
struct DexMethodId{
	u2 classIdx;		/*类的类型，指向DexTypeId列表的索引*/
	u2 protoIdx;		/*声明类型，指向DexProtoId列表的索引*/
	u4 nameIdx;		/*方法名，指向DexStringId列表的索引*/
}
```

我们直接分析一下第一个数据，“01 00, 04 00， 00 00 00 00”，

首先，classIdx，为1，对应DexTypeId列表的索引1，也就是HelloWorld；

其次，protoIdx，为4，对应DexProtoId列表中的索引4，也就是void()；

最后，nameIdx，为0，对应DexStringId列表中的索引0，也就是<clinit>。

因此，第一个数据就出来了，即void HelloWorld.\<clinit>() 。



### dex_class_defs

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex23.png?raw=true)

classDefsSize字段，为1，也就是只有一个类定义，classDefsOff，为1C0h，我们找到它的偏移位置。

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex24.png?raw=true)

接下来的数据结构是DexClassDef，请看

```c++
struct DexClassDef{
	u4 classIdx;		/*类的类型，指向DexTypeId列表的索引*/
	u4 accessFlags;		/*访问标志*/
	u4 superclassIdx;	/*父类类型，指向DexTypeId列表的索引*/
	u4 interfacesOff;	/*接口，指向DexTypeList的偏移*/
	u4 sourceFileIdx;	/*源文件名，指向DexStringId列表的索引*/
	u4 annotationsOff;	/*注解，指向DexAnnotationsDirectoryItem结构*/
	u4 classDataOff;	/*指向DexClassData结构的偏移*/
	u4 staticValuesOff;	/*指向DexEncodedArray结构的偏移*/
}
```

我们直接根据结构开始分析吧。

classIdx为1，对应DexTypeId列表的索引1，找到是HelloWorld，确实是我们源程序中的类的类型。

accessFlags为1，它是类的访问标志，对应的值是一个以ACC_开头的枚举值，1对应的是 ACC_PUBLIC，你可以在010Editor中看一下，说明我们的类是public的。

superclassIdx的值为3，找到DexTypeId列表中的索引3，对应的是java.lang.object，说明我们的类的父类类型是Object的。

interfaceOff指向的是DexTypeList结构，我们这里是0说明没有接口。如果有接口的话直接对应到DexTypeList，就和之前我们分析的一样了，这里不多解释，有兴趣的可以写一个有接口的类验证下。

再下来sourceFileIdx指向的是DexStringId列表的索引，代表源文件名，我们这里位4，找一下对应到了字符串"HelloWorld.java"，说明我们类程序的源文件名为HelloWorld.java。

annotationsOff字段指向注解目录接口，根据类型不同会有注解类、注解方法、注解字段与注解参数，我们这里的值为0，说明没有注解，这里也不过多解释，有兴趣可以自己试试。

接下来是classDataOff了，它指向的是DexClassData结构的位置偏移，DexClassData中存储的是类的数据部分，我们开始详细分析一下它，首先，还是先找到偏移位置3F8h：

![image](https://github.com/aprz512/pic4aprz512/blob/master/Blog/Android-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Matrix/dex25.png?raw=true)

我们看看DexClassData数据结构：

```c++
struct DexClassData{
	DexClassDataHeader	        header;			/*指定字段与方法的个数*/
	DexField* 			staticFields;		/*静态字段，DexField结构*/
	DexField*			instanceFields；	/*实例字段，DexField结构*/
	DexMethod*			directMethods;		/*直接方法，DexMethod结构*/
	DexMethod*			virtualMethods;		/*虚方法，DexMethod结构*/
}
```

```c++
struct DexClassDataHeader{
	u4 staticFieldsSize;	/*静态字段个数*/
	u4 instanceFieldsSize;	/*实例字段个数*/
	u4 directMethodsSize;	/*直接方法个数*/
	u4 virtualMethodsSize;  /*虚方法个数*/
}
 
struct DexField{
	u4 fieldIdx;		/*指向DexFieldId的索引*/
	u4 accessFlags;		/*访问标志*/
}
 
struct DexMethod{
	u4 methodIdx;		/*指向DexMethodId的索引*/
	u4 accessFlags;		/*访问标志*/
	u4 codeOff;		/*指向DexCode结构的偏移*/

```

注意，在这些结构中的u4不是指的占用4个字节，而是指它们是uleb128类型（占用1~5个字节）的数据。

这里面就不具体分析了，可以自己打开 010 就知道里面的字段是什么意思了。就是描述这个类的静态字段，实例字段，各种方法。字段的访问表示等等。在方法的描述中，有个 dexCode，里面描述的方法的指令集。



### 参考文档

[一篇文章带你搞懂DEX文件的结构](https://blog.csdn.net/sinat_18268881/article/details/55832757)

