---
layout: '../../layouts/MarkdownPost.astro'
title: 'JVM虚拟机之字节码指令集解析'
pubDate: 2023-03-13
description: 'Java 字节码对于虚拟机，就好像汇编语言对于计算机，属于基本执行指令'
author: 'Suwian'
cover:
    url: 'https://s2.loli.net/2023/03/13/fogTvyiu7kEzsLx.png'
    square: 'https://s2.loli.net/2023/03/13/fogTvyiu7kEzsLx.png'
    alt: 'cover'
tags: ["JVM"]
theme: 'light'
featured: true
---
## 概述

 **官方文档**： https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html



Java 虚拟机的指令由**一个字节长度**的、代表着某种特定操作含义的数字（称为操作码，Opcode)，以及跟随其后的**零至多个**代表此操作所需参数（称为操作数， Operand） 而构成。由于 Java 虚拟机采用面向操作数栈而不是寄存器的结构，所以大多数的指令都不包含操作数， 只有一个操作码。

由于限制了Java虚拟机操作码的长度为一个字节(0,255)，这意味着指令集的操作码总数不可能超过 256 条。

### 执行模型

如果不考虑异常处理的话，那么 Java 虚拟机的解释器可以使用下面这个伪代码当做最基本的执行模型来理解：

``` java
do{
自动计算PC寄存器的值加1
根据PC寄存器的指示位置，从字节码流中取出操作码；
if(字节码存在操作数)从字节码流中取出操作数；
执行操作码所定义的操作；
}while(字节码长度>0);
```

### 字节码与数据类型

在 Java 虚拟机的指令集中，大多数的指令都包含了其操作所对应的数据类型信息。例如，`iload` 指令用于从局部变量表中加载 int 型的数据到操作数栈中，而 `fload` 指令加载的则是 float 类型的数据。

对于大部分与数据类型相关的字节码指令，它们的操作码助记符中都有特殊的字符来表明专门为哪种数据类型服务：

- i 代表对 int 类型的数据操作

- l 代表 long

- s 代表 short

- b 代表 byte

- c 代表 char

- f 代表 float

- d 代表 double

也有一些指令的助记符中没有明确地指明操作类型的字母， 如 `arraylength` 指令，它没有代表数据类型的特殊字符，但操作数永远只能是一个数组类型的对象。

还有另外一些指令，如无条件跳转指令 `goto` 则是与数据类型无关的。

值得注意的是，大部分的指令都没有支持整数类型 byte、char 和 short，甚至没有任何指令支持 boolean 类型。编译器会在编译期或运行期将 byte 和 short 类型的数据带符号扩展（Sign-Extend）为相应的 int 类型数据，将 boolean 和 char 类型数据零位扩展（Zero-Extend）为相应的 int 类型数据。与之类似，在处理 boolean、byte、short 和 char 类型的数组时，也会转换为使用对应的 int 类型的字节码指令来处理。因此，大多数对于 boolean、byte、short 和 char 类型数据的操作， 实际上都是使用相应的 int 类型作为运算类型。

### 指令分析

由于完全介绍和学习这些指令需要花费大量时间。为了让大家能够更快地熟悉和了解这些基本指令，这里将的字节码指令集按用途大致分成 9 类。

- 加载与存储指令

- 算术指令

- 类型转换指令

- 对象的创建与访问指令

- 方法调用与返回指令

- 操作数栈管理指令

- 比较控制指令

- 异常处理指令

- 同步控制指令

(说在前面)在做值相关操作时。

- 一个指令，可以从局部变量表、常量池、堆中对象、方法调用、系统调用中等取得数据，这些数据（可能是值可能是对象的引用）被压入操作数栈。

- 一个指令，也可以从操作数栈中取出一到多个值（pop多次），完成赋值、加减乘除、方法传参、系统调用等等操作。

## 加载与存储指令

**作用**

加载和存储指令用于将数据从栈帧的局部变量表和操作数栈之间来回传递。

**常用指令**

- [局部变量压栈指令] 将一个局部变量加载到操作数栈：`xload`、xload\_\<n\> （其中 x 为i、1、f、d、 a，n 为0到3）
- [常量入栈指令] 将一个常量加载到操作数栈：`bipush`、`sipush`、`ldc`、`ldc_w`、`ldc2_w`、`aconst_null`、`iconst_m1`、iconst\_\<i\>、lconst\_\<l\>、fconst\_\<f\>、dconst\_\<d\> （其中 i 、l、f、d 的范围在不同的指令是不一样的，下节会详细解释）
- [出栈装入局部变量表指令] 将一个数值从操作数栈存储到局部变量表：`xstore`、`xstore_n`（其中 x 为 i、1、f、d、a，n 为 0 到 3)；`xastore`（其中 x 为 i、l、f、d、a、b、c、s）
- 扩充局部变量表的访问索引的指令：wide。

上面所列举的指令助记符中，有一部分是以尖括号结尾的（例如 iload\_\<n\> ）。这些指令助记符实际上代表了一组指令 （例如 iload\_\<n\>代表了`iload_0`、`1load_1`、`iload_2 ` 和  `iload_3`  这几个指令）。这几组指令都是某个带有一个操作数的通用指令（例如 iload）的特殊形式，**对于这若干组特殊指令来说，它们表面上没有操作数，不需要进行取操作数的动作，但操作数都隐含在指令中**。除此之外，它们的语义与原生的通用指令完全一致（例如 `iload_0` 的语义与操作数为 0 时的 `iload` 指令语义完全一致 ）。在尖括号之间的字母指定了指令隐含操作数的数据类型，\<n\> 代表非负的整数，\<i\> 代表是 int 类型数据，\<l\> 代表 long 类型，\<f\> 代表 float 类型，\<d\> 代表 double 类型。

值得注意的是，操作 byte、char、short 和 boolean 类型数据时，经常用 int 类型的指令来表示。

### 复习：再谈操作数栈与局部变量表

**操作数栈（Operand Stacks）**

我们知道，Java字节码是Java虚拟机所使用的指令集。因此，它与Java虚拟机基于栈的计算模型是密不可分的。在解释执行过程中，每当为Java方法分配栈桢时，Java虚拟机往往需要开辟一块额外的空间作为操作数栈，来存放计算的操作数以及返回结果。

具体来说便是：执行每一条指令之前，Java虚拟机要求该指令的操作数已被压入操作数栈中。在执行指令时，Java虚拟机会将该指令所需的操作数弹出，并且将指令的结果重新压入栈中。

![op-1.png|inline](https://s2.loli.net/2023/03/13/hruCIxMGfHLV8T6.png)

以加法指令 `iadd` 为例。假设在执行该指令前，栈顶的两个元素分别为 int 值 1 和 int 值 2，那么`iadd` 指令将弹出这两个 int，并将求得的和 int 值 3 压入栈中。

![op-2.png|inline](https://s2.loli.net/2023/03/13/G1X4LB5ocdixTvj.png)

由于`iadd` 指令只消耗栈顶的两个元素，因此，对于离栈顶距离为 2 的元素，即图中的问号，`iadd` 指令并不关心它是否存在，更加不会对其进行修改。



**局部变量表（Local Variables）**

Java 方法栈桢的另外一个重要组成部分则是局部变量表，字节码程序可以将计算的结果缓存在局部变量表之中。

实际上，Java虚拟机将局部变量区当成一个数组，依次存放this指针（仅非静态方法），所传入的参数，以及字节码中的局部变量。和操作数栈一样，long 类型以及 double 类型的值将占据两个 slot，其余类型仅占据一个 slot。

![lv.png|inline](https://s2.loli.net/2023/03/13/kW3CDT2ebNtEd4X.png)

``` java
public void foo(long l, float f) {
    {
        int i = e;
    }
    {
        String s = "Hello, World";
    }
}
```

![lv-2.png|inline](https://s2.loli.net/2023/03/13/uojtkEYgHOL43D1.png)

this 表示当前类的引用，long 类型的值占两个槽位，i 和 s变量由于分别在各自代码块中，没有共同的生命周期，所以占同一个槽位（即槽位复用）。

![bytecode-3.png](https://s2.loli.net/2023/03/13/zOfhaA4MVNUwYDS.png)

在栈帧中，与性能调优关系最为密切的部分就是局部变量表。局部变量表中的变量也是重要的垃圾回收根节点（GCRoots），只要被局部变量表中直接或间接引用的对象都不会被回收。

### 局部变量压栈指令

局部变量压栈指令将给定的局部变量表中的数据压入操作数栈。

这类指令大体可以分为：

- xload_\<n\>（ x 为 i、l、f、d、a，n为0到3）

- xload（x 为 i、l、f、d、a） 

说明：在这里，x的取值表示数据类型。



指令 xload_n 表示将第 n 个局部变量压入操作数栈。比如 `iload_1`、`fload_0`、`aload_0` 等指令。其中 `aload_0` 表示将局部变量表中的索引为 0 的一个对象引用压栈。

指令 `xload` 通过指定参数的形式，把局部变量压入操作数栈，当使用这个命令时，表示局部变量的数量可能超过了 4 个，比如指令`iload`、`fload` 等.

``` java
public void load(int num, Object obj, long count, boolean flag, short[] arr) {
    System.out.println(num);
    System.out.println(obj);
    System.out.println(count);
    System.out.println(flag);
    System.out.println(arr);
}
```

字节码执行过程：

![bytecode-1.png|inline](https://s2.loli.net/2023/03/13/o4N5eEqWK3lY8ng.png)

PS：其中操作数栈为了展示方便直接叠加了上去，正常（例如上例中的打印输出）会有入栈出栈的操作。

### 常量入栈指令

常量入栈指令的功能是将常数压入操作数栈，根据数据类型和入栈内容的不同，又可以分为 const 系列、push 系列 ldc 指令。

**指令 const 系列**

用于对特定的常量入栈，入栈的常量隐含在指令本身里。指令有：

- iconst_\<i\>（i 从 -1到5）：`iconst_m1` 将 -1 压入操作数栈，`iconst_x`  ( x 为到5）将 x 压入栈
-  lconst_\<l\>（l 从 0到1）：`lconst_0` 、`lconst_1` 分别将长整数 0 和 1 压入栈
-  fconst_\<f\> （f 从 0到2）：`fconst_0`、`fconst_1`、`fconst_2` 分别将浮点数0、1、2 压入栈
- dconst_\<d\> （d 从0到1）：`dconst_0` 和 `dconst_1` 分别将 double 型 0 和 1 压入栈
- aconst_null：`aconst_null` 将 null 了压入操作数栈

从指令的命名上不难找出规律，指令助记符的第一个字符总是喜欢表示数据类型，i 表示整数，l 表示长整数，f 表示浮点数，d 表示双精度浮点，习惯上用 a 表示对象引用。如果指令隐含操作的参数，会以下划线形式给出。



**指令 push 系列**

主要包括 `bipush` 和 `sipush`。它们的区别在于接收数据类型的不同，`bipush` 接收 8 位整数作为参数， `sipush` 接收 16 位整数，它们都将参数压入栈。



**指令 ldc 系列**

如果以上指令都不能满足需求，那么可以使用万能的 `ldc` 指令，它可以接收一个8位的参数。该参数指向常量池中的 int、 float 或者String 的索引，将指定的内容压入堆栈。

类似的还有`ldc_w`，它接收两个8位参数，能支持的索引范围大于 `ldc`。如果要压入的元素是 long 或者 double 类型的，则使用`ldc2_w`指令，使用方式都是类似的。

**总结如下**

| 类型                         | 常数指令           | 范围                          |
| ---------------------------- | ------------------ | ----------------------------- |
| int(boolean,byte,char,short) | iconst             | [-1, 5]                       |
|                              | bipush             | [-128, 127]                   |
|                              | sipush             | [-32768, 32767]               |
|                              | ldc、ldc_w、ldc2_w | any int value                 |
| long                         | lconst             | 0, 1                          |
|                              | ldc、ldc_w、ldc2_w | any long value                |
| float                        | fconst             | 0, 1, 2                       |
|                              | ldc、ldc_w、ldc2_w | any float value               |
| double                       | dconst             | 0, 1                          |
|                              | ldc、ldc_w、ldc2_w | any double value              |
| reference                    | aconst             | null                          |
|                              | ldc、ldc_w、ldc2_w | String literal, Class literal |

**示例如下**

![push-1.png|inline](https://s2.loli.net/2023/03/13/qypCJfzQSmhorjY.png)

![push-2.png](https://s2.loli.net/2023/03/13/ifl2rkqsxeYa5GL.png)

注意：常量入栈指令中的n和局部变量压栈指令中的 n 不一样，此处的 n 代表数值或者对象，而不是局部变量表中的下标

### 出栈入局部变量表指令

出栈装入局部变量表指令用于将操作数栈中栈顶元素弹出后，装入局部变量表的指定位置，用于给局部变量赋值。

这类指令主要以 `store` 的形式存在,比如 

- xstore（ x 为 i、l、f、d、a），指令 xstore 由于没有隐含参数信息，故需要提供一个 byte 类型的参数类指定目标局部变量表的位置。
- xstore_n （ x 为 i、l、f、d、a，n 为 0 至 3），例如：指令 `istore_n` 将从操作数栈中弹出一个整数，并把它贿值给局部变量索引为 n 的位置

一般说来，类似像 store 这样的命令需要带一个参数，用来指明将弹出的元素放在局部变量表的第几个位置。但是，为了尽可能压缩指令大小，使用专门的 `istore_1`指令表示将弹出的元素放置在局部变量表第 1 个位置。类似的还有 `istore_0`、`istore_2`、`istore_3`，它们分别表示从操作数栈顶弹出一个元素，存放在局部变量表第 0、2、3 个位置。

由于局部变量表前几个位置总是非常常用，因此这种做法虽然增加了指令数量，但是可以大大压缩生成的字节的体积 。如果局部变量表很大，需要存储的槽位大于3，那么可以使用 istore 指令，外加一个参数，用来表示需要存放的槽位位置。



**示例如下**

![bytecode-2.png|inline](https://s2.loli.net/2023/03/13/CWaMG8q5UrTj1Il.png)

解释说明（由于只是分析，没有调用这个方法，所以全部使用的变量名称来代替具体的值，真实的情况是存储实际的值）：

- 首先该方法被调用的时候，形式参数 k 和 d 都是有确定的值，由于该方法不是静态方法，所以局部变量表中的第一个位置（槽位）存储 this，而第二个位置存储 k 具体的值
- 然后第三个和第四个位置储存 d 具体的值，由于 d 是 double 类型，所以需要占据两个槽位
- 数据已经准备好了，那就来看字节码，首先 `iload_1`是将局部变量表中下标为 1 的 k 值取出来压入操作数栈中，然后`iconst_2`是将常量池中的整型值 2 压入操作数栈，`iadd`让操作数栈弹出的 k 值和整型值 2 执行相加操作，之后将相加的结果值 m 压入操作数栈中。（图中在执行弹栈和压栈操作之后，为了展示方便并没有删除操作数栈中的 k 值和 2，真正的操作是弹栈之后 k 值和 2 就会从操作数栈中弹出，之后操作数栈中就没有 k 值和 2 了，栈顶就是 m 值了）
- 然后 `istore_4`是将操作数栈中的 m 值弹出栈，然后放在局部变量表中下标为 4 的位置
- `idc2_w #13<12>`代表将 long 型值 12 压入操作数栈，`istore5` 是将值 12 弹栈之后放入局部变量表中下标为 5 的位置，由于 12 是 long 型，所以占据两个位置（槽位），`ldc #15\<atguigu\>`代表将字符串 "atguigu" 压入操作数栈，`astore 7`代表将字符串 "atguigu" 弹栈之后放入局部变量表中下标为 7 的位置
- `idc #16<10.0>`代表将 float 类型数据 10.0 压入操作数栈，`fstore 8`代表将 10.0 弹出栈，然后放入局部变量表中下标为 8 的位置
- `idc2_w #17<10.0>` 代表将 10.0 压入操作数栈，`dstore2` 代表将 10.0 弹出栈，之后将 10.0 放入下标为 2 和 3 的操作（double 类型数据占据两个槽位）



## 算术指令

**作用**

算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新压入操作数栈。



**分类**

大体上算术指令可以分为两种：对**整型数据**进行运算的指令与对**浮点类型数据**进行运算的指令。在每一大类中，都有针对 Java 虚拟机具体数据类型的专用算术指令，但没有直接支持 byte、short、char 和 boolean 类型的算术指令，对于这些数据的运算，都使用 int 类型的指令来处理。此外，在处理 boolean、byte、short 和 char 类型的数组时，也会转换为使用对应的 int 类型的字节码指令来处理。



**运算时溢出**

数据运算可能会导致溢出，例如两个很大的正整数相加，结果可能是一个负数。其实 Java 虚拟机规范并无明确规定过整型数据溢出的具体结果，仅规定了在处理整型数据时，只有除法指令以及求余指令中当出现除数为 0 时会导致虚拟机抛出异常 ArithmeticException。



**运算模式**

- **向最接近数舍入模式**：JVM 要求在进行浮点数计算时，所有的运算结果都必须舍入到适当的精度，非精确结果必须舍入为可被表示的最接近的精确值，如果有两种可表示的形式与该值一样接近，将优先选择最低有效位为零的；

- **向零舍入模式**：将浮点数转换为整数时，采用该模式，该模式将在目标数值类型中选择一个最接近但是不大于原值的数字作为最精确的舍入结果；



**NaN 值的出现**

当一个操作产生溢出时，将会使用有符号的无穷大表示，如果某个操作结果没有明确的数学定义的话，将会使用 NaN 值来表示。而且所有使用 NaN 值作为操作数的算术操作，结果都会返回 NaN，示例如下：

![NaN-1.png|inline](https://s2.loli.net/2023/03/14/kHFBcbivht1q4CE.png)

### 所有的算术指令

| 算数指令   | 备注         | int(boolean,byte,char,short) | long | float         | double        |
| ---------- | ------------ | ---------------------------- | ---- | ------------- | ------------- |
| 加法指令   |              | iadd                         | ladd | fadd          | dadd          |
| 减法指令   |              | isub                         | lsub | fsub          | dsub          |
| 乘法指令   |              | imul                         | lmul | fmul          | dmul          |
| 除法指令   |              | idiv                         | ldiv | fdiv          | ddiv          |
| 求余指令   | remainder    | irem                         | lrem | frem          | drem          |
| 取反指令   | negation     | ineg                         | lneg | fneg          | dneg          |
| 自增指令   |              | iinc                         |      |               |               |
| 位运算指令 | 按位或指令   | ior                          | lor  |               |               |
|            | 按位与指令   | iand                         | land |               |               |
|            | 按位异或指令 | ixor                         | lxor |               |               |
| 比较指令   |              |                              | lcmp | fcmpg / fcmpl | dcmpg / dcmpl |

**示例一**

``` java
public static int bar(int i) {
	return ((i + 1) - 2) * 3 / 4;
}
```

![ex-1.png|inline](https://s2.loli.net/2023/03/14/BfF8eOvU9ITcjul.png)



**示例二**

``` java
public void add() {
	byte i = 15;
	int j = 8;
	int k = i + j;
}
```

![ex-2.png|inline](https://s2.loli.net/2023/03/14/IiZesH57LjcX1CM.png)

![ex-3.png|inline](https://s2.loli.net/2023/03/14/TF9W8Kn14ukHr2g.png)

![ex-5.png|inline](https://s2.loli.net/2023/03/14/BWL2YgSFXH8Qjtz.png)



**示例三**

``` java
public static void main(String[] args) {
	int x = 500;
	int y = 100;
	int a = x / y;
	int b = 50;
	System.out.println(a + b);
}
```

![ex-6.png|inline](https://s2.loli.net/2023/03/14/tGmlVUfpIZ96bcv.png)

![ex-7.png|inline](https://s2.loli.net/2023/03/14/7Z2LA4btz5qEshW.png)

### 比较指令说明

比较指令的作用是比较栈顶两个元素的大小，并将比较结果入栈。比较指令有:dcmpg、dcmpl、fcmpg、fcmpl、lcmp，与前面讲解的指令类似，首字符 d 表示 double 类型，f 表示 float，l 表示 long



## 类型转换指令

## 对象的创建与访问指令

## 方法调用与返回指令

## 操作数栈管理指令

## 控制转移指令

## 异常处理指令

## 同步控制指令

