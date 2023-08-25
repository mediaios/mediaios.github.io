---
layout: post
title : POSIX线程API
description: 对POSIX线程API的详细介绍
category: blog
tag: c,pthread,posix
---

## POSIX线程

POSIX线程是POSIX的线程标准，定义了创建和操纵线程的一套API。实现POSIX线程标准的库常被称作Pthreads,一般用于Unix-like POSIX 系统，如Linux、 Solaris。但是Microsoft Windows上的实现也存在，例如直接使用Windows API实现的第三方库pthreads-w32；而利用Windows的SFU/SUA子系统，则可以使用微软提供的一部分原生POSIX API。

### API的具体内容

Pthreads定义了一套C语言的类型、函数与常量，它以pthread.h头文件和一个线程库实现。

Pthreads API中大致共有100个函数调用，全都以"pthread_"开头，并可以分为四类：

* 线程管理，例如创建线程，等待(join)线程，查询线程状态等。
* Mutex：创建、摧毁、锁定、解锁、设置属性等操作。
* 条件变量（Condition Variable）：创建、摧毁、等待、通知、设置与查询属性等操作。
* 使用了读写锁的线程间的同步管理

POSIX的Semaphore API可以和Pthreads协同工作，但这并不是Pthreads的标准。因而这部分API是以"sem_"打头，而非"pthread_"。

### 数据类型

* pthread_t：线程句柄.出于移植目的，不能把它作为整数处理，应使用函数pthread_equal()对两个线程ID进行比较。获取自身所在线程id使用函数pthread_self()。
* pthread_attr_t：线程属性。主要包括scope属性、detach属性、堆栈地址、堆栈大小、优先级。主要属性的意义如下：

	* __detachstate，表示新线程是否与进程中其他线程脱离同步。如果设置为PTHREAD_CREATE_DETACHED，则新线程不能用pthread_join()来同步，且在退出时自行释放所占用的资源。缺省为PTHREAD_CREATE_JOINABLE状态。可以在线程创建并运行以后用pthread_detach()来设置。一旦设置为PTHREAD_CREATE_DETACHED状态，不论是创建时设置还是运行时设置，则不能再恢复到PTHREAD_CREATE_JOINABLE状态。
	* __schedpolicy，表示新线程的调度策略，包括SCHED_OTHER（正常、非实时）、SCHED_RR（实时、轮转法）和SCHED_FIFO（实时、先入先出）三种，缺省为SCHED_OTHER，后两种调度策略仅对超级用户有效。运行时可以用过pthread_setschedparam()来改变。
	* __schedparam，一个struct sched_param结构，目前仅有一个sched_priority整型变量表示线程的运行优先级。这个参数仅当调度策略为实时（即SCHED_RR或SCHED_FIFO）时才有效，并可以在运行时通过pthread_setschedparam()函数来改变，缺省为0。系统支持的最大和最小的优先级值可以用函数sched_get_priority_max和sched_get_priority_min得到。
	* __inheritsched，有两种值可供选择：PTHREAD_EXPLICIT_SCHED和PTHREAD_INHERIT_SCHED，前者表示新线程使用显式指定调度策略和调度参数（即attr中的值），而后者表示继承调用者线程的值。缺省为PTHREAD_EXPLICIT_SCHED。
	* __scope，表示线程间竞争CPU的范围，也就是说线程优先级的有效范围。POSIX的标准中定义了两个值：PTHREAD_SCOPE_SYSTEM和PTHREAD_SCOPE_PROCESS，前者表示与系统中所有线程一起竞争CPU时间，后者表示仅与同进程中的线程竞争CPU。目前LinuxThreads仅实现了PTHREAD_SCOPE_SYSTEM一值。

* pthread_barrier_t：同步屏障数据类型。
* pthread_mutex_t：mutex数据类型。
* pthread_cond_t：条件变量数据类型。

## 线程操纵函数

### pthread_create()

#### 功能

创建一个新线程。

#### 简介

	#include <pthread.h>
	
	       int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
	                          void *(*start_routine) (void *), void *arg);
	                        
编译和链接使用 -pthread .

#### 描述
	
pthread_create()函数在程序中开启一个新的线程。新线程启动执行通过调用 start_routine() 函数，arg 是传递给 start_routine() 的唯一参数。 

新的线程被终止于下列方法之一：

* 当程序调用 pthread_exit(3)，指出一个退出状态的值提供给程序中调用 pthread_join(3)的线程。
* start_routine()函数中的 return。
* 使用函数 pthread_cancel(3)取消线程执行。

attr是设置新创建线程的属性的，该属性可以使用 pthread_attr_init(3) 进行初始化。如果 attr 设置为 NULL，则创建的线程利用的就是默认的属性。

在返回之前，如果成功创建了一个线程，则线程的ID会在 thread指向的缓冲区中存储。

#### 返回值

成功时，返回 0， 失败时返回错误码。

#### 示例


	#include <pthread.h>
	#include <string.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <unistd.h>
	#include <errno.h>
	#include <ctype.h>
	
	/* prototype for thread routine */
	void print_message_function ( void *ptr );
	
	/* struct to hold data to be passed to a thread
	 this shows how multiple data items can be passed to a thread */
	typedef struct str_thdata
	{
	    int thread_no;
	    char message[100];
	} thdata;
	
	
	int main(int argc,char *argv[])
	{
	    pthread_t thread1, thread2;  /* thread variables */
	    thdata data1, data2;         /* structs to be passed to threads */
	    
	    /* initialize data to pass to thread 1 */
	    data1.thread_no = 1;
	    strcpy(data1.message, "Hello!");
	    
	    /* initialize data to pass to thread 2 */
	    data2.thread_no = 2;
	    strcpy(data2.message, "Hi!");
	    
	    /* create threads 1 and 2 */
	    pthread_create (&thread1, NULL, (void *) &print_message_function, (void *) &data1);
	    pthread_create (&thread2, NULL, (void *) &print_message_function, (void *) &data2);
	    
	    /* Main block now waits for both threads to terminate, before it exits
	     If main block exits, both threads exit, even if the threads have not
	     finished their work */
	    pthread_join(thread1, NULL);
	    pthread_join(thread2, NULL);
	    
	    /* exit */
	    exit(0);
	} /* main() */
	
	/**
	 * print_message_function is used as the start routine for the threads used
	 * it accepts a void pointer
	 **/
	void print_message_function ( void *ptr )
	{
	    thdata *data;
	    data = (thdata *) ptr;  /* type cast to a pointer to thdata */
	    
	    /* do the work */
	    printf("Thread %d says %s \n", data->thread_no, data->message);
	    
	    pthread_exit(0); /* exit */
	} /* print_message_function ( void *ptr ) */



### pthread_exit()

#### 功能

终止正在运行的线程。

#### 简介

	#include <pthread.h>
	void pthread_exit(void *retval);
	
### pthread_cancel()

#### 功能

向一个线程发送取消请求。

#### 简介

	#include <pthread.h>
	int pthread_cancel(pthread_t thread);

#### 描述

pthread_cancel() 函数向一个线程发送一个取消请求。目标线程什么时候回应以及回不回应取决于线程的两个属性：cancelability state 和 type.

线程的 cancelability state 取决于 pthread_setcancelstate(3),这个属性可以被设置为 enabled 和 disabled. 如果一个线程的 cancelability state 属性是 enabled 的，那么一个此时该线程如果接收到取消请求才有效。

发送成功并不意味着线程会终止。

若是在整个程序退出时，要终止各个线程，应该在成功发送 cancel 指令后，使用 pthread_join()函数，等待指定的线程硬完全退出之后，再继续执行；否则，很容易产生“段错误”。

#### 返回值

成功则返回0，不成功则返回非0 。


### pthread_join()

#### 功能

等待一个线程的结束，线程间同步的操作。

#### 简介

	#include <pthread.h>
	int pthread_join(pthread_t thread, void **retval);
	
#### 描述

pthread_join()函数，以阻塞的方式等待thread指定的线程结束，当函数返回时，被等待的线程资源被收回。如果线程已经结束，那么该函数会立即返回，并且thread指定的线程必须是joinable的。

#### 返回值

成功则返回0，不成功则返回错误码。


### pthread_kill()



### pthread_cleanup_push()



####返回值

* pthread_cleanup_pop()


* pthread_setcancelstate()


* pthread_setcanceltype()


## 线程属性函数

* pthread_attr_init()
* pthread_attr_setdetachstate()
* pthread_attr_getdetachstate()
* pthread_attr_setscope()
* pthread_attr_setschedparam()
* pthread_attr_getschedparam()
* pthread_attr_destroy()

## mutex函数

* pthread_mutex_init() ：初始化互斥锁
* pthread_mutex_destroy()：删除互斥锁
* pthread_mutex_lock() ：占有互斥锁（阻塞操作）
* pthread_mutex_trylock() ：试图占有互斥锁（不阻塞操作）。即，当互斥锁空闲时，将占有该锁；否则，立即返回。
* pthread_mutex_unlock() ：释放互斥锁
*  pthread_mutexattr_() ：互斥锁属性相关的函数

## 条件变量函数

* pthread_cond_init() ：初始化条件变量
* pthread_cond_destroy()：销毁条件变量
* pthread_cond_signal(): 发送一个信号给正在当前条件变量的线程队列中处于阻塞等待状态的线程，使其脱离阻塞状态，唤醒后继续执行。如果没有线程处在阻塞等待状态，pthread_cond_signal也会成功返回。一般只给一个阻塞状态的线程发信号。假如有多个线程正在阻塞等待当前条件变量，则根据各等待线程优先级的高低确定哪个线程接收到信号开始继续执行。如果各线程优先级相同，则根据等待时间的长短来确定哪个线程获得信号。但pthread_cond_signal在多处理器上可能同时唤醒多个线程，当只能让一个被唤醒的线程处理某个任务时，其它被唤醒的线程就需要继续wait。POSIX规范要求pthread_cond_signal至少唤醒一个pthread_cond_wait上的线程，有些实现为了简便，在单处理器上也会唤醒多个线程。所以最好对pthread_cond_wait()使用while循环对条件变量是否满足做条件判断。
* pthread_cond_wait(): 等待条件变量的特殊条件发生；pthread_cond_wait() 必须与一个pthread_mutex配套使用。该函数调用实际上依次做了3件事：对当前pthread_mutex解锁、把当前线程挂起到当前条件变量的线程队列、被其它线程的信号唤醒后对当前pthread_mutex申请加锁。如果线程收到一个信号被唤醒，将被配套的互斥锁重新锁住，pthread_cond_wait() 函数将不返回直到线程获得配套的互斥锁。需要注意的是，一个条件变量不应该与多个互斥锁配套使用。
* pthread_cond_broadcast(): 某些应用，如线程池，pthread_cond_broadcast唤醒全部线程，但我们通常只需要一部分线程去做执行任务，所以其它的线程需要继续wait.
* pthread_condattr_(): 条件变量属性相关的函数
* pthread_key_create(): 分配用于标识进程中线程特定数据的pthread_key_t类型的键
* pthread_key_delete(): 销毁现有线程特定数据键
* pthread_setspecific(): 为指定线程的特定数据键设置绑定的值
* pthread_getspecific(): 获取调用线程的键绑定值，并将该绑定存储在 value 指向的位置中

## 同步屏障函数

* pthread_barrier_init()
* pthread_barrier_wait()
* pthread_barrier_destory()

## 工具函数

* pthread_equal(): 对两个线程的线程标识号进行比较
* pthread_detach(): 分离线程
* pthread_self(): 查询线程自身线程标识号
* pthread_once()： 某些需要仅执行一次的函数。其中第一个参数为pthread_once_t类型，是内部实现的互斥锁，保证在程序全局仅执行一次。
* 信号量函数，包含在semaphore.h中：
* sem_open：创建或者打开已有的命名信号量。可分为二值信号量与计数信号量。命名信号量可以在进程间共享使用。
* sem_close：关闭一个信号灯，但没有将它从系统中删除。命名信号灯是随内核持续的，即使当前没有进程打开着某个信号灯，它的值仍然保持。
* sem_unlink：从系统中删除信号灯。
* sem_getvalue：返回所指定信号灯的当前值。如果该信号灯当前已上锁，那么返回值或为0，或为某个负数，其绝对值就是等待该信号灯解锁的线程数。
* sem_wait：申请共享资源，所指定信号灯的值如果大于0，那就将它减1并立即返回，就可以使用申请来的共享资源了。如果该值等于0，调用线程就被进入睡眠状态，直到该值变为大于0，这时再将它减1，函数随后返回。sem_wait操作必须是原子操作。
* sem_trywait：申请共享资源，当所指定信号灯的值已经是0时，后者并不将调用线程投入睡眠。相反，它返回一个EAGAIN错误。
* sem_post：释放共享资源。与sem_wait恰相反。
* sem_init：初始化非命名（内存）信号量
* sem_destroy：摧毁非命名信号量

## 共享内存函数
* mmap：把一个文件或一个POSIX共享内存区对象映射到调用进程的地址空间。使用该函数的目的： 1.使用普通文件以提供内存映射I/O 2.使用特殊文件以提供匿名内存映射。 3.使用shm_open以提供无亲缘关系进程间的Posix共享内存区。
* munmap： 删除一个映射关系
* msync：文件与内存同步函数
* shm_open：创建或打开共享内存区
* shm_unlink：删除一个共享内存区对象的名字，删除一个名字仅仅防止后续的open,msq_open或sem_open调用取得成功。
* ftruncate:调整文件或共享内存区大小
* fstat来获取有关该对象的信息