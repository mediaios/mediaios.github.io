---
layout: post
title: 在终端中编译并运行c和c++及oc文件
description: 介绍c、c++、oc在终端中如何编译运行
category: blog
tag: c语言
---



## 前言

如果是在Mac下的话，首先我们需要先安装XCode，以及其Command line tool（命令行工具）

## c

我们通过gcc编译c程序. 首先编写一个hello.c文件，具体内容是：

```
#include <stdio.h>
int main(){
    printf("hello world!!!\n");
    return 0;
}
```

然后我们执行指令编译`.c`文件，会产生一个`hello.o`文件。最后我们通过`.o`选项进行连接生成可执行文件`hello`，然后运行可执行文件就会看到输出结果。

```
zhangqis-Mac-mini:acm qizhang$ touch hello.c
zhangqis-Mac-mini:acm qizhang$ vim hello.c
zhangqis-Mac-mini:acm qizhang$ gcc -c hello.c 
zhangqis-Mac-mini:acm qizhang$ ls
first		first.cpp	hello.c		hello.o
zhangqis-Mac-mini:acm qizhang$ gcc hello.o -o hello
zhangqis-Mac-mini:acm qizhang$ ls
first		first.cpp	hello		hello.c		hello.o
zhangqis-Mac-mini:acm qizhang$ ./hello 
hello world!!!
zhangqis-Mac-mini:acm qizhang$ 

```

其中编译和连接可以通过一行指令来完成`gcc hello.c -o hello`,下面是示例：


```
zhangqis-Mac-mini:acm qizhang$ ls
first		first.cpp	hello.c
zhangqis-Mac-mini:acm qizhang$ gcc hello.c -o hello
zhangqis-Mac-mini:acm qizhang$ ls
first		first.cpp	hello		hello.c
zhangqis-Mac-mini:acm qizhang$ ./hello 
hello world!!!
zhangqis-Mac-mini:acm qizhang$ 
```

## c++

虽然GCC也可以编译C++文件，但是使用G++（GCC用于编译C++的特殊版本）会更方便。首先我们可以直接通过`g++ first.cpp`来编译，但是会生成默认的`a.out`可执行文件，一般我们都通过`-o`选项对输出的可执行文件进行重命名操作。



编写c++程序(first.cpp)

```
#include <iostream>
using namespace std;

int main(){
	int a,b;
	cin>>a>>b;
	cout<<a+b<<endl;
	return 0;
}
```

下面是编译并输出示例

```
zhangqis-Mac-mini:acm qizhang$ ls
first.cpp
zhangqis-Mac-mini:acm qizhang$ g++ first.cpp -o first
zhangqis-Mac-mini:acm qizhang$ ls
first		first.cpp
zhangqis-Mac-mini:acm qizhang$ ./first 
15 46
61
zhangqis-Mac-mini:acm qizhang$ 
```

当然也可以通过GCC来编译C++文件，只要在再加入链接库即可。

```
lzhangqis-Mac-mini:acm qizhang$ ls
first.cpp
zhangqis-Mac-mini:acm qizhang$ gcc first.cpp -lstdc++ -o first
zhangqis-Mac-mini:acm qizhang$ ls
first		first.cpp
zhangqis-Mac-mini:acm qizhang$ ./first 
15 89
104
zhangqis-Mac-mini:acm qizhang$ 
```
  
## Objective-C

如果我们需要编译OC的源文件，我们需要用到`clang`指令（clang好像也可以编译C/C++代码?!）。我们还是先创建一个OC的源文件。

hello.m文件代码：

```
#import <Foundation/Foundation.h>

int main(int argc,const char * argv[]){
	@autoreleasepool{
		NSLog(@"hello world!!!");
	}
	return 0;
}
```

编译并输出文件`clang -fobjc-arc -framework Foundation hello.m -o hello`

示例：

```
zhangqis-Mac-mini:acm qizhang$ clang -fobjc-arc -framework Foundation hello.m -o hello
zhangqis-Mac-mini:acm qizhang$ ls
hello	hello.m
zhangqis-Mac-mini:acm qizhang$ ./hello 
2017-07-18 11:49:29.597 hello[26367:2845383] hello world!!!
```