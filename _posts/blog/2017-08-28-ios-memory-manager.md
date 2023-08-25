---
layout: post
title: ios中的内存管理
description: 介绍IOS中的内存管理知识，以及开发中的注意事项
category: blog
tag: iOS,Ojbective-C
---

## 概述

内存管理是IOS开发中非常重要的一个模块，现在虽然有了ARC技术，但是深入理解内存管理对把控app内存消耗是至关重要的。这篇文章具体包括：

* 理解引用计数
* ARC内存管理
* autorelease pool

## 理解引用计数

`Objective-C`使用引用计数来管理内存。每个对象都有一个可以递增或递减的计数器，如果想使某个对象继续存活，那就递增其引用计数，用完了之后就递减其计数。当计数变为0，就表示没人关注此对象了，于是就可以把它销毁了。

### 引用计数工作原理

在引用计数架构下，对象有个计数器，泳衣表示当前有多少个事物想令此对象继续存活下去。在`Objective-C`中叫做`引用计数`。NSObject协议声明了下面三个方法用于操作计数器：

* retain : 递增保留计数
* release : 递减保留计数
* autorelease : 待稍后清理“自动释放池”时，在递减保留计数。

为避免不经意间使用了无效对象，一般调用玩release之后都会清空指针。这就能保证不会出现可能指向无效对象的指针。eg:

```
NSNumber *number = [[NSNumber alloc] initWithInt:1337];
[array addObject:number];
[number release];
number = nil;
```

### 属性存取方法中的内存管理

若属性为"strong 关系"，则设置的属性值会保留。如下：

```
- (void)setFoo:(id)foo{
	[foo retain];
	[_foo release];
	_foo = foo;
}
```

保留新值释放旧值，然后更新实例变量令其指向新值。

### 自动释放池

在OC的引用计数架构中，自动释放池是一项重要特性。调用release会立刻递减对象的保留计数，然而有时候可以不掉用它，改为调用autorelease,此方法会在稍后递减计数，通常是在下一次“事件循环”时递减，不过也可能执行的更早。

此特性很有用，尤其是在方法中返回对象时更应该用它。

```
- (NSString *)stringValue
{
	NSString *str = [[NSSstring alloc] initWithFormat:@"I am this:%@",self];
	return str;
}
```

此时返回的str对象其保留计数比期望值要多1 。保留计数多1，就意味着调用者要负责处理多出来的这一次保留操作。必须设法将其抵消。但是，不能在方法内释放str,否则还没等方法返回，系统就把该对象回收了。所以，这里应该使用autorelease,它会在稍后释放对象。改写上面程序如下：

```
- (NSString *)stringValue
{
	NSString *str = [[NSString alloc] initWithFormat:@"I am this:%@",self];
	return [str autorelease];
}
```

修改之后，stringValue方法把NSString对象返回给调用者时，此对象必然存活。所以我们能够像下面这样使用它：

```
NSString *str = [self stringValue];
NSLog(@"The string is:%@",str);
```

但是，假如要持有此对象的话，那就需要保留并于稍后释放：

```
_instanceVariable = [[self stringValue] retain];
//...
[_instanceVariable release];
```

## ARC内存管理

ARC 则是帮助我们做对象内存管理的一套机制，使得我们以前在 MRC 模式下管理内存工作量能在 ARC 模式下得到缓解。可见 ARC 是编译时特性，它没有改变 Objective-C 引用计数式内存管理的本质，更不是 GC（垃圾回收）。

在 ARC 特性下有 4 种与内存管理息息相关的变量所有权修饰符：

* `__strong`
* `__weak`
* `__autoreleasing`
* `__unsafe_unretaied`

`变量所有权修饰符`并不是`属性修饰符`,这里做一个对照关系：

* `assign` 对应的所有权类型是`__unsafe_unretained`
* `copy` 对应的所有权类型是`__strong`
* `retain`对应的所有权类型是`__strong`
* `strong`对应的所有权类型是`__strong`
* `unsafe_unretained`对应的所有权类型是`__unsafe_unretained`

以上除了`weak`外，其它的属性修饰符在MRC模式下也是有效的。另外，`__strong`、`__weak`、`__autoreleasing`修饰的自动变量会自动初始化为nil 

### __strong

`__strong` 表示强引用，对应定义 property 时用到的 `strong`。当对象没有任何一个强引用指向它时，它才会被释放。如果在声明引用时不加修饰符，那么引用将默认是强引用。当需要释放强引用指向的对象时，需要保证所有指向对象强引用置为 nil。`__strong` 修饰符是 id 类型和对象类型默认的所有权修饰符。

### __weak

`__weak` 表示弱引用，对应定义 property 时用到的 `weak`。弱引用不会影响对象的释放，而当对象被释放时，所有指向它的弱引用都会自定被置为 nil，这样可以防止野指针。`__weak` 最常见的一个作用就是用来避免强引用循环。但是需要注意的是，`__weak` 修饰符只能用于 iOS5 以上的版本，在 iOS4 及更低的版本中使用 `__unsafe_unretained` 修饰符来代替。

`__weak`的几个使用场景：

* 在 Delegate 关系中防止强引用循环。在 ARC 特性下，通常我们应该设置 Delegate 属性为 `weak` 的。但是这里有一个疑问，我们常用到的 UITableView 的 delegate 属性是这样定义的： `@property (nonatomic, assign) id<UITableViewDelegate> delegate;`，为什么用的修饰符是 `assign` 而不是 `weak`？其实这个 `assign` 在 ARC 中意义等同于 `__unsafe_unretaied`（后面会讲到），它是为了在 ARC 特性下兼容 iOS4 及更低版本来实现弱引用机制。一般情况下，你应该尽量使用 `weak`。
* 在 Block 中防止强引用循环。
* 用来修饰指向由 Interface Builder 创建的控件。比如：`@property (weak, nonatomic) IBOutlet UIButton *testButton;`

### __autoreleasing

在 ARC 模式下，我们不能显示的使用 autorelease 方法了，但是 autorelease 的机制还是有效的，通过将对象赋给 `__autoreleasing` 修饰的变量就能达到在 MRC 模式下调用对象的 autorelease 方法同样的效果。

`__autoreleasing` 修饰的对象会被注册到 Autorelease Pool 中，并在 Autorelease Pool 销毁时被释放，和 MRC 特性下的 autorelease 的意义相同。定义 property 时不能使用这个修饰符，因为任何一个对象的 property 都不应该是 autorelease 类型的。

在 ARC 模式下，显式的使用 __autoreleasing 的场景很少见，但是 autorelease 的机制却依然在很多地方默默起着作用。我们来看看这些场景：

* 方法返回值。
* 访问 `__weak` 修饰的变量。
* id 的指针或对象的指针(id *)

#### 方法返回值

有如下示例：

```
- (NSObject *)object {
    NSObject *o = [[NSObject alloc] init];
    return o;
}
```

这里 o 的所有权修饰符是默认的 `__strong`。由于 return 使得 o 超出其作用域，它强引用持有的对象本该被释放，但是由于该对象作为函数返回值，所以一般情况下编译器会自动将其注册到 Autorelease Pool 中（注意这里是一般情况下，在一些特定情况下，ARC 机制提出了巧妙的运行时优化方案来跳过 autorelease 机制，见后面章节）。这是 autorelease 机制默默起作用的一个例子。

这涉及到两个角色的问题。一个角色是调用方法接收返回值的接收方。当参数被作为返回值 return 之后，接收方如果要接着使用它就需要强引用它，使它 retainCount +1，用完后再清理，使它 retainCount -1。有持有就有清理，这是接收方的责任。另一个角色就是返回对象的方法，即提供方。在方法中创建了对象并作为返回值时，一方面你创建了这个对象你就得负责释放它，有创建就有释放，这是创建者的责任。另一方面你得保证返回时对象没被释放以便方法外的接收方能拿到有效的对象，否则你返回的是 nil，有何意义呢。所以就需要找一个合理的机制既能延长这个对象的生命周期，又能保证对其释放。这个机制就是 `autorelease` 机制。

当对象作为参数从方法返回时，会被放到正在使用的 Autorelease Pool 中，由这个 Autorelease Pool 强引用这个对象而不是立即释放，从而延长了对象的生命周期，Autorelease Pool 自己销毁的时候会把它里面的对象都顺手清理掉，从而保证了对象会被释放.

Autorelease Pool 是与线程一一映射的，这就是说一个 autoreleased 的对象的延迟释放是发生在它所在的 Autorelease Pool 对应的线程上的。因此，在方法返回值的这个场景中，如果 Autorelease Pool 的 drain 方法没有在接收方和提供方交接的过程中触发，那么 autoreleased 对象是不会被释放的（除非严重错乱的使用线程）。

通常，Autorelease Pool 的销毁会被安排在很好的时间点上：

* Run Loop 会在每次 loop 到尾部时销毁 Autorelease Pool。
* GCD 的 dispatched blocks 会在一个 Autorelease Pool 的上下文中执行，这个 Autorelease Pool 不时的就被销毁了（依赖于实现细节）。NSOperationQueue 也是类似。
* 其他线程则会各自对他们对应的 Autorelease Pool 的生命周期负责。

至此，我们知道了为何方法返回值需要 autorelease 机制，以及这一机制是如何保障接收方能从提供方那里获得依然鲜活的对象。


**ARC 模式下方法返回值跳过 autorelease 机制的优化方案**

在 MRC 时代，当我们自己创建了对象并把它作为方法的返回值返回出去时，需要手动调用对象的 autorelease 方法，如上节所讲的利用 autorelease 机制正确返回对象。到了 ARC 时代，ARC 需要保持对 MRC 代码的兼容，这就意味着 MRC 的实现和 ARC 的实现可以相互替换，而对象接收方和对象提供方无需知道对方是 MRC 实现还是 ARC 实现也能正确工作。比如，当基于 MRC 实现的代码调用你的一个 ARC 实现的方法来获取一个对象，那么你的方法必须同样采用上文所讲的 autorelease 机制来返回对象以确保对象接收方能正确获得对象。所以，即使在 ARC 模式下对象的 autorelease 方法不再能被显示调用，但是 autorelease 的机制仍然是在默默的工作着，只是编译器在帮你实践这一机制。

但是，ARC 还提出了巧妙的运行时优化方案来跳过 autorelease 机制。这个过程是这样的：当方法的调用方和实现方的代码都是基于 ARC 实现的时候，在方法 return 的时候，ARC 会调用 `objc_autoreleaseReturnValue()` 替代前面说的 autorelease。在调用方持有方法返回对象的时候（也就是做 retain 的时候），ARC 会调用 `objc_retainAutoreleasedReturnValue()`。在调用 `objc_autoreleaseReturnValue()` 时，它会在栈上查询 return address 来确定 return value 是否会被传给 `objc_retainAutoreleasedReturnValue()`。如果没传，那么它就会走前文所讲的 autorelease 的过程。如果传了（这表明返回值能顺利从提供方交接给接收方），那么它就跳过 autorelease 并同时修改 return address 来跳过 `objc_retainAutoreleasedReturnValue()`，从而一举消除了 `autorelease` 和 `retain` 的过程。这个方案可以在 MRC-to-ARC 调用、ARC-to-ARC 调用以及 ARC-to-MRC 调用中正确工作，并在符合条件的一些 ARC-to-ARC 调用中消除 autorelease 机制。

#### 访问__weak修饰的变量

在访问 `__weak` 修饰的变量时，实际上必定会访问注册到 Autorelease Pool 的对象。如下来年两段代码是相同的效果：

```
id __weak obj1 = obj0;
NSLog(@"class=%@", [obj1 class]);
// 等同于：
id __weak obj1 = obj0;
id __autoreleasing tmp = obj1;
NSLog(@"class=%@", [tmp class]);
```

为什么会这样呢？因为 `__weak`修饰符只持有对象的弱引用，而在访问对象的过程中，该对象有可能被废弃，如果把被访问的对象注册到 Autorelease Pool 中，就能保证 Autorelease Pool 被销毁前对象是存在的。

#### id 的指针或对象的指针(id *)

另一个隐式地使用 __autoreleasing 的例子就是使用 id 的指针或对象的指针(id *) 的时候。

例子：

```
NSError *__autoreleasing error; 
￼if (![data writeToFile:filename options:NSDataWritingAtomic error:&error]) { 
　　NSLog(@"Error: %@", error); 
}
// 即使上面你没有写 __autoreleasing 来修饰 error，编译器也会帮你做下面的事情：
NSError *error; 
NSError *__autoreleasing tempError = error; // 编译器添加 
if (![data writeToFile:filename options:NSDataWritingAtomic error:&tempError]) 
￼{ 
　　error = tempError; // 编译器添加 
　　NSLog(@"Error: %@", error); 
}
```

error 对象在你调用的方法中被创建，然后被放到 Autorelease Pool 中，等到使用结束后随着 Autorelease Pool 的销毁而释放，所以函数外 error 对象的使用者不需要关心它的释放。

在 ARC 中，所有这种指针的指针类型（id *）的函数参数如果不加修饰符，编译器会默认将他们认定为 `__autoreleasing` 类型。

有一点特别需要注意的是，某些类的方法会隐式地使用自己的 Autorelease Pool，在这种时候使用 `__autoreleasing` 类型要特别小心。比如 NSDictionary 的 enumerateKeysAndObjectsUsingBlock 方法：

```
- (void)loopThroughDictionary:(NSDictionary *)dict error:(NSError **)error {
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
          // do stuff  
          if (there is some error && error != nil) {
                *error = [NSError errorWithDomain:@"MyError" ￼code:1 userInfo:nil];
          }
￼
    }];
￼}
```

上面的代码中其实会隐式地创建一个 Autorelease Pool，类似于：

```
- (void)loopThroughDictionary:(NSDictionary *)dict error:(NSError **)error {
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
          @autoreleasepool {  // 被隐式创建。
              if (there is some error && error != nil) {
                    *error = [NSError errorWithDomain:@"MyError" ￼code:1 userInfo:nil];
              }
￼          }
    }];
    // *error 在这里已经被dict的做枚举遍历时创建的 Autorelease Pool释放掉了。
￼} 
```

为了能够正常的使用 *error，我们需要一个 strong 类型的临时引用，在 dict 的枚举 Block 中是用这个临时引用，保证引用指向的对象不会在出了 dict 的枚举 Block 后被释放，正确的方式如下：

```
- (void)loopThroughDictionary:(NSDictionary *)dict error:(NSError **)error {
　　NSError * __block tempError; // 加 __block 保证可以在Block内被修改。
　　[dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) { 
　　　　if (there is some error) { 
　　　　　　*tempError = [NSError errorWithDomain:@"MyError" ￼code:1 userInfo:nil]; 
　　　　} ￼ 
　　}] 
　　if (error != nil) { 
　　　　*error = tempError; 
　　} ￼
}
```

### ARC模式规则

ARC 模式下，还有一些需要注意的规则：

* 不能显式使用 retain/release/retainCount/autorelease。
* 不能使用 NSAllocateObject/NSDeallocateObject。
* 需要遵守内存管理的方法命名规则。在 ARC 模式和 MRC 模式下，以 alloc/new/copy/mutableCopy 开头的方法在返回对象时都必须返回给调用方所应当持有的对象。在 ARC 模式下，追加一条：以 init 开头的方法必须是实例方法并且必须要返回对象。返回的对象应为 id 类型或声明该方法的类的对象类型，或是该类的超类型或子类型。该返回的对象并不注册到 Autorelease Pool 中，基本上只是对 alloc 方法返回值的对象进行初始化处理并返回该对象。需要注意的是：`- (void)initialize;` 方法虽然是以 init 开头但是并不包含在上述规则中。
* 不要显式调用 dealloc。
* 使用 @autoreleasepool 块替代 NSAutoreleasePool。
* 不能使用区域（NSZone）
* 对象型变量不能作为 C 语言结构体（struct/union）的成员。
* 显式转换 id 和 void *。

### Toll-Free Bridging

Toll-Free Briding 保证了在程序中，可以方便和谐的使用 Core Foundation 类型的对象和Objective-C 类型的对象。

在 MRC 时代，由于 Objective-C 类型的对象和 Core Foundation 类型的对象都是相同的 release 和 retain 操作规则，所以 Toll-Free Bridging 的使用比较简单，但是自从切换到 ARC 后，Objective-C 类型的对象内存管理规则改变了，而 Core Foundation 依然是之前的机制，换句话说，Core Foundation 不支持 ARC。

这个时候就必须要要考虑一个问题了，在做 Core Foundation 与 Objective-C 类型转换的时候，用哪一种规则来管理对象的内存。显然，对于同一个对象，我们不能够同时用两种规则来管理，所以这里就必须要确定一件事情：哪些对象用 Objective-C（也就是 ARC）的规则，哪些对象用 Core Foundation 的规则（也就是 MRC）的规则。或者说要确定对象类型转换了之后，内存管理的 ownership 的改变。于是苹果在引入 ARC 之后对 Toll-Free Bridging 的操作也加入了对应的方法与修饰符，用来指明用哪种规则管理内存，或者说是内存管理权的归属。这些方法和修饰符分别是：

* __bridge（修饰符）
* __bridge_retained（修饰符） or CFBridgingRetain（函数）
* __bridge_transfer（修饰符） or CFBridgingRelease（函数）

#### __bridge

只是声明类型转变，但是不做内存管理规则的转变。

eg:

```
CFStringRef s1 = (__bridge CFStringRef) [[NSString alloc] initWithFormat:@"Hello, %@!", name];
```

只是做了 NSString 到 CFStringRef 的转化，但管理规则未变，依然要用 Objective-C 类型的 ARC 来管理 s1，你不能用 CFRelease() 去释放 s1。

#### __bridge_retained or CFBridgingRetain

表示将指针类型转变的同时，将内存管理的责任由原来的 Objective-C 交给Core Foundation 来处理，也就是，将 ARC 转变为 MRC。

eg:

```
NSString *s1 = [[NSString alloc] initWithFormat:@"Hello, %@!", name];
￼CFStringRef s2 = (__bridge_retained CFStringRef)s1;
// or CFStringRef s2 = (CFStringRef)CFBridgingRetain(s1);
￼// do something with s2
//...
￼CFRelease(s2); // 注意要在使用结束后加这个
```

我们在第二行做了转化，这时内存管理规则由 ARC 变为了 MRC，我们需要手动的来管理 s2 的内存，而对于 s1，我们即使将其置为 nil，也不能释放内存。

#### __bridge_transfer or CFBridgingRelease

这个修饰符和函数的功能和上面那个 __bridge_retained 相反，它表示将管理的责任由 Core Foundation 转交给 Objective-C，即将管理方式由 MRC 转变为 ARC。

eg:

```
CFStringRef result = CFURLCreateStringByAddingPercentEscapes(. . .);
￼NSString *s = (__bridge_transfer NSString *)result;
//or NSString *s = (NSString *)CFBridgingRelease(result);
￼return s;
```

这里我们将 result 的管理责任交给了 ARC 来处理，我们就不需要再显式地将 CFRelease() 了。

对了，这里你可能会注意到一个细节，和 ARC 中那个 4 个主要的修饰符（__strong, __weak, …）不同，这里修饰符的位置是放在类型前面的，虽然官方文档中没有说明，但看官方的头文件可以知道。记得别把位置写错。

## Autorelease Pool

译文[Using Autorelease Pool Blocks](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/memorymgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI)

### About autorelease pool

Autorelease Pool是自动释放池。

Autorelease Pool的一个使用场景是在需要延迟释放某些对象的情况时，可以把他们先放到对应的Autorelease Pool中，等Autorelease Pool生命周期结束时再一起释放。这些对象会被发送autorelease消息。在非ARC时代，我们显示地发送autorelease消息，在ARC时代，系统会帮我们做这些事情。

使用Autorelease Pool时候的语法：

```
@autoreleasepool {
    // Code that creates autoreleased objects.
}
```

而且可以嵌套：

```
@autoreleasepool {
    // . . .
    @autoreleasepool {
        // . . .
    }
    . . .
}
```

Cocoa中总是会期望代码在一个Autorelease Pool Block中运行，否则那些被autorelease的对象就不能被释放了，这就内存泄露了。在AppKit和UIKit框架中，当处理一个事件循环的一次迭代的时候（比如一次鼠标点击事件或一次tap事件），会自动在一个autorelease pool block里完成，因此通常你不需要自己去创建一个autorelease pool block，甚至你都很少看到这种代码。

但是，你仍然会在下列几种情况下用到你自己的autorelease pool block：

* 你编写是命令行工具的代码，而不是基于UI框架的代码。
* 你需要写一个循环，里面会创建很多临时的对象。
	* 这时候你可以在循环内部的代码块里使用一个autorelease pool block，这样这些对象就能在一次迭代完成后被释放掉。这种方式可以降低内存最大占用。
* 当你大量使用辅助线程。
	* 你需要在线程的任务代码中创建自己的autorelease pool block。
	
### 使用Autorelease Pool Block来降低内存占用峰值

eg:

```
NSArray *urls = <# An array of file URLs #>;
for (NSURL *url in urls) {
 
    @autoreleasepool {
        NSError *error;
        NSString *fileContents = [NSString stringWithContentsOfURL:url
                                         encoding:NSUTF8StringEncoding error:&error];
        /* Process the string, creating and autoreleasing more objects. */
    }
}
``` 

上面的例子中，for循环的每次迭代处理一个文件，我们把这个处理过程放在一个autorelease pool block中，这样任何在里面创建的autorelease的对象都会在这次迭代结束后被释放，这样就不会占用很多内存了。

在autorelease pool block执行完后，所有曾在里面autorelease的对象都可以被认为是释放了的，所以就别再给它们发消息了。如果你确实要穿越autorelease pool block用到临时对象，那可以在autorelease pool block里retain它，之后再autorelease它，看下面例子：

```
– (id)findMatchingObject:(id)anObject {
 
    id match;
    while (match == nil) {
        @autoreleasepool {
 
            /* Do a search that creates a lot of temporary objects. */
            match = [self expensiveSearchForObject:anObject];
 
            if (match != nil) {
                [match retain]; /* Keep match around. */
            }
        }
    }
 
    return [match autorelease];   /* Let match go and return it. */
}
```

### Autorelease Pool Block和线程

在Cocoa中，每个线程去维护它自己的autorelease pool block的栈，所以当我们自己要写一些线程代码，我们可能就要自己去创建自己的autorelease pool block了，尤其当你的线程是长时间工作、可能生产出大量的autorelease的对象的时候。

如果你分发的线程不会产生对Cocoa的调用，那么你也许就不需要使用autorelease pool block了。

### autorelease的实现

既然说到 Autorelease Pool，这里就多说一下在 MRC 时代 autorelease 的实现机制，了解这个机制对于我们理解 Objective-C 的内存管理是很有帮助的。

在讨论实现之前，我们首先来看个问题，我们都知道 autorelease 机制的一个典型的应用场景就是参数传递返回值。那为什么方法返回值的时候需要用到 autorelease 特性呢？这是因为你在方法中创建了一个对象，在返回之前你总不能把它释放了吧，这样你返回的就是 nil 了，而返回后，方法都调用结束了。参数值 return 出去后，外面接收它的要使用它就要强引用它，给它 retainCount +1，不用了就清理，retainCount -1，这是方法外面的事，但对于你这个方法来说你创建了它却没清理它，这是不负责任地搞内存泄露啊！所以得有一套机制保证创建这个对象的方法能在方法调用结束后清理掉这个对象，这就是 autorelease 机制。

在 Objective-C 中编程人员可以通过 autorelease 机制设定变量的作用域，这其中就离不开 Autorelease Pool。结合 Autorelease Pool 来使用 autorelease 的方法大致如下：

```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];  
id obj = [[NSObject alloc] init];  
[obj autorelease];  
// Do something...
[pool drain]; 
```

上面的代码最后一行执行的时候就会自动调用 `[obj release];`。

这个过程中调用` [obj autorelease]` 究竟发生了什么呢？我们可以查看 GNUstep 的源代码来看看：

```
// GNUstep/modules/core/base/Source/NSObject.m autorelease
- (id)autorelease {  
	[NSAutoreleasePool addObject:self];  
}
```

autorelease 实例方法的本质就是调用 NSAutoreleasePool 对象的 addObject 类方法。下面来看看这个类方法的实现，由于源码比较复杂，下面对代码进行了简化，大致如下：

```
// GNUstep/modules/core/base/Source/NSAutoreleasePool.m addObject
+ (void)addObject:(id)anObj {  
	NSAutoreleasePool *pool = 取得正在使用的 NSAutoreleasePool 对象;  
	if (pool != nil) {  
		[pool addObject:anObj];  
	} else {  
		NSLog(@"NSAutoreleasePool 对象非存在状态下调用 autorelease");  
	}  
}
```

什么叫正在使用的 NSAutoreleasePool 的对象呢？比如下面：

```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];  
id obj = [[NSObject alloc] init];  
[obj autorelease]; 
```

被赋值的 pool 变量就是。

而这个例子里面：

```
NSAutoreleasePool *pool0 = [[NSAutoreleasePool alloc] init];  
	NSAutoreleasePool *pool1 = [[NSAutoreleasePool alloc] init];  
		NSAutoreleasePool *pool2 = [[NSAutoreleasePool alloc] init];  
			id obj = [[NSObject alloc] init];  
			[obj autorelease];  
		[pool2 drain];  
	[pool1 drain];  
[pool0 drain];
```

则是最内侧的 pools。

下面看看实例方法的实现：

```
// GNUstep/modules/core/base/Source/NSAutoreleasePool.m addObject
- (void)addObject:(id)anObj   {  
	[array addObject:anObj];  
} 
```

实际的 GNUstep 实现使用的是链表结构，这跟在 NSMutableArray 中添加对象是一样的。如果调用 NSObject 类的 autorelease 实例方法，该对象将被追加到正在使用的 NSAutoreleasePool 对象中的数组里。


再看看看 `[pool drain];` 的实现。

```
//GNUstep/modules/core/base/Source/NSAutoreleasePool.m drain
- (void)drain {  
	[self dealloc];  
}  
- (void)dealloc {  
	[self emptyPool];  
	[array release];  
}  
- (void)emptyPool {  
	for (id obj in array) {  
		[obj release];  
	}  
} 
```

调用了好几个方法，最终是对于 Autorelease Pool 的数组里的对象都调用了 release 方法。

### NSThread、NSRunLoop 和 NSAutoreleasePool

根据苹果官方文档中对 NSRunLoop 的描述，我们可以知道每一个线程，包括主线程，都会拥有一个专属的 NSRunLoop 对象，并且会在有需要的时候自动创建。

在主线程的 NSRunLoop 对象（在系统级别的其他线程中应该也是如此，比如通过 dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0) 获取到的线程）的每个 event loop 开始前，系统会自动创建一个 Autorelease Pool ，并在 event loop 结束时 drain 。我们上面提到的场景 1 中创建的 autoreleased 对象就是被系统添加到了这个自动创建的 Autorelease Pool 中，并在这个 Autorelease Pool 被 drain 时得到释放。

另外，NSAutoreleasePool 中还提到，每一个线程都会维护自己的 Autorelease Pool 堆栈。换句话说 Autorelease Pool 是与线程紧密相关的，每一个 Autorelease Pool 只对应一个线程。

弄清楚 NSThread、NSRunLoop 和 NSAutoreleasePool 三者之间的关系可以帮助我们从整体上了解 Objective-C 的内存管理机制