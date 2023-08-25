---
layout: post
title: IOS程序在main函数前所做的事情
description: 看看 main 函数之前都发生了什么
category: blog
tag: ios
---

直观上看，ios程序的入口是main函数。实际上操作系统在加载我们的二进制可执行文件时执行到`main`函数之前做了很多事情。比如 OC的runtime运行库就是在main之前建立好的。下面我们通过一个截图来看一下main函数之前系统到底做了哪些事情。

从OC的runtime的入口函数着手，`_objc_init`,我们加一个符号断电，可以看到如下结果：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180427IosMain/main_01.png)

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180427IosMain/main_02.png)

从上图可以看到，程序的入口是 `_dyld_start` 。 

## 动态链接库

iOS 中用到的所有系统 framework 都是动态链接的，类比成插头和插排，静态链接的代码在编译后的静态链接过程就将插头和插排一个个插好，运行时直接执行二进制文件；而动态链接需要在程序启动时去完成“插插销”的过程，所以在我们写的代码执行前，动态连接器需要完成准备工作。

Xcode中有一个link列表：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180427IosMain/main_03.png)

这些 framework 将会在动态链接过程中被加载，另外还有隐含 link 的 framework，可以测试出来：先找到可执行文件，我这里叫 QiDemo 的工程，模拟器路径下找到 QiDemo.app，可执行文件默认同名，再通过 otool命令：

```
qis-Mac-mini:Debug-iphoneos qi$ cd QiDemo.app/
qis-Mac-mini:QiDemo.app qi$ ls
Base.lproj			QiDemo
Info.plist			_CodeSignature
PkgInfo				embedded.mobileprovision
qis-Mac-mini:QiDemo.app qi$ otool -L QiDemo 
QiDemo:
	/System/Library/Frameworks/Foundation.framework/Foundation (compatibility version 300.0.0, current version 1452.23.0)
	/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
	/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.1)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.50.4)
	/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation (compatibility version 150.0.0, current version 1452.23.0)
	/System/Library/Frameworks/UIKit.framework/UIKit (compatibility version 1.0.0, current version 3698.52.10)
```
从上面结果可以看出，有两个默认添加的lib：`libobjc`即 objc和rutime, `libSystem`中包含了很多系统级别的lib，熟知的有：

* libdispatch(GCD)
* libsystem_c (C语言库)
* libsystem_block (Block)
* libcommonCrypto (加密库，比如通常用的md5)

这些lib都是dylib格式（如windows中的dll），系统使用动态库有以下几点好处：

* 代码公用 ： 很多程序都动态连接了这些lib，但它们在内存和磁盘中公用一份
* 易于维护 ： 由于被依赖的lib是程序执行时才link的，所以这些lib很容易做更新，比如`libSystem.dylib`是`libSystem.B.dylib`的替身，那天想升级直接换成`libSystem.C.dylib` 然后在替换替身就行了。
* 减少可执行文件体积：相比静态链接，动态链接在编译时不需要打进去，所以可执行文件的体积要小很多。

## dyld

dyld（the dynamic link editor），Apple 的动态链接器，系统 kernel 做好启动程序的初始准备后，交给 dyld 负责，援引并翻译《 Mike Ash 这篇 blog 》对 dyld 作用顺序的概括：

1. 从 kernel 留下的原始调用栈引导和启动自己
2. 将程序依赖的动态链接库递归加载进内存，当然这里有缓存机制
3. non-lazy 符号立即 link 到可执行文件，lazy 的存表里
4. non-lazy 符号立即 link 到可执行文件，lazy 的存表里
5. 找到可执行文件的 main 函数，准备参数并调用
6. 程序执行中负责绑定 lazy 符号、提供 runtime dynamic loading services、提供调试器接口
7. 程序main函数 return 后执行 static terminator
8. 某些场景下 main 函数结束后调 libSystem 的 _exit 函数

## ImageLoader

当然这个 image 不是图片的意思，这里的ImageLoader指的是镜像加载，负责从镜像文件中取出各种我们需要的信息，主要是决议了一些需要在动态链接时的符号。 

我们的程序在运行的时候除了自己的可执行镜像，还会依赖许多其他的库，有些在我们编译链接时给link进去了，这是静态链接，比如Podfile里面配置的第三方库（framework,.a文件,.so文件），有些是我们的程序启动时链接的库，比如最常见的libobjc.A.dylib，里面包含了objc的runtime。

它大概表示一个二进制文件（可执行文件或 so 文件），里面是被编译过的符号、代码等，所以 ImageLoader 作用是将这些文件加载进内存，且每一个文件对应一个ImageLoader实例来负责加载。

1. 在程序运行时它先将动态链接的 image 递归加载 （也就是上面测试栈中一串的递归调用的时刻）
2. 再从可执行文件 image 递归加载所有符号

当然所有这些都发生在我们真正的main函数执行前。


那么我们的iOS/MacOSX应用程序是如何动态加载的呢？还是从刚才的堆栈和对照dyld源码，runtime源码探索

搜索`_dyld_start`发现启动的部分在文件`dyldStartup.s`，里面是一坨汇编，就不细看了。
循着堆栈的踪迹，看到在函数`dyld::_main`这里，根据源码中的注释可以看到，`dyld::_main`是dyld的入口，内核在加载dyld时，从 `dyld_start` -> `dyld::main` `dyld:main`做一些注册操作并返回真正的应用程序的main函数地址.

```
//
// Entry point for dyld.  The kernel loads dyld and jumps to __dyld_start which
// sets up some registers and call this function.
//
// Returns address of main() in target program which __dyld_start jumps to
//
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
    ....
}
```

`dyld`在加载到`libdispatch.dylib`时，会调用到`_objc_init`，看下这个函数的代码

```
//objc-os.mm
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();
        
    // Register for unmap first, in case some +load unmaps something
    _dyld_register_func_for_remove_image(&unmap_image);
    dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_2_images);
    dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 
    0/*not batch*/, &load_images);
}
```

前面是一些环境的初始化，重点看后面的dyld_xxx方法，这几个方法向dyld注册了回调:

1. _dyld_register_func_for_remove_image(&unmap_image);就如注释所说，先注册一个remove_image的方法，为了避免用户在+(void)load里面去unmap一些东西。
2. dyld_register_image_state_change_handler这里注册了一个回调，当dyld_image状态为dyld_image_state_bound时，触发回调。
3. dyld_register_image_state_change_handler这里注册了一个回调，当dyld_image状态为dyld_image_state_dependents_initialized时，触发回调。

当我们在category或者类中添加+(void) load方法时，会在这个时机调用。
具体我们在load_images中可以看到有如下代码片段

```
 const char *
load_images(enum dyld_image_states state, uint32_t infoCount,
            const struct dyld_image_info infoList[])
{
        bool found;

    // Return without taking locks if there are no +load methods here.
        found = false;
        //这里会找所有的class以及category的列表里面是否有+load方法
        for (uint32_t i = 0; i < infoCount; i++) {
            if (hasLoadMethods((const headerType *)infoList[i].imageLoadAddress)) {
                found = true;
                break;
            }
        }
        if (!found) return nil;
        
        
        recursive_mutex_locker_t lock(loadMethodLock);

        // Discover load methods
        // 搜集所有的+load方法
        {
            rwlock_writer_t lock2(runtimeLock);
            found = load_images_nolock(state, infoCount, infoList);
        }
        //调用+load方法
        if (found) {
            call_load_methods();
        }
        ...
    }
    
    bool 
load_images_nolock(enum dyld_image_states state,uint32_t infoCount,
                   const struct dyld_image_info infoList[])
    {
        bool found = NO;
        uint32_t i;
    
        i = infoCount;
        while (i--) {
            const headerType *mhdr = (headerType*)infoList[i].imageLoadAddress;
            if (!hasLoadMethods(mhdr)) continue;
            // 搜集镜像中的load方法
            prepare_load_methods(mhdr);
            found = YES;
        }
    
        return found;
    }
    
  
  
  void prepare_load_methods(const headerType *mhdr)
    {
        size_t count, i;
    
        runtimeLock.assertWriting();
    
        //1. 先搜集class中的+load方法
        classref_t *classlist = 
            _getObjc2NonlazyClassList(mhdr, &count);
        for (i = 0; i < count; i++) {
            schedule_class_load(remapClass(classlist[i]));
        }
    
        category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
        //2. 再搜集category中的+load方法
        for (i = 0; i < count; i++) {
            category_t *cat = categorylist[i];
            Class cls = remapClass(cat->cls);
            if (!cls) continue;  // category for ignored weak-linked class
            realizeClass(cls);
            assert(cls->ISA()->isRealized());
            add_category_to_loadable_list(cat);
        }
    }
    
    //递归实现，可以看到这里是先找父类中的+load方法，并放入到一个列表里面，所以在最终的class的load方法列表里面，父类的+load方法是先于子类的+load方法的
    //大家可以看下add_class_to_loadable_list的实现，里面的分配内存策略使用了指数增长策略
    static void schedule_class_load(Class cls)
    {
        if (!cls) return;
        assert(cls->isRealized());  // _read_images should realize
    
        if (cls->data()->flags & RW_LOADED) return;
    
        // Ensure superclass-first ordering
        schedule_class_load(cls->superclass);
    
        add_class_to_loadable_list(cls);
        cls->setInfo(RW_LOADED); 
    }
```

```
 void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

为什么要重复调用call_class_loads和call_category_loads呢，答案是由于这个函数被设计为可重入的，重入时这个函数直接return。这样当我们在+load函数中又去map了其他的镜像，这个镜像又有许多其他新的包含+load的类和分类，这里就是为了保证新添加的+load可以调用到。
在call_class_loads中也可以看到，是从之前保存类的+load方法的列表头部开始一个一个取出来并执行的，所有我们的父类的+load要先于子类的+load执行。
call_load_methods执行完毕后，所有+load方法就执行完毕了。

在上面第二步有一个回调map_2_images，我们来看看这个做了什么。

```
 const char *
map_images_nolock(enum dyld_image_states state, uint32_t infoCount,
                  const struct dyld_image_info infoList[])
    {
        static bool firstTime = YES;
        static bool wantsGC = NO;
        uint32_t i;
        header_info *hi;
        header_info *hList[infoCount];
        uint32_t hCount;
        size_t selrefCount = 0;
    
        // Perform first-time initialization if necessary.
        // This function is called before ordinary library initializers. 
        // fixme defer initialization until an objc-using image is found?
        if (firstTime) {
            preopt_init();
        }
    
        // Find all images with Objective-C metadata.
        hCount = 0;
        i = infoCount;
        //收集infoList中的所有header信息并存储在一个单链表
        while (i--) {
            const headerType *mhdr = (headerType *)infoList[i].imageLoadAddress;
    
            hi = addHeader(mhdr);
            if (!hi) {
                // no objc data in this entry
                continue;
            }
    
            if (mhdr->filetype == MH_EXECUTE) {
                // Size some data structures based on main executable's size
                size_t count;
                _getObjc2SelectorRefs(hi, &count);
                selrefCount += count;
                _getObjc2MessageRefs(hi, &count);
                selrefCount += count;
            }
    
            hList[hCount++] = hi;
            
        }
    
        if (firstTime) {
            //注册一些我们常用的函数，比如load,alloc,dealloc等等
            //并使用selrefCount的值 去创建一个capacity为这个值的全局Map
            sel_init(wantsGC, selrefCount);
            //初始化AutoreleasePool，和一个SideTable，SideTable是干啥的也没弄清楚，暂时不管，这里不重要
            arr_init();
        }
        //重点是这个，这个会去读取hList里面的所有的image
        //中的classes，categories，protocols并存储在内存中
        //使我们之后使用runtime方法提供了前提，比如方法替换，消息转发等等
        _read_images(hList, hCount);
    
        firstTime = NO;
    
        return NULL;
    }

```

看完回调的实现后，回到dyld在加载libdipsatch.dylib后，再加载其他镜像时，就会跑到这些回调，所有这些镜像内部的OC对象，方法，分类，协议等信息都会在runtime中存储好。一切就绪后，就开始跑到我们main函数了。

## runtime 与 +load

刚才讲到 libSystem 是若干个系统 lib 的集合，所以它只是一个容器 lib 而已，而且它也是开源的，里面实质上就一个文件，init.c，由 libSystem_initializer 逐步调用到了 _objc_init，这里就是 objc 和 runtime 的初始化入口。

除了 runtime 环境的初始化外，_objc_init中绑定了新 image 被加载后的 callback：

```
dyld_register_image_state_change_handler(
dyld_image_state_bound, 1, &map_images);
dyld_register_image_state_change_handler(
dyld_image_state_dependents_initialized, 0, &load_images);
```

可见 dyld 担当了 runtime 和 ImageLoader 中间的协调者，当新 image 加载进来后交由 runtime 大厨去解析这个二进制文件的符号表和代码。继续上面的断点法，断住神秘的 +load 函数：

清楚的看到整个调用栈和顺序：

1. dyld 开始将程序二进制文件初始化
2. 交由 ImageLoader 读取 image，其中包含了我们的类、方法等各种符号
3. 由于 runtime 向 dyld 绑定了回调，当 image 加载到内存后，dyld 会通知 runtime 进行处理
4. runtime 接手后调用 map_images 做解析和处理，接下来 load_images 中调用 call_load_methods 方法，遍历所有加载进来的 Class，按继承层级依次调用 Class 的 +load 方法和其 Category 的 +load 方法

至此，可执行文件中和动态库所有的符号（Class，Protocol，Selector，IMP，…）都已经按格式成功加载到内存中，被 runtime 所管理，再这之后，runtime 的那些方法（动态添加 Class、swizzle 等等才能生效）

## 关于 +load 方法的几个问题

Q: 重载自己 Class 的 +load 方法时需不需要调父类？
A: runtime 负责按继承顺序递归调用，所以我们不能调 super

Q: 在自己 Class 的 +load 方法时能不能替换系统 framework（比如 UIKit）中的某个类的方法实现
A: 可以，因为动态链接过程中，所有依赖库的类是先于自己的类加载的

Q: 重载 +load 时需要手动添加 @autoreleasepool 么？
A: 不需要，在 runtime 调用 +load 方法前后是加了 `objc_autoreleasePoolPush()` 和 `objc_autoreleasePoolPop()` 的。

Q: 想让一个类的 +load 方法被调用是否需要在某个地方 import 这个文件
A: 不需要，只要这个类的符号被编译到最后的可执行文件中，+load 方法就会被调用（Reveal SDK 就是利用这一点，只要引入到工程中就能工作）

## 总结

整个事件由 dyld 主导，完成运行环境的初始化后，配合 ImageLoader 将二进制文件按格式加载到内存。

动态链接依赖库，并由 runtime 负责加载成 objc 定义的结构，所有初始化工作结束后，dyld 调用真正的 main 函数。

值得说明的是，这个过程远比写出来的要复杂，这里只提到了 runtime 这个分支，还有像 GCD、XPC 等重头的系统库初始化分支没有提及（当然，有缓存机制在，它们也不会玩命初始化），总结起来就是 main 函数执行之前，系统做了茫茫多的加载和初始化工作，但都被很好的隐藏了，我们无需关心。


## 参考

* [BLOGARCHIVEWEIBOGITHUBRSS
iOS 程序 main 函数之前发生了什么](https://blog.sunnyxx.com/2014/08/30/objc-pre-main/)
* [App进入Main之前做的事情](http://tutudev.com/2016/06/30/iosbeforemain/)