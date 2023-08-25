---
layout: post
title: 顺序表
description: 描述顺序表的概念以及常用的操作
category: blog
tag: seqlist,顺序表
---

## 顺序表的定义

### 顺序存储方法

把线性表的节点按逻辑次序依次存放在一组地址连续的存储单元里的方法。

### 顺序表（Seqential List）

用顺序存储方法存储的线性表简称为顺序表。

## 节点a1的存储地址

不失一般性，设线性表中所有结点的类型相同，则每个结点所占用存储空间大小亦相同。假设表中每个结点占用c个存储单元，其中第一个单元的存储地址则是该结点的存储地址，并设表中开始结点a1的存储地址（简称为基地址）是LOC（a1），那么结点ai的存储地址LOC（ai）可通过下式计算：`LOC（ai）= LOC（a1）+（i-1）*c `

在顺序表中，每个结点ai的存储地址是该结点在表中的位置i的线性函数。只要知道基地址和每个结点的大小，就可在相同时间内求出任一结点的存储地址。是一种随机存取结构。

## 顺序表类型定义

	 #define ListSize 100 //表空间的大小可根据实际需要而定，这里假设为100
	  typedef int DataType; //DataType的类型可根据实际情况而定，这里假设为int
	  typedef struct {
	      DataType data[ListSize]；//向量data用于存放表结点
	      int length；//当前的表长度
	     }SeqLi

关于书序表类型定义，有以下注意点：

* 用向量这种顺序存储的数组类型存储线性表的元素外，顺序表还应该用一个变量来表示线性表的长度属性，因此用结构类型来定义顺序表类型。
* 存放线性表结点的向量空间的大小ListSize应仔细选值，使其既能满足表结点的数目动态增加的需求，又不致于预先定义过大而浪费存储空间。
* 由于C语言中向量的下标从0开始，所以若L是SeqList类型的顺序表，则线性表的开始结点a1和终端结点an分别存储在L．data[0]和L．Data[L．length-1]中。
* 若L是SeqList类型的指针变量，则a1和an分别存储在L-＞data[0]和L-＞data[L-＞length-1]中。

## 书序表的特点

顺序表是用向量实现的线性表，向量的下标可以看作结点的相对地址。因此顺序表的的特点是逻辑上相邻的结点其物理位置亦相邻。

## 顺序表的代码实现

头文件：

	#ifndef qi_define_linerlist_h
	#define qi_define_linerlist_h
	
	#include <stdio.h>
	
	#define MAXSIZE 100
	#define TRUE 1
	#define FALSE 0
	
	typedef  struct {
	    int data[MAXSIZE];
	    int last;
	}SeqList;
	
	SeqList SeqListInit();
	SeqList SeqListInsert(SeqList L,int i,int x);
	void ShowData(SeqList L);
	SeqList SeqListDelete(SeqList L,int i);
	SeqList SeqListMerge(SeqList Sla, SeqList Slb);
	
	#endif

具体实现：

	#include "qi_define_linerlist.h"
	
	// 初始化顺序表
	SeqList SeqListInit()
	{
	    SeqList L;
	    L.last = 0;  // 顺序表长度为 0
	    return L;
	}
	
	// 查找元素
	// 在顺序表 L 中查找值为 x 的元素。若查找成功，则返回它在顺序表中的位置，否则返回 -1
	int SeqListLocate(SeqList L,int x)
	{
	    int i = 1;
	    while (i < L.last && L.data[i-1] != x) {
	        i++;
	    }
	    return i < L.last ? i : 0;
	}
	
	// 判断线性表是否有元素
	int IsEmpty(SeqList L)
	{
	    return L.last == 0 ? TRUE : FALSE;
	}
	
	// 插入运算
	SeqList SeqListInsert(SeqList L,int i,int x)
	{
	    int j;
	    if(L.last == MAXSIZE)
	    {
	        printf("表满\n");            // 表满，退出
	        exit(0);
	    }
	    if(i < 1 || i > L.last + 1)
	    {
	        printf("插入位置错误\n");     // 插入位置错，退出
	        exit(0);
	    }
	    for(j = L.last - 1; j >= i - 1; j--)
	    {
	        L.data[j + 1] = L.data[j];  // 第i个位置后的数组元素逐一后移
	    }
	    L.data[i - 1] = x;              // 新元素插入到第i个位置
	    L.last++;                       // 顺序表长度增1
	    return (L);                     // 返回插入元素后的顺序表
	}
	
	// 删除运算
	SeqList SeqListDelete(SeqList L,int i)
	{
	    int j;
	    if (i < 1 || i > L.last) {
	        printf("位置非法\n");
	        exit(0);
	    }
	    for (j = i; j <= L.last -1; j++) {
	        L.data[j -1]  = L.data[j];
	    }
	    L.last--;
	    return (L);
	}
	
	// 合并非递减有序顺序表
	SeqList SeqListMerge(SeqList Sla, SeqList Slb)
	{// 将非递减有序的顺序表Sla和Slb合并成一个新的顺序表Slc，Slc也是非递减有序的
	    // 初始化
	    SeqList Slc;
	    Slc.last = 0;
	    int i = 0, j = 0, k = -1;
	    while(i < Sla.last && j < Slb.last)     // Sla和Slb均非空
	    {
	        if(Sla.data[i] <= Slb.data[j])
	        {
	            Slc.data[++k] = Sla.data[i++];  // Sla中元素插入Slc
	        }
	        else
	        {
	            Slc.data[++k] = Slb.data[j++];  // Slb中元素插入Slc
	        }
	    }
	    while(i < Sla.last)
	    {
	        Slc.data[++k] = Sla.data[i++];      // Slb已空，将Sla表的剩余部分复制到新表
	    }
	    while(j < Slb.last)
	    {
	        Slc.data[++k] = Slb.data[j++];      // Sla已空，将Slb表的剩余部分复制到新表
	    }
	    Slc.last = k + 1;
	    return (Slc);
	}
	
	// 展示SeqList的内容
	void ShowData(SeqList L)
	{
	    int i = 0;
	    for (; i < L.last; ++i)
	    {
	        printf("%d\t", L.data[i]);
	    }
	    printf("\n");
	}
	
	
[代码下载——github](https://github.com/MaxwellQi/DataStructure/tree/master/DataStructureStudy/DataStructureStudy/seqlist)