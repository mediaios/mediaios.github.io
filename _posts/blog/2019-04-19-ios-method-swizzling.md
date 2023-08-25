---
layout: post
title: 初探Method Swizzling及其应用场景
description: 学习ios中的Method Swizzling的使用方法及应用场景
category: blog
tag: runtime,ios runtime, ios Method Swizzling, Method Swizzling
---

## 关于Method Swizzling 

最近对一些ios的apm系统比较感兴趣，所以就研究了一些相关的技术。首先从最基本的`Method Swizzling`开始。

`Method Swizzling`是OC runtime提供的一种动态替换方法实现的技术，我们利用它可以替换系统或者我们自定义类的方法实现，进而达到我们的特殊目的。 

代码-github： [MethodSwizzling](https://github.com/mediaios/training/tree/master/MethodSwizzling)

### Method Swizzling原理 

为什么`Method Swizzling`能替换一个类的方法呢？我们首先要理解一下其替换的原理。 

OC中的方法在`runtime.h`中的定义如下： 

```
struct objc_method{
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
}
```

* method_name: 方法名
* method_types: 方法类型,主要存储着方法的参数类型和返回值类型
* method_imp: 方法的实现，函数指针

由此，我们也可以发现 OC 中的方法名是不包括参数类型的，也就是说下面两个方法在 runtime 看来就是同一个方法：

```
- (void)viewWillAppear:(BOOL)animated;
- (void)viewWillAppear:(NSString *)string;
```

原则上，方法的名称 `method_name` 和方法的实现 `method_imp` 是一一对应的，而 Method Swizzling 的原理就是动态地改变它们的对应关系，以达到替换方法实现的目的。

## `Method Swizzling`应用 

### runtime中和方法替换相关的函数 

#### class_getInstanceMethod

```
OBJC_EXPORT Method _Nullable
class_getInstanceMethod(Class _Nullable cls, SEL _Nonnull name);
```

* 作用：获取一个类的实例方法
* cls : 方法所在的类
* name: 选择子的名称(选择子就是方法名称)

#### class_getClassMethod

```
OBJC_EXPORT Method _Nullable
class_getClassMethod(Class _Nullable cls, SEL _Nonnull name);
```

* 作用：获取一个类的类方法
* cls : 方法所在的类
* name: 选择子名称

#### method_getImplementation

```
OBJC_EXPORT IMP _Nonnull
method_getImplementation(Method _Nonnull m);
```

* 作用: 根据方法获取方法的指针
* m : 方法 

#### method_getTypeEncoding

```
OBJC_EXPORT const char * _Nullable
method_getTypeEncoding(Method _Nonnull m);
```

* 作用: 获取方法的参数和返回值类型描述 
* m : 方法 


#### class_addMethod

```
OBJC_EXPORT BOOL
class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, 
                const char * _Nullable types);
```

* 作用: 给类添加一个新方法和该方法的实现
* 返回值： yes,表示添加成功; No,表示添加失败
* cls： 将要添加方法的类
* name: 将要添加的方法名
* imp: 实现这个方法的指针 
* types: 要添加的方法的返回值和参数

#### method_exchangeImplementations

```
OBJC_EXPORT void
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2);
```

* 交换两个方法 

#### class_replaceMethod 

```
OBJC_EXPORT IMP _Nullable
class_replaceMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, 
                    const char * _Nullable types) ;
```
* 作用： 指定替换方法的实现 
* cls :  将要替换方法的类
* name:  将要替换的方法名 
* imp:   新方法的指针
* types: 新方法的返回值和参数描述


### 替换一个类的实例方法

eg： 替换`UIViewController`中的`viewDidLoad`方法。 

```
#import "UIViewController+MI.h"
#import <objc/runtime.h>

@implementation UIViewController (MI)

+ (void)load{
    
    // 替换ViewController中的viewDidLoad方法
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        SEL origin_selector  = @selector(viewDidLoad);
        SEL swizzed_selector = @selector(mi_viewDidLoad);
        
        Method origin_method = class_getInstanceMethod([self class], origin_selector);
        Method swizzed_method = class_getInstanceMethod([self class], swizzed_selector);
        
        BOOL did_add_method = class_addMethod([self class],
                                              origin_selector,
                                              method_getImplementation(swizzed_method),
                                              method_getTypeEncoding(swizzed_method));
        if (did_add_method) {
            NSLog(@"debugMsg: ViewController类中没有viewDidLoad方法(可能在其父类h中)，所以先添加后替换");
            class_replaceMethod([self class],
                                swizzed_selector,
                                method_getImplementation(origin_method),
                                method_getTypeEncoding(origin_method));
        }else{
            NSLog(@"debugMsg: 直接交换方法");
            method_exchangeImplementations(origin_method, swizzed_method);
        }
        
    });
}

- (void)mi_viewDidLoad
{
    [self mi_viewDidLoad];
    NSLog(@"debugMsg: 替换成功");
}
```

对以上代码做简要说明： 

* 在category的`+(void)load`方法中添加替换的代码，在类初始加载的时候会被自动调用。
* dispath_once确保只执行一次
* 先调用`class_addMethod`方法，保证即使父类中存在要替换的方法，也能替换成功


替换成功。控制台信息： 

```
2019-04-17 17:25:16.937849+0800 MethodSwizzling[4975:639584] debugMsg: 直接交换方法
2019-04-17 17:25:17.025214+0800 MethodSwizzling[4975:639584] debugMsg: 替换成功
```

### 替换一个类的实例方法到另一个类

当我们对私有类库(不知道该类的头文件，只知道有这个类并且已知该类中的一个方法)，此时我们需要hook这个类的方法到一个新类中。 

eg： 我们要hook person类中有一个`speak:`方法方法: 

```
#import "Person.h"

@implementation Person

- (void)speak:(NSString *)language
{
    NSLog(@"person speak language: %@",language);
}

+ (void)sleep:(NSUInteger)hour
{
    NSLog(@"person sleep: %lu",hour);
}

@end
```

我们新建`ChinesePerson`，hook `speak:`方法到`ChinesePerson`中。 

```
#import "ChinesePerson.h"
#import <objc/runtime.h>

@implementation ChinesePerson

+ (void)load
{
    Class origin_class  = NSClassFromString(@"Person");
    Class swizzed_class = [self class];
    
    SEL origin_selector = NSSelectorFromString(@"speak:");
    SEL swizzed_selector = NSSelectorFromString(@"mi_speak:");
    
    Method origin_method = class_getInstanceMethod(origin_class, origin_selector);
    Method swizzed_method = class_getInstanceMethod(swizzed_class, swizzed_selector);
    
    BOOL add_method = class_addMethod(origin_class,
                                      swizzed_selector,
                                      method_getImplementation(swizzed_method),
                                      method_getTypeEncoding(swizzed_method));
    if (!add_method) {
        return;
    }
    
    swizzed_method = class_getInstanceMethod(origin_class, swizzed_selector);
    if (!swizzed_method) {
        return;
    }
    
    BOOL did_add_method = class_addMethod(origin_class,
                                          origin_selector,
                                          method_getImplementation(swizzed_method),
                                          method_getTypeEncoding(swizzed_method));
    if (did_add_method) {
        class_replaceMethod(origin_class,
                            swizzed_selector,
                            method_getImplementation(origin_method),
                            method_getTypeEncoding(origin_method));
    }else{
        method_exchangeImplementations(origin_method, swizzed_method);
    }
    
}

- (void)mi_speak:(NSString *)language
{
    if ([language isEqualToString:@"Chinese"]) {
        [self mi_speak:language];
    }
}
```


替换成功。控制台信息（只打印汉语）：

```
2019-04-17 17:25:17.025362+0800 MethodSwizzling[4975:639584] person speak language: Chinese
```

### 替换类方法 

eg: 我们替换person类中的`sleep:`方法： 

```
#import "Person+MI.h"
#import <objc/runtime.h>

@implementation Person (MI)
+ (void)load
{
    Class class = [self class];
    SEL origin_selector  = @selector(sleep:);
    SEL swizzed_selector = @selector(mi_sleep:);
    
    Method origin_method = class_getClassMethod(class, origin_selector);
    Method swizzed_method = class_getClassMethod(class,swizzed_selector);
    
    if (!origin_method || !swizzed_method) {
        return;
    }
    
    IMP origin_imp = method_getImplementation(origin_method);
    IMP swizzed_imp = method_getImplementation(swizzed_method);
    const char* origin_type = method_getTypeEncoding(origin_method);
    const char* swizzed_type = method_getTypeEncoding(swizzed_method);
    
    // 添加方法到MetaClass中
    Class meta_class = objc_getMetaClass(class_getName(class));
    class_replaceMethod(meta_class, swizzed_selector, origin_imp, origin_type);
    class_replaceMethod(meta_class, origin_selector, swizzed_imp, swizzed_type);
    
}

+ (void)mi_sleep:(NSUInteger)hour
{
    if (hour >= 7) {
        [self mi_sleep:hour];
    }
}
@end

```

控制台打印（睡眠大于等于7小时才打印----呼吁健康睡眠）：

```
2019-04-17 17:25:17.025465+0800 MethodSwizzling[4975:639584] person sleep: 8
```

类方法的hook和实例方法的hook有两点不同：

* 获取Method的方法变更为`class_getClassMethod(Class cls, SEL name)`,不是`class_getInstanceMethod(Class cls, SEL name)`;
* 对于类方法的动态添加,需要将方法添加到MetaClass中,因为实例方法记录在class的method-list中, 类方法是记录在meta-class中的method-list中的.


### 替换类簇中的方法 

```
#import "MIMutableDictionary.h"
#import <objc/runtime.h>

@implementation MIMutableDictionary

+ (void)load
{
    Class origin_class = NSClassFromString(@"__NSDictionaryM");
    Class swizzed_class = [self class];
    SEL origin_selector = @selector(setObject:forKey:);
    SEL swizzed_selector = @selector(mi_setObject:forKey:);
    Method origin_method = class_getInstanceMethod(origin_class, origin_selector);
    Method swizzed_method = class_getInstanceMethod(swizzed_class, swizzed_selector);
    IMP origin_imp = method_getImplementation(origin_method);
    IMP swizzed_imp = method_getImplementation(swizzed_method);
    const char* origin_type = method_getTypeEncoding(origin_method);
    const char* swizzed_type = method_getTypeEncoding(swizzed_method);

    class_replaceMethod(origin_class, swizzed_selector, origin_imp, origin_type);
    class_replaceMethod(origin_class, origin_selector, swizzed_imp, swizzed_type);
}

- (void)mi_setObject:(id)objContent forKey:(id<NSCopying>)keyContent
{
    if (objContent && keyContent) {
        NSLog(@"执行了进去");
        [self mi_setObject:objContent forKey:keyContent];
    }
}


@end
```

## 应用

不推荐在项目中过多的使用`Method Swizzling`，不然的话原生的类都被hook的非常乱，项目出问题时非常难定位问题。有一篇文章说它是ios中的一个毒瘤。 [iOS届的毒瘤-MethodSwizzling](https://www.valiantcat.cn/index.php/2017/11/03/53.html)

尽管如此，我们还是了解并梳理一下其应用场景来感受一下这种技术。

### 防止数组取值时越界crash

不仅仅是数组，`NSDictionary`的利用runtime防止崩溃是同样的原理。 

```
#import "NSArray+Safe.h"
#import <objc/runtime.h>

@implementation NSArray (Safe)

+ (void)load
{
    // objectAtIndex: 方式取元素
    Method origin_method = class_getInstanceMethod(objc_getClass("__NSArrayI"), @selector(objectAtIndex:));
    Method replaced_method = class_getInstanceMethod(objc_getClass("__NSArrayI"), @selector(safeObjectAtIndex:));
    method_exchangeImplementations(origin_method, replaced_method);
    
    Method origin_method_muta = class_getInstanceMethod(objc_getClass("__NSArrayM"), @selector(objectAtIndex:));
    Method replaced_method_muta = class_getInstanceMethod(objc_getClass("__NSArrayM"), @selector(safeMutableObjectAtIndex:));
    method_exchangeImplementations(origin_method_muta, replaced_method_muta);
    
    // 直接通过数组下标的方式取元素
    Method origin_method_sub = class_getInstanceMethod(objc_getClass("__NSArrayI"), @selector(objectAtIndexedSubscript:));
    Method replaced_method_sub = class_getInstanceMethod(objc_getClass("__NSArrayI"), @selector(safeObjectAtIndexedSubscript:));
    method_exchangeImplementations(origin_method_sub, replaced_method_sub);
    
    Method origin_method_muta_sub = class_getInstanceMethod(objc_getClass("__NSArrayM"), @selector(objectAtIndexedSubscript:));
    Method replaced_method_muta_sub = class_getInstanceMethod(objc_getClass("__NSArrayM"), @selector(safeMutableObjectAtIndexedSubscript:));
    method_exchangeImplementations(origin_method_muta_sub, replaced_method_muta_sub);
    
}

- (id)safeObjectAtIndex:(NSUInteger)index
{
    if (self.count > index && self.count) {
        return [self safeObjectAtIndex:index];
    }
    NSLog(@"errorMsg:数组[NSArray]越界...");
    return nil;
}

- (id)safeMutableObjectAtIndex:(NSUInteger)index
{
    if (self.count > index && self.count) {
        return [self safeMutableObjectAtIndex:index];
    }
    NSLog(@"errorMsg:数组[NSMutableArray]越界...");
    return nil;
}

-(id)safeObjectAtIndexedSubscript:(NSUInteger)index
{
    if (self.count > index && self.count) {
        return [self safeObjectAtIndexedSubscript:index];
    }
    NSLog(@"errorMsg:数组[NSArray]越界...");
    return nil;
}

- (id)safeMutableObjectAtIndexedSubscript:(NSUInteger)index
{
    if (self.count > index && self.count) {
        return [self safeMutableObjectAtIndexedSubscript:index];
    }
    NSLog(@"errorMsg:数组[NSMutableArray]越界...");
    return nil;
}

@end

```

使用： 

```
- (void)test2
{
    NSArray *arr = @[@"a",@"b",@"c",@"d",@"e",@"f"];
    NSLog(@"atIndex方式： %@",[arr objectAtIndex:10]);
    NSLog(@"下标方式: %@",arr[10]);
}
```

控制台输出log： 

```
2019-04-18 19:14:18.139417+0800 MethodSwizzling[25379:1703659] errorMsg:数组[NSArray]越界...
2019-04-18 19:14:18.139536+0800 MethodSwizzling[25379:1703659] atIndex方式： (null)
2019-04-18 19:14:18.139793+0800 MethodSwizzling[25379:1703659] errorMsg:数组[NSArray]越界...
2019-04-18 19:14:18.139868+0800 MethodSwizzling[25379:1703659] 下标方式: (null)
```

### 改变app中所有按钮的大小

常规做法是遍历视图中的所有子视图，把所有的button整体改变。 此时我们使用runtime改变按钮的大小。 

```
#import "UIButton+Size.h"
#import <objc/runtime.h>

@implementation UIButton (Size)

+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 增大所有按钮大小
        Method origin_method = class_getInstanceMethod([self class], @selector(setFrame:));
        Method replaced_method = class_getInstanceMethod([self class], @selector(miSetFrame:));
        method_exchangeImplementations(origin_method, replaced_method);
        
    });
   
}

- (void)miSetFrame:(CGRect)frame
{
    frame = CGRectMake(frame.origin.x, frame.origin.y, frame.size.width+20, frame.size.height+20);
    NSLog(@"设置按钮大小生效");
    [self miSetFrame:frame];
}

@end
```

### 处理按钮重复点击

如果重复过快的点击同一个按钮，那么就会多次触发和按钮绑定的事件。处理这种case的方式有很多种，通过`Method Swillzing`也能解决这种问题。 

.h文件： 

```
#import <UIKit/UIKit.h>

NS_ASSUME_NONNULL_BEGIN

@interface UIButton (QuickClick)
@property (nonatomic,assign) NSTimeInterval delayTime;
@end

NS_ASSUME_NONNULL_END
```


```
#import "UIButton+QuickClick.h"
#import <objc/runtime.h>

@implementation UIButton (QuickClick)
static const char* delayTime_str = "delayTime_str";

+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Method originMethod =   class_getInstanceMethod(self, @selector(sendAction:to:forEvent:));
        Method replacedMethod = class_getInstanceMethod(self, @selector(miSendAction:to:forEvent:));
        method_exchangeImplementations(originMethod, replacedMethod);
    });
}

- (void)miSendAction:(nonnull SEL)action to:(id)target forEvent:(UIEvent *)event
{
    if (self.delayTime > 0) {
        if (self.userInteractionEnabled) {
            [self miSendAction:action to:target forEvent:event];
        }
        self.userInteractionEnabled = NO;
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW,
                                     (int64_t)(self.delayTime * NSEC_PER_SEC)),
                                     dispatch_get_main_queue(), ^{
                                         self.userInteractionEnabled = YES;
                                     });
    }else{
        [self miSendAction:action to:target forEvent:event];
    }
}

- (NSTimeInterval)delayTime
{
    return [objc_getAssociatedObject(self, delayTime_str) doubleValue];
}

- (void)setDelayTime:(NSTimeInterval)delayTime
{
    objc_setAssociatedObject(self, delayTime_str, @(delayTime), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```

## 代码

github： [MethodSwizzling](https://github.com/mediaios/training/tree/master/MethodSwizzling)


## 参考

* [Method Swizzling的各种姿势](http://www.tanhao.me/code/160723.html/)
* [Objective-C Method Swizzling 的最佳实践](http://blog.leichunfeng.com/blog/2015/06/14/objective-c-method-swizzling-best-practice/)
* [Method Swizzling](https://nshipster.com/method-swizzling/)
* [iOS届的毒瘤-MethodSwizzling](https://www.valiantcat.cn/index.php/2017/11/03/53.html)