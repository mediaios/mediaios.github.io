---
layout: post
title: 预处理
description: 记录C语言中预处理指令的使用
category: blog
tag: c,define
---

## 预处理指令

### 1.预处理的分类

大多数的预处理指令分为以下三种类型：

* **宏定义** `#define` 指令定义一个宏， `#undef`指令删除一个宏定义。
* **文件包含** `#include` 指令让一个指定文件的内容被包含到程序中。
* **条件编译** `#if` , `#ifdef` , `#ifndef` ,`#elif` , `#else` 和 `#endif` 指令可以根据编译器可以测试的条件来讲一段文本块包含到程序中或排除在程序之外。
* **其它** `#error`,`#line`和`#pragma` 指令等。 

### 2.预处理指令的特点

所有的预处理指令都有以下特点：

* **指令都以#开始** #号不需要在一行的行首，只要它之前只有空白字符就行。在#号之后是指令名，接着是指令所需要的其它信息。
* **在指令的符号之间可以插入任意数量的空格或横向制表符** 例如下面是合法的：

	```
	#         define          N          100 
	```
	 
* **指令总是在第一个换行符处结束，除非明确地指明要继续** 如果想在下一行继续指令，我们必须在当前行的末尾使用 `\` 字符。例如我们利用下面指令定义了一个宏来表示硬盘容量，按字节计算：

```
#define DISK_CAPACITY (SIDES *                  \
							TRACK_PER_SIDE *         \
							BYTES_PER_TRACK *        \
							BYTES_PER_SECTOR	)
```

* **指令可以出现在程序中的任何地方** 我们通常将`#define`和`#include`指令放在文件的开始，其他指令则放在后面，甚至在函数定义的中间。
* **注释可以与指令放在同一行** 实际上，在一个宏定义的后面加一个注释来解释宏的意义。 

## 宏定义

### 1.简单宏

定义格式：

`#define 标识符 替换列表`

eg:

```
#define STE_LEN 80
#define TRUE 1
```

### 2.带参数的宏

定义格式：

`#define 标识符(x1,x2,...,xn) 替换列表`

注意：在宏的名字和左括号之间必须没有空格。如果有空格，预处理器会认为是在定义一个简单的宏，其中（x1,x2,……,xn）是替换列表的一部分。

宏定义的本质也就是代码替换。在使用到宏的地方，会把宏定义的内容代码直接拷贝过来。

eg: （注意宏内容中的小括号的使用）

```
	#define MAX(x,y) ((x)>(y) ? (x):(y));
```

