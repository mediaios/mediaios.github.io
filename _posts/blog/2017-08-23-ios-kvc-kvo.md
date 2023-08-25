---
layout: post
title: KVC与KVO的用法以及其原理解析
description: 介绍KVC与KVO的用法以及内部实现原理
category: blog
tag: iOS,Ojbective-C,KVO,KVC
---

## 概述

KVC和KVO是在日常开发中经常遇到的问题，并且在面试的时候多有被问到。这篇文章主要介绍KVC与KVO的用法以及内部实现原理。

* KVC的基本原理以及其简单应用
* KVO的基本原理以及其应用

以下是详细介绍。

## KVC

KVC( Key-Value-Coding),用于键值编码。KVC 是一种可以直接通过字符串的名字 key 来访问类属性的机制，而不是通过调用 `setter`、`getter` 方法去访问。我们可以通过在运行时动态的访问和修改对象的属性。而不是在编译时确定，KVC 是 iOS 开发中的黑魔法之一。

### KVC的使用

KVC定义了一种按名称访问对象属性的机制，支持这种访问的主要方法有如下几类。

#### 设置值

为属性设置值：

```
// value的值为OC对象，如果是基本数据类型要包装成NSNumber
- (void)setValue:(id)value forKey:(NSString *)key;

// keyPath键路径，类型为xx.xx
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;

// 它的默认实现是抛出异常，可以重写这个函数做错误处理。
- (void)setValue:(id)value forUndefinedKey:(NSString *)key;
```

#### 获取值

获取属性的值：

```
- (id)valueForKey:(NSString *)key;

- (id)valueForKeyPath:(NSString *)keyPath;

// 如果Key不存在，且没有KVC无法搜索到任何和Key有关的字段或者属性，则会调用这个方法，默认是抛出异常
- (id)valueForUndefinedKey:(NSString *)key;
```

#### 其它

以下是其它的KVC方法：

```
// 允许直接访问实例变量，默认返回YES。如果某个类重写了这个方法，且返回NO，则KVC不可以访问该类。
+ (BOOL)accessInstanceVariablesDirectly;

// 这是集合操作的API，里面还有一系列这样的API，如果属性是一个NSMutableArray，那么可以用这个方法来返回
- (NSMutableArray *)mutableArrayValueForKey:(NSString *)key;

// 如果你在setValue方法时面给Value传nil，则会调用这个方法
- (void)setNilValueForKey:(NSString *)key;

// 输入一组key，返回该组key对应的Value，再转成字典返回，用于将Model转到字典。
- (NSDictionary *)dictionaryWithValuesForKeys:(NSArray *)keys;

// KVC提供属性值确认的API，它可以用来检查set的值是否正确、为不正确的值做一个替换值或者拒绝设置新值并返回错误原因。
- (BOOL)validateValue:(id)ioValue forKey:(NSString *)inKey error:(NSError)outError;
```

### 示例

定义一个`Chinese`类：

```
/* Chinese.h */
#import <Foundation/Foundation.h>

@interface Chinese : NSObject
{
    @private
    int _age;
}
@property (nonatomic,strong,readonly) NSString *name;
@property (nonatomic,assign,getter=isMale) BOOL male;
- (void)log;
@end


/* Chinese.m */
#import "Chinese.h"

@implementation Chinese
- (void)log
{
    NSLog(@"Chinese: name=%@,age=%d,male=%d",_name,_age,(int)_male);
}
@end

```

这个类有私有 private 变量和只读 readonly 变量，如果用一般的 setter 和 getter，在类外部是不能访问到私有变量的，不能设值给只读变量，那是不是就拿它没办法了呢?然而使用 KVC 可以做到。

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    Chinese *chinese = [Chinese new];
    [chinese log];
    
    // 设置 readonly value
    [chinese setValue:@"Jack" forKey:@"name"];
    
    // 设置 private value
    [chinese setValue:@24 forKey:@"age"];
    [chinese setValue:@1 forKey:@"male"];
    [chinese log];
    
    // 获取 readonly value
    NSLog(@"name: %@", [chinese valueForKey:@"_name"]);
    
    // 获取 private value
    NSLog(@"age: %d", [[chinese valueForKey:@"_age"] intValue]);
    NSLog(@"male: %d", [[chinese valueForKey:@"isMale"] boolValue]);
}

```

运行结果：

```
2017-08-23 18:50:35.011510+0800 DemoKVO[4417:1074197] Chinese: name=(null),age=0,male=0
2017-08-23 18:50:35.011632+0800 DemoKVO[4417:1074197] Chinese: name=Jack,age=24,male=1
2017-08-23 18:50:35.011712+0800 DemoKVO[4417:1074197] name: Jack
2017-08-23 18:50:35.011850+0800 DemoKVO[4417:1074197] age: 24
2017-08-23 18:50:35.011880+0800 DemoKVO[4417:1074197] male: 1
```

### KVC的实现细节

```
- (void)setValue:(id)value forKey:(NSString *)key;
```
1. 首先搜索 setter 方法，有就直接赋值。
2. 如果上面的 setter 方法没有找到，再检查类方法`+ (BOOL)accessInstanceVariablesDirectly`
	* 返回 NO，则执行`setValue：forUNdefinedKey：`
	* 返回 YES，则按`_<key>，_<isKey>，<key>，<isKey>`的顺序搜索成员名。
3. 还没有找到的话，就调用`setValue:forUndefinedKey:`


```
- (id)valueForKey:(NSString *)key;
```
1. 首先查找 getter 方法，找到直接调用。如果是 bool、int、float 等基本数据类型，会做 NSNumber 的转换。
2. 如果没查到，再检查类方法`+ (BOOL)accessInstanceVariablesDirectly`
	* 返回 NO，则执行`valueForUNdefinedKey:`
	* 返回 YES，则按`_<key>,_is<Key>,<key>,is<Key>`的顺序搜索成员名。
3. 还没有找到的话，调用`valueForUndefinedKey:`

### KVC与点语法的比较

用 KVC 访问属性和用点语法访问属性的区别：

1. 用点语法编译器会做预编译检查，访问不存在的属性编译器会报错，但是用 KVC 方式编译器无法做检查，如果有错误只能运行的时候才能发现（crash）。
2. 相比点语法用 KVC 方式 KVC 的效率会稍低一点，但是灵活，可以在程序运行时决定访问哪些属性。
3. 用 KVC 可以访问对象的私有成员变量。

### KVC的应用

以下是KVC在开发中比较常见的应用。

#### 字典转模型

```
- (void)setValuesForKeysWithDictionary:(NSDictionary *)keyedValues;
```

#### 集合类操作符

获取集合类的 count，max，min，avg，svm。确保操作的属性为数字类型，否则会报错。

```
NSMutableArray *bookList = [NSMutableArray array];
for (int i = 0; i <= 20; i++)  {
  Book *book = [[Book alloc] initWithName:[NSString stringWithFormat:@"book%d",i] price:i*10];
  [bookList addObject:book];
}
   
Student *student = [[Student alloc] initWithBookList:bookList];
   
Teacher *teacher = [[Teacher alloc] initWithStudent:student];

// KVC获取数组
for (Book *book in [student valueForKey:@"bookList"]) {
  NSLog(@"bookName : %@ \t price : %f",book.name,book.price);
}
   
NSLog(@"All book name  : %@",[teacher valueForKeyPath:@"student.bookList.name"]);
NSLog(@"All book name  : %@",[student valueForKeyPath:@"bookList.name"]);
NSLog(@"All book price : %@",[student valueForKeyPath:@"bookList.price"]);
   
// 计算（确保操作的属性为数字类型，否则会报错。)  五种集合运算符
NSLog(@"count of book price : %@",[student valueForKeyPath:@"bookList.@count.price"]);
NSLog(@"min of book price : %@",[student valueForKeyPath:@"bookList.@min.price"]);
NSLog(@"avg of book price : %@",[student valueForKeyPath:@"bookList.@max.price"]);
NSLog(@"sum of book price : %@",[student valueForKeyPath:@"bookList.@sum.price"]);
NSLog(@"avg of book price : %@",[student valueForKeyPath:@"bookList.@avg.price"]);
```

运行结果：

```
All book price : (
    0,
    10,
    20,
    30,
    40,
    50,
    60,
    70,
    80,
    90,
    100
)
2017-01-18 20:45:26.640887 KVC-Demo[58294:5509308] count of book price : 11
2017-01-18 20:45:26.640956 KVC-Demo[58294:5509308] min of book price : 0
2017-01-18 20:45:26.641039 KVC-Demo[58294:5509308] avg of book price : 100
2017-01-18 20:45:26.641220 KVC-Demo[58294:5509308] sum of book price : 550
2017-01-18 20:45:26.641300 KVC-Demo[58294:5509308] avg of book price : 50
```

#### 修改私有属性

1) 修改TextField的placeholder:

```
[_textField setValue:[UIColor redColor] forKeyPath:@"_placeholderLabel.textColor"];   

[_textField setValue:[UIFont systemFontOfSize:14] forKeyPath:@“_placeholderLabel.font"];
```

2) 修改 UIPageControl 的图片：

```
[_pageControl setValue:[UIImage imageNamed:@"selected"] forKeyPath:@"_currentPageImage"];

[_pageControl setValue:[UIImage imageNamed:@"unselected"] forKeyPath:@"_pageImage"];
```

### 总结

键值编码是一种间接访问对象的属性使用字符串来标识属性，而不是通过调用存取方法直接或通过实例变量访问的机制，非对象类型的变量将被自动封装或者解封成对象，很多情况下会简化程序代码。

优点：

1. 不需要通过 setter、getter 方法去访问对象的属性，可以访问对象的私有属性
2. 可以轻松处理集合类(NSArray)。

缺点：

1. 一旦使用KVC你的编译器无法检查出错误，即不会对设置的键、键值路径进行错误检查。
2. 执行效率要低于 setter 和 getter 方法。因为使用 KVC 键值编码，它必须先解析字符串，然后在设置或者访问对象的实例变量。
3. 使用 KVC 会破坏类的封装性。

## KVO

KVO(Key-Value-Observing),用于观察键值。任何对象都允许观察其它对象的属性，并且可以接收其它对象的属性发生变化时的通知。它是一个观察者模式。

`NSKeyValueObserving `是`NSObject`的一个非正式协议。从协议的角度看，是定义了一套让开发者遵守的规范和使用的方法。在 Cocoa 的 MVC 框架中，架起 ViewController 和 Model 沟通的桥梁。

### KVO的使用

以下包含KVO常用的几种使用案例。

#### 注册于移除注册

比如我们现在有一个`Person`类，我们要观察`Person`(被观察者)类对象的变化，那么就可以在该类的对象上调用名为`NSKeyValueObserverRegistration`的category方法将观察者对象与被观察者对象注册与移除注册：

```
@interface NSObject(NSKeyValueObserverRegistration)

- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context;
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath context:(nullable void *)context NS_AVAILABLE(10_7, 5_0);
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;

@end
```

这两个方法的定义在 `Foundation/NSKeyValueObserving.h `中，`NSObject`，`NSArray`，`NSSet`均实现了以上方法，因此我们不仅可以观察普通对象，还可以观察数组或结合类对象。在该头文件中，我们还可以看到 NSObject 还实现了 `NSKeyValueObserverNotification` 的 category 方法。

#### 为属性赋值

将观察者与被观察者注册好之后，就可以对被观察者对象的属性进行操作，这些变更操作就会被通知给观察者对象。注意，只有遵循 KVO 方式来设置属性，观察者对象才会获取通知，也就是说遵循使用属性的 setter 方法，或通过 key-path 来设置：

```
    [person setAge:10];
    [person setValue:[NSNumber numberWithInt:10] forKey:@"age"];
```

#### 处理变更通知

观察者需要实现名为 `NSKeyValueObserving` 的 category 方法来处理收到的变更通知：

```
@interface NSObject(NSKeyValueObserving)
- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change context:(nullable void *)context;

@end

```

### 示例

#### 实现观察者类

首先定义一个观察者类：

```
/* PersonObserver.h */
#import <Foundation/Foundation.h>

@interface PersonObserver : NSObject

@end


/* PersonObserver.m */
#import "PersonObserver.h"

@implementation PersonObserver


- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    if ([keyPath isEqualToString:@"age"]) {
        Class classInfo = (__bridge Class)context;
        NSString *className = [NSString stringWithCString:object_getClassName(classInfo) encoding:NSUTF8StringEncoding];
        
        NSLog(@" >> class: %@, Age changed", className);
        NSLog(@" old age is %@", [change objectForKey:@"old"]);
        NSLog(@" new age is %@", [change objectForKey:@"new"]);
    }else if([keyPath isEqualToString:@"info"])
    {
        Class classInfo = (__bridge Class)context;
        NSString *className = [NSString stringWithCString:object_getClassName(classInfo) encoding:NSUTF8StringEncoding];
        NSLog(@" >> class: %@, Info changed", className);
        NSLog(@" old info is %@", [change objectForKey:@"old"]);
        NSLog(@" new info is %@", [change objectForKey:@"new"]);
        
    }else{
        [super observeValueForKeyPath:keyPath
                             ofObject:object
                               change:change
                              context:context];
    }
}
@end

```

一定要注意：在实现处理变更通知方法 observeValueForKeyPath 时，要将不能处理的 key 转发给 super 的 observeValueForKeyPath 来处理。

使用示例：

```
#import "ViewController.h"
#import "Person.h"
#import "PersonObserver.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    PersonObserver *observer = [[PersonObserver alloc] init];
    
    Person *person = [[Person alloc] init];
    [person addObserver:observer forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:(__bridge void * _Nullable)([Person class])];
    [person setAge:10];
//    [person setValue:[NSNumber numberWithInt:10] forKey:@"age"];
    [person removeObserver:observer forKeyPath:@"age"];

}
```
运行结果如下：

```
2017-08-23 16:15:53.420238+0800 DemoKVO[4320:1052355]  >> class: Person, Age changed
2017-08-23 16:15:53.420343+0800 DemoKVO[4320:1052355]  old age is 5
2017-08-23 16:15:53.420450+0800 DemoKVO[4320:1052355]  new age is 10
```

#### 实现被观察者类

上面的被观察者类(Person)如何实现呢？其实现方式有两种，一种是手动实现，另一种是自动实现。


1）手动实现

```
/* Person.h */
#import <Foundation/Foundation.h>

@interface Person : NSObject
{
    int age;
}

- (void)setAge:(int)tage;
@end


/* Person.m */
#import "Person.h"
@implementation Person

- (id)init
{
    self = [super init];
    if (self) {
        age = 5;
    }
    return self;
}

- (int)age
{
    return age;
}

- (void)setAge:(int)tage
{
    [self willChangeValueForKey:@"age"];
    age = tage;
    [self didChangeValueForKey:@"age"];
}

+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
{
    if ([key isEqualToString:@"age"]) {
        return NO;
    }
    return [super automaticallyNotifiesObserversForKey:key];
}

@end

```

首先，需要手动实现属性的 `setter `方法，并在设置操作的前后分别调用 `willChangeValueForKey:` 和 `didChangeValueForKey`方法，这两个方法用于通知系统该 key 的属性值即将和已经变更了；
其次，要实现类方法 `automaticallyNotifiesObserversForKey`，并在其中设置对该 key 不自动发送通知（返回 NO 即可）。这里要注意，对其它非手动实现的 key，要转交给 super 来处理。

2）自动实现

自动实现键值观察就非常简单了，只要使用了自动属性即可。(在这里我用Student类替代Person类)

```
/*  Student.h */
#import <Foundation/Foundation.h>

@interface Student : NSObject
@property (nonatomic,assign) int age;
@end

/*  Student.m */
#import "Student.h"

@implementation Student
@synthesize age;

- (id)init
{
    self = [super init];
    if (self) {
        age = 15;
    }
    return self;
}
@end
```

运行结果：

```
2017-08-23 16:15:53.420731+0800 DemoKVO[4320:1052355]  >> class: Student, Age changed
2017-08-23 16:15:53.420751+0800 DemoKVO[4320:1052355]  old age is 15
2017-08-23 16:15:53.420763+0800 DemoKVO[4320:1052355]  new age is 18
```

### 键值观察依赖键

有时候一个属性的值依赖于另一对象中的一个或多个属性，如果这些属性中任一属性的值发生变更，被依赖的属性值也应当为其变更进行标记。因此，object 引入了依赖键。

#### 观察依赖键

观察依赖键的方式与前面描述的一样，下面先在 Observer 的 `observeValueForKeyPath:ofObject:change:context:` 中添加处理变更通知的代码：

```
#import "PersonObserver.h"

@implementation PersonObserver


- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    if ([keyPath isEqualToString:@"age"]) {
        Class classInfo = (__bridge Class)context;
        NSString *className = [NSString stringWithCString:object_getClassName(classInfo) encoding:NSUTF8StringEncoding];
        
        NSLog(@" >> class: %@, Age changed", className);
        NSLog(@" old age is %@", [change objectForKey:@"old"]);
        NSLog(@" new age is %@", [change objectForKey:@"new"]);
    }else if([keyPath isEqualToString:@"info"])
    {
        Class classInfo = (__bridge Class)context;
        NSString *className = [NSString stringWithCString:object_getClassName(classInfo) encoding:NSUTF8StringEncoding];
        NSLog(@" >> class: %@, Info changed", className);
        NSLog(@" old info is %@", [change objectForKey:@"old"]);
        NSLog(@" new info is %@", [change objectForKey:@"new"]);
        
    }else{
        [super observeValueForKeyPath:keyPath
                             ofObject:object
                               change:change
                              context:context];
    }
}
@end
```

#### 实现依赖键

在这里，观察的是 `TeacherWrapper` 类的 `info` 属性，该属性是依赖于 `Teacher` 类的 `age` 和 `grade` 属性。为此，我在 `Teacher` 中添加了 `grade` 属性：

```
/* Teacher.h */
#import <Foundation/Foundation.h>

@interface Teacher : NSObject
@property (nonatomic,readwrite) int grade;
@property (nonatomic,readwrite) int age;
@end

/* Teacher.m */
#import "Teacher.h"

@implementation Teacher
@synthesize age;
@synthesize grade;
@end
```

下面来看看 `TeacherWrapper` 中的依赖键属性是如何实现的。

```
/* TeacherWrapper.h */
#import <Foundation/Foundation.h>
@class Teacher;
@interface TeacherWrapper : NSObject
{
    @private
    Teacher * _teacher;
}
@property (nonatomic,assign) NSString* info;
@property (nonatomic,retain) Teacher *teacher;

- (id)init:(Teacher *)aTeacher;
@end


/* TeacherWrapper.m */
#import "TeacherWrapper.h"
#import "Teacher.h"

@implementation TeacherWrapper

- (id)init:(Teacher *)aTeacher
{
    self = [super init];
    if (self) {
        _teacher = aTeacher;
    }
    return self;
}

- (NSString *)info
{
    return [[NSString alloc] initWithFormat:@"%d#%d", [_teacher grade], [_teacher age]] ;
}

- (void)setInfo:(NSString *)theInfo
{
    NSArray *array = [theInfo componentsSeparatedByString:@"#"];
    [_teacher setGrade:[[array objectAtIndex:0] intValue]];
    [_teacher setAge:[[array objectAtIndex:1] intValue]];
}

+ (NSSet *)keyPathsForValuesAffectingInfo
{
    NSSet * keyPaths = [NSSet setWithObjects:@"teacher.age", @"teacher.grade", nil];
    return keyPaths;
}

@end

```
首先，要手动实现属性 `info` 的 `setter/getter` 方法，在其中使用 `Teacher` 的属性来完成其 `setter` 和 `getter`。

其次，要实现 `keyPathsForValuesAffectingInformation` 或 `keyPathsForValuesAffectingValueForKey:` 方法来告诉系统 `info` 属性依赖于哪些其他属性，这两个方法都返回一个key-path 的集合。在这里要多说几句，如果选择实现 `keyPathsForValuesAffectingValueForKey`，要先获取 super 返回的结果 set，然后判断 key 是不是目标 key，如果是就将依赖属性的 key-path 结合追加到 super 返回的结果 set 中，否则直接返回 super的结果。
在这里，`info` 属性依赖于 `teacher` 的 `age` 和 `grade` 属性，`teacher` 的 `age/grade` 属性任一发生变化，`info` 的观察者都会得到通知。

使用实例：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    Teacher *teacher = [[Teacher alloc] init];
    TeacherWrapper *wrapper = [[TeacherWrapper alloc] init:teacher];
    [wrapper addObserver:observer forKeyPath:@"info" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:(__bridge void* _Nullable)([TeacherWrapper class])];
    [teacher setAge:30];
    [teacher setGrade:1];
    [wrapper removeObserver:observer forKeyPath:@"info"];
    
}

```

输出结果：

```
2017-08-23 16:15:53.421590+0800 DemoKVO[4320:1052355]  >> class: TeacherWrapper, Info changed
2017-08-23 16:15:53.421626+0800 DemoKVO[4320:1052355]  old info is 0#0
2017-08-23 16:15:53.421653+0800 DemoKVO[4320:1052355]  new info is 0#30
2017-08-23 16:15:53.421690+0800 DemoKVO[4320:1052355]  >> class: TeacherWrapper, Info changed
2017-08-23 16:15:53.421714+0800 DemoKVO[4320:1052355]  old info is 0#30
2017-08-23 16:15:53.421736+0800 DemoKVO[4320:1052355]  new info is 1#30
```

### KVO实现原理

当某个类的对象第一次被观察时，系统就会在运行期动态地创建该类的一个派生类，在这个派生类中重写基类中任何被观察属性的 setter 方法。 派生类在被重写的 setter 方法实现真正的通知机制，就如前面手动实现键值观察那样。这么做是基于设置属性会调用 setter 方法，而通过重写就获得了 KVO 需要的通知机制。当然前提是要通过遵循 KVO 的属性设置方式来变更属性值，如果仅是直接修改属性对应的成员变量，是无法实现 KVO 的。 同时派生类还重写了 class 方法以“欺骗”外部调用者它就是起初的那个类。然后系统将这个对象的 isa 指针指向这个新诞生的派生类，因此这个对象就成为该派生类的对象了，因而在该对象上对 setter 的调用就会调用重写的 setter，从而激活键值通知机制。此外，派生类还重写了 dealloc 方法来释放资源。

#### 派生类NSKVONotifying_Person剖析

在这个过程，被观察对象的 isa 指针从指向原来的 Person 类，被 KVO 机制修改为指向系统新创建的子类 NSKVONotifying_Person 类，来实现当前类属性值改变的监听。

所以当我们从应用层面上看来，完全没有意识到有新的类出现，这是系统“隐瞒”了对 KVO 的底层实现过程，让我们误以为还是原来的类。但是此时如果我们创建一个新的名为 NSKVONotifying_Person 的类()，就会发现系统运行到注册 KVO 的那段代码时程序就崩溃，因为系统在注册监听的时候动态创建了名为 NSKVONotifying_Person 的中间类，并指向这个中间类了。

因而在该对象上对 setter 的调用就会调用已重写的 setter，从而激活键值通知机制。这也是 KVO 回调机制，为什么都俗称 KVO 技术为黑魔法的原因之一吧：内部神秘、外观简洁。

#### 子类setter方法剖析

`KVO` 在调用存取方法之前总是调用 `willChangeValueForKey:`，通知系统该 keyPath 的属性值即将变更。 当改变发生后，`didChangeValueForKey:` 被调用，通知系统该 keyPath 的属性值已经变更。 之后，`observeValueForKey:ofObject:change:context:` 也会被调用。

重写观察属性的 `setter` 方法这种方式是在运行时而不是编译时实现的。 `KVO` 为子类的观察者属性重写调用存取方法的工作原理在代码中相当于：

```
- (void)setName:(NSString *)newName
{
    [self willChangeValueForKey:@"name"];    // KVO在调用存取方法之前总调用
    [super setValue:newName forKey:@"name"]; // 调用父类的存取方法
    [self didChangeValueForKey:@"name"];     // KVO在调用存取方法之后总调用
}
```

#### 总结

KVO 的本质就是监听对象的属性进行赋值的时候有没有调用 setter 方法

1. 系统会动态创建一个继承于 `Person` 的 `NSKVONotifying_Person`
2. `person` 的 `isa` 指针指向的类 `Person` 变成 `NSKVONotifying_Person`，所以接下来的 `person.age` = `newAge` 的时候，他调用的不是 `Person` 的 `setter` 方法，而是 `NSKVONotifying_Person`（子类）的 `setter` 方法
3. 重写`NSKVONotifying_Person`的`setter`方法：`[super setName:newName]`
4. 通知观察者告诉属性改变。

#### KVO监听数组

KVO 不能直接监听数组的变化，因为KVO监听的是内存地址的变化，数组的内存地址没有发生变化。所以利用KVO监听数组的变化时，需要对数组进行一个模型封装。具体代码如下：

```
//
//  ViewController.m
//  KVO_Demo01
//
//  Created by qi on 24/04/2018.
//  Copyright © 2018 tvu. All rights reserved.
//

#import "ViewController.h"

@interface Model : NSObject
@property (strong,nonatomic)NSMutableArray *modelArray;
@end

@implementation Model

-(NSMutableArray *)modelArray{
    if(!_modelArray){
        _modelArray = [NSMutableArray array];
    }
    return _modelArray;
}

@end

@interface ViewController ()

@property (strong,nonatomic)Model *model;

@end

@implementation ViewController

- (Model*)model
{
    if (!_model) {
        _model = [Model new];
        [_model addObserver:self forKeyPath:@"modelArray" options:NSKeyValueObservingOptionNew context:NULL];
    }
    return _model;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}

-(void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context{
    
    if ([keyPath isEqualToString:@"modelArray"]) {
        NSLog(@"%ld",self.model.modelArray.count);
    }
}

- (IBAction)onpressedButtonAddEle:(id)sender {
    NSObject *obj = [[NSObject alloc] init];
    [[self.model mutableArrayValueForKeyPath:@"modelArray"] addObject:obj];
}

- (IBAction)onpressedButtonInserEle:(id)sender {
    NSObject *obj = [[NSObject alloc] init];
    [[self.model mutableArrayValueForKeyPath:@"modelArray"] insertObject:obj atIndex:0];
}

- (IBAction)onpressedButtonRemoveEle:(id)sender {
    
    int arrayCount = (int)[[self.model mutableArrayValueForKeyPath:@"modelArray"] count];
    if (arrayCount == 0) {
        return;
    }
    
    [[self.model mutableArrayValueForKeyPath:@"modelArray"] removeObjectAtIndex:0];
}

- (IBAction)onpressedButtonAddArray:(id)sender {
    NSArray *elementArray = @[@"1",@"2",@"3"];
    [[self.model mutableArrayValueForKeyPath:@"modelArray"] addObjectsFromArray:elementArray];
}

- (IBAction)onpressedButtonAddArrayOnce
{
    NSArray *elementArray = @[@"1",@"2",@"3"];
      NSIndexSet *set = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange([self.model mutableArrayValueForKeyPath:@"modelArray"].count, elementArray.count)];
    [[self.model mutableArrayValueForKeyPath:@"modelArray"] insertObjects:elementArray atIndexes:set];
}

- (void)dealloc{
     [self.model removeObserver:self forKeyPath:@"modelArray"];
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

@end

```

对以上代码总结：
* 使用 mutableArrayValueForKey: 代理来获取 NSMutableArray 属性。
* 使用 addObjectsFromArray 向数组中添加元素时，每往数组中添加一个元素都会触发一次 KVO 的执行
* 从 KVO 的通知中可以获取触发这次通知的操作类型，这里是往数组中添加元素，kind 的数值是 2，即 NSKeyValueChangeInsertion
* 从 KVO 的通知中还可获取到新添加的对象以及该对象在数组中的索引值

如果我们想一次性往数组中加入多个元素(如 addObjectsFromArray )，但是只想让其触发一次 KVO 的执行。应该按照如下方法：

使用 NSMutableArray 的这个接口 ` - (void)insertObjects:(NSArray *)objects atIndexes:(NSIndexSet *)indexes`  。 具体实现看上述代码。




#### 代码下载

GitHub: 
[ios_kvc_kvo](https://github.com/MaxwellQi/ios_kvc_kvo)