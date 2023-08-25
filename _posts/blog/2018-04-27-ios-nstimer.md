---
layout: post
title: NSTimer的使用以及注意事项
description: NSTimer的使用，如何避免使用中避免坑
category: blog
tag: ios
---
## 初始化NSTimer

`NSTimer`的初始化方法分为两种情况，一种是以schedule开头另一种是非schedule开头的方法。

以`schedule`开头的方法会自动把timer加入到当前`run loop`，到了设定的时间点就会触发指定的方法 ；而不是以`schedule`开头的方法则需要手动添加timer到一个`run loop`中才有效。`run loop`在运行时一般有两个mode,一个是`defaultmode`,一个是`trackingmode`，正常情况下`run loop`使用`defaultmode`，`schedule`生成的timer会默认添加到`defaultmode`中。

## 示例

### 以schedule开头的方法创建timer

下面我们来检测一下使用以`schedule`方式创建的timer ：

```
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic,strong) NSTimer  *mytimer;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    _mytimer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(timecount) userInfo:nil repeats:YES];
    
}

- (void)timecount{
    NSDate* date = [NSDate date];
    NSDateFormatter *formatter = [[NSDateFormatter alloc]init];
    [formatter setDateFormat:@"hh:mm:ss"];
    NSString *time = [formatter stringFromDate:date];
    NSLog(@"%@,%@",time,[NSThread currentThread]);
}

@end
```

运行程序打印结果如下：

```
2018-04-27 16:08:24.831872+0800 NSTimerDemo_01[15190:448471] 04:08:24,<NSThread: 0x604000074f80>{number = 1, name = main}
2018-04-27 16:08:25.830206+0800 NSTimerDemo_01[15190:448471] 04:08:25,<NSThread: 0x604000074f80>{number = 1, name = main}
```

由此可见使用`schedule`开头创建的timer确实默认添加到了当前runloop的`defaultmode`中。

### 以非schedule开头的方法创建timer

下面用以非`schedule`开头的方法创建timer:

```
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic,strong) NSTimer  *mytimer;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
   
    _mytimer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(timecount) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:_mytimer forMode:NSDefaultRunLoopMode];  // 这个runloop是在主线程中开启的
    
}

- (void)timecount{
    NSDate* date = [NSDate date];
    NSDateFormatter *formatter = [[NSDateFormatter alloc]init];
    [formatter setDateFormat:@"hh:mm:ss"];
    NSString *time = [formatter stringFromDate:date];
    NSLog(@"%@,%@",time,[NSThread currentThread]);
}

@end
```

在以上代码中，如果仅仅是调用`[NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(timecount) userInfo:nil repeats:YES]`实例化timer而不把timer放到一个`run loop`中，那么这个timer是不会执行的，即是无效的。所以我们在使用不是以`schedule`开头的方法实例化timer时，一定要注意将该timer添加到一个runloop中去。以下是运行结果：

```
2018-04-27 16:14:55.234761+0800 NSTimerDemo_01[15265:452956] 04:14:55,<NSThread: 0x60400006e180>{number = 1, name = main}
2018-04-27 16:14:56.233079+0800 NSTimerDemo_01[15265:452956] 
```

上面代码中是将timer添加到主线程的`runloop` . 我们也可以用以下方法将timer添加到非主线程的`runloop`中去。

```
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic,strong) NSTimer  *mytimer;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    [NSThread detachNewThreadSelector:@selector(useNSTimer) toTarget:self withObject:nil];
}

- (void)timecount{
    NSDate* date = [NSDate date];
    NSDateFormatter *formatter = [[NSDateFormatter alloc]init];
    [formatter setDateFormat:@"hh:mm:ss"];
    NSString *time = [formatter stringFromDate:date];
    NSLog(@"%@,%@",time,[NSThread currentThread]);
}

- (void)useNSTimer{
    _mytimer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(timecount) userInfo:nil repeats:YES];
    [self performSelector:@selector(stopcount) withObject:nil afterDelay:10];
    [[NSRunLoop currentRunLoop] addTimer:_mytimer forMode:NSRunLoopCommonModes];
    // runloop在子线程上是需要手动开启的
    [[NSRunLoop currentRunLoop] run];
}

- (void)stopcount
{
    [_mytimer invalidate];     // 如果不invalidate，会导致timmer无法停止
    _mytimer = nil;
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

@end
```

上面代码中使用`NSThread`创建一个线程。在线程中创建timer并把timer添加到该线程的`runloop`中去。有一点需要注意：非主线程的`runloop`需要手动启动。 在上面的代码中，还加入了停止timer的代码`[self performSelector:@selector(stopcount) withObject:nil afterDelay:10]`,在10s之后停止timer . 

`[timer invalidate]`方法是停止timer的方法，如果不调用这个方法而直接使用`_timer = nil`的话，会导致timer无法停止 。 

## 使用NSTimer的注意事项

在使用NSTimer时有以下注意事项。

### 掺杂着scrollview使用时


run loop在运行时一般有两个mode，一个defaultmode，一个trackingmode，正常情况下run loop使用defaultmode，scheduled生成的timer会默认添加到defaultmode中，当我们互动scrollview时，run loop切换到trackingmode运行，于是我们发现定时器失效了。为了使定时器在我们滑动scrollview时也能正常运行，我们需要确保defaultmode和trackingmode里都添加了我们生成的timer。如：

```
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:_timeInterval target:self selector:@selector(addone) userInfo:nil repeats:YES];
 [[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
```
或者

```
NSTimer *timer = [NSTimer timerWithTimeInterval:_timeInterval target:self selector:@selector(addone) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
```

### 引用问题

使用NSTimer时，timer会保持对target和userInfo参数的强引用。只有当调取了NSTimer的invalidate方法时，NSTimer才会释放target和userInfo。生成timer的方法中如果repeats参数为NO，则定时器触发后会自动调取invalidate方法。如果repeats参数为YES，则需要程序员手动调取invalidate方法才能释放timer对target和userIfo的强引用。

在使用repeats参数为YES的定时器时，如果在使用完定时器时后没有调取invalidate方法，导致target和userInfo没有被释放，则可能会形成循环引用情况，从而影响内存释放。

### NSTimer的弊端和注意下事项

* 必须保证有一个活跃的 runloop，子线程的 runloop 是默认关闭的，这时如果不激活 runloop，performSelector 和 scheduledTimerWithTimeInterval 的调用将是无效的。
* NSTimer 的创建与撤销必须在同一个线程操作、performSelector 的创建与撤销必须在同一个线程操作。
* 内存管理有潜在泄露的风险。
* 如果不是通过 invalidate 来关闭是无法停止的，还有持有 self，造成对象无法释放。

正是由于上面的这些弊端，所以推荐使用 dispatch 的 timer

## GCD定时器

### GCD定时器的创建和执行

```
- (void)startGCDTimer{
    // GCD定时器
    static dispatch_source_t _timer;
    NSTimeInterval period = 1.0; //设置时间间隔
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    dispatch_source_set_timer(_timer, dispatch_walltime(NULL, 0), period * NSEC_PER_SEC, 0); //每秒执行
    // 事件回调
    dispatch_source_set_event_handler(_timer, ^{
        // 在主线程中执行
        dispatch_async(dispatch_get_main_queue(), ^{   
            NSLog(@"Do Count"); 
        });
    });
    
    dispatch_resume(_timer);
}
```

dispatch_source_t必须是全局或者static变量。

### GCD定时器的停止

```
-(void) stopGCDTimer{
    // 停止
    if(_timer){
        dispatch_source_cancel(_timer);
        _timer = nil;
    }
}
```

### GCD定时器的暂停和继续

```
-(void) pauseGCDTimer{
    // 暂停     
    if(_timer){
        dispatch_suspend(_timer);
    }
}
-(void) resumeGCDTimer{
    // 继续
    if(_timer){
        dispatch_resume(_timer);
    }
}
```