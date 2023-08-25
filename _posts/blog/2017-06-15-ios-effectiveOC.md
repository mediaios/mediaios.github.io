---
layout: post
title: Effective Objective-C 2.0
description: Effective Objective-C 2.0读书笔记
category: blog
tag: iOS,Ojbective-C
---

## 说明

这是我对《Objective-C高级编程：iOS与OS X多线程和内存管理》的阅读笔记。

## OC中的属性

### 属性特质

在为OC中的一个类定义属性时，一定要注意。因为属性的各种特质设定会影响编译器的存取方法。

```
@property (nonatomic,readwrite,copy) NSString *firstName;
```

属性可以拥有的特质分为四类：

**原子性**

在默认情况下，有编译器合成的方法会通过锁定机制确保其原子性(atomic)。如果属性具备nonatomic特质，则不使用同步锁。

** 内存管理语义**

* assign "设置方法"只会执行针对“纯量类型”（例如：CGFloat,NSInteger等）的简单赋值操作
* strong 此特质表明该属性定义了一种“拥有关系”。为这种属性设置新值时，设置方法会先保留新值，并释放旧值，然后再将新值设置上去。
* weak 次特质表明该属性定义了一种“非拥有关系”。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似，然而在属性所指的对象遭到摧毁是，属性值也会清空。
* unsafe_unretained  此特质的语义和assign相同，但是它适用于“对象类型”，该特质表明一种非拥有关系，当目标对象遭到摧毁时，属性不会自动清空，这一点和weak有区别。
* copy  此特质表达的属性关系与strong类似。然而设置方法并不保留新值，而是“拷贝”。当属性类型为NSString*时，经常用此特质来保护其封装性，因为传递给设置方法的新值有可能指向一个NSMutableString类的实例。这个类是NSString的子类，表示一种可以修改其值的字符串，此时若不是拷贝字符串，那么设置完属性后，字符串的值就可能会在对象不知情的情况下遭人更改。所以，这时就要拷贝一份“不可变”的字符串，确保对象中的字符串值不会无意间变动。只要实现属性所用的对象是“不可变的”，就应该在设置新属性值是拷贝一份。

小示例：

```
@interface EOCPerson : NSManagedObject

@property (copy) NSString *firstName;
@property (copy) NSString *lastName;

- (id)initWithFirstName:(NSString *)firstName lastName:(NSString *)lastName;

@end

// 在实现类中可以这样写
- (id)initWithFirstName:(NSString *)firstName lastName:(NSString *)lastName
{
    if(self = [super init])
    {
        _firstName = [firstName copy];
        _lastName = [lastName copy];
    }
    return self;
}
```

**atomic和nonatomic**

具备atomic特质的获取方法会通过锁定机制来确保其操作的原子性。也就是说如果两个线程读写同一属性，那么不论何时总能看到有效的属性值。如果使用nonatomic的话，那么当其中一个线程正在改写某属性值是，另一个线程可能会突然闯入，把尚未修改好的属性值读取出来。发生这种情况时，线程读到的属性值可能不对。

在ios开发中，几乎所有的属性都声明成nonatomic。这样做是历史原因：在ios中，同步锁的开销较大，这样会带来性能问题。一般情况下并不要求属性必须是原子的，因为这样并不能保证线程安全。若要实现线程安全操作，还需采用更深层的锁定机制才行。在ios开发中，一般都会使用nonatomic属性，但是在Mac OS X开发中，使用atomic属性通常不会有性能瓶颈。


## 比价对象相等

### 对象相等判断规则

我们知道：

* `==`操作符比较的是两个指针是否相等，而不是指所指的对象是否相等。
* 应该使用NSObject中的`isEquals`方法来判断两个对象是否相等。

### 比较对象相等的方法

我们知道，两个类型不同的对象是不相等的，某些类型的对象已经提供了对象相等的判断方法。如果现在已经知道两个对象的类型相同，那么就可以使用下面方法判断两个对象是否相等：

```
NSString *foo = @"Badger 123";
NSString *bar = [NSStringsstringWithFormat:@"Badger %i",123];
BOOL equalsA = [foo == bar]; //  equalsA = NO
BOOL equalsB = [foo isEqual:bar]; // equalsB = YES
BOOL equalsC = [foo isEqualsToString:bar]; // eaualsC = YES;
```

NSObject 协议中有两个用于判断等同性的关键方法：

```
- (BOOL)isEqual:(id)object;
- (NSUInteger)hash;
```

NSObject类对这两个方法的默认实现是：当且仅当其“指针值”完全相等时，这两个对象才相等。若想在自定义的对象中正确复写这些方法，就必须先理解其规则。如果`isEqual:`方法判定两个对象相等，那么其`hash`方法也必须返回同一个值。但是如果两个对象的hash方法返回同一个值，那么`isEqual:`方法未必会认为两者相等。

比如有下面这个类：
```
@interface EOCPerson : NSObject
@property (nonatomic,copy) NSString *firstName;
@property (nonatomic,copy) NSString *lastName;
@property (nonatomic,assign) NSUInteger age;
@end

```
我们认为，如果两个EOCPerson对象的所有字段均相等，那么这两个对象就相等。于是`isEqual:`方法可以写成：

```
- (BOOL)isEqual:(id)object
{
    if(self == object) return YES;
    if([self calss] != [object class]) return NO;
    
    EOCPerson *otherPerson = (EOPerson *)object;
    if(![_firstName isEqualToString:otherPerson.firstName]) return NO;
    if(![_lastName isEqualToString:otherPerson.lastName]) return NO;
    if(_age != otherPerson.age) return NO;
    
    return YES;
}

- (NSUInteger)hash
{
    return 1337;
}
```
上面的写法不是最合适的，因为在collection中使用这种对象将会产生性能问题。因为collection在检索哈希表时，会用对象的哈希码做索引。假如collection是用set实现的，那么set可能会根据哈希码把对象分装到不同的数组中。在向set中添加新对象是，要根据其哈希码找到与之相关的那个数组，一次检查其中各个元素，看数组中已有的对象是否和将要添加的新对象相等
。如果相等，那就说明要添加的对象已经在set里面了。由此可知，如果令每个对象都返回相同的哈希码，那么在set中已有1000000个对象的情况下，若是继续向其中添加对象，则需要将这10000000个对象全部扫描一遍。下面是计算哈希码的另一种搞笑方法：

```
- (NSUInteger)hash
{
    NSUInteger firstNameHash = [_firstName hash];
    NSUInteger lastNameHash = [_lastName hash];
    NSUInteger ageHash = _age;
    return firstNameHash ^ lastNameHash ^ ageHash;
}
```
这种方法既能保持较高的效率，又能使生成的哈希码至少位于一定范围之内，而不会过于频繁重复。当然，此算法的哈希码还是会碰撞，不过至少可以保证哈希码有多种可能的取值。

## 以“类族”模式，隐藏实现细节

“类族”是一种很有用的模式，可以隐藏“抽象基类”背后的实现细节。OC的系统框架普遍使用此模式。

### 创建类族

假设现在有一个处理创建雇员的类，每个雇员都有“名字”和“薪水”两个属性，管理者可以命令其执行日常工作。但是，各种雇员的工作内容却不相同。经理在带领雇员做项目时，无需关心每个人如何完成其工作，仅需只是其开工即可。

首先定义抽象基类：
```
typedef NS_ENUM(NSUInteger,EOCEmployeeType)
{
    EOCEmployeeTypeDeveloper,
    EOCEmployeeTypeDesigner,
    EOCEmployeeTypeFinance
};

@interface EOCEmployee:NSObject
@property (copy) NSString *name;
@property NSUInteger salary;

+ (EOCEmployee *)employeeWithType:(EOCEmployeeType)type;
- (void)doADaysWork;
@end

@implementation EOCEmployee

+ (EOCEmployee *)employeeWithType:(EOCEmployeeType)type
{
    switch(type){
        case EOCEmployeeTypeDeveloper:
            return [EOCEmployeeTypeDeveloper new];
            break;
        case EOCEmployeeTypeDesigner:
            return [EOCEmployeeTypeDesigner new];
            break;
        case EOCEmployeeTypeFinance:
            return [EOCEmployeeTypeFinance new];
            break;
    }
}

- (void)doADaysWork
{
    // subclass implement this
}
@end

```

每个“实体子类”都从基类继承而来。例如：
```
@interface EOCEmployeeDeveloper : EOCEmployee
@end

@implementation EOCEmployeeDeveloper

- (void)doADaysWork
{
    [self wirteCode];
}

@end
```

### cocoa里的类族

若要判断某对象是否位于类族中，不要直接检测两个“类对象”是否相等，应该采用下列代码：

```
id mybeAnArray = /* ... */;
if([mybeAnArray isKindOfClass:(NSArray class)]){
    // will be hit
}
```

## 理解ojjc_msgSend的作用

在对象上调用方法是OC中经常使用的功能。用OC属于来说，这叫“传递消息”。消息有“名称”或“选择子”，可以接受参数，而且可能还有返回值。

在OC中，如果向某对象传递消息，那就会使用动态绑定机制决定需要调用的方法。在底层，所有方法都是普通的C语言函数，然而对象收到消息后，究竟该调用哪个方法则完全于运行期决定，甚至可以在程序运行时改变，这些特性使得OC称为一门真正的动态语言。给对象发消息，可以这样写：

```
id returnValue = [someObject messageName:parameter];
```

在本例中，someObject叫做“接收者”，messageName叫做“选择子”。选择子与参数合起来称为“消息”。编译器看到此消息后，将其转换为一条标准的C语言函数调用，所调用的函数乃是消息传递机制中的核心函数，叫做 objc_msgSend,其原型如下：

```
void objc_msgSend(id self,SEL cmd,...);
```

这是个参数可变的函数，能接受两个或两个以上的参数。第一个参数代表接收者，第二个参数代表选择子，后续参数就是消息中的那些参数，其顺序不变。选择子值得就是方法的名字。编译器会把刚才那个例子中的消息转换为如下函数：

```
id returnValue = objc_msgSend(  someObject,
                                @selector(messageName:)
                                parameter);
```

前面讲的这部分内容只描述了部分消息的调用过程，其它“边界情况”则需要交由OC运行环境中的另一些函数来处理。

* objc_msgSend_stret  如果待发送的消息要返回结构体，那么可交由此函数处理。
* objc_msgSend_fpret  如果消消息返回的是浮点数，那么可交由此函数处理。
* objc_msgSendSuper   如果要给超累发消息，例如[super message:parameter]，那么就交由此函数处理。

## 消息转发机制

若想令类能理解某条消息，我们必须以程序代码实现出对应的方法才行。但是，在编译期向类发送了其无法解读的消息并不会报错，因为在运行期可以继续向类中添加方法，所以编译器在编译时还无法确知类中到底会不会有某个方法的实现。当对象接收到无法解读的消息后，就会启动“消息转发”机制，程序员可经由此过程告诉对象应该如何处理未知消息。

例如当你在控制台中看到以下消息时，就是消息转发处理的消息：

```
- [_NSCFNumber lowercaseString]: unrecognized selector sent to 
instance 0x87
*** Terminating app due to uncaught exception
'NSInvalidArgumentException',reason: '-[_NSCFNumber lowercaseString]: unrecognized selector sent to instance 0x87'
```
上面这段异常信息是由NSObject的“doesNotRecognizeSelector:”方法抛出的。本例所列举的异常并不奇怪，因为NSNumber类里面本来就没有名为lowercaseString的方法。控制台中看到的那个__NSFCNumber是为了实现“无缝桥接”而使用的内部类，配置NSNumber对象是也会一并创建此对象。在本例中，消息转发过程以应用程序崩溃而告终，不过开发者在编写自己的类时，可于转发过程中设置挂钩，泳衣执行预定的逻辑，而不使应用程序崩溃。

消息转发分为两大阶段。第一阶段先征询接收者所属的类，看其是否能动态添加方法，以处理当前这个“未知的选择子”，这叫做“动态方法解析”。第二阶段涉及“完整的消息转发机制”。如果运行期系统已经把第一阶段执行完了，那么接收者自己就无法再以动态新增方法的手段来响应包含该选择子的消息了。此时，运行期系统会请求接收者以其他手段来处理消息相关的方法调用。这又细分两小步。首先，请接收者看看有没有其他对象能处理这条消息。若有，则运行期系统会把消息转给那个对象，于是消息转发过程结束，一切如常。若没有“备援的接收者”，则启动完整的消息转发机制，运行期系统会把与消息有关的全部细节都封装到NSInvocation对象中，再给接收者最后一次机会，令其设法解决当前还未处理的这条消息。

### 动态方法解析

对象在收到无法解读的消息后，首先将调用其所属类的下列方法：

```
+ (BOOL)resolveInstanceMethos:(SEL)selector;
```
该方法的参数就是那个未知的选择子，其返回值为BOOL类型，表示这个类是否能新增一个示例方法用以处理此选择子。在继续往下执行转发机制前，本类有机会新增一个处理此选择子的方法。假如尚未实现的方法不是实例方法而是类方法，那么运行期系统就会调用另外一个方法，该方法，叫做`resolveClassMethod:` 。

使用这种办法的前提是：相关方法的实现代码已经写好，只等着运行的时候动态插在类里面就可以了。此方案常用来实现@dynamic属性，比如说,要访问CoreData框架中NSManagerObjects对象的属性时就可以这么做，因为实现这些属性所需的存取方法在编译器就能确定。

下列代码演示了如何用“resolveInstanceMethod:”来实现@dynamic属性：

```
id autoDictionaryGetter(id self,SEL _cmd);
void autoDictionarySetter(id self,SEL _cmd,id value);

+ (BOOL)resolveInstanceMethod:(SEL)selector
{
    NSString *selectorString = NSStringFromSeletor(selector);
    if(/* selector is from a @dynamic property */){
        if([selectorString hasPrefix:@"set"]){
            class_addMethod(self,
                            selector,
                            (IMP)autoDictionarySetter,
                            "v@:@");
        }else{
            class_addMethod(self,
                            selector,
                            (IMP)autoDictionaryGetter,
                            "@@:");
        }
        return YES;
    }
    return [super resolveInstanceMethod:selector];
}
```

首先将选择子化为字符串，然后检测其是否表示设置方法。若前缀为set，则表示设置方法，否则就是获取方法。不管哪种情况，都会把处理该选择子的方法加到类里面，所添加的方法是用纯C函数实现的。

### 备援接收者

当前接收者还有第二次机会能处理未知的选择子，在这一步中，运行期系统会问它：能不能把这条消息转给其它接收者来处理。与该步骤对应的处理方法如下：

```
- (id)forwardingTargetForSelector:(SEL)selector
```

方法参数代表未知的选择子，若当前接收者能找到备选对象，则将其返回，若找不到，就返回nil。 通过此方案，我们可以用“组合”来模拟出“多重继承”的某些特性。在一个兑现国内部，可能还有一些列其它对象，该对象可经由此方法将能够处理某些选择子的相关内部对象返回，这样的话，在外界看来，好像是该对象亲自处理了这些消息似得。

### 完整的消息转发机制

如果转发算法已经来到这一步的话，那么唯一能做的就是穷完整的消息转发机制了。首先创建NSInvocation对象，把与尚未处理的那条消息有关的全部细节都封于其中。此对象包含选择子、目标及参数。在触发NSInvocation对象时，“消息派发系统”将亲自出马，把消息指派给目标对象。

此步骤会调用下列方法来转发消息：
```
- (void)forwardInvocation:(NSInvocation *)invocation
```

这个方法可以实现得很简单：秩序改变调用目标，事消息在新目标上得以调用即可。

实现此方法时，若发现某调用操作不应由本类处理，则需调用超类的同名方法。这样的话，集成体系中的每个类都有机会处理此调用请求，直至NSObjet 。 如果最后调用了NSObject类的方法，那么该方法还会继而调用`doesNotRecognizeSelector:`以抛出异常，此异常表明选择子最终未能得到处理。

### 已完整的例子演示动态方法解析

为了说明消息转发机制的意义，下面示范如何以动态方法解析来实现@dynamic属性。假设要编写一个类似于“字典”的对象，它里面可以容纳其它对象，只不过开发者要直接通过属性来存取其中的数据。

该类的接口可以写成：
```
#import <Foundation/Foundation.h>

@interface EOCAutoDictionary: NSObject
@property (nonatomic,strong) NSString *string;
@property (nonatomic,strong) NSNumber *number;
@property (nonatomic,strong) NSDate *date;
@property (nonatomic,strong) id opaqueueobject;

@end
```

在类的内部，每个属性的值还是会存放在字典里，所以我们先在勒种编写如下代码，并将属性声明为@dynamic，这样的话，编译器就不会为其自动生成实例变量及存取方法了：

```
#import "EOCAutoDictionary.h"
#import <objc/runtime.h>

@interface EOCAutoDictionary()
@property (nonatomic,strong) NSMutableDictionary *backingStore;
@end

@implementation EOCAutoDictionary

@dynamic string,number,date,opaqueobject;

- (id)init
{
    if(self = [super init])
    {
        _backingStore = [NSMutableDictionary new];
    }
    return self;
}

- (BOOL)resolveInstanceMethod:(SEL)selector
{
    NSString *selectorString = NSStringFromSelector(selector);
    if([selectorString hasPrefix:@"set"])
    {
        class_addMethod(self,
                        selector,
                        (IMP)autoDictionarySetter,
                        "v@:@");
    }else{
        class_addMethod(self,
                        selector,
                        (IMP)autoDictionaryGetter,
                        "@@:");
    }
    return YES;
}


@end

```

## 内存管理

### 理解引用计数

看下面这段代码：

```
NSMutableArray *array = [[NSMutableArray alloc] init];  // array: count = 1

NSNumber *number = [[NSNumber alloc] initWithInt:1337]; // number: count = 1

[array addObject:number]; // number: count = 2
[number release];
[array release];

```

为了避免不经意间使用了无效对象，一般调用完release之后都会清空指针。这就能保证不会出现可能指向无效对象的指针，这种指针通常称为“悬挂指针”。 可以按照下面的方法来写：

```
NSNumber *number = [[NSNumber alloc] initWithInt:1337];
[array addObject:number];
[number release];
number = nil;
```

### 属性存取方法中的内存管理

若属性为`strong`关系，则设置的属性值会保留。比方说，有个名叫foo的属性由名为_foo的实例变量实现，那么该属性的设置方法会是这样：

```
- (void)setFoo:(id)foo
{
    [foo retain];
    [_foo release];
    _foo = foo;
}
```

此方法会保留新值并释放旧值，然后更新实例变量，令其指向新值。顺序很重要。

自动释放池中的释放操作要等到下一次事件循环是才会执行。

### 在dealloc方法中只释放引用并解除监听

