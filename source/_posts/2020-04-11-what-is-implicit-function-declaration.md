---
layout: next
title: 隐式函数声明[-Wimplicit-function-declaration]
date: 2020-04-11 14:33:10
categories: C
tags: C
---

## 什么是隐式函数声明

C语言中，函数调用前不一定要声明。如果没有声明，编译器会自动按照一种隐式声明规则，为调用函数的C代码产生汇编代码。

<!-- more -->

## 忽略隐式函数声明警告的危害
编译so库时会出现未定义符号，导致加载该so的程序执行出错。举例如下:

```C
// add.c
int add(int a, int b) {
	return a + b;
}
int sum(int a, int b, int c) { // 调用add方法，求三数和
	retunr ad(a, b) + c; // 这里本意调用add(a, b)，笔误写成ad(a, b)
}
```

将add.c编译成libadd.so，编译不报错，仅提示隐式函数声明警告，用ldd -r查看该so, 发现未定义符号ad。
![](image1.png)

```
$ ldd -r libadd.so
undefined symbol: ad (./libadd.so)
```

此时写一个main程序，调用libadd.so中的sum方法求三数之和
```C
#include<stdio.h>
extern int sum(int, int, int);
int main() {
	printf("%d\n", sum(1, 2, 3));
	return 0;
}
```

编译main程序出错，错误原因在于**编写add.c时将符号add错写成了ad，由于忽视隐式函数声明警告，导致该错误没有在编译add.c的时候及时发现。**

```
$ gcc main.c -L. -ladd
./libadd.so: undefined reference to `ad'
collect2: error: ld returned 1 exit status
```

* 对于用dlopen动态加载该so库的程序，如指定RTLD_NOW，dlopen立即失败；如指定RTLD_LAZY延迟绑定，则dlopen成功，但dlsym加载引用了未定义符号的函数(如sum函数)会出错，示例main程序如下：

```C
#include<dlfcn.h>
#include<stdio.h>
#include<stdlib.h>
int main() {
	void *handle = dlopen("./libadd.so", RTLD_LAZY);
	if(handle == NULL) {
		fprintf(stderr, "%s\n", dlerror());
		exit(-1);
	}
	int (*add)(int, int);
	int (*sum)(int, int, int);

	add = dlsym(handle, "add");	// add方法正常执行
	printf("1 + 2 = %d\n", add(1, 2));

	sum = dlsym(handle, "sum");	// 出错，报未定义符号ad
	printf("1 + 2 + 3 = %d\n", sum(1, 2, 3));
	return 0;
}
```

```
$ gcc main.c -ldl
$ ./a.out
1 + 2 = 3
./a.out: symbol lookup error: ./libadd.so: undefined symbol: ad
```
虽然编译成功，libadd.so可以加载，add函数可以执行，但执行sum函数时报未定义符号错。如及时处理隐式函数声明警告，可以在编译add.c的时候就发现该问题。

## 总结

* 函数先声明再使用，包含必要的头文件。

* 重视编译器的隐式函数声明警告, 可开启-Werror选项检查, 不要简单使用-Wno忽略该编译告警。
