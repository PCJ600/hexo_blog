---
layout: next
title: ARMv8 A64汇编中立即数范围问题分析
date: 2021-11-21 20:21:19
categories:
- Assembly
tags:
- Assembly
- troubleshooting
---

## 问题描述
**ARMv8 A64汇编中，立即数是如何表示的？不同的指令对于立即数的表示有差异吗？**

在Stackoverflow上发现类似的讨论：[https://stackoverflow.com/questions/30904718/range-of-immediate-values-in-armv8-a64-assembly](https://stackoverflow.com/questions/30904718/range-of-immediate-values-in-armv8-a64-assembly)

<!-- more -->

问题复现：（环境： `Linux debian 4.19.0-10-amd64`)

1、编写 `hello.s`

```assembly
ADD X12, X10, 0xFEF
AND X12, X10, 0xFEF
```

如果你是`x86_64`环境，需先安装`aarch64`编译工具：

```shell
apt-get install binutils-aarch64-linux-gnu
```

2、编译`hello.s`，发现对于相同立即数`0xFEF`，<font color='red'>AND指令编译出`immediate out of range`错，但ADD指令编译却不出错，**为什么？？?**</font>

```
# aarch64-linux-gnu-as hello.s -o hello.o
hello.s: Assembler messages:
hello.s:1: Error: immediate out of range at operand 3 -- `and x12,x10,0xfef'
```



## 原因分析

查询ARMv8指令集手册(`C4.1 A64 instruction set encoding`)，先来看下A64 指令集编码格式：
![](image1.png)


可以看出**A64指令长度固定为32位，其中op0(占4位)用于区分不同类型的指令，剩下的28位也不可能全用于表示立即数**。因此，和x86/x64这种变长指令集对比，A64中立即数表示范围非常有限。



解释这个问题，需要查询指令集手册，对比ADD, AND指令的立即数编码格式差异。

### ADD(immediate)

ADD指令的编码格式和伪码，可以参考手册 `C6.2.4 ADD(immediate)`，或下图：
![](image2.png)


* `imm12`表示立即数，长度12位，范围0~4095，共4096种取值。

* 读伪代码可知，`sh`取值为1时，将`imm12`左移12位得到立即数。因此又有0 << 12, 1 << 12, 2 << 12, ... 4095 << 12 共4096种取值。前后加起来一共8192种取值。
* 此例中<font color = 'red'>**0xFEF = 4079，显然落在0~4095范围内，所以ADD指令编译不出问题。**</font>

利用`objdump`查看`ADD X12, X10, 0xFEF` 指令的二进制码：

```
# aarch64-linux-gnu-objdump -d hello.o
Disassembly of section .text:
0000000000000000 <.text>:
   0:   913fbd4c        add     x12, x10, #0xfef
```

 `0x913fbd4c` = `1001 0001 0011 1111 1011 1101 0100 1100` b， 可以看出这条ADD指令的编码和指令集手册规定的编码格式是一致的。（各字段对应的关系参考上图）



### AND(immediate)

AND指令的编码格式和伪码参考手册 `C6.2.12 AND(immediate)`，或下图：
![](image3.png)
![](image4.png)

AND指令的伪码不是很直观，详细解释参考老外的这篇[文章](https://dinfuehr.github.io/blog/encoding-of-immediate-values-on-aarch64/)，非常清楚（参考`Logical Immediates`小节）

**这里给出直观的结论：**

AND指令支持的立即数个数为5334个，结果参考：[full output](https://gist.github.com/dinfuehr/51a01ac58c0b23e4de9aac313ed6a06a)

对于AND指令支持的立即数a，必定能通过如下方式得到：

* 首先，一定能找到一个长度为n的二进制模式串s（n要求必须是2 , 4,  8, 16, 32, 64 中的一个）。

* 且模式串s必须由 m个连续的1组成的二进制串(这个串长度也是n，m要求必须在[1, n - 1]之间) 通过循环右移任意次数得到。

* 最后将这个n位的模式串s拼接 64 / n 次，得到1个64位的二进制串，它的值等于a。



**N, imms, immr字段的作用：**

`N` 和 `imms`共同决定**二进制模式串s的长度**，以及**模式串s有几个连续的1**。

`immr`： r即rotate，表示模式串s的循环右移次数，范围 0 ~ 63。

N和imms的取值组合参考下表：

| N    | imms |      |      |      |      |      | 模式串长度 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---------- |
| 0    | 1    | 1    | 1    | 1    | 0    | x    | 2 位       |
| 0    | 1    | 1    | 1    | 0    | x    | x    | 4 位       |
| 0    | 1    | 1    | 0    | x    | x    | x    | 8 位       |
| 0    | 1    | 0    | x    | x    | x    | x    | 16 位      |
| 0    | 0    | x    | x    | x    | x    | x    | 32 位      |
| 1    | x    | x    | x    | x    | x    | x    | 64 位      |

对于给定的`N`和`imms`，**x可以取0或1，但不允许所有的x同时取1**，因此对于n位的模式串，x的取值组合有 n - 1种情况。

以下举几个例子帮助理解：

* `N` = 0 `imms` = **1 1 1 0** <font color = 'red'>**1 0**</font>， 表示模式串长度为4，且模式串中连续1的个数为 10 'b + 1 = 3，即 0111。此时`immr`的范围[0, 3]
  * `immr` = 0， 表示立即数为 0111 0111 0111 0111  0111 0111 0111 0111  0111 0111 0111 0111  0111 0111 0111 0111 = 0x7777777777777777
  * `immr` = 2, 表示模式串循坏右移2次 （0111 -> 1101)，立即数为 1101 1101 1101 1101 1101 1101 1101 1101 1101 1101 1101 1101 1101 1101 1101 1101 = 0xdddddddddddd

* `N` = 0 `imms` = **1 1 1 1 0** <font color = 'red'>**0**</font>，表示模式串长度为2，且模式串中连续1的个数为 0 'b + 1 = 1，即 01。此时`immr`的范围[0, 1]
  * `immr` = 0，表示立即数 01 01 ...... 01 b(连续32个01组成)，即 0x5555555555555555
  * `immr` = 1，表示模式串循坏右移1次（01 -> 10），立即数 10 10 ......  10 b(连续32个10组成)，即 0xaaaaaaaaaaaaaaaa

可以看出：

* 对于长度为n位的模式串，连续1的个数可以是[1, n - 1]，又因为s位模式串循环右移任意次数是[0, n -1]，所以一共拼接出 n * (n - 1)个不同的立即数。

* 长度n的取值由N和imms共同决定，取值范围 {2, 4, 8, 16, 32, 64}

综上， AND指令支持的立即数总数为 2 * 1 + 4 * 3 + 8 * 7 + ...... + 64 * 63 = 5334。

可以手撕代码，打印出这5334个符合条件的立即数，参考：[https://dinfuehr.github.io/blog/encoding-of-immediate-values-on-aarch64/](https://dinfuehr.github.io/blog/encoding-of-immediate-values-on-aarch64/)

回到问题， 0xfef的二进制串为1111 1110 1111，你会发现 <font color ='red'>**N, imms, immr不管取什么值，都无法得到1111 1110 1111这个串，所以AND指令的编译会报错**</font>。



## 参考资料

【1】[《ARM Architecture Reference Manual, for ARMv8-A architecture profile》](https://developer.arm.com/documentation/ddi0487/latest/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile)

【2】  [ENCODING OF IMMEDIATE VALUES ON AARCH64](https://dinfuehr.github.io/blog/encoding-of-immediate-values-on-aarch64/)
