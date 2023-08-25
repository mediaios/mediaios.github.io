---
layout: post
title: 单向链表
description: 描述单向链表的概念以及常用的操作
category: blog
tag: 单链表
---

## 概念

指向单向链表第一个节点的指针称为链表的头指针。一个链表由头指针指向第一个节点，每个节点的指针域指向其后继节点，最后一个节点的指针域为空。

链表根据指针域的个数分为单链表和双向链表两种。

如果链表中的每个节点只有一个指针域，则该链表为单链表(singly linked list)。 单链表各节点的指针域通常指向其后继节点。在这里我们介绍的是单链表。

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171016linkedlist/onlink_01.png)


### 单链表节点类

节点类如下：

```
class Node
{
public:
	int data; // 假设要存储的数据类型是int
	Node *next; // 指向其后继节点	
};
```

完整的单链表节点类声明如下：

```
class OnelinkNode
{
public:
	int data;
	OnelinkNode *next;
	OnlinkNode(int k = 0,OnelinkNode *nextnode = NULL)    // 构造节点，内联函数
	{
		data = k;
		next = NULL;
	}
	~OnelinkNode(){}		// 析构函数
}
```

## 单链表类的设计与实现

### 单链表类的声明

```
#ifndef Onelink_hpp
#define Onelink_hpp

#include <stdio.h>

class OnelinkNode
{
public:
    int data;
    OnelinkNode *next;
    OnelinkNode(int k = 0, OnelinkNode *nextnode = NULL)  // 构造节点，内联函数
    {
        data = k;
        next = NULL;
    }
    ~OnelinkNode(){};
};

class Onelink
{
public:
    OnelinkNode *head;  // 单链表的头指针
    Onelink(int n = 0); // 构造函数,以n个自然数建立单链表
    ~Onelink();
    bool isEmpty()const // 判断单链表是否为空
    {
        return head == NULL;
    }
    bool isFull()const  // 判断单链表是否已满(总是不满的)
    {
        return false;
    }
    
    int length() const; // 返回单链表长度
    OnelinkNode * index(int i)const;    // 定位，返回指向第 i 个节点的指针
    int get(int i)const;    // 返回第 i 个数据元素值
    bool set(int i,int k);  // 设置第 i 个元素值为k
    OnelinkNode * insert(OnelinkNode *p,int k);  // k 值插入作为p节点的后继节点
    
    bool remove(OnelinkNode *p);    // 删除 p 节点的后继节点
    void output(OnelinkNode *p)const;   // 输出以 p 为头指针的单链表
    void ouput()const;  // 输出以 head 为头指针的单链表
    
    

    
};


#endif /* Onelink_hpp */
```

### 单链表类的实现


#### 建立单链表

当构造函数没有参数即n=0时，建立一条空链表，head=NULL； 否则以 n 个自然数 1 ~ n 建立一条单链表，每次新创建的节点链入原链表的末尾。

```
/*
 *  建立单链表
 */
Onelink::Onelink(int n)
{
    head = NULL;    // n = 0时，构造空链表
    if (n > 0) {
        int i = 1;
        OnelinkNode *rear,*q;
        head = new OnelinkNode(i++);
        rear = head;
        while (i <= n) {
            q = new OnelinkNode(i++);
            rear->next = q;     // q节点链入 rear 节点之后
            rear = q;           // rear 指向新的链尾节点
        }
    }
}
```

#### 销毁单链表

Onelink 类的西沟函数撤销一条单链表，使 head=NULL, 并释放该单链表所占用的内存空间。

```
/*
 *  销毁单链表
 */
Onelink::~Onelink()
{
    OnelinkNode *p = head, *q;
    while (p != NULL) {
        q = p;
        p = p->next;
        delete q;
    }
    head = NULL;
}
```

#### 其它实现

```
int Onelink::length() const
{
    int linkLen = 0;
    OnelinkNode *p = head;
    while (p != NULL) {
        linkLen++;
        p = p->next;
    }
    return linkLen;
}

/*
 *  定位，返回指向第 i 个节点的指针
 */
OnelinkNode * Onelink::index(int i)const
{
    if (i < 0) return NULL;
    int j = 0;
    OnelinkNode *p = head;
    while (p != NULL && j < i) {
        j++;
        p = p->next;
    }
    return p;
}

int Onelink::get(int i)const
{
    OnelinkNode *p = index(i);
    if (p != NULL) {
        return p->data;
    }
    return -32768;
    
}

/*
 *  设置第 i 个元素值为k
 */
bool Onelink::set(int i,int k)
{
    OnelinkNode *p = index(i);
    if (p != NULL) {
        p->data = k;
        return true;
    }
    return false;
}

/*
 *  k 值插入作为p节点的后继节点
 */
OnelinkNode * Onelink::insert(OnelinkNode *p,int k)
{
    OnelinkNode *q = new OnelinkNode(k);
    if (p == NULL) {
        p = q;
    }else{
        q->next = p->next;
        p->next = q;
    }
    return p;
    
}


/*
 *  删除 p 节点的后继节点
 */
bool Onelink::remove(OnelinkNode *p)
{
    if (p != NULL) {
        OnelinkNode *q = p->next;
        if (q != NULL) {
            p->next = q->next->next;
            delete q;
            return true;
        }
    }
    return false;
}

/*
 *  输出以 p 为头指针的单链表
 */
void Onelink::output(OnelinkNode *p)const
{
    while (p != NULL) {
        printf("%d ",p->data);
        p = p->next;
    }
    printf("\n");
    
}

/*
 *  输出以 head 为头指针的单链表
 */
void Onelink::ouput()const
{
    printf("Onelink:  ");
    output(head);
}
```

## 测试代码

```
#include <iostream>
#include "Onelink.hpp"


/*
 * 单向链表逆转
 */
void reverse(Onelink &h1)
{
    OnelinkNode *p = h1.head, *q, *front = NULL;
    while (p != NULL) {
        q = p->next;
        p->next = front;
        front = p;
        p = q;
    }
    h1.head = front;
}

int main(int argc, const char * argv[]) {

    Onelink h1(5);
    h1.ouput();
    reverse(h1);
    
    printf("Reverse!\n");
    h1.ouput();
    
    return 0;
}

```

代码下载：

[完整实现- Data_stru](https://github.com/MaxwellQi/Data_stru/tree/master/DataStruct_C%2B%2B)
