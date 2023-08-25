---
layout: post
title: 多线程中的资源同步
description: 从多语言多角度和分析在多线程的环境中如何让共享资源保持同步
category: blog
tag: ios , resource synchronize
---

## 关于资源同步

所谓的资源同步就是在多个线程中，共享同一资源，由于每个线程都对这一公共的资源都有操作，所以有可能回发生：A线程正在利用资源r做操作时，而资源r却在此时被线程B所修改了。这就有可能会引起一系列的问题。所以资源同步就是指：当线程A利用资源r的时候，不允许其它线程访问资源r，这就是多线程中的资源同步。

* 在单线程中，资源一定是同步的
* 只有多线程中的共享资源(公共资源)才需要资源同步

### 资源同步发生的场景

在平常的编码中，当我们声明某一个变量时，我们首先要想到这个变量的使用场景：即是不是在多线程环境下，该变量(资源)是不是在不同的线程中都被操作(包括读取和修改),如果存在，那么就需要对这个资源做线程同步。

## OC中的资源同步

OC中的资源同步有多种方式，下面我具体罗列出来：

* 用@synchronized 关键字
* 使用NSLock 加锁的方式(锁一定要成对出现，不然的话就会发生死锁)
* 把对共有资源的操作放到同一个串行队列中


### 资源同步问题场景

用多线程来模拟一个买票的程序，如果不加资源同步的话，买票的结果就是乱七八糟的，100张票有可能被卖了150次。在这里：

* 票的总张数是一定的
* 一个线程代表一个买票窗口

实现以上场景：

```
@interface ViewController ()
@property (nonatomic,assign) int ticketsCounts;  // 票总张数
@property (nonatomic,strong) NSThread *sellThread1;  // 卖票线程1
@property (nonatomic,strong) NSThread *sellThread2; // 卖票线程2

@property (nonatomic,strong) NSLock *ticketsLock;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    self.ticketsCounts = 100;
    
    self.sellThread1 = [[NSThread alloc] initWithTarget:self selector:@selector(sellWindow01_NotThreadSelf) object:nil];
    self.sellThread2 = [[NSThread alloc] initWithTarget:self selector:@selector(sellWindow02_NotThreadSelf) object:nil];
    NSLog(@"开始卖票..");
    [self.sellThread1 start];
    [self.sellThread2 start];
    
    self.ticketsLock = [[NSLock alloc] init];
}

- (void)sellWindow01_NotThreadSelf
{
    while (self.ticketsCounts) {

        if (self.ticketsCounts == 0) {
            [self.sellThread1 cancel];
            NSLog(@"window01: 票卖完了... ");
            return;
        }
        self.ticketsCounts -= 1;
        NSLog(@"window01 卖了一张票,剩余票数：%d ",self.ticketsCounts);
        usleep(1000*1000); // 休眠1s

    }

}

- (void)sellWindow02_NotThreadSelf
{
    while (self.ticketsCounts) {
        
        if (self.ticketsCounts == 0) {
            [self.sellThread1 cancel];
            NSLog(@"window02: 票卖完了... ");
            return;
        }
        self.ticketsCounts -= 1;
        NSLog(@"window02 卖了一张票,剩余票数：%d ",self.ticketsCounts);
        usleep(1000*1000); // 休眠1s
        
    }
    
}


/*
  触摸屏幕停止卖票
 */
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    [self.sellThread1 cancel];
    [self.sellThread2 cancel];
    NSLog(@"停止买票，现在剩余票数=%d",self.ticketsCounts);
}


- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}


```

我们看一下运行结果，很显然-剩余票数是混乱的。

![](https://raw.githubusercontent.com/MaxwellQi/Thread/master/images/image_01.png)


### 解决资源不同步问题

我们这里只介绍在OC语言中资源同步的方法。

#### @synchronized

直接在卖票操作时，使用`@synchronized`关键字同步资源。

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    self.ticketsCounts = 100;
        
    self.sellThread1 = [[NSThread alloc] initWithTarget:self selector:@selector(sellWindow01) object:nil];
    self.sellThread2 = [[NSThread alloc] initWithTarget:self selector:@selector(sellWindow02) object:nil];
    NSLog(@"开始卖票..");
    [self.sellThread1 start];
    [self.sellThread2 start];
    
    
    self.ticketsLock = [[NSLock alloc] init];
}

- (void)sellWindow01
{

/* 同步代码块  */
    @synchronized (self) {
        
        while (self.ticketsCounts) {
            
            if (self.ticketsCounts == 0) {
                [self.sellThread1 cancel];
                NSLog(@"window01: 票卖完了... ");
                return;
            }
            self.ticketsCounts -= 1;
            NSLog(@"window01 卖了一张票,剩余票数：%d ",self.ticketsCounts);
            usleep(1000*1000); // 休眠1s
            
        }
        
    }
}

- (void)sellWindow02
{
/** 同步代码块  **/
    @synchronized (self) {
        
        while (self.ticketsCounts) {
            
            if (self.ticketsCounts == 0) {
                [self.sellThread2 cancel];
                NSLog(@"window02: 票卖完了... ");
                return;
            }
            self.ticketsCounts -= 1;
            NSLog(@"window02 卖了一张票,剩余票数：%d ",self.ticketsCounts);
            usleep(1000*1000); // 休眠1s
            
        }

        
    }
}

```

我们可以看到，此时资源就同步了。

![](https://raw.githubusercontent.com/MaxwellQi/Thread/master/images/image_02.png)

#### NSLock

OC为我们提供了NSLoc，我们可以在对引发资源不同步的操作加上锁，这样也可以保证资源同步。

```
- (void)sellWindow01
{    
/** 加锁 **/
    [self.ticketsLock lock];
        while (self.ticketsCounts) {

            if (self.ticketsCounts == 0) {
                [self.sellThread1 cancel];
                NSLog(@"window01: 票卖完了... ");
                return;
            }
            self.ticketsCounts -= 1;
            NSLog(@"window01 卖了一张票,剩余票数：%d ",self.ticketsCounts);
            usleep(1000*1000); // 休眠1s

        }
    [self.ticketsLock unlock];
    

}

- (void)sellWindow02
{    
 /** 加锁 **/
        [self.ticketsLock lock];
        while (self.ticketsCounts) {

            if (self.ticketsCounts == 0) {
                [self.sellThread2 cancel];
                NSLog(@"window02: 票卖完了... ");
                return;
            }
            self.ticketsCounts -= 1;
            NSLog(@"window02 卖了一张票,剩余票数：%d ",self.ticketsCounts);
            usleep(1000*1000); // 休眠1s

        }
        [self.ticketsLock unlock];
    

}
```

 
利用宏定义，来简化加锁操作并带有日志：
 
```
 
#define TVULOCK(weblock)\
do{\
[weblock lock];\
NSLog(@"%s----lock-----%d",__func__,__LINE__);\
}while(0)
	
#define TVUUNLOCK(weblock)\
do{\
[weblock unlock];\
NSLog(@"%s----unlock-----%d",__func__,__LINE__);\
}while(0)
	
```

运行结果如图所示：

![](https://raw.githubusercontent.com/MaxwellQi/Thread/master/images/image_03.png)


#### 利用GCD串行队列

使用GCD串行队列也可以做资源同步。具体做法是：

*  创建一个串行队列
*  把公共代码放到串行队列中去执行

```
@property (nonatomic,strong) dispatch_queue_t phoneQueue;


- (dispatch_queue_t)phoneQueue
{
    if (!_phoneQueue) {
        _phoneQueue = dispatch_queue_create("phone_queue", DISPATCH_QUEUE_SERIAL);
    }
    return _phoneQueue;
}



- (void)sellWindow01
{      
    /** 放入串行队列中 **/
    dispatch_async(self.phoneQueue, ^{
        while (self.ticketsCounts) {
            
            if (self.ticketsCounts == 0) {
                [self.sellThread1 cancel];
                NSLog(@"window01: 票卖完了... ");
                return;
            }
            self.ticketsCounts -= 1;
            NSLog(@"window01 卖了一张票,剩余票数：%d ",self.ticketsCounts);
            usleep(1000*1000); // 休眠1s
            
        }
    });
    

}

- (void)sellWindow02
{
    /** 放入串行队列中 **/
    
    dispatch_async(self.phoneQueue, ^{
        while (self.ticketsCounts) {

            if (self.ticketsCounts == 0) {
                [self.sellThread2 cancel];
                NSLog(@"window02: 票卖完了... ");
                return;
            }
            self.ticketsCounts -= 1;
            NSLog(@"window02 卖了一张票,剩余票数：%d ",self.ticketsCounts);
            usleep(1000*1000); // 休眠1s
        }
        
    });
    

}

```

运行结果如图所示：

![](https://raw.githubusercontent.com/MaxwellQi/Thread/master/images/image_04.png)





