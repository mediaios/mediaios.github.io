---
layout: post
title: GCD高级用法
description: 
category: blog
tag: ios, Objective-C,GCD
---

## dispatch_after 

`dispatch_after`方法是延时把任务添加到队列，而不是在指定的时间后立即执行，时间到后，它只将任务添加到队列上。

用法示例： 

```
- (void)testDispatchAfter
{
    dispatch_time_t afterT = dispatch_time(DISPATCH_TIME_NOW,3*NSEC_PER_SEC);
    dispatch_queue_t queue = dispatch_queue_create("test.queue", DISPATCH_QUEUE_SERIAL);
    dispatch_after(afterT, queue, ^{
        NSLog(@"hello word!!!");
        NSLog(@"%@",[NSThread currentThread]);
    });
}
```

## dispatch_once

`dispatch_once`函数能保证某段代码在程序运行的过程中只被执行1次，并且在多线程环境下`dispatch_once`也能保证线程安全。

单例写法示例： 

```
// Singleton.h

#import <Foundatiion/Foundation.h>
@interface Singleton : NSObject

+ (instancetype)shareInstance;
@end

// Singleton.m

#import "Singleton.h"

@implementation Singleton

static Singleton * _instance;

+ (instancetype)shareInstance
{
	static dispatch_once_t onceToken;
	dispatch_once(&onceToken,^{
		_instance = [[super allocWithZone:NULL] init];
	});
	return _instance;
}

+ (id)allocWithZone:(struct _NSZone *)zone
{
	return [Singleton shareInstance];
}

- (id)copyWithZone:(struct _NSZone *)zone
{
	return [Singleton shareInstance];
}

@end
```

调用并打印结果： 

```
// 调用
Singleton *obj1 = [Singleton shareInstance];
NSLog(@"obj1=%@",obj1);
Singleton *obj2 = [Singleton shareInstance];
NSLog(@"obj2=%@",obj1);
Singleton *obj3 = [[Singleton alloc] init];
NSLog(@"obj3=%@",obj1);
[NSThread detachNewThreadWithBlock:^{
    Singleton *obj4 = [[Singleton alloc] init];
    NSLog(@"obj4=%@ ，%@",obj1,[NSThread currentThread]);
}];

// 打印结果：

2016-07-05 16:24:55.670511+0800 Demo_01[61620:3391932] obj1=<Singleton: 0x600002bc0620>
2016-07-05 16:24:55.670638+0800 Demo_01[61620:3391932] obj2=<Singleton: 0x600002bc0620>
2016-07-05 16:24:55.670734+0800 Demo_01[61620:3391932] obj3=<Singleton: 0x600002bc0620>
2016-07-05 16:24:55.673431+0800 Demo_01[61620:3391990] obj4=<Singleton: 0x600002bc0620> ，<NSThread: 0x600003c892c0>{number = 3, name = (null)}

```

## GCD栅栏：dispatch_barrier_async 

如果为GCD并行队列中的多个任务添加栅栏(`dispatch_barrier_async`)的话，应用会首先执行`dispatch_barrier_async`之前添加到队列中的任务，然后执行`dispatch_barrier_async`之后的任务。

示例： 

```
- (void)testBarrier
{
    dispatch_queue_t currQue = dispatch_queue_create("test.barrier", DISPATCH_QUEUE_CONCURRENT);
 
 /*** 对于GlobalQueue，栅栏不会生效 ***/ 
//    dispatch_queue_t currQue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_async(currQue, ^{
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---111----");
        }
    });
    
    dispatch_async(currQue, ^{
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---222----");
        }
    });
    
    dispatch_barrier_sync(currQue, ^{
        NSLog(@"test---barrier----");
    });
    
    
    dispatch_async(currQue, ^{
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---333----");
        }
    });
    
    dispatch_async(currQue, ^{
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---444----");
        }
    });
}

// 执行结果： 
2016-07-05 16:34:37.041258+0800 Demo_01[61714:3401990] test---111----
2016-07-05 16:34:37.041258+0800 Demo_01[61714:3401991] test---222----
2016-07-05 16:34:39.041504+0800 Demo_01[61714:3401991] test---222----
2016-07-05 16:34:39.041504+0800 Demo_01[61714:3401990] test---111----
2016-07-05 16:34:41.043611+0800 Demo_01[61714:3401990] test---111----
2016-07-05 16:34:41.043611+0800 Demo_01[61714:3401991] test---222----
2016-07-05 16:34:41.043852+0800 Demo_01[61714:3401938] test---barrier----
2016-07-05 16:34:43.047195+0800 Demo_01[61714:3401990] test---333----
2016-07-05 16:34:43.047195+0800 Demo_01[61714:3401989] test---444----
2016-07-05 16:34:45.052114+0800 Demo_01[61714:3401989] test---444----
2016-07-05 16:34:45.052114+0800 Demo_01[61714:3401990] test---333----
2016-07-05 16:34:47.056917+0800 Demo_01[61714:3401989] test---444----
2016-07-05 16:34:47.056916+0800 Demo_01[61714:3401990] test---333----
```

注意：它只对`dispatch_queue_create(label, attr)`接口创建的并发队列有作用，如果是`Global Queue(dispatch_get_global_queue)`，这个barrier就不起作用。

## GCD快速迭代方法： dispatch_apply

`dispatch_apply`是同步执行的函数，不会立刻返回，在执行完block中的任务后才返回。功能是把一项任务提交到队列中多次执行，队列可以是串行也可以是并行。 为了不阻塞主线程，`dispatch_apply`正确使用方法是把`dispatch_apply`放在异步队列中调用，然后执行完成后通知主线程。

示例代码： 

```
- (void)testDispatchApply
{
    dispatch_queue_t applyQue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_apply(6, applyQue, ^(size_t cCount) {
        NSLog(@"%zu , %@",cCount,[NSThread currentThread]);
    });
}

// 打印结果： 
2016-07-05 16:51:27.155613+0800 Demo_01[61896:3421480] 0 , <NSThread: 0x600000132b00>{number = 1, name = main}
2016-07-05 16:51:27.155616+0800 Demo_01[61896:3421594] 1 , <NSThread: 0x60000015cac0>{number = 3, name = (null)}
2016-07-05 16:51:27.155692+0800 Demo_01[61896:3421599] 2 , <NSThread: 0x60000016eac0>{number = 4, name = (null)}
2016-07-05 16:51:27.155720+0800 Demo_01[61896:3421594] 4 , <NSThread: 0x60000015cac0>{number = 3, name = (null)}
2016-07-05 16:51:27.155723+0800 Demo_01[61896:3421480] 3 , <NSThread: 0x600000132b00>{number = 1, name = main}
2016-07-05 16:51:27.155764+0800 Demo_01[61896:3421599] 5 , <NSThread: 0x60000016eac0>{number = 4, name = (null)}

```

从上面运行结果可以看出，0-6打印的顺序不定。 因为在并发队列中异步执行任务，所以各个任务的执行时间长短不定，最后结束顺序也不定。`dispatch_apply`函数会等待全部的任务执行完毕，再去执行接下来的代码。

## GCD的队列组：dispatch_group

有时候我们会有这样的需求：分别异步执行2个耗时任务，然后当2个耗时任务都执行完毕后再回到主线程执行任务。这时候我们可以用到 GCD 的队列组。

* 调用队列组的 `dispatch_group_async` 先把任务放到队列中，然后将队列放入队列组中。或者使用队列组的 `dispatch_group_enter`、`dispatch_group_leave` 组合来实现
`dispatch_group_async`。
* 调用队列组的 `dispatch_group_notify` 回到指定线程执行任务。或者使用 `dispatch_group_wait` 回到当前线程继续向下执行（会阻塞当前线程）。

### dispatch_group_notify

监听group中任务的完成状态，当所有的任务都执行完成后，追加到group中，并执行任务。

测试代码：

```
- (void)testGroupNotify
{
    dispatch_group_t group = dispatch_group_create();
    NSLog(@"group---begin----");
    dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
        // task 1
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---111----,%@",[NSThread currentThread]);
        }
    });
    
    dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
        // task 2
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---222----,%@",[NSThread currentThread]);
        }
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"group 的任务都执行完了....,%@",[NSThread currentThread]);
    });
    
    NSLog(@"group---end----");
}

打印结果：
2016-07-05 17:23:55.341531+0800 Demo_01[62204:3454601] group---begin----
2016-07-05 17:23:55.341649+0800 Demo_01[62204:3454601] group---end----
2016-07-05 17:23:57.346892+0800 Demo_01[62204:3454645] test---111----,<NSThread: 0x6000029a7f00>{number = 3, name = (null)}
2016-07-05 17:23:57.346892+0800 Demo_01[62204:3454644] test---222----,<NSThread: 0x6000029ecec0>{number = 4, name = (null)}
2016-07-05 17:23:59.350834+0800 Demo_01[62204:3454644] test---222----,<NSThread: 0x6000029ecec0>{number = 4, name = (null)}
2016-07-05 17:23:59.350834+0800 Demo_01[62204:3454645] test---111----,<NSThread: 0x6000029a7f00>{number = 3, name = (null)}
2016-07-05 17:24:01.354572+0800 Demo_01[62204:3454645] test---111----,<NSThread: 0x6000029a7f00>{number = 3, name = (null)}
2016-07-05 17:24:01.354583+0800 Demo_01[62204:3454644] test---222----,<NSThread: 0x6000029ecec0>{number = 4, name = (null)}
2016-07-05 17:24:01.354822+0800 Demo_01[62204:3454601] group 的任务都执行完了....,<NSThread: 0x6000029aa940>{number = 1, name = main}
```

从以上结果可以看出，当所有的任务都执行完成后，才执行`dispatch_group_notify`block中的内容。

### dispatch_group_wait

暂停当前线程(阻塞当前线程),等待指定的group中的任务执行完成后，才会往下继续执行。

测试代码： 

```
- (void)testGroupWait
{
    dispatch_group_t group = dispatch_group_create();
    NSLog(@"group---begin----");
    dispatch_group_async(group, dispatch_get_global_queue(0, 0) , ^{
        // task 1
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---111----,%@",[NSThread currentThread]);
        }
    });
    
    dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
        // task 2
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---222----,%@",[NSThread currentThread]);
        }
    });
    
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    NSLog(@"group---end----");
}

// 打印日志

2016-07-05 17:33:15.104723+0800 Demo_01[62300:3465899] group---begin----
2016-07-05 17:33:17.105353+0800 Demo_01[62300:3465928] test---222----,<NSThread: 0x600003dc5e80>{number = 4, name = (null)}
2016-07-05 17:33:17.105354+0800 Demo_01[62300:3465931] test---111----,<NSThread: 0x600003df1280>{number = 3, name = (null)}
2016-07-05 17:33:19.105610+0800 Demo_01[62300:3465928] test---222----,<NSThread: 0x600003dc5e80>{number = 4, name = (null)}
2016-07-05 17:33:19.105610+0800 Demo_01[62300:3465931] test---111----,<NSThread: 0x600003df1280>{number = 3, name = (null)}
2016-07-05 17:33:21.110782+0800 Demo_01[62300:3465928] test---222----,<NSThread: 0x600003dc5e80>{number = 4, name = (null)}
2016-07-05 17:33:21.110782+0800 Demo_01[62300:3465931] test---111----,<NSThread: 0x600003df1280>{number = 3, name = (null)}
2016-07-05 17:33:21.110966+0800 Demo_01[62300:3465899] group---end----
```

根据以上结果可以看出，当所有执行任务完成后，才执行`dispatch_group_wait`之后的操作。但是使用`dispatch_group_wait`会阻塞当前线程。

### dispatch_group_enter,dispatch_group_leave

* `dispatch_group_enter`标志着一个任务追加到group，相当于group中未执行完的任务数+1
* `dispatch_group_leave`标志着一个任务离开了group，相当于group中未执行完的任务-1
* 当group中未执行完毕的任务为0时，才会使`dispatch_group_wait`接触阻塞，以及执行追加到`dispatch_group_notify`中的任务。

示例代码：

```
- (void)testGroupEnterLeave
{
    dispatch_group_t group = dispatch_group_create();
    NSLog(@"group---begin----");
    
    dispatch_queue_t globalQue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_enter(group);
    dispatch_async(globalQue, ^{
        // task 1
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---111----,%@",[NSThread currentThread]);
        }
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(globalQue, ^{
        // task 2
        for (int i = 0; i < 3; i++) {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"test---222----,%@",[NSThread currentThread]);
        }
        dispatch_group_leave(group);
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
         NSLog(@"group 的任务都执行完了....,%@",[NSThread currentThread]);
    });
    NSLog(@"group---end----");
}

// 执行结果：
2016-07-05 18:02:56.490714+0800 Demo_01[62537:3494536] group---begin----
2016-07-05 18:02:56.490831+0800 Demo_01[62537:3494536] group---end----
2016-07-05 18:02:58.494196+0800 Demo_01[62537:3494579] test---111----,<NSThread: 0x60000123e300>{number = 3, name = (null)}
2016-07-05 18:02:58.494176+0800 Demo_01[62537:3494576] test---222----,<NSThread: 0x600001210380>{number = 4, name = (null)}
2016-07-05 18:03:00.496831+0800 Demo_01[62537:3494579] test---111----,<NSThread: 0x60000123e300>{number = 3, name = (null)}
2016-07-05 18:03:00.496831+0800 Demo_01[62537:3494576] test---222----,<NSThread: 0x600001210380>{number = 4, name = (null)}
2016-07-05 18:03:02.502133+0800 Demo_01[62537:3494576] test---222----,<NSThread: 0x600001210380>{number = 4, name = (null)}
2016-07-05 18:03:02.502130+0800 Demo_01[62537:3494579] test---111----,<NSThread: 0x60000123e300>{number = 3, name = (null)}
2016-07-05 18:03:02.502411+0800 Demo_01[62537:3494536] group 的任务都执行完了....,<NSThread: 0x600001272940>{number = 1, name = main}
```

由以上结果可以看出，当所有任务执行完成之后，才执行dispatch_group_notify中的任务。这里的`dispatch_group_enter`,`dispatch_group_leave`组合，其实就等同于`dispatch_group_async`.

## GCD信号量：dispatch_semaphore

GCD 中的信号量是指 `Dispatch Semaphore`，是持有计数的信号。类似于过高速路收费站的栏杆。可以通过时，打开栏杆，不可以通过时，关闭栏杆。在 `Dispatch Semaphore` 中，使用计数来完成这个功能，计数为0时等待，不可通过。计数为1或大于1时，计数减1且不等待，可通过。
`Dispatch Semaphore` 提供了三个函数。

作者：行走少年郎
链接：https://juejin.im/post/5a90de68f265da4e9b592b40
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。


* `dispatch_semaphore_create` : 创建一个semaphore并初始化信号的总量
* `dispatch_semaphore_signal` : 发送一个信号，让信号总量+1
* `dispatch_semaphore_wait` : 可以使信号量-1，当信号总量为0时就会一直等待(阻塞所在线程)，否则就可以正常执行。

信号量的使用前提是：想清楚你需要处理哪个线程等待（阻塞），又要哪个线程继续执行，然后使用信号量。

`Dispatch semaphore`在实际开发中主要用于：

* 保持线程同步，将异步执行任务转换为同步执行任务
* 保证线程安全，为线程加锁

### Dispatch Semaphore线程同步

以下是实例代码： 

```
- (void)testSemaphore
{
    NSLog(@"semaphore----begin-----");
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    __block int num = 0;
    dispatch_async(queue, ^{
       // task 1
        [NSThread sleepForTimeInterval:2];
        NSLog(@"test---111----,%@",[NSThread currentThread]);
        num = 100;
        dispatch_semaphore_signal(semaphore);
    });
    
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    NSLog(@"semaphore----end,num=%d-----",num);
}

// 结果log： 
2016-07-05 18:44:37.984193+0800 Demo_01[62873:3533641] semaphore----begin-----
2016-07-05 18:44:39.987655+0800 Demo_01[62873:3533694] test---111----,<NSThread: 0x600002f72380>{number = 3, name = (null)}
2016-07-05 18:44:39.987850+0800 Demo_01[62873:3533641] semaphore----end,num=100-----
```

从以上结果可以看到： `semaphore----end`是在执行完  `number = 100;` 之后才打印的。而且输出结果 number 为 100。
这是因为异步执行不会做任何等待，可以继续执行任务。异步执行将任务1追加到队列之后，不做等待，接着执行`dispatch_semaphore_wait`方法。此时 `emaphore == 0`，当前线程进入等待状态。然后，异步任务1开始执行。任务1执行到`dispatch_semaphore_signal`之后，总信号量，此时 semaphore == 1，`dispatch_semaphore_wait`方法使总信号量减1，正在被阻塞的线程（主线程）恢复继续执行。最后打印`semaphore----end,num=100-----`。这样就实现了线程同步，将异步执行任务转换为同步执行任务。


### Dispatch Semaphore为线程加锁

直接上示例代码： 

```
static dispatch_semaphore_t semaphoreLock;
/**
 * 线程安全：使用 semaphore 加锁
 * 初始化火车票数量、卖票窗口(线程安全)、并开始卖票
 */
- (void)initTicketStatusSave {
    NSLog(@"currentThread---%@",[NSThread currentThread]);  // 打印当前线程
    NSLog(@"semaphore---begin");
    
    semaphoreLock = dispatch_semaphore_create(1);
    
    self.ticketSurplusCount = 50;
    
    // queue1 代表北京火车票售卖窗口
    dispatch_queue_t queue1 = dispatch_queue_create("net.bujige.testQueue1", DISPATCH_QUEUE_SERIAL);
    // queue2 代表上海火车票售卖窗口
    dispatch_queue_t queue2 = dispatch_queue_create("net.bujige.testQueue2", DISPATCH_QUEUE_SERIAL);
    
    __weak typeof(self) weakSelf = self;
    dispatch_async(queue1, ^{
        [weakSelf saleTicketSafe];
    });
    
    dispatch_async(queue2, ^{
        [weakSelf saleTicketSafe];
    });
}

/**
 * 售卖火车票(线程安全)
 */
- (void)saleTicketSafe {
    while (1) {
        // 相当于加锁
        dispatch_semaphore_wait(semaphoreLock, DISPATCH_TIME_FOREVER);
        
        if (self.ticketSurplusCount > 0) {  //如果还有票，继续售卖
            self.ticketSurplusCount--;
            NSLog(@"%@", [NSString stringWithFormat:@"剩余票数：%d 窗口：%@", self.ticketSurplusCount, [NSThread currentThread]]);
            [NSThread sleepForTimeInterval:0.2];
        } else { //如果已卖完，关闭售票窗口
            NSLog(@"所有火车票均已售完");
            
            // 相当于解锁
            dispatch_semaphore_signal(semaphoreLock);
            break;
        }
        
        // 相当于解锁
        dispatch_semaphore_signal(semaphoreLock);
    }
}

// 运行结果：
2016-07-05 18:57:38.584484+0800 Demo_01[62984:3547371] currentThread---<NSThread: 0x600003761440>{number = 1, name = main}
2016-07-05 18:57:38.584587+0800 Demo_01[62984:3547371] semaphore---begin
2016-07-05 18:57:38.584868+0800 Demo_01[62984:3547420] 剩余票数：49 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:38.787199+0800 Demo_01[62984:3547562] 剩余票数：48 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:38.988251+0800 Demo_01[62984:3547420] 剩余票数：47 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:39.193709+0800 Demo_01[62984:3547562] 剩余票数：46 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:39.398121+0800 Demo_01[62984:3547420] 剩余票数：45 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:39.598349+0800 Demo_01[62984:3547562] 剩余票数：44 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:39.798766+0800 Demo_01[62984:3547420] 剩余票数：43 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:40.000612+0800 Demo_01[62984:3547562] 剩余票数：42 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:40.206008+0800 Demo_01[62984:3547420] 剩余票数：41 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:40.411411+0800 Demo_01[62984:3547562] 剩余票数：40 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:40.613849+0800 Demo_01[62984:3547420] 剩余票数：39 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:40.816190+0800 Demo_01[62984:3547562] 剩余票数：38 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:41.016651+0800 Demo_01[62984:3547420] 剩余票数：37 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:41.217007+0800 Demo_01[62984:3547562] 剩余票数：36 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:41.417335+0800 Demo_01[62984:3547420] 剩余票数：35 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:41.620219+0800 Demo_01[62984:3547562] 剩余票数：34 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:41.824861+0800 Demo_01[62984:3547420] 剩余票数：33 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:42.029526+0800 Demo_01[62984:3547562] 剩余票数：32 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:42.234322+0800 Demo_01[62984:3547420] 剩余票数：31 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:42.434863+0800 Demo_01[62984:3547562] 剩余票数：30 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:42.635091+0800 Demo_01[62984:3547420] 剩余票数：29 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:42.839669+0800 Demo_01[62984:3547562] 剩余票数：28 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:43.045046+0800 Demo_01[62984:3547420] 剩余票数：27 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:43.250166+0800 Demo_01[62984:3547562] 剩余票数：26 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:43.454881+0800 Demo_01[62984:3547420] 剩余票数：25 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:43.655830+0800 Demo_01[62984:3547562] 剩余票数：24 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:43.859918+0800 Demo_01[62984:3547420] 剩余票数：23 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:44.061356+0800 Demo_01[62984:3547562] 剩余票数：22 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:44.265429+0800 Demo_01[62984:3547420] 剩余票数：21 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:44.470066+0800 Demo_01[62984:3547562] 剩余票数：20 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:44.670666+0800 Demo_01[62984:3547420] 剩余票数：19 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:44.875000+0800 Demo_01[62984:3547562] 剩余票数：18 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:45.080218+0800 Demo_01[62984:3547420] 剩余票数：17 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:45.285013+0800 Demo_01[62984:3547562] 剩余票数：16 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:45.487857+0800 Demo_01[62984:3547420] 剩余票数：15 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:45.689099+0800 Demo_01[62984:3547562] 剩余票数：14 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:45.889486+0800 Demo_01[62984:3547420] 剩余票数：13 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:46.094864+0800 Demo_01[62984:3547562] 剩余票数：12 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:46.297988+0800 Demo_01[62984:3547420] 剩余票数：11 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:46.503409+0800 Demo_01[62984:3547562] 剩余票数：10 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:46.706188+0800 Demo_01[62984:3547420] 剩余票数：9 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:46.911561+0800 Demo_01[62984:3547562] 剩余票数：8 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:47.116288+0800 Demo_01[62984:3547420] 剩余票数：7 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:47.316923+0800 Demo_01[62984:3547562] 剩余票数：6 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:47.519103+0800 Demo_01[62984:3547420] 剩余票数：5 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:47.723633+0800 Demo_01[62984:3547562] 剩余票数：4 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:47.923917+0800 Demo_01[62984:3547420] 剩余票数：3 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:48.124625+0800 Demo_01[62984:3547562] 剩余票数：2 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:48.324965+0800 Demo_01[62984:3547420] 剩余票数：1 窗口：<NSThread: 0x600003720000>{number = 3, name = (null)}
2016-07-05 18:57:48.525209+0800 Demo_01[62984:3547562] 剩余票数：0 窗口：<NSThread: 0x600003700ac0>{number = 4, name = (null)}
2016-07-05 18:57:48.725410+0800 Demo_01[62984:3547420] 所有火车票均已售完
2016-07-05 18:57:48.725557+0800 Demo_01[62984:3547562] 所有火车票均已售完

```

## GCD存在的一些问题 


### 在主队列的主线程中调用主线程执行同步任务会产生死锁

示例代码如下： 

```
dispatch_queue_t mainQue = dispatch_get_main_queue();
dispatch_sync(mainQue,^{
    NSLog(@"test...");
});
```

执行这段代码会发现程序崩溃，block里边的MainQueue不会打印出来。 反之用异步API去执行这段代码就没问题.

```
dispatch_queue_t mainQueue = dispatch_get_main_queue();
dispatch_async(mainQueue,^{
NSLog("MainQueue");            
});
```

这段代码崩溃的原因是：主线程的获取和执行都是在主队列的主线程里边执行的，然后碰到了需要同步执行的代码，主线程就会阻塞到同步代码的地方，等待同步代码执行完毕之后继续执行。可是由于FIFO协议的存在，串行队列先进先出，由于主队列主线程的事件是先进去的，所以同步代码会等待主队列主线程代码执行完毕才会继续执行。这样就产生了死锁，程序崩溃。
如果将这段代码放到其他异步队列然后由主线程执行就不会崩溃了，代码如下：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
dispatch_queue_t mainQueue = dispatch_get_main_queue();
dispatch_sync(mainQueue,^{
NSLog(@"MainQueue");
});
});

```

### 在串行队列的同步任务里再执行一个同步任务，会发生死锁 

```
dispatch_queue_t serial_queue = dispatch_queue_create("com.haley.com", DISPATCH_QUEUE_SERIAL);
dispatch_sync(serial_queue, ^{
NSLog(@"串行1--同步1");
dispatch_sync(serial_queue, ^{
NSLog(@"串行1--同步2");
});
});

```

程序崩溃，NSLog(@"串行1--同步1");会被执行，NSLog(@"串行1--同步2");不会被执行。崩溃原因和第一条是相同的，严格来说第一条只是第二条的一种特殊情况。

### 在串行队列的异步任务中再嵌套执行同步任务，也会发生死锁

```
dispatch_queue_t serial_queue = dispatch_queue_create("com.haley.com", DISPATCH_QUEUE_SERIAL);
dispatch_async(serial_queue, ^{
NSLog(@"串行1--异步1");
dispatch_sync(serial_queue, ^{
NSLog(@"串行1----同步1");
});
[NSThread sleepForTimeInterval:2.0];
});

```

NSLog(@"串行1--异步1");会执行，NSLog(@"串行1----同步1");不会执行。同样的，由于串行队列一次只能执行一个任务，任务结束后，才能执行下一个任务。所以异步任务的结束需要等里面同步任务结束，而里面同步任务的开始需要等外面异步任务结束，所以就相互等待，发生死锁了。

### dispatch_apply导致的死锁

```
dispatch_queue_t queue = dispatch_queue_create("com.multithread.tempApp", DISPATCH_QUEUE_SERIAL);
dispatch_apply(3, queue, ^(size_t fitstApplyCount) {
NSLog(@"first apply loop count: %tu", fitstApplyCount);
//再来一个dispatch_apply！死锁！
dispatch_apply(3, queue, ^(size_t secondApplyCount) {
NSLog(@"second apply loop count %tu", secondApplyCount);
});
});

```
