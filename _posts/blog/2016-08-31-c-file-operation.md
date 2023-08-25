---
layout: post
title : 文件操作
description: c语言中的文件操作
category: blog
tag: c,file, stream
---

## 文件概述

所有的文件都通过流进行输入、输出操作。与文本流和二进制流对应，文件可以分为文本文件和二进制文件两大类。

* 文本文件，也称为ASCII文件，这种文件在保存时，每个字符对应一个字节，用于存放对应的ASCII码。
* 二进制文件，不是保存ASCII码，而是按二进制的编码方式来保存文件内容。

## 文件基本操作

文件的基本操作包括文件的打开和关闭。除了标准的输入、输出文件外，其他所有的文件都必须先打开再使用，而使用后也必须关闭该文件。

### 文件指针

文件指针是一个指向文件有关信息的指针，这些信息包括文件名、状态和当前位置，他们保存在一个结构体变量中。在使用文件时需要在内存中为其分配空间，用来存放文件的基本信息。该结构体类型是由系统定义的，C语言规定该类型为 FILE 型，其声明如下：

	typedef	struct __sFILE {
		unsigned char *_p;	/* current position in (some) buffer */
		int	_r;		/* read space left for getc() */
		int	_w;		/* write space left for putc() */
		short	_flags;		/* flags, below; this FILE is free if 0 */
		short	_file;		/* fileno, if Unix descriptor, else -1 */
		struct	__sbuf _bf;	/* the buffer (at least 1 byte, if !NULL) */
		int	_lbfsize;	/* 0 or -_bf._size, for inline putc */
	
		/* operations */
		void	*_cookie;	/* cookie passed to io functions */
		int	(*_close)(void *);
		int	(*_read) (void *, char *, int);
		fpos_t	(*_seek) (void *, fpos_t, int);
		int	(*_write)(void *, const char *, int);
	
		/* separate buffer for long sequences of ungetc() */
		struct	__sbuf _ub;	/* ungetc buffer */
		struct __sFILEX *_extra; /* additions to FILE to not break ABI */
		int	_ur;		/* saved _r when _r is counting ungetc data */
	
		/* tricks to meet minimum requirements even when malloc() fails */
		unsigned char _ubuf[3];	/* guarantee an ungetc() buffer */
		unsigned char _nbuf[1];	/* guarantee a getc() buffer */
	
		/* separate buffer for fgetln() when line crosses buffer boundary */
		struct	__sbuf _lb;	/* buffer for fgetln() */
	
		/* Unix stdio files get aligned to block boundaries on fseek() */
		int	_blksize;	/* stat.st_blksize (may be != _bf._size) */
		fpos_t	_offset;	/* current lseek offset (see WARNING) */
	} FILE;
	
从上面的结构中可以发现使用typedef定义了一个FILE为该结构体类型，在编写程序时可直接使用上面定义的 FILE 类型来定义变量，注意在定义变量是不必将结构体内容全部给出，只需要如下形式：

	FILE *fp; // fp是一个指向 FILE 类型的指针变量
	
### 文件的打开

fopen 函数用来打开一个文件，打开文件的操作就是创建一个流。fopen 函数的原型在 stdio.h 中，起吊用的一般形式为：

	FILE *fp;
	fp = fopen(文件名，使用文件方式);

其中文件名就是要被打开的文件的名字，”使用文件方式“ 是指对打开的文件要进行读还是写。文件的大开发方式有如下：

| 文件使用方式 | 含义 |
|:-----------: | :--------------:|
| r (只读) | 打开一个文本文件，只允许读数据 |
| w (只写) | 打开或建立一个文本文件，只允许写数据 |
| a (追加) | 打开一个文本文件，并在文件末尾写数据 |
| rb (只读) | 打开一个二进制文件，只允许读数据 |
| wb (只写) | 打开或建立一个二进制文件，只允许写数据 |
| ab (追加) | 打开一个二进制文件，并在文件末尾写数据 | 
| r+ (读写) | 打开一个文本文件，只允许读和写 |
| w+ (读写) | 打开或建立一个文本文件，允许读写 |
| a+ (读写) | 打开一个文件，允许读，火灾文件末追加数据 |
| rb+ (读写) | 打开一个二进制文件，允许读和写 |
| wb+ (读写) | 打开或建立一个二进制文件，允许读和写 |
| ab+ (读写) | 打开一个二进制文件，语序读，火灾文件末追加数据 |

如果使用 fopen 函数打开文件成功，则返回一个有确定指向的 FILE 类型指针；若打开失败，则返回 NULL 。通常打开失败的原因有以下几个方面：

* 指定的路径不存在
* 文件名中含有无效字符
* 以 r 模式打开一个不存在的文件

### 文件的关闭

文件使用完毕后，应使用 fclose 函数将其关闭。 fclose 函数和 fopen 一样，原型也在 stdio.h 中，调用的一般形式为：

	fclose(文件指针);

fclose 函数也返回一个值，当正常完成关闭文件操作时， fclose 函数返回值为 0 ，否则返回 EOF。 在程序结束之前应关闭所有文件，这样做的目的是防止因为没有关闭文件二造成数据流失。

## 文件的读写

### fputc 函数

	ch = fputc(ch,fp);

该函数的作用是把一个字符写到磁盘文件(fp所指向的文件) 中去。其中 ch 是要输出的字符，它可以是一个字符常量，也可以是一个字符变量。 fp是文件指针变量。如果函数输出成功则返回值就是输出的字符；如果失败，则返回 EOF 。

eg:

	void writeCharToFile()
	{
	    FILE *fp;
	    char ch;
	    fp = fopen("/Users/qizhang/Desktop/exp01.txt", "w");
	    if (fp == NULL) {
	        printf("can not open file\n");
	        exit(0);
	    }
	    ch = getchar();
	    while (ch != '#') {
	        fputc(ch, fp);
	        ch = getchar();
	    }
	    fclose(fp);
	}

### fgetc 函数

	ch = fgetc(fp);

该函数的作用是从指定的文件（fp指向的文件）读入一个字符赋给ch 。需要注意的是，该文件必须是以读或读写的方式打开。当函数遇到文件结束符时将返回一个文件结束标志 EOF .

eg: 从刚刚的文件中读数据

	void readCharFromFile()
	{
	    FILE *fp;
	    char ch;
	    fp = fopen("/Users/qizhang/Desktop/exp01.txt", "r");
	    ch = fgetc(fp);
	    while (ch != EOF) {
	        putchar(ch);
	        ch = fgetc(fp);
	    }
	    fclose(fp);
	}

### fputs 函数

fputs 函数和 fputc 函数类似，区别在于 fputc 每次只向文件中写入一个字符，而 fputs 函数每次向文件中写入一个字符串。

	fputs(字符串,文件指针);
	
该函数的作用是向指定的文件写入一个字符串，其中字符串可以是字符串常量，也可以是字符数组名、指针或变量。

eg:

	void writeStringToFile()
	{
	    FILE *fp;
	    char filename[30],str[30];
	    printf("please input filename:\n");
	    scanf("%s",filename);
	    fp = fopen(filename, "w");
	    if (fp == NULL) {
	        printf("Can not open! \n press any key to continue:\n");
	        getchar();
	        exit(0);
	    }
	    printf("please input string:\n");
	    getchar();
	    gets(str);
	    fputs(str, fp);
	    fclose(fp);
	}
	
### fgets 函数

fgets 函数与 fgetc 函数类似，区别在于 fgetc 每次从文件中读出一个字符，而 fgets 函数每次从文件中读出一个字符串。

	fgets(字符数组名,n,文件指针);
该函数的作用是从指定的文件中读一个字符串到字符数组中。 n表示所得到的字符串中红的字符的个数 (包含'\0')。

eg:

	void readStringFromFile()
	{
	    FILE *fp;
	    char filename[30],str[30];
	    printf("please input filename:\n");
	    scanf("%s",filename);
	    fp = fopen(filename,"r");
	    if(fp == NULL) {
	        printf("can not open!\n press any key to continue\n");
	        getchar();
	        exit(0);
	    }
	    fgets(str, sizeof(str), fp);  // 只获取30个长度
	    printf("%s",str);
	    fclose(fp);
	}

### fprintf 函数

fprintf函数和fscanf函数与printf和scanf函数的作用相似，它们最大的区别就是读写的对象不同，fprintf和fscanf函数读写的对象不是终端而是磁盘文件。

	ch = fprintf(文件类型指针,格式字符串,输出列表);
	fprintf(fp,"%d",i); // 将整形变量i输出到fp指向的文件中

eg:

	void writeToFile()
	{
	    FILE *fp;
	    int i = 88;
	    char filename[80];
	    printf("please input filename:\n");
	    scanf("%s",filename);
	    fp = fopen(filename, "w");
	    if (fp == NULL) {
	        printf("can not open!\npress any key to continue\n");
	        getchar();
	        exit(0);
	    }
	    fprintf(fp, "%c",i); // 将88以字符形式写入fp所指的文件中
	    fclose(fp);
	}
	
### fscanf 函数

fscanf函数的一般形式如下：

	fscanf(文件类型指针,格式字符串,输入列表);
	fscanf(fp,"%d",&i);

它的作用是读入fp所指向的文件中的i的值。

eg:

	void readOperationUSEfscanf()
	{
	    FILE *fp;
	    char i,j;
	    char filename[30]; // 定义一个字符型数组
	    printf("please input filename:\n");
	    scanf("%s",filename); // 输入文件名
	    fp = fopen(filename, "r");
	    if (fp == NULL) {
	        printf("can not open!\npress any key to continue\n");
	        getchar();
	        exit(0);
	    }
	    for (i = 0; i < 5; i++) {
	        fscanf(fp, "%c",&i);
	        printf("%d is %5d\n",i+1);
	    }
	    fclose(fp);
	}

### fread函数和fwrite函数

fputc 和 fgetc 函数每次只能读写文件中的一个字符，但是在编写程序的过程中往往需要对整块数据进行读写，例如对一个结构体类型变量值进行读写。此时就需要使用 fread 和 fwrite 函数。

fread 函数的一般形式如下：

	fread(buffer,size,count,fp);

该函数的作用是从 fp 所指的文件中读入 count 次，每次读 size 字节，读入的字节存在 buffer 地址中。

fwrite 函数的一般形式如下：

	fwrite(buffer,size,count,fp);

它的作用是将 buffer 地址开始的信息输出 count 次，每次写 size 字节到 fp 所指向的文件中。参数说明如下：

* buffer: 一个指针。对于 fwrite 来说就是要输出数据的地址（起始地址）；对于 fread 来说是所要读入的数据存放的地址。
* size: 要读写的字节数。
* count: 要读写多少 size 字节的数据项。
* fp: 文件型指针。

eg:

	fread(a,2,3,fp);

其含义是从 fp 所指文件中每次读两个字节送入实数组a中，连续读 3 次。

	fwrite(a,2,3,fp);

其含义是将 a 数组中的信息每次输出两个字节到 fp 所指向的文件中，连续输出 3 次。

	struct address_list
	{
	    char name[10];
	    char adr[20];
	    char tel[15];
	}info[100];
	
	void save(char *name,int n)
	{
	    FILE *fp;
	    int i;
	    fp = fopen(name, "wb");
	    if (fp == NULL) {
	        printf("can not open file\n");
	        exit(0);
	    }
	    for (int i = 0; i < n; i++) {
	        if (fwrite(&info[i], sizeof(struct address_list), 1, fp) != 1) {
	            printf("file write error\n");
	        }
	        fclose(fp);
	    }
	}
	
	void show(char *name,int n)
	{
	    int i;
	    FILE *fp;
	    fp = fopen(name, "rb");
	    if (fp == NULL) {
	        printf("can not open file\n");
	        exit(0);
	    }
	    for (i = 0; i < n; i++) {
	        fread(&info[i], sizeof(struct address_list), 1, fp);
	        printf("%15s%20s%20s\n",info[i].name,info[i].adr,info[i].tel);
	    }
	    fclose(fp);
	}
	
	int main()
	{
	    int i,n;
	    char filename[50];
	    printf("how may ?\n");
	    scanf("%d",&n);
	    printf("please input filename:\n");
	    scanf("%s",filename);
	    printf("please input name,address,telephone:\n");
	    for (int i= 0; i < n; i++) {
	        printf("NO%d",i+1);
	        scanf("%s%s%s",info[i].name,info[i].adr,info[i].tel);
	        save(filename, n);
	    }
	    
	    show(filename, n);
	    
	    return 0;
	}

## 文件的定位

在对文件进行操作时，往往不需要从头开始，秩序对其中指定的内容进行操作，这是就需要使用文件定位函数来实现文件的随机读取。

### fseek 函数

借助缓冲型 I/O 系统中的 fseek 函数可以完成随机读写操作。 fseek 函数的一般形式如下：

	fseek(文件类型指针,位移量,起始点);

该函数的作用是移动文件内部位置指针。其中，“文件类型指针”指向被移动文件，“位移量”表示移动的字节数，要求位移量是 long 类型数据，以便在文件长度大于 64KB 时不会出错。当用常量表示位移量时，要求加后缀 “L” ；参数“起始点”表示从何处开始计算位移量，规定的起始点有3种：文件首、文件当前位置和文件尾。如图：

| 起始点 | 表示符号 | 数字表示 |
|:------|:-------:|--------:|
| 文件手 | SEEK-SET| 0       |
| 当前位置| SEEK-CUR| 1      |
| 文件末尾| SEEK-END| 2      |

	fseek(fp,-20L,1); 
表示将位置指针从当前位置向后退20个字节。fseek 函数一般用于二进制文件。在文本文件中由于要进行转换，往往计算的位置会出现错误。文件的随机读写在移动位置指针之后进行，即可用前面介绍的任一种读写函数进行读写。

eg:

	void fseekDemo()
	{
	    FILE *fp;
	    char filename[30],str[50];
	    printf("please input filename:\n");
	    scanf("%s",filename);
	    fp = fopen(filename, "wb");
	    if (fp == NULL) {
	        printf("can not open!\n press any key to continue\n");
	        getchar();
	        exit(0);
	    }
	    printf("please input string:\n");
	    getchar();
	    gets(str);
	    fputs(str, fp);
	    fclose(fp);
	    
	    fp = fopen(filename, "rb");
	    if (fp == NULL) {
	        printf("can not open!\npress any key to continue\n");
	        getchar();
	        exit(0);
	    }
	    fseek(fp, 5L, 0); // 将文件指向距文件首5个字节的位置，指向字符串中的第6个字符
	    fgets(str, sizeof(str), fp);
	    putchar('\n');
	    puts(str);
	    fclose(fp);
	}

### rewind 函数

	void rewind(文件类型指针);
该函数的作用是使位置指针重新返回文件的开头，该函数没有返回值。

eg:输出文件中的内容

	void rewindDemo()
	{
	    FILE *fp;
	    char ch,filename[50];
	    printf("please input filename:\n");
	    scanf("%s",filename);
	    fp = fopen(filename, "r");
	    if (fp == NULL) {
	        printf("can not open this file.\n");
	        exit(0);
	    }
	    ch = fgetc(fp);
	    while (ch != EOF) {
	        putchar(ch);
	        ch = fgetc(fp);
	    }
	    rewind(fp);
	    ch = fgetc(fp);
	    while (ch != EOF) {
	        putchar(ch);
	        ch = fgetc(fp);
	    }
	    fclose(fp);
	}

### ftell 函数

	long ftell(文件类型指针);
该函数的作用是得到流式文件中的当前位置，用相对于文件开头的位移量来表示。当 ftell 函数返回值为 -1L 时，表示出错。

eg:(求字符串的长度)

	void getStrLength()
	{
	    FILE *fp;
	    int n;
	    char ch,filename[50];
	    printf("please input filename:\n");
	    scanf("%s",filename);
	    fp = fopen(filename, "r");
	    if (fp == NULL) {
	        printf("can not open this file.\n");
	        exit(0);
	    }
	    ch = fgetc(fp);
	    while (ch != EOF) {
	        putchar(ch);
	        ch = fgetc(fp);
	    }
	    n = ftell(fp);
	    printf("\nthe length of the string is:%d\n",n);
	    fclose(fp);
	}

