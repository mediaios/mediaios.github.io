---
layout: post
title: Objective-C中的runtime
description: 介绍Objective-C中的runtime
category: blog
tag: iOS,Ojbective-C,KVO,KVC
---
## 概述

`Objective-C`是面相运行时的语言（runtime oriented language），就是说它会尽可能的把编译和链接时要执行的逻辑延迟到运行时。这就给了你很大的灵活性，你可以按需要把消息重定向给合适的对象，你甚至可以交换方法的实现，等等；这就需要使用runtime，runtime可以做对象自省查看他们正在做的和不能做的（don’t respond to）并且适当地分发消息。
`Objective-C`的Runtime是一个运行时库（Runtime Library），它是一个主要使用C和汇编写的库，为C添加了面相对象的能力并创造了`Objective-C`。这就是说它在类信息（Class Information）中被加载，完成所有的方法分发，方法转发，等等。

`Objective-C`是一门简单的语言，95%是C。只是在语言层面上加了些关键字和语法。真正让`Objective-C`如此强大的是它的运行时。它很小但却很强大。它的核心是消息分发。

Runtime是开源的，你可以去[apple runtime](https://opensource.apple.com/tarballs/objc4/)下载Runtime的源码。

这篇文章主要包含的内容有：

* 理解类和对象的本质
* runtime在开发中的应用

## 基础

### 认识类和对象的本质

描述`Objective-C`对象所有的数据结构定义都在Runtime的头文件里,例如 `id`,`Class`等。

#### id

`id`类型在`usr/include/objc/objc.h`头文件中被定义如下：

```
/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

`objc_object`又被定义为如下：

```
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

```

由此可见，每个对象结构体的首个成员是Class类的变量。该变量定义了对象所属的类，通常称为isa指针。

#### Class

`Class`类型在`usr/include/objc/objc.h`头文件中被定义如下：

```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
```

`objc_class`被定义在`usr/include/objc/runtime.h`文件中，内容如下：

```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
```

对`objc_class`中的变量说明：

* isa : 结构体的首个变量也是isa指针，这说明Class本身也是Objective-C中的对象。类作为对象时的 isa 指针指向的是元类(Meta Class)。
* super_class : 它定义了本类的超类。类对象所属类型（isa指针所指向的类型）是另外一个类，叫做“元类”
* ivars : 成员变量列表，类的成员变量都在ivars里面
* methodLists : 方法列表，类的实例方法都在methodLists里，类方法在元类的methodLists里面。methodLists是一个指针的指针，通过修改该指针指向指针的值，就可以动态的为某一个类添加成员方法。这也就是Category实现的原理，同时也说明了Category只可以为对象添加成员方法，不能添加成员变量。
* cache : 方法缓存列表，objc_msgSend（下文详解）每调用一次方法后，就会把该方法缓存到cache列表中，下次调用的时候，会优先从cache列表中寻找，如果cache没有，才从methodLists中查找方法。提高效率。

#### 元类(Meta Class)

上面讲到一个类也是一个对象，那么它必然也是某一种类的实例，这种类就是：元类(Meta Class)。就如类是对应的实例的描述一样，元类则是类作为对象时的描述。元类的方法列表对应的则是类方法(Class Method)列表，这正是类作为一个对象时所需要的。当我们像 `[NSObject alloc]` 这样给一个类发送消息时，Runtime 就会去对应的元类查找其类方法列表，并匹配调用。

那在接着往下探究：元类又是谁的实例呢？它的 isa 又指向谁呢？答案如下图所示。

![](http://upload-images.jianshu.io/upload_images/1321491-dda0360cd4769dbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

元类的 isa 都指向根元类(Root Meta Class)，也就是说元类都是根元类(Root Meta Class)的实例。而根元类(Root Meta Class)的 isa 则指向自己，这样就不会无休止的链下去了。

在图中还能看到类的继承关系以及对应的元类的继承关系，已经比较清晰了，不再详述。

#### SEL

SEL是选择子的类型，选择子指的就是方法的名字。在`usr/include/objc/objc.h`的头文件中的定义如下：

```
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```

它就是个映射到方法的C字符串，SEL类型代表着方法的签名，在类对象的方法列表中存储着该签名与方法代码的对应关系，每个方法都有一个与之对应的SEL类型的对象，根据一个SEL对象就可以找到方法的地址，进而调用方法。

#### Method

Method代表类中的某个方法的类型，在`usr/include/objc/runtime.h`的头文件中的定义如下：

```
/// An opaque type that represents a method in a class definition.
typedef struct objc_method *Method;
```

objc_method 在`runtime.h`中定义如下：

```
struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

```

* method_name : 方法名
* method_types: 方法类型，主要存储着方法的参数类型和返回值类型
* method_imp : 方法的实现，函数指针

` class_copyMethodList(Class cls, unsigned int *outCount)` 可以使用这个方法获取某个类的成员方法列表。

#### Ivar

Ivar代表类中实例变量的类型,在runtime头文件中定义如下：

```
/// An opaque type that represents an instance variable.
typedef struct objc_ivar *Ivar;
```

objc_ivar的定义如下：

```
struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
} 
```

`class_copyIvarList(Class cls, unsigned int *outCount)` 可以使用这个方法获取某个类的成员变量列表。

#### objc_property_t

objc_property_t是属性，在runtime的头文件中的的定义如下：

```
/// An opaque type that represents an Objective-C declared property.
typedef struct objc_property *objc_property_t;
```

`class_copyPropertyList(Class cls, unsigned int *outCount)` 可以使用这个方法获取某个类的属性列表。

#### IMP

IMP在runtime的头文件中的的定义如下：

```
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif

```

IMP是一个函数指针，它是由编译器生成的。当你发起一个消息后，这个函数指针决定最终执行哪段代码。


#### Cache

Cache在runtime的头文件中的的定义如下：

```
typedef struct objc_cache *Cache                             OBJC2_UNAVAILABLE;
```

objc_cache的定义如下：

```
struct objc_cache {
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    Method buckets[1]                                        OBJC2_UNAVAILABLE;
};
```

每调用一次方法后，不会直接在isa指向的类的方法列表（methodLists）中遍历查找能够响应消息的方法，因为这样效率太低。它会把该方法缓存到cache列表中，下次的时候，就直接优先从cache列表中寻找，如果cache没有，才从isa指向的类的方法列表（methodLists）中查找方法。提高效率。

### 发送消息(objc_msgSend)

在Objective-C中，调用方法是经常使用的。用Objective-C的术语来说，这叫做“传递消息”（pass a message）。消息有“名称”（name）或者“选择子”（selector），也可以接受参数，而且可能还有返回值。
如果向某个对象传递消息，在底层，所有的方法都是普通的C语言函数，然而对象收到消息之后，究竟该调用哪个方法则完全取决于运行期决定，甚至可能在运行期改变，这些特性使得Objective-C变成一门真正的动态语言。
给对象发送消息可以这样来写：

```
id returnValue = [someObject message:parm];
```

someObject叫做“接收者”（receiver），message是“选择子”（selector），选择子和参数结合起来就叫做“消息”（message）。编译器看到此消息后，将其转换成C语言函数调用，所调用的函数乃是消息传递机制中的核心函数，叫做`objc_msgSend`，其原型如下：

```
id objc_msgSend (id self, SEL _cmd, ...);
```

后面的...表示这是个“参数个数可变的函数”，能接受两个或两个以上的参数。第一个参数是接收者（receiver），第二个参数是选择子（selector），后续参数就是消息中传递的那些参数（parm），其顺序不变。

编译器会把上面的那个消息转换成:

```
id returnValue objc_mgSend(someObject, @selector(message:), parm);
```

传递消息的几种函数：

* `objc_msgSend` : 普通的消息都会通过该函数发送
* `objc_msgSend_stret` : 消息中有结构体作为返回值时，通过此函数发送和接收返回值。
* `objc_msgSend_fpret ` : 消息中返回的是浮点数，可交由此函数处理。
* `objc_msgSendSuper ` : 和`objc_msgSend`类似，这里把消息发送给超类。
* `objc_msgSendSuper_stret ` : 和`objc_msgSend_stret`类似，这里把消息发送给超类。
* `objc_msgSendSuper_fpret ` : 和`objc_msgSend_fpret`类似，这里把消息发送给超类。

编译器会根据情况选择一个函数来执行。`objc_msgSend`发送消息的原理：

* 第一步：检测这个selector是不是要被忽略的。
* 第二步：检测这个target对象是不是nil对象。（nil对象执行任何一个方法都不会Crash，因为会被忽略掉）
* 第三步：首先会根据target对象的isa指针获取它所对应的类（class）。
* 第四步：优先在类（class）的cache里面查找与选择子（selector）名称相符，如果找不到，再到methodLists查找。
* 第五步：如果没有在类（class）找到，再到父类（super_class）查找，再到元类（metaclass），直至根metaclass。
* 第六步：一旦找到与选择子（selector）名称相符的方法，就跳至其实现代码。如果没有找到，就会执行消息转发（message forwarding）。

### 消息转发

当一个对象在收到无法解读的消息之后，它会将消息实施转发。转发的主要步骤如下：
消息转发步骤:

* 第一步：对象在收到无法解读的消息后，首先调用`resolveInstanceMethod：`方法决定是否动态添加方法。如果返回YES，则调用`class_addMethod`动态添加方法，消息得到处理，结束；如果返回NO，则进入下一步；
* 第二步：当前接收者还有第二次机会处理未知的选择子，在这一步中，运行期系统会问：能不能把这条消息转给其他接收者来处理。会进入`forwardingTargetForSelector:`方法，用于指定备选对象响应这个selector，不能指定为self。如果返回某个对象则会调用对象的方法，结束。如果返回nil，则进入下一步；
* 第三步：这步我们要通过`methodSignatureForSelector:`方法签名，如果返回nil，则消息无法处理。如果返回methodSignature，则进入下一步；
* 第四步：这步调用`forwardInvocation：`方法，我们可以通过anInvocation对象做很多处理，比如修改实现方法，修改响应对象等，如果方法调用成功，则结束。如果失败，则进入`doesNotRecognizeSelector`方法，抛出异常，此异常表示选择子最终未能得到处理。

```
/**
 消息转发第一步：对象在收到无法解读的消息后，首先调用此方法，可用于动态添加方法，方法决定是否动态添加方法。如果返回YES，则调用class_addMethod动态添加方法，消息得到处理，结束；如果返回NO，则进入下一步；
 */
+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    return NO;
}

/**
 当前接收者还有第二次机会处理未知的选择子，在这一步中，运行期系统会问：能不能把这条消息转给其他接收者来处理。会进入此方法，用于指定备选对象响应这个selector，不能指定为self。如果返回某个对象则会调用对象的方法，结束。如果返回nil，则进入下一步；
 */
- (id)forwardingTargetForSelector:(SEL)aSelector
{
    return nil;
}

/**
 这步我们要通过该方法签名，如果返回nil，则消息无法处理。如果返回methodSignature，则进入下一步。
 */
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    if ([NSStringFromSelector(aSelector) isEqualToString:@"study"])
    {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

/**
 这步调用该方法，我们可以通过anInvocation对象做很多处理，比如修改实现方法，修改响应对象等，如果方法调用成功，则结束。如果失败，则进入doesNotRecognizeSelector方法。
 */
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    [anInvocation setSelector:@selector(play)];
    [anInvocation invokeWithTarget:self];
}

/**
 抛出异常，此异常表示选择子最终未能得到处理。
 */
- (void)doesNotRecognizeSelector:(SEL)aSelector
{
    NSLog(@"无法处理消息：%@", NSStringFromSelector(aSelector));
}

```

![](http://upload-images.jianshu.io/upload_images/1321491-98dff019ac1cbe31.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意： 接收者在每一步中均有机会处理消息，步骤越靠后，处理消息的代价越大。最好在第一步就能处理完，这样系统就可以把此方法缓存起来了。

### 方法交换

在Objective-C中，对象收到消息之后，究竟会调用哪种方法需要在运行期才能解析出来。查找消息的唯一依据是选择子(selector)，选择子(selector)与相应的方法(IMP)对应，利用Objective-C的动态特性，可以实现在运行时偷换选择子（selector）对应的方法实现，这就是方法交换（method swizzling）。
每个类都有一个方法列表，存放着selector的名字和方法实现的映射关系。IMP有点类似函数指针，指向具体的Method实现。

![类的方法列表会把每个选择子都映射到相关的IMP之上](http://upload-images.jianshu.io/upload_images/1321491-fab02075750e2129.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

类的方法列表会把每个选择子都映射到相关的IMP之上

![](http://upload-images.jianshu.io/upload_images/1321491-34dff4504826ae5c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以新增选择子，也可以改变某个选择子所对应的方法实现，还可以交换两个选择子所映射到的指针。

#### 方法交换的API

Objective-C中提供了三种API来动态替换类方法或实例方法的实现：

1. `class_replaceMethod`替换类方法的定义。
	
	```
	class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
	```
	
2. `method_exchangeImplementations`交换两个方法的实现。

	```
	method_exchangeImplementations(Method m1, Method m2)
	```
	
3. `method_setImplementation`设置一个方法的实现

	```
	method_setImplementation(Method m, IMP imp)
	```

先说下这三个方法的区别：

* `class_replaceMethod：`当类中没有想替换的原方法时，该方法调用`class_addMethod`来为该类增加一个新方法，也正因如此，`class_replaceMethod`在调用时需要传入types参数，而其余两个却不需要。
* `method_exchangeImplementations：`内部实现就是调用了两次`method_setImplementation`方法。

使用场景：

```
+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{

        SEL originalSelector = @selector(willMoveToSuperview:);
        SEL swizzledSelector = @selector(myWillMoveToSuperview:);

        Method originalMethod = class_getInstanceMethod(self, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(self, swizzledSelector);

        BOOL didAddMethod = class_addMethod(self, 
                                            originalSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(self, 
                                swizzledSelector, 
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)myWillMoveToSuperview:(UIView *)newSuperview
{
    NSLog(@"WillMoveToSuperview: %@", self); 
    [self myWillMoveToSuperview:newSuperview];
}
```

#### 总结

1. `class_replaceMethod`，当需要替换的方法有可能不存在时，可以考虑使用该方法。
2. `method_exchangeImplementations`，当需要交换两个方法的时使用。
3. `method_setImplementation`是最简单的用法，当仅仅需要为一个方法设置其实现方式时实现。

## 示例

### 实例、类、父类、元类关系结构示例代码

我创建了两个类，其继承关系是`Chinese->People->NSObject`。 下面就用runtime提供的方法来打印相关的信息。

```
#import "ViewController.h"
#import <objc/runtime.h>
#import "People.h"
#import "Chinese.h"

@interface ViewController ()

@end

@implementation ViewController


/* 类、父类、元类关系结构的示例代码 */
- (void)testClass
{
    Chinese *ch = [[Chinese alloc] init];
    NSLog(@"获取Chinese对象ch所属的类是：%@,其父类是：%@",object_getClass(ch),class_getSuperclass(object_getClass(ch)));
    Class cls = objc_getMetaClass("Chinese");

    NSLog(@"元类是：%@, 元类的父类：%@, 元类的isa:%@",cls,class_getSuperclass(cls),object_getClass(cls));

    People *peo = [[People alloc] init];
    NSLog(@"获取People对象peo所属的类是：%@,其父类是：%@",object_getClass(peo),class_getSuperclass(object_getClass(peo)));
    cls = objc_getMetaClass("People");
    NSLog(@"元类是：%@, 元类的父类：%@, 元类的isa:%@",cls,class_getSuperclass(cls),object_getClass(cls));

    
    
    cls = objc_getMetaClass("UIView");
    NSLog(@"元类是： %@,父类是：%@, cls的isa是： %@", cls, class_getSuperclass(cls), object_getClass(cls)); // Print: YES, UIView, UIResponder, NSObject

    
    cls = objc_getMetaClass("NSObject");
    NSLog(@"元类是： %@, 父类是：%@, cls的isa是：%@", cls, class_getSuperclass(cls), object_getClass(cls)); // Print: YES, NSObject, NSObject, NSObject
}

```

打印信息如下：

```
获取Chinese对象ch所属的类是：Chinese,其父类是：People
元类是：Chinese, 元类的父类：People, 元类的isa:NSObject
获取People对象peo所属的类是：People,其父类是：NSObject
元类是：People, 元类的父类：NSObject, 元类的isa:NSObject
元类是： UIView,父类是：UIResponder, cls的isa是： NSObject
元类是： NSObject, 父类是：NSObject, cls的isa是：NSObject
```

`object_getClass()`可以获得当前对象 `isa`。这里以 `Chinese` 相关的打印信息为例，来解释一下：

```
元类是：Chinese, 元类的父类：People, 元类的isa:NSObject
```

首先我们通过 object_getClass() 获取实例 sub 所属的 Class(isa) 是 `Chinese`；通过 class_getSuperclass() 我们可以获取 `Chinese` 对应的父类是 `People`；通过 objc_getMetaClass() 指定类名，我们可以获取对应的元类，通过 class_isMetaClass() 我们可以判断一个 Class 是否为元类 。


### 动态操作类与实例

```
- (void)runtimeConstuct
{
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
    
    
    // 创建并注册一个类，并往类中添加方法
    Class cls = objc_allocateClassPair(People.class, "Janpanese", 0);
    class_addMethod(cls, @selector(testRuntimeMethod), (IMP)testRuntimeMethodIMP, "i@:@");
    objc_registerClassPair(cls);
    
    
    // 2: Create instance of class, print some info about class and associated meta class.
    id sub = [[cls alloc] init];
    NSLog(@"类是：%@, 父类是：%@", object_getClass(sub), class_getSuperclass(object_getClass(sub))); // Print: Janpanese, People
    Class metaCls = objc_getMetaClass("Janpanese");
    NSLog(@"元类是：%@, 父类是：%@,metaCls的isa是：%@", metaCls, class_getSuperclass(metaCls), object_getClass(metaCls)); // Print: YES, Janpanese, SuperClass, NSObject

    
    
    // 3: Methods of class.
    unsigned int outCount = 0;
    Method *methods = class_copyMethodList(cls, &outCount);
    for (int32_t i = 0; i < outCount; i++) {
        Method method = methods[i];
        NSLog(@"方法名：%@, %s", NSStringFromSelector(method_getName(method)), method_getTypeEncoding(method));
    }
    // Print: testRuntimeMethod, i@:@
    free(methods);
    
    
    // 4: Call method.
    int32_t result = (int) [sub performSelector:@selector(testRuntimeMethod) withObject:@{@"a":@"para_a", @"b":@"para_b"}];
    NSLog(@"函数返回值：%d", result); // Print: 99
    
    
    // 5: Destory instances and class.
    // Destroy instances of cls class before destroy cls class.
    sub = nil;
    // Do not call this function if instances of the cls class or any subclass exist.
    objc_disposeClassPair(cls);
    
#pragma clang diagnostic pop
    
}

```

打印信息如下：

```
类是：Janpanese, 父类是：People
元类是：Janpanese, 父类是：People,metaCls的isa是：NSObject
方法名：testRuntimeMethod, i@:@
testRuntimeMethodIMP : {
    a = "para_a";
    b = "para_b";
}
函数返回值：99
```

在上面的代码中，我们在运行时动态创建了 `People` 的一个子类：`Janpanese`；接着为这个类添加了方法和实现；打印了 `Janpanese` 的类、父类、元类相关信息；遍历和打印了 `Janpanese` 的方法的相关信息；调用了 `Janpanese` 的方法；最后销毁了实例和类。

对上面代码的说明：

* 我们看到了几行` #pragma clang diagnostic...` 代码，这是用于忽略编译器对于未声明的 @selector 的 warning。因为我们的代码中我们需要动态的为一个类创建方法，所以必然不会事先声明。
* `class_addMethod()` 函数的最后一个参数 types 是描述方法返回值和参数列表的字符串，我们的代码中的用到的 `i@:@` 四个字符分别对应着：返回值 int32_t、参数 id self、参数 SEL _cmd、参数 NSDictionary *dic。这个其实就是类型编码(Type Encoding)的概念。在 Objective-C 中，为了协助 Runtime 系统，编译器会将每个方法的返回值和参数列表编码为一个字符串，这个字符串会与方法对应的 selector 关联。更详细的知识可以查阅[ Type Encodings](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)
* 使用 `objc_registerClassPair()` 函数需要注意，你不能注册已经注册过的类。
* 使用 `objc_disposeClassPair()` 函数需要注意，如果一个类的实例和子类还存在时，不要去销毁一个类。

关于更多 Runtime 函数的使用细节可以查阅 [Objective-C Runtime Reference](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/index.html)
