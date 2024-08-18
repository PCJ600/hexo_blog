---
layout: next
title: 使用GDB修改二进制文件
date: 2021-10-16 20:11:59
categories: GDB
tags: GDB
---

### 背景介绍
GDB不仅可以用来调试程序，还可以直接修改被调试程序的二进制文件。这种方式相比于改源码重新编译、`gdb attach`有什么优势呢？考虑以下企业生产环境中的几个调试场景：

* **需要修改的二进制文件是其他领域的，你没有源码和编译工程**，让相关领域出调试对接件比较费时，但你只想临时改一行别人的代码，几分钟内完成验证。
* **调试环境上，使用gdb attach进程方式有困难**：
  * 被调试的服务（进程）没有启动断点(可定位性很差)，或者gdb手动拉起的方法非常复杂，等服务正常启动后再attach已经赶不上打断点的时机。
  * 长时间gdb挂住业务进程导致触发丢心跳复位。
  * 你不确定修改的二进制文件同时被几个进程加载，但你诉求很明确，就是直接改文件，对所有进程生效。

以下举一个简单的例子，介绍GDB修改程序二进制文件的技巧：
<!-- more -->

### 问题举例

有一个需修改的二进制文件，C码如下：

```C
int main() {
    int grade = 65;
    if (grade <= 60) {			// 这里写错了，需要修改成 >= 60及格
        printf("pass\n");
    } else {
        printf("fail\n");
    }
    return 0;
}
```

程序执行结果如下：

```
# gcc main.c -o main && ./main
fail
```

<font color = 'red'>【问题】</font>这里需要将`grade <= 60` 要改成 `grade > 60`。只通过GDB修改二进制文件的方式怎么实现？

### 技巧&解决步骤

1、缺省情况下，`gdb`是以只读方式加载程序的。需要先通过命令行指定加载方式为可写，再通过file命令加载二进制文件：

```
(gdb) set write on
(gdb) show write
Writing into executable and core files is on.
(gdb) file main
Reading symbols from main...(no debugging symbols found)...done.
```

2、结合C码和汇编代码，定位出需修改的汇编指令：

```assembly
(gdb) disassemble /mr main
Dump of assembler code for function main:
   0x0000000000001135 <+0>:     55                      push   %rbp
   0x0000000000001136 <+1>:     48 89 e5                mov    %rsp,%rbp
   0x0000000000001139 <+4>:     48 83 ec 10             sub    $0x10,%rsp
   0x000000000000113d <+8>:     c7 45 fc 41 00 00 00    movl   $0x41,-0x4(%rbp)
   0x0000000000001144 <+15>:    83 7d fc 3c             cmpl   $0x3c,-0x4(%rbp)        # 0x3c = 60, 对应C码：grade <= 60
   0x0000000000001148 <+19>:    7f 0e                   jg     0x1158 <main+35>
   0x000000000000114a <+21>:    48 8d 3d b3 0e 00 00    lea    0xeb3(%rip),%rdi        # 0x2004
   0x0000000000001151 <+28>:    e8 da fe ff ff          callq  0x1030 <puts@plt>
   0x0000000000001156 <+33>:    eb 0c                   jmp 0x1164 <main+47>
   0x0000000000001158 <+35>:    48 8d 3d aa 0e 00 00    lea    0xeaa(%rip),%rdi        # 0x2009
   0x000000000000115f <+42>:    e8 cc fe ff ff          callq  0x1030 <puts@plt>
   0x0000000000001164 <+47>:    b8 00 00 00 00          mov    $0x0,%eax
   0x0000000000001169 <+52>:    c9                      leaveq
   0x000000000000116a <+53>:    c3                      retq

```

不难发现， `if (grade <= 60)` 对应的指令为：`0x1148 <+19>: 7f 0e jg 0x1158 <main+35>`， 这里只需要将`jg`指令改`jle`指令即可。

3、`jg`, `jle`指令格式参考[指令集手册](https://www.intel.cn/content/www/cn/zh/architecture-and-technology/64-ia-32-architectures-software-developer-vol-2a-manual.html)，如下：
![](image1.png)
所以只需要将 `7f 0e` 改成`7e 0e`即可。

4、修改二进制代码（注意大小端和指令长度），用`gdb`的`set`命令修改地址处的内容，方法如下：

```
(gdb) disassemble /mr main
Dump of assembler code for function main:
   ...
   0x0000000000001148 <+19>:    7f 0e   jg     0x1158 <main+35>
   ...
(gdb) set *(short *)0x1148 = 0xe7e  (指令长度为2个字节，这里是小端序)
(gdb) disassemble /mr main
Dump of assembler code for function main:
   ...
   0x0000000000001148 <+19>:    7e 0e   jle    0x1158 <main+35>
   ...
End of assembler dump.
```

退出`gdb`, 执行`main`程序输出pass，说明修改生效：

```
# ./main
pass
```

### 思考

如果改成70分以上及格，如何修改？如果是aarch64格式的二进制呢？

注意：涉及到立即数的修改，x86_64和aarch64差异很大。aarch64中不同的汇编指令，对立即数的存储方式和表示范围都不同，具体操作时需查询对应的指令集手册。

### 参考资料

【1】[《100个gdb小技巧》](https://wizardforcel.gitbooks.io/100-gdb-tips/content/)

【2】[《ARM Architecture Reference Manual, for ARMv8-A architecture profile》](https://developer.arm.com/documentation/ddi0487/latest/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile)

【3】[《64-ia-32-architectures-software-developer-vol-2a-manual》](https://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-software-developer-vol-2a-manual.html)


