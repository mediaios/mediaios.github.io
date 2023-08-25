---
layout: post
title: 浅拷贝深拷贝
description: IOS中的浅拷贝和深拷贝
category: blog
tag: ios
---

## 1.浅拷贝&深拷贝

浅复制：浅复制只复制指向对象的指针，并不复制对象本身。

深复制：深复制是直接复制整个对象到另一块内存中。

NSObject提供了`copy`和`mutableCopy` 方法，`copy`复制后对象是不可变对象（immutable），`mutableCopy`复制后的对象是可变对象（mutable），与原始对象是否可变无关。

### 1.1 非集合类对象的copy&mutableCopy

非集合类对象指的是NSString、NSNumber之类的对象，深复制会复制引用对象的内容，而浅复制只复制引用这些对象的指针。因此，如果对象A被浅复制到对象B，对象B和对象A引用的是同一内存地址的实例变量或属性。

#### 1.1.1不可变对象的copy&mutableCopy

示例：

```
        NSString *str = @"hello";
        NSString *str2 = str;
        NSString *strCopy = [str copy];
        NSMutableString *strMutableCopy = [str mutableCopy];
        
        NSLog(@"\n str : %p\n  str2 : %p \n  strCopy : %p \n strMutableCopy : %p",str,str2,strCopy,strMutableCopy);
```

输出结果：

```
 str : 0x100001040
 str2 : 0x100001040 
 strCopy : 0x100001040 
 strMutableCopy : 0x102822e10
```

通过以上示例可以看到，str,str2,strCopy的内存地址一致，即指向了同一块内存区域，进行了浅复制的操作。而 strMutableCopy与另外三个变量内存地址不同，系统为其分配了新的内存，即进行了深复制操作。

#### 1.1.2 可变对象的copy&mutableCopy

示例：

```
        NSMutableString *mStr = [NSMutableString stringWithString:@"hello"];
        NSString *mStrCopy = [mStr copy];
        NSMutableString *mutableStrCopy = [mStr copy];
        NSMutableString *mutableStrMutableCopy = [mStr mutableCopy];
        
        [mStr appendString:@"A"];
//        [mutableStrCopy appendString:@"B"];   //  crash
        [mutableStrMutableCopy appendString:@"C"];
        
        NSLog(@"\n mStr : %p \n mStrCopy :%p \n mutableStrCopy :%p  \n mutableStrMutableCopy :%p",mStr,mStrCopy,mutableStrCopy,mutableStrMutableCopy);
```

输出结果：

```
 mStr : 0x10050d240 
 mStrCopy :0x10050d320 
 mutableStrCopy :0x10050d340  
 mutableStrMutableCopy :0x10050d360
```

上面示例中，运行到`[mutableStrCopy appendString:@"B"]`会崩溃，因为通过`copy`方法获得的字符串是不可变字符串。  我们还看到了上面四个对象的地址各不相同，因此这里的`copy`和`mutableCopy`执行的均是深复制。

结合上面的两个例子，我们可以得出结论：

* 对于不可变对象，执行`copy`操作是指针复制，执行`mutableCopy`是内容复制
* 对于可变对象，执行`copy`和`mutableCopy`操作均是内容复制

```
[immutable copy];           // 浅复制
[immutable mutableCopy];    // 深复制
[mutable copy];				 // 深复制
[mutable mutableCopy];		 // 深复制
```

### 1.2 容器类对象的深复制与浅复制

容器类对象是指`NSArray`,`NSDictionary`等。容器类对象的深复制与浅复制如图：

![](https://github.com/pro648/tips/wiki/images/CopyCollectionCopy.png)

对于容器类，需要探讨的是复制后容器内元素的变化，而非容器本身内存地址是否发生变化。

#### 1.2.1 容器类对象的浅复制

有很多方法可以对集合进行浅复制。当对集合进行浅复制时，将复制原始集合中元素指针到新的集合，即原始集合中元素引用计数加一。

示例：

```
        NSMutableString *china = [NSMutableString stringWithString:@"China"];
        NSMutableString *american = [NSMutableString stringWithString:@"American"];
        NSMutableString *japan = [NSMutableString stringWithString:@"Japan"];
        NSArray *array = [NSArray arrayWithObjects:china, american, japan, nil];
        
        NSArray *array2 = [array copy];
        NSMutableArray *mutableArray3 = [array mutableCopy];
        NSArray *array4 = [[NSArray alloc] initWithArray:array copyItems:NO];
        
        NSMutableString *tempString = array2.firstObject;
        [tempString appendString:@"*ShangHai"];
        
        NSLog(@"array : %p, \n array2 %p, \n mutableArray3 %p, \n array4 %p",array, array2, mutableArray3, array4);
        NSLog(@"array %@, \n array2 %@, \n mutableArray3 %@, \n array4 %@",array , array2, mutableArray3, array4);
```

运行结果：

```
2018-06-01 09:44:42.790895+0800 MyTest[6947:1180092]
 array : 0x100416b10, 
 array2 0x100416b10, 
 mutableArray3 0x100416c20, 
 array4 0x100416b10
2018-06-01 09:44:42.791216+0800 MyTest[6947:1180092] array (
    "China*ShangHai",
    American,
    Japan
), 
 array2 (
    "China*ShangHai",
    American,
    Japan
), 
 mutableArray3 (
    "China*ShangHai",
    American,
    Japan
), 
 array4 (
    "China*ShangHai",
    American,
    Japan
)
```
从以上示例中可以看到，array和array2数组内存地址相同;mutableArray3与其它数组的地址不同。这是因为`mutableCopy`的对象会被分配新的内存。

观察数组内元素，发现修改array2数组内第一个元素，四个数组第一个元素都发生了变化，所以这里只是进行了浅复制。

#### 1.2.2 容器类对象的深复制

有两种方式对容器类对象进行深复制：

* 使用`initWithArray:copyItems:`方法，其中第二个参数为`YES`
* 使用归档解档


使用`initWithArray:copyItems:`方法进行深复制时，第二个参数设置为`YES`,如果使用该方法对集合进行深复制，那么集合内每个元素都会收到`copyWithZone:`消息。我们平常使用`copy`,`mutableCopy`方法时，系统会把`copy`和`mutableCopy`自动替换为`copyWithZone:`和`mutableCopyWithZone:`。如果集合元素遵守了`NSCopying`协议，元素被复制到新的集合；如果元素不遵守`NSCopying`协议，用这样的方式进行深复制会在运行时产生错误。

`copyWithZone:`产生的是浅复制，所以，这种方法只能产生一层深复制 one-level-deep copy，如果集合内元素仍然是集合，则子集合内元素不会被深复制，只对子集合内元素指针进行复制。

示例：

```
        NSMutableString *china1 = [NSMutableString stringWithString:@"China"];
        NSMutableString *american1 = [NSMutableString stringWithString:@"American"];
        NSMutableString *japan1 = [NSMutableString stringWithString:@"Japan"];
        NSArray *array = [NSArray arrayWithObjects:china1, american1, japan1, nil];
        
        NSArray *array2 = [array copy];
        NSMutableArray *mutableArray3 = [array mutableCopy];
        NSArray *array4 = [[NSArray alloc] initWithArray:array copyItems:YES];
        
        NSMutableString *tempString = array2.firstObject;
        [tempString appendString:@"*ShangHai"];
        
        NSLog(@"array : %p, \n array2 %p, \n mutableArray3 %p, \n array4 %p",array, array2, mutableArray3, array4);
        NSLog(@"array %@, \n array2 %@, \n mutableArray3 %@, \n array4 %@",array , array2, mutableArray3, array4);
```

运行结果：

```
2018-06-01 10:27:12.051617+0800 MyTest[7194:1304768] array : 0x10060aa00, 
 array2 0x10060aa00, 
 mutableArray3 0x10060ab10, 
 array4 0x10060ada0
2018-06-01 10:27:12.051989+0800 MyTest[7194:1304768] array (
    "China*ShangHai",
    American,
    Japan
), 
 array2 (
    "China*ShangHai",
    American,
    Japan
), 
 mutableArray3 (
    "China*ShangHai",
    American,
    Japan
), 
 array4 (
    China,
    American,
    Japan
)
```

可以看到 `array4`数组内第一个元素与其它数组第一个元素内存地址不同了，即进行了一层深复制。

这种对集合进行深复制的方法，对其它类型集合也有效。如词典中initWithDictionary: withItems:方法。

如果你的数组内元素是另一个数组，想要进行完全深复制，可以使用归档、解归档方法。使用该方法时，归档对象要遵守NSCoding协议。

使用归档解档方法，进行完全深复制：

```
 // 1.创建一个可变数组，数组第一个元素是另一个可变数组，第二个元素是另一个不可变数组。
    NSMutableString *hue = [NSMutableString stringWithString:@"hue"];
    NSMutableString *saturation = [NSMutableString stringWithString:@"saturation"];
    NSMutableString *brightness = [NSMutableString stringWithString:@"brightness"];
    NSMutableArray *hsbArray1 = [NSMutableArray arrayWithObjects:hue, saturation, brightness, nil];
    NSArray *hsbArray2 = [NSArray arrayWithObjects:hue, saturation, brightness, nil];
    NSMutableArray *hsbArray3 = [NSMutableArray arrayWithObjects:hsbArray1, hsbArray2, nil];
    
    // 2.通过归档、解档进行完全深复制。
    NSData *dataArea = [NSKeyedArchiver archivedDataWithRootObject:hsbArray3];
    NSMutableArray *hsbArray4 = [NSKeyedUnarchiver unarchiveObjectWithData:dataArea];
    
    // 3.输出hsbArray3和hsbArray4数组第一个元素内存地址。
    NSLog(@"Memory location of \n hsbArray3.firstObject = %p, \n hsbArray4.firstObject = %p",hsbArray3.firstObject, hsbArray4.firstObject);
```

上面代码中，可变数组`hsbArray3`第一个元素是可变数组`hsbArray1`，第二个元素是不可变数组`hsbArray2`。

使用归档、读取归档方法深复制后，在控制台输出`hsbArray3`和`hsbArray4`第一个元素内存地址。输出如下：

```
 hsbArray3.firstObject = 0x10064cba0, 
 hsbArray4.firstObject = 0x1028183d0
```

可以看到hsbArray3和hsbArray4数组内元素内存地址不同，即进行了一层深复制。

在trueDeepCopy方法内，继续为hsbArray4数组内第一个元素tempArray1可变数组添加字符串对象。为hsbArray4第二个元素hsbArray2数组添加字符串对象。最后输出hsbArray3和hsbArray4数组内容。

```
 NSMutableString *hue = [NSMutableString stringWithString:@"hue"];
    NSMutableString *saturation = [NSMutableString stringWithString:@"saturation"];
    NSMutableString *brightness = [NSMutableString stringWithString:@"brightness"];
    NSMutableArray *hsbArray1 = [NSMutableArray arrayWithObjects:hue, saturation, brightness, nil];
    NSArray *hsbArray2 = [NSArray arrayWithObjects:hue, saturation, brightness, nil];
    NSMutableArray *hsbArray3 = [NSMutableArray arrayWithObjects:hsbArray1, hsbArray2, nil];
    
    // 2.通过归档、解档进行完全深复制。
    NSData *dataArea = [NSKeyedArchiver archivedDataWithRootObject:hsbArray3];
    NSMutableArray *hsbArray4 = [NSKeyedUnarchiver unarchiveObjectWithData:dataArea];
    
    // 3.输出hsbArray3和hsbArray4数组第一个元素内存地址。
    NSLog(@"Memory location of \n hsbArray3.firstObject = %p, \n hsbArray4.firstObject = %p",hsbArray3.firstObject, hsbArray4.firstObject);

    
    // 4.为hsbArray4第一个元素添加字符串。
    NSMutableArray *tempArray1 = hsbArray4.firstObject;
    [tempArray1 addObject:@"hsb"];
    
    // 5.hsbArray4第二个元素是hsbArray2，而hsbArray2是不可变数组，这一步将产生错误。
    //    NSMutableArray *tempArray2 = hsbArray4[1];
    //    [tempArray2 addObject:@"Color"];
    
    // 6.输出数组内容。
    NSLog(@"Contents of \n hsbArray3 %@, \n hsbArray4 %@",hsbArray3, hsbArray4);
```

因为hsbArray4第二个元素是hsbArray2副本，而hsbArray2是不可变数组，这一步将产生错误。注释掉5部分代码后，控制台输出如下：

```
2018-06-01 11:38:42.699946+0800 MyTest[7383:1396001] Memory location of 
 hsbArray3.firstObject = 0x10280ab80, 
 hsbArray4.firstObject = 0x1006258e0
2018-06-01 11:38:42.700043+0800 MyTest[7383:1396001] Contents of 
 hsbArray3 (
        (
        hue,
        saturation,
        brightness
    ),
        (
        hue,
        saturation,
        brightness
    )
), 
 hsbArray4 (
        (
        hue,
        saturation,
        brightness,
        hsb
    ),
        (
        hue,
        saturation,
        brightness
    )
)
Program ended with exit code: 0
```

可以看到只有hsbArray4数组第一个元素内对象发生了改变，所以，使用归档、读取归档进行的是完全深复制。

复制集合时，该集合、集合内元素的可变性可能会受到影响。每种方法对任意深度集合中对象的可变性有稍微不同的影响。

1. copyWithZone:创建对象的最外层 surface level不可变，所有更深层次对象的可变性不变。
2. mutableCopyWithZone:创建对象的最外层 surface level可变，所有更深层次对象的可变性不变。
3. initWithArray: copyItems:第二个参数为NO，此时，所创建数组最外层可变性与初始化的可变性相同，所有更深层级对象可变性不变。
4. initWithArray: copyItems:第二个参数为YES，此时，所创建数组最外层可变性与初始化的可变性相同，下一层级是不可变的，所有更深层级对象可变性不变。
5. 归档、解档复制的集合，所有层级的可变性与原始对象相同。

### 1.3 自定义对象的深复制浅复制

自定义对象，需要实现`<NSCopying>`协议,才能进行copy. 代码如下所示：

```
#import <Foundation/Foundation.h>


@interface Person : NSObject
@property (nonatomic,strong) NSString *name;
@property (nonatomic,assign) NSUInteger age;

- (void)setName:(NSString *)name andAge:(NSUInteger)age;
@end


#import "Person.h"
@interface Person()<NSCopying>

@end

@implementation Person

- (id)copyWithZone:(NSZone *)zone
{
    Person *person = [[[self class] allocWithZone:zone] init];
    person.name = self.name;
    person.age  = self.age;
    return person;
}

- (void)setName:(NSString *)name andAge:(NSUInteger)age
{
    _name = name;
    _age  = age;
}

@end

#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    Person *person = [[Person alloc] init];
    [person setName:@"James" andAge:35];
    
    Person *personCopy = [person copy];
    [personCopy setName:@"Tom" andAge:23];
    
    NSLog(@"\n person:%p \n personCopy:%p",person,personCopy);
    return 0;
}
```

运行结果：

```
 person:0x102813d50 
 personCopy:0x102813e50
```

