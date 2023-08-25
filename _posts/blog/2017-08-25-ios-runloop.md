---
layout: post
title: iOS中的runloop
description: 梳理iOS中的runloop
category: blog
tag: iOS,Ojbective-C,runloop
---

## RunLoop

网上有一篇非常详细的关于RunLoop的博客： [深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)



## OC中的performselector:方法 



### NSObject中的方法 

```objective-c
- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
```

上面三个方法，均为同步执行，与线程无关。在主线程和子线程中均可调用，它只是简单的发送一个消息。 



### 延迟方法

```objective-c
- (void)performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay inModes:(NSArray<NSRunLoopMode> *)modes;
- (void)performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay;
```

上面两个方法均为异步的。如果在主线程，会执行selector中的方法；如果在其它线程，默认不会执行。因为这两个方法会在当前线程的runloop中创建一个timer，并且runloop的mode为default. 下面是苹果对这两个方法的注释： 

```objective-c
This method sets up a timer to perform the aSelector message on the current thread’s run loop. The timer is configured to run in the default mode (NSDefaultRunLoopMode). When the timer fires, the thread attempts to dequeue the message from the run loop and perform the selector. It succeeds if the run loop is running and in the default mode; otherwise, the timer waits until the run loop is in the default mode.
```

因为子线程中的runloop默认没有开启，所以这两个方法在子线程中调用时不会执行。 当你在子线程中开启runloop时一定要在这两个方法的后面开启，因为run方法是尝试开启当前线程中的runloop,如果该线程中并没有任何事件(source,timer,observer)的话，并不会成功开启。 代码如下所示： 

```objective-c
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(queue, ^{
 // [[NSRunLoop currentRunLoop] run];  // 此时runloop不会被成功开启，selector中的方法不会执行
    [self performSelector:@selector(invocationMethod) withObject:nil afterDelay:3];
    [[NSRunLoop currentRunLoop] run]; // run方法是尝试开启当前线程中的runloop，如果该线程中并没有任何事件(source,timer,observer)的话，并不会成功开启。
});
```

对于该performSelector延迟方法而言，如果在主线程中调用，那么selector方法也是在主线程中执行；如果是在子线程中调用，那么selector也是在该子线程中执行。



### 多参传递 

除了使用Dictionary和Array以及OC对象之外，还有两种方法可以做多值传递。 

方法一： 采用runtime中的`objc_msgSend()`方法

```objective-c
    // 多参传递:方法1，采用 objc_msgSend() 方法
    NSNumber *age = [NSNumber numberWithInt:10];
    NSString *name = @"Bob";
    NSString *gender = @"Male";
    NSArray *parents = @[@"Su",@"Ting"];
    
    SEL selector = NSSelectorFromString(@"getAge:name:gender:parents:");
    NSArray *array = @[age,name,gender,parents];
    ((void(*)(id,SEL,NSNumber*,NSString*,NSString*,NSArray*))objc_msgSend)(self,selector,age,name,gender,parents);


- (void)getAge:(NSNumber *)age name:(NSString *)name gender:(NSString *)gender parents:(NSArray *)parents
{
    NSLog(@"%d----%@---%@---%@",[age intValue],name,gender,parents[0]);
}
```



方法二：采用invocatiion

```objective-c
 // 多参传递:方法1，采用 objc_msgSend() 方法
    NSNumber *age = [NSNumber numberWithInt:10];
    NSString *name = @"Bob";
    NSString *gender = @"Male";
    NSArray *parents = @[@"Su",@"Ting"];
    
    SEL selector = NSSelectorFromString(@"getAge:name:gender:parents:");
    NSArray *array = @[age,name,gender,parents];    
    // 多参传递:方法2
    [self performSelector:selector withObject:array];

- (id)performSelector:(SEL)aSelector withObject:(id)object
{
    NSMethodSignature *signature = [[self class] instanceMethodSignatureForSelector:aSelector];
    if (signature == nil) {
        return nil;
    }
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
    invocation.target = self;
    invocation.selector = aSelector;
    
    //减去 self _cmd
    NSInteger paramtersCount = signature.numberOfArguments - 2;
    
    for (int i = 0; i < paramtersCount; i++) {
        id obj = object[i];
        if ([obj isKindOfClass:[NSNull class]]) continue;
        [invocation setArgument:&obj atIndex:i+2];
    }
    [invocation invoke];
    
    id returnValue = nil;
    if (signature.methodReturnLength > 0) { //如果有返回值的话，才需要去获得返回值
        [invocation getReturnValue:&returnValue];
    }
    return returnValue;
}

- (void)getAge:(NSNumber *)age name:(NSString *)name gender:(NSString *)gender parents:(NSArray *)parents
{
    NSLog(@"%d----%@---%@---%@",[age intValue],name,gender,parents[0]);
}
```

