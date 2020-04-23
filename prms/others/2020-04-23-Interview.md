---
layout:     post
title:      Interview
subtitle:    
date:       2020-04-23
author:     gitKong
header-img: img/post-bg-miui6.jpg
catalog: true
tags:
    - iOS
---

# runtime相关问题

# 结构模型

1、绍下runtime的内存模型（isa、对象、类、metaclass、结构体的存储信息等）

```
isa 是一个共有体，64位指针，指向Class，存储一些状态信息和部分引用计数

对象的结构体是objc_object，其中成员有isa指针，指向的是类对象

类的结构体是objc_class，继承自objc_object，其中有4个成员变量，isa指向元类；superclass指向父类；cache_t用于缓存方法等信息，加速方法调用；class_data_bits_t结构体就是存储实例对象的方法、属性和协议等信息

元类也是类对象，调用类方法的时候为了和对象查找方法的机制一致，才引入元类，存储类的类方法，其中isa指向根元类，superclass指向父元类

```

2、为什么要设计metaclass

```
实例对象的方法等信息存储到类对象中，那么类方法就要设计一个样的机制来存储，所以引入元类。
```

3、class_copyIvarList & class_copyPropertyList区别

```
class_copyIvarList  是包括h/m文件中成员变量以及类的属性；而class_copyPropertyList仅仅包含h/m文件中由property声明的属性
```

4、class_rw_t 和 class_ro_t 的区别

```
class_rw_t 表示类的可读写指针，class_ro_t表示类的可读指针。
class_rw_t结构体成员变量中有class_ro_t结构体、类的方法列表、属性列表以及遵循的协议列表等数据，而class_ro_t结构体成员变量也有类的方法列表、属性列表以及遵循的协议列表。
编译期间，通过objc_class结构体中的data()函数获取到的class_rw_t实际上是class_ro_t，编译期间会把类的相关信息存储到class_ro_t中，
当objc加载到运行时的时候，通过realizeClass方法，新增一个class_rw_t结构体后，赋值class_io_t结构体，并通过methodizeClass把class_io_t的方法列表、属性列表、协议列表信息存储到class_rw_t的methods、propertys、protocols中。
```

5、category如何被加载的,两个category的load方法的加载顺序，两个category的同名方法的加载顺序

```
category 在objc加载到运行时的时候（即objc_init中）加载，最终可在_read_images方法中实现。
类的load先于category，而category同名方法加载顺序取决于编译顺序
```

6、category & extension区别，能给NSObject添加Extension吗，结果如何

```
extension是类的拓展，是类的一部分，常用于添加私有属性，编译期间就跟随类加载，并且必须有一个类的源码才能为一个类添加extension，所以无法为系统的类添加extension。

category是分类，是在运行时的时候加载，无法添加属性。
```

7、消息转发机制，消息转发机制和其他语言的消息机制优劣对比
[http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)

```
当消息传递无法找到合适的方法实现，就会进入消息转发阶段，首先会进行动态方法解析，一般在此方法中动态新增方法实现，如果没有，就进入下一步寻找合适的消息处理对象，有则继续走消息传递，没有就进入消息重定向，获取合适的方法签名（方法的参数类型、返回值类型信息），如果有签名，则进入最后一步消息执行，否则提示找不到方法。

其他语言一般都是静态绑定，就是在编译期就能决定运行时所调用的函数；
而OC的消息转发机制是动态绑定，在运行期才能确定调用函数

优点：可以动态新增方法、改变篡改 receiver等
缺点：执行效率相对慢、安全性较低。
```

8、在方法调用的时候，方法查询-> 动态解析-> 消息转发 之前做了什么

```
方法调用的时候，会转变为objc_msgSend或者其他msgSend方法，这个方法的的第一个参数是方法receiver接受者、第二个参数是方法的SEL（方法名）,其余都是附加参数。

objc_msgSend 汇编实现中CacheLookup 宏就是查询方法是否缓存，在此之前会判断receiver是否为空，空则不处理，再次会获取对象的isa指向，获取对应类对象


1、检查忽略的Selector，比如当我们运行在有垃圾回收机制的环境中，将会忽略retain和release消息。

2、检查receiver是否为nil。

```

```
ENTRY    _objc_msgSend
    MESSENGER_START

    NilTest    NORMAL

    GetIsaFast NORMAL        // r11 = self->isa
    CacheLookup NORMAL        // calls IMP on success

    NilTestSupport    NORMAL

    GetIsaSupport       NORMAL

// cache miss: go search the method lists
LCacheMiss:
    // isa still in r11
    MethodTableLookup %a1, %a2    // r11 = IMP
    cmp    %r11, %r11        // set eq (nonstret) for forwarding
    jmp    *%r11            // goto *imp

    END_ENTRY    _objc_msgSend
```

9、IMP、SEL、Method的区别和使用场景

```
Method 结构体中有是三个成员变量：
SEL name 方法选择器，表示方法名 
char*types 表示方法的类型编码Type Encoding，以返回值类型+参数类型的组合编码
IMP imp 表示方法的实现，第一个参数消息的接受者；第二个参数代表方法的选择子；... 代表可选参数，前面的 id 代表返回值。

Method是存储在class_rw_t结构体中的method_array_t methods成员变量或class_ro_t结构体中的method_list_t baseMethodList成员变量中存储，表示一个方法信息

方法调用，通过SEL查找到对应的Method，然后执行IMP实现。
```

10、load、initialize方法的区别什么？在继承关系中他们有什么区别

```
load方法表示：当类第一次加载的时候调用
initialize方法表示：当类第一次被使用的时候调用

继承中的关系：
load方法：
1、子类没有实现，会调用父类的
2、子类和父类都实现，则父类先调用

initialize方法：
1、子类没有实现，会调用父类的方法多次
2、子类和父类都实现，则父类先调用

与分类的关系：

load：
主类与分类的加载顺序是: 主类优先于分类加载，无关编译顺序；
分类间的加载顺序取决于编译的顺序: 先编译先加载，后编译则后加载

initialize:
当有多个 Category 都实现，会覆盖类中的方法，只执行一个(最后编译那个)。
```

```
+load

+load方法会在runtime加载类、分类时调用
每个类、分类的+load，在程序运行过程中只调用一次
调用顺序：
1、先调用类的+load
√ 按照编译先后顺序调用（先编译，先调用）
√ 调用子类的+load之前会先调用父类的+load
2、再调用分类的+load
√ 按照编译先后顺序调用（先编译，先调用）

+initialize

+initialize方法会在类第一次接收到消息时调用
调用顺序
1、先调用父类的+initialize，再调用子类的+initialize
2、(先初始化父类，再初始化子类，每个类只会初始化1次)
+initialize和+load的很大区别是，+initialize是通过objc_msgSend进行调用的，所以有以下特点
√ 如果子类没有实现+initialize，会调用父类的+initialize（所以父类的+initialize可能会被调用多次）
√ 如果分类实现了+initialize，就覆盖类本身的+initialize调用
```

# 11、category的实现原理，能否添加属性？

```
分类的实现原理是将category中的方法，属性，协议数据放在category_t结构体中，在加载到运行时（objc_init）的时候，remethodizeClass函数将结构体内的方法列表拷贝到类对象的方法列表中。

Category可以添加属性，但是并不会自动生成成员变量及set/get方法。因为category_t结构体中并不存在成员变量。
实例对象的方法、属性、成员变量、协议在编译期就会确立好内存存储，存储在class_ro_t中，当运行时加载的时候，才会把class_ro_t数据同步到class_rw_t中。
而分类也是运行时的时候加载的，因此无法添加成员变量。
```

# 12、_objc_msgForward函数是做什么的，直接调用它将会发生什么？

```
_objc_msgForward是 IMP 类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，_objc_msgForward会尝试做消息转发。

直接调用：会跳过方法查询步骤，直接进入转发。

例子：JSPatch 中利用方法替换，把要监听的方法替换为_objc_msgForward，
然后在forwardInvocation方法中获取NSInvocation对象，包含方法的参数信息，
然后传递到js中
```

---

# 内存管理

1、weak的实现原理？SideTable的结构是什么样的

```
通过__weak 修饰的对象，最终会调用storeWeak函数，参数location表示weak指针，参数newObj表示被引用的对象指针。

1、首先会获取全局的sideTable Map，根据被引用的对象指针，获取对应的sideTable结构体。
2、判断该对象是否在sideTable的weak_table_t中有weak_entry_t，如果有，则表示已经有weak引用过，
则直接把新的weak指针添加到weak_entry_t的weak_refeners_t数组中，否则通过weak指针和weak引用对象新建weak_entry_t，存到weak_table中。

SideTable是一个结构体，其中有是三个成员变量，weak_table_t weak_table；表示存储的弱引用表；spinlock slock；互斥锁用于线程安全；RefcountMap refcnts;引用计数的map

```

2、关联对象的应用？系统如何实现关联对象的

```
如何实现：

通过objc_setAssociatedObject把object、key、value、policy入参，函数内部处理：
1、通过AssociationManager单例获取全局AssociationHashMap哈希表
2、新建ObjectAssociationMap C++map类，通过object参数的地址取反后作为Key，
ObjectAssociationMap对象作为value存到全局AssociationHashMap表中

其中ObjectAssociationMap 哈希map是以参数key为key，ObjcAssociation C++类为value来存储；而ObjcAssociation的成员变量就是参数value和policy

应用：
1、因为关联对象的销毁是伴随着实例对象销毁，通过监控关联对象的销毁间接监控对象销毁，可做一些自动释放操作，如为 KVO 创建一个关联的观察者。
2、为现有的类添加私有变量以帮助实现细节；
3、为现有的类添加公有属性；

```

3、关联对象的如何进行内存管理的？关联对象如何实现weak属性

```
内存管理：
关联对象会在对象释放的时候自动释放，即对象的dealloc中会检测类是否有关联对象，有就调用_object_remove_assocations函数释放

weak的实现前提是，存在一个__weak 指针指向到被引用对象的地址，dealloc时通过引用的对象查找到weak指针，然后做置空操作。
而关联对象跟NSObject 对象并不存在指针这样的中间媒介，因此只提供了OBJC_ASSOCIATION_ASSIGN

管理对象如何实现weak属性：
1、新增NSObject类，声明一个weak修饰的 id属性
2、在对应分类中新增关联对象，用weak修饰，在set和get方法中通过新增的NSObject类包装此关联对象进行存取

```

4、Autoreleasepool的原理？所使用的的数据结构是什么

```
数据结构：
由一个个page大小为4096字节的AutoreleasePoolPage节点组成的双向链表。

原理：
Autoreleasepool 是通过AutoreleasePoolPage 的Push和Pop方法对autorelease对象进行入栈和出栈
AutoreleasePool结构体初始化后会调用AutoreleasePoolPage 的Push方法，然后先添加哨兵对象（nil）到当前page的next指针指向位置，并返回的是哨兵对象（nil），Pop的入参也是这个哨兵对象。
autorelease的对象会添加到当前page的next指针指向位置，当AutoreleasePool结构体 析构的时候，会调用AutoreleasePoolPage的Pop方法，然后根据哨兵对象的地址，查找比它高地址的对象 发送release消息
```

5、ARC的实现原理？ARC下对retain & release做了哪些优化

```
ARC的实现原理：
通过编译期自动插入retain、release、autorelease来控制引用计数，当引用计数为0的时候会自动释放对象；
其中objc_retain会将引用计数存储在isa.extra_rc变量（8个字节），当extra_rc溢出（上溢）后就会将extra_rc的一半引用计数迁移到SideTable的RefcountMap中。
objc_release会先处理isa.extra_rc变量的引用计数，当处理完（下溢）后把SideTable的RefcountMap中引用计数转回到extra_rc中。

```
[https://juejin.im/entry/5976a063f265da6c261d8b13](https://juejin.im/entry/5976a063f265da6c261d8b13)

[http://blog.sunnyxx.com/2014/10/15/behind-autorelease/](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

```
优化：

// ARC
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    if (YES) {  // if语句目的是让person对象的生命周期仅在当前作用域有效.
        Person *_person = [Person newPerson]; // 这里只对`new`关键字开头的方法进行测试, 其他三种方法可自行验证.
        NSLog(@"if语句执行中 %@", _person);
    }

    NSLog(@"if语句执行完");

}

@end

解析：

id objc_autoreleaseReturnValue(id obj)
{
#if SUPPORT_RETURN_AUTORELEASE
    assert(tls_get_direct(AUTORELEASE_POOL_RECLAIM_KEY) == NULL);

    if (callerAcceptsFastAutorelease(__builtin_return_address(0))) {
        tls_set_direct(AUTORELEASE_POOL_RECLAIM_KEY, obj);
        return obj;
    }
#endif

    return objc_autorelease(obj);
}

objc_autoreleaseReturnValue通过判断对象是否想持有，如果是，则通过TLS存储对象并直接返回。上诉例子中_person 是强引用，那么就表示想持有，正常来说，此时是需要retain一次。


id objc_retainAutoreleasedReturnValue(id obj)
{
#if SUPPORT_RETURN_AUTORELEASE
    if (obj == tls_get_direct(AUTORELEASE_POOL_RECLAIM_KEY)) {
        tls_set_direct(AUTORELEASE_POOL_RECLAIM_KEY, 0);
        return obj;
    }
#endif
    return objc_retain(obj);
}

优化通过调用objc_retainAutoreleasedReturnValue代替retain，内部通过TLS判断当前对象是否跟存储的一致，一致则直接返回，无需retain

ARC下为了免除autorelease和retain这两步多余的操作来提高性能，使用函数objc_autoreleaseReturnValue来替代autorelease、
函数objc_retainAutoreleasedReturnValue来替代retain，方法内部利用TLS(线程局部存储)存储对象，避免retain和autorelease的调用，提高性能。

```

6、ARC下哪些情况会造成内存泄漏

```
1、使用CoreFoundation对象
2、循环引用：代理、block、NSTimer
3、CGImageRef、C的malloc等
```

7、autoreleasepool的创建和销毁时机
[http://blog.sunnyxx.com/2014/10/15/behind-autorelease/](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

```
UIKit通过RunLoopObserver在runloop的两次sleep间，对AutoreleasePool进行pop和push，将这次Loop中产生的Autorelease对象进行释放

在没有手加Autorelease Pool的情况下，Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop

自动释放池的创建和释放，销毁的时机如下所示

kCFRunLoopEntry; // 进入runloop之前，创建一个自动释放池
kCFRunLoopBeforeWaiting; // 休眠之前，销毁自动释放池，创建一个新的自动释放池
kCFRunLoopExit; // 退出runloop之前，销毁自动释放池
```

8、ARC下，release 和 retain的插入时机是在哪？

```
objc_storeStrong的时候插入release和retain，也就是setter方法

autoreleasepool的时候插入，当对象标记为autorelease后加入到pool，在当前runLoop休眠前会把autoreleasepool的对象调用release
```

9、Runloop的实现逻辑
[https://blog.ibireme.com/2015/05/18/runloop/](https://blog.ibireme.com/2015/05/18/runloop/)
s
```
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```

10、什么时候用@autoreleasepool

根据[Apple的文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html)，使用场景如下：

```
*   1、写基于命令行的的程序时，就是没有UI框架，如AppKit等Cocoa框架时。
*   2、写循环，循环里面包含了大量临时创建的对象。
*   3、创建了新的线程。（非Cocoa程序创建线程时才需要）,子线程runloop默认
       不启动，所以不会开启一个autoreleasePool，需要手动启动或者添加
*   4、长时间在后台运行的任务。
```

11、NSThread中是否需要手动添加@autoreleasepool

```
NSThread在iOS7之后才自动创建线程中的AutoreleasePool，在此之前还是需要这样处理的，当然，新建NSThread后，最好也手动创建一个，苹果文档也是这样建议
```

11、子线程的@autoreleasepool什么时候释放

```
1.子线程在使用autorelease对象时，如果没有autoreleasepool会在
autoreleaseNoPage中懒加载一个出来。

2.在runloop的run:beforeDate，以及一些source的callback中，有
autoreleasepool的push和pop操作，总结就是系统在很多地方都差不多
autorelease的管理操作。

3.就算插入没有pop也没关系，在线程exit的时候会释放资源，执行
AutoreleasePoolPage::tls_dealloc，在这里面会清空autoreleasepool。
```

---


# 其他

1、Method Swizzle注意事项

```
分两种情况：
一、交换的都是本类或者父类的方法
1、交换的方法不能只有声明没有实现，否则陷入死循环
2、本类中没有这个method，需要先add_method

二、交换的不是同一个类的方法
注意消息接收者self是隐式参数，交换方法并没有交换实例，方法内使用self需要注意，避免crash
```

2、属性修饰符atomic的内部实现是怎么样的?能保证线程安全吗

```
atomic 表示原子性，内部利用自旋锁来确保属性的读写是线程安全的，因此性能消耗相对高。但是不能确保外部使用时线程安全的。
就是说属性的setter和getter是线程安全的，除此之外，都不能保证线程安全。
```

3、iOS 中内省的几个方法有哪些？内部实现原理是什么

```
1、- isKindOfClass   自己或者父类的isa跟传入的aClass比较
2、- isMemberOfClass   自己的isa跟传入的aClass比较
3、- respondsToSelector
4、+ instancesRespondToSelector
```

```
isKindOfClass 判断是否是自己或者父类, isMemberOfClass 判断是否是自己
-(BOOL)isKindOfClass:(Class)aClass{
            for (Class tcls = isa; tcls; tcls-> superclass) { 
                    if (tcls == aClass) {
                             return YES; 
                    }
            }
        return NO;
}

-(BOOL)isMemberOfClass:(Class)aClass{
          return isa == aClass;
}

bool class_respondsToSelector_inst(Class cls, SEL sel, id inst)
{
    IMP imp;

    if (!sel  ||  !cls) return NO;

    // Avoids +initialize because it historically did so.
    // We're not returning a callable IMP anyway.
    imp = lookUpImpOrNil(cls, sel, inst, 
                         NO/*initialize*/, YES/*cache*/, YES/*resolver*/);
    return bool(imp);
}

IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}
```

4、class、objc_getClass、object_getclass 方法有什么区别?

```
class：返回类对象
Class objc_getClass(const char *aClassName) 根据类名返回对于的类对象
Class object_getClass(id obj) 根据obj参数（instance对象、class对象、meta-class对象）返回类对象、元类对象、基元类对象
```

5、Type Encoding是什么

```
Type 类型对于静态语言来说，一般在编译期用来做静态语法检测，以及确立内存的分配；
而对于OC动态语言来说，要实现运行时的消息分发，就需要把类型信息保存起来，这个就是Type Encoding。
方法的类型信息是通过返回值类型+参数类型组合编码而成。具体写法参考文档描述。
```

6、[self performSelector:@selector(newObjc)]; 有什么问题？

```
编译不通过。new开头的方法，认为是构造方法，构造的对象自身会持有，痛performSelector会造成内存泄漏。
```

7、@synchronized 的原理，如何自己实现？
[http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

[http://mrpeak.cn/blog/synchronized/](http://mrpeak.cn/blog/synchronized/)

```
编译时候，会转化成objc_sync_enter、objc_sync_exit插在block代码的起点和末尾

synchronized中传入的object的内存地址，被用作key，通过hash map对应的一个系统维护的递归锁。不管是传入什么类型的object，只要是有内存地址，就能启动同步代码块的效果

@synchronized 后面需要紧跟一个 OC 对象，它实际上是把这个对象当做锁来使用。这是通过一个哈希表来实现的，OC 在底层使用了一个互斥锁的数组(你可以理解为锁池)，通过对对象去哈希值来得到对应的互斥锁。
```

8、@synchronized (self){} 有什么问题？

```
1、多处地方共用一个锁，而共用一个锁的代码块，都必须按照顺序执行，降低代码效率。
2、外部可能使用当前对象作为锁，当两个公共锁交替使用时，容易引起死锁问题
```

9、事务的特征

```
事务具有四个特征：原子性（ Atomicity ）、一致性（ Consistency ）、隔离性（ Isolation ）和持久性（ Durability ）。这四个特性简称为 ACID 特性。
1 、原子性
事务是数据库的逻辑工作单位，不可分割，事务中包含的各操作要么都做，要么都不做
2 、一致性
事务执行的结果必须是使数据库从一个一致性状态变到另一个一致性状态。因此当数据库只包含成功事务提交的结果时，就说数据库处于一致性状态。如果数据库系统 运行中发生故障，有些事务尚未完成就被迫中断，这些未完成事务对数据库所做的修改有一部分已写入物理数据库，这时数据库就处于一种不正确的状态，或者说是 不一致的状态。
3 、隔离性
一个事务的执行不能其它事务干扰。即一个事务内部的操作及使用的数据对其它并发事务是隔离的，并发执行的各个事务之间不能互相干扰。 事务的隔离级别有4级，可以查看这篇博客，[关于MySQL的事务处理及隔离级别](http://blog.sina.com.cn/s/blog_4c197d420101awhc.html)
4 、持续性
也称永久性，指一个事务一旦提交，它对数据库中的数据的改变就应该是永久性的，不能回滚。接下来的其它操作或故障不应该对其执行结果有任何影响。

```

---

---

# NSNotification相关

1、实现原理（结构设计、通知如何存储的、name&observer&SEL之间的关系等）

```
结构设计：
NSNotification 通知对应实例，一个通知对应一个，其中有属性name（通知名）、object（附带对象）、userInfo（配置信息）
NSNotificationCenter，通知单例，负责监听和发送通知
NSNotificationQueue 通知队列，用于异步发送消息，这个异步并不是开启线程，而是把通知存到双向链表实现的队列里面，等待某个时机触发时调用NSNotificationCenter的发送接口进行发送通知

如何存储：
1、监听时设置name；
存储到一个map 表中，其中name为key，value为key是object，value是Observation结构体为节点的链表。
Observation有其中的三个成员变量：id observer 观察这，接收通知的对象，SEL selector 响应的方法，next指针，指向下一个Observation

2、监听时只设置object：
存储到一个map 表中，其中object为key，value是Observation结构体为节点的链表。

3、监听时没设置name和object：
直接存储到Observation结构体为节点的链表中
```

2、通知的发送时同步的，还是异步的
NSNotificationCenter接受消息和发送消息是在一个线程里吗？如何异步发送消息

```
通知发送需要通过NSNotificationCenter同步发送
没指定线程处理接收消息的话，默认是接收和发送在同一个线程
可通过NSNotificationQueue实现异步发送消息，注意不是开启线程，而是把通知存储到双向链表中，在RunLoop的某个时刻进行发送，依赖runloop，三种可选状态NSPostingStyle
```

3、NSNotificationQueue是异步还是同步发送？在哪个线程响应

```
NSNotificationQueue最终还是通过NSNotificationCenter进行同步发送，注册通知时没指定线程处理的话，响应的线程就是发送的线程
```

4、NSNotificationQueue和runloop的关系

```
NSNotificationQueue依赖runloop，使用时需确保对应线程的runloop开启
因为queue的通知是在runloop的某个时刻，通过center发送的
```

5、如何保证通知接收的线程在主线程

```
1、可使用block注册通知，指定mainQueue响应，这个监听需要手动移除通知
2、通过在主线程注册一个machport，用作线程通信
3、GCD切换线程
```

```
MachPort的工作方式其实是将NSMachPort的对象添加到一个线程所
对应的RunLoop中，并给NSMachPort对象设置相应的代理。在其他线
程中调用该MachPort对象发消息时会在MachPort所关联的线程中执行
相关的代理方法。
```

[https://cloud.tencent.com/developer/article/1018076](https://cloud.tencent.com/developer/article/1018076)


6、页面销毁时不移除通知会崩溃吗

```
iOS9.0之前，通知中心对observer进行unsafe_unretained 引用，需要手动移除，否则会出现野指针访问；
iOS9.0之后，通知中心对observer进行weak引用，无需手动移除通知，但是如果使用block的API监听，需要手动移除。
```

7、多次添加同一个通知会是什么结果？多次移除通知呢

```
多次添加同一个通知，当Post发送时，会调用多次回调。因为添加时没去重
多次移除一般没影响，确保通知注册和销毁的调用时机即可，否则可能出现注册后就被移除。
```

8、下面的方式能接收到通知吗？为什么
```
// 发送通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];
// 接收通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
```

```
接收不到，因为监听时设置了name和object，则存储方式是通过name和object组合存储的，
所以发送的时候不能单单通过name找到对应的Observation，无法调用存储的方法。
```

---

---

# Runloop & KVO

# Runloop

1、app如何接收到触摸事件的

```
分两个阶段

系统响应阶段：
1、触摸屏幕后，事件会交由IOKit处理
2、IOKit将事件封装成IOHIDEvent事件，通过mach port传递给SpringBoard进程
3、SpringBoard进程接收到事件后，触发当前进程主线程runloop的source事件
（如果app在后台，则当前进程（SpringBoard）自行处理，在前台则通过mach port传递给app）

App响应阶段：
1、App进程的mach port接收到SpringBoard的触摸事件后，触发runloop的source1回调
2、source1触发source0回调，将接收到的IOHIDEvent事件封装成UIEvent事件
3、source0回调中将UIEvent事件添加到UIApplication对象的事件队列中
4、事件传递，寻找事件响应者。
```

2、为什么只有主线程的runloop是开启的

```
RunLoop和线程的一一对应的，对应的方式是以key-value的方式保存在一个全局字典中
而主线程的RunLoop会在初始化全局字典时创建，保证应用程序能正常接收处理事件。子线程的runloop是通过懒加载的方式创建，默认关闭，节省资源。
```

3、为什么只在主线程刷新UI

```
1、UIKit 中的类并不是线程安全的类，大部分都是使用nonatomic，UI操作涉及到渲染访问各种View对象的属性，如果异步操作下会存在读写问题，而为其加锁则会耗费大量资源并拖慢运行速度

2、Application是在主线程中对事件进行传递，那么UI处理响应事件则必须要在主线程中处理
```

4、PerformSelector和runloop的关系

```
PerformSelector 是唤醒runloop的事件源，可以是timer或者source0；
runloop唤醒的事件源有两种：
1、input source，包括source0和source1，
其中source0是App内部事件，包括App自己管理的UIEvent、CFSocket。source1由RunLoop和内核管理，source1带有mach_port_t，可以接收内核消息并触发回调
2、定时器
```

5、如何使线程保活

```
默认情况一个线程创建出来，运行完要做的事情，线程就会消亡
可在对应线程中创建runloop，并添加事件源，可以是port、timer
```

---

# KVO

1、实现原理

```
通过运行时新建对应类的子类，把对象的isa指针指向新建的子类中，在子类中添加对应监听属性的setter方法，然后插入监听回调方法
```

2、如何手动关闭kvo

```
1、可调用系统提供的方法：removeObserve移除监听
2、关闭KVO本质就是把isa指向回原来的类，可通过object_setClass函数设置
```

3、通过KVC修改属性会触发KVO么

```
会。通过KVC修改属性或者成员变量都会触发KVO，
KVC内部调用了 willChangeValueForKey 和 didChangeValueForKey
KVC寻找过程为：找setKey: - _setKey - 找成员变量key,isKey,_isKey,_key
```

4、哪些情况下使用kvo会崩溃，怎么防护崩溃

```
设置监听，当类销毁时没有及时移除监听

可利用关联对象进行自动移除，或者使用Facebook开源的 KVOViewcontroller，
```

5、kvo的优缺点

```
优点：
1、KVO提供了一种简单的方法实现两个对象间的同步。例如：model和view之间同步；
2、能够对非我们创建的对象，即内部对象的状态改变作出响应，而且不需要改变内部对象（SKD对象）的实现；
3、能够提供观察的属性的最新值以及先前值；
4、用key paths来观察属性，因此也可以观察嵌套对象；
5、完成了对观察对象的抽象，因为不需要额外的代码来允许观察值能够被观察

缺点：
1、我们观察的属性必须使用strings来定义。因此在编译器不会出现警告以及检查；
2、对属性重构将导致我们的观察代码不再可用；
3、观察多个属性时，需要些复杂的if判断条件语句；
4、当释放观察者时不需要移除观察者。
```

6、如何手动添加KVO监听属性

```
手动调用willChangeValueForKey:和didChangeValueForKey:
```

7、KVO如何监听数组元素变化

```
1、手动添加willChangeValueForKey和didChangeValueForKey方法

元素变化的时候手动插入两个方法

2、通过- (NSMutableArray *)mutableArrayValueForKey:(NSString *)key;

key是当前数组名，监听返回的数组，必须通过返回的数组做增删操作才可以
```

---
---

# Block

1、block的内部实现，结构体是什么样的

```
底层结构体是_main_block_imp_0结构体，成员变量有：__block_impl 结构体、_main_block_desc_0结构体，还有捕抓的变量。
__block_impl有成员变量：void *isa（指向类对象）、void *FuncPtr（指针，指向block代码块的实现函数地址）、int flags，用来确定是否增加对象的引用计数

_main_block_desc_0有成员变量size_t Block_size，表示block的内存占用


内部实现逻辑：

block初始化时，会构造_main_block_imp_0结构体，同时底层实现静态函数_main_block_func_0
来包装block块中的代码，其中首参数就是_main_block_imp_0指针，用来获取捕抓的变量；
构造_main_block_imp_0时传入参数_main_block_func_0地址、_main_block_desc_0地址，以及捕抓到的变量，
内部把_main_block_func_0赋值给__block_impl的FuncPtr，isa指向对应的类对象

当block调用时，通过强转_main_block_imp_0指针地址为__block_impl指针（因为首地址一样），通过
__block_impl获取FuncPrt，调用_main_block_func_0函数

```

2、block是类吗，有哪些类型

```
是类，结构体内部有isa指针，指向对应的类对象
类型有三种
__NSGlobalBlock__ 全局block，没有访问auto变量，存储在数据区
__NSStackBlock__ 栈block，访问了auto变量，存储在栈区
__NSMallocBlock__ 堆block，栈block调用copy后，存储在堆区

auto自动变量，离开作用域就销毁，通常局部变量前面自动添加auto关键字
全局变量不是auto变量，block不会捕抓

ARC下，默认四种情况是会被调用copy
1、block作为函数返回值
2、赋值给__strong指针
3、使用Cocoa API中带有usingBlock
4、GCD
```

3、一个int变量被 __block 修饰与否的区别？

```
单纯int 变量：block内部捕抓的是值，访问方式为值传递，因此block中修改int值，外部的变量是不会变更

__block 修饰的int变量：内部会把int变量包装为__Block_byref_变量名_0的结构体指针，从而实现引用传递。

而这个结构体内部的成员变量__forwarding 结构体指针、int变量，block copy后栈中block的forwarding指针指向堆中的block，
堆中的__forwarding指向自己，__forwarding 保证在栈上或者堆上都能正确访问对应变量
```

4、block的变量截获

```
对于auto 变量，通过值传递保存到block（_main_block_imp_0结构体）中。

对于static静态变量，通过引用传递保存到block（_main_block_imp_0结构体）中。

对于全局变量，block是不会捕抓到，可在block中直接访问。

对于__block 修饰的变量，内部会转换成__block_byref_变量名_0的结构体指针，内部__forwarding指针存储变量的地址。
保存到block（_main_block_imp_0结构体）中。

对于OC对象的变量，内部通过__strong id 变量名形式来捕抓对象，只是指针赋值，不影响对象的引用计数。
同时多了两个方法用作管理起内存，_main_block_copy_0 负责把对象拷贝到堆中的__main_block_imp_0结构体中；_main_block_dispose_0 负责把堆中的对象移除。

当block对象销毁的时候，block中捕抓到的对象也会被销毁。
```

5、block在修改NSMutableArray，需不需要添加__block

```
如果修改可变数组的指针，则需要添加__block
如果修改数组指针指向的内存，则不需要添加__block

因为block是拷贝了一份数组指针，两个指针同时都是指向同一个地址，因此可以访问同一个内存
```

6、怎么进行内存管理的

```
对block本身的内存管理：通过block_copy 和 block_release管理，对应retain和release

1、对于NSGlobalBlock 无引用外部变量，上诉四个操作无效。

2、对于NSStackBlock 引用外部变量，通过copy或block_copy会拷贝到堆区，release、block_release 、retain无效，作用域结束后会自动释放。

3、对于NSMallocBlock 栈block copy后就是堆block，release或block_release会引用计数-1，copy、block_copy、retain都会引用计数+1

对block引用变量的内存管理

变量被block捕抓并持有后，block copy后会把block以及其捕抓的变量都从栈拷贝到堆中，
当block对象销毁的时候，block持有的对象引用-1，当引用计数为0的时候，就销毁。
```

![block的内存管理操作.png](https://upload-images.jianshu.io/upload_images/1085031-ab5ceeccd9acd3bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

7、block可以用strong修饰吗

```
2014年9月后的一次编译器优化之后，如果用strong修饰block，编译器会自动将blcok从stack拷贝到heap上
```

8、解决循环引用时为什么要用__strong、__weak修饰

```
block的循环引用发生原因：

block内部会强引用访问的对象的，而block也被访问的对象持有时，则循环引用。
ARC下通过__weak 修饰访问的对象，block中捕抓为__strong NSObject * __weak obj;为弱引用，因此可避免循环引用。
通过__strong修饰，当对象被多次访问时，内部是为了防止访问block中捕抓的对象被释放，通过__strong强引用，当block消耗的时候，引用计数也会-1
```

9、block发生copy时机

```
主动调用copy相关方法

ARC下，默认四种情况是会被调用copy
1、block作为函数返回值
2、Block被强引用，Block被赋值给__strong或者id类型
3、使用Cocoa API中带有usingBlock
4、GCD
```

10、Block访问对象类型的auto变量时，在ARC和MRC下有什么区别

```
首先block访问auto变量，那么此时为NSStackBlock，当作用域结束后会自动销毁

ARC下，block会强引用访问的对象

MRC下，block不会强引用访问的对象，即使使用__block修饰，也不会强引用，只是会复制指针

MRC 环境下，block 截获外部用 __block 修饰的变量，不会增加对象的引用计数，调用_Block_retain_object 引用计数+1
ARC 环境下，block 截获外部用 __block 修饰的变量，会增加对象的引用计数，没有调用_Block_retain_object

在 MRC 环境下，可以通过 __block 来打破循环引用，在 ARC 环境下，则需要用 _weak 来打破循环引用

```

```
1、mrc 下栈上的 block，在 arc 中并且符合一定条件的情况下会自动被拷贝到堆上
2、block 内部如果使用了 __block 修饰的局部变量，在 mrc 下 block 
不会对这个变量产生强引用，在 arc 下 block 会对这个变量产生强引
用。block 从栈上拷贝到堆上的时候，如果是在 mrc 下，那么 block 结
构体中的 bref 结构部不会从栈上拷贝到堆上，所以 bref 不会持有变
量；如果是在 arc 下，那么 bref 也会跟着一起拷贝到堆上，所以 bref 
会强引用变量。
3、arc 下，解决循环引用可以使用 __weak，
__unsafe__unretained，或者使用__block 并且在块内将变量置 nil，
当代码块内的代码执行完的时候，block 就不会再持有局部变量了。如
果是在 mrc 下，可以使用 __unsafe__unretained、__block 解决循环
引用的问题。
```

[__Block 在 ARC 和 MRC 环境的区别](https://ioscaff.com/articles/276)
[https://juejin.im/post/5b5c51d0e51d45199358c80d](https://juejin.im/post/5b5c51d0e51d45199358c80d)


---
---

# 多线程

1、iOS开发中有多少类型的线程？分别对比

```
主线程：
程序开始默认创建的线程，对应runloop是默认启动，处理UI显示以及事件处理，跟随程序的生命周期

子线程：
开发者手动创建的线程，用来处理耗时操作，对应runloop是默认关闭，一般线程处理完后会自动销毁

子线程runloop run后，当runloop销毁后线程才会销毁。
```

2、GCD有哪些队列，默认提供哪些队列

```
1、全局并发队列（默认提供）：dispatch_get_global_queue(long identifier, unsigned long flags)；系统调度队列，支持任务并发执行
其中identifier是队列的优先级，全局并发队列提供4中优先级，但不能保证实时性。

2、主队列（默认提供）:dispatch_get_main_queue()；主队列中的任务都会在主线程上执行

3、串行队列:dispatch_queue_create("xxx", DISPATCH_QUEUE_SERIAL)；开发创建的队列，在对应线程中串行执行任务

4、并行队列:dispatch_queue_create("xxx", DISPATCH_QUEUE_CONCURRENT)；开发创建的并发队列

```

3、GCD有哪些方法api

```
dispatch_get_global_queue：根据优先级获取全局队列

dispatch_get_main_queue：获取主队列

dispatch_queue_create：生成队列

dispatch_async(队列，block)：开启异步任务

dispathc_sync(队列，block)：开启同步任务

dispatch_set_target_queue：设置队列优先级或者实现串行执行任务(https://www.jianshu.com/p/8f07a9bdd1cd)

dispatch_once：确保程序中只执行一次

dispatch_after：延时执行

dispatch_group：设置任务组，可等待组内任务完成后再处理其他

dispatch_queue_set_specific：标记队列，FMDB用来判断是否同一个队列，避免死锁
```

4、GCD主线程 & 主队列的关系

```
主线程中有且只有一条主队列，在主线程创建时创建；主队列的任务都会在主线程中执行
```

5、如何实现同步，有多少方式就说多少

```
1、使用串行队列，任务FIFO
2、使用嵌套
3、使用锁
4、利用dispatch_set_target_queue，目标队列为串行队列
5、dispatch_group
6、dispatch_barrier_async 栅栏，等待队列中任务完成后才完成，读写锁也可以利用这个实现
```

6、dispatch_once实现原理
[源码解答](http://lingyuncxb.com/2018/02/01/GCD%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902%20%E2%80%94%E2%80%94%20dispatch-once%E7%AF%87/)
[最新源码解析](https://juejin.im/post/5c8fbad5e51d4531050a67dd#heading-1)


```
内部实现：

1、把dispatch_once_t的指针地址强转为dispatch_once_gate_t结构体，内部有一个dgo_once指针成员变量
2、通过dgo_once判断是否有标记DLOCK_ONCE_DONE，表示block已经执行过，是则return，否则下一步
3、通过原子操作（0则标记为n，返回true；否则返回false）确保block执行一次，block执行完成后标记DLOCK_ONCE_DONE
4、原子操作判断返回false，则意味着其他线程也进来了，此时dispatch_once_wait加锁等待block执行完成。
```

![流程图](https://upload-images.jianshu.io/upload_images/1085031-93cd239b77841ce7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


7、什么情况下会死锁

```
产生死锁的四个必要条件：
（1） 互斥条件：一个资源每次只能被一个进程使用。
（2） 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
（3） 不剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺。
（4） 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。
这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之
一不满足，就不会发生死锁。
```

```
死锁一般出现在线程处理中。
GCD 的sync同步执行会阻塞当前线程，等待sync执行的队列任务完成后，才会继续执行当前线程的任务。

当前线程的队列与sync的队列一致，则会形成死锁。
当前串行队列里面同步执行当前串行队列就会死锁，解决的方法就是将同步的串行队列放到另外一个线程就能够解决

因为队列一样，任务按FIFO执行，而因为同步阻塞的原因，当前线程任务需要等sync的任务完成，同个队列任务，sync的任务也要等当前线程的任务完成

并发队列的任务支持多任务同时执行，因此即使同步执行，也不会阻塞
异步执行不会造成死锁，因为异步不影响当前线程任务执行
```

8、有哪些类型的线程锁，分别介绍下作用和使用场景

```
1、递归锁：嵌套使用
2、自旋锁：轮询等待，任务比较频繁调用的情况下使用，会占用cpu较多
3、互斥锁：线程会休息唤醒，在任务不是很频繁调用的情况下使用，降低cpu
4、读写锁：
```

9、NSOperationQueue中的maxConcurrentOperationCount默认值

```
默认值是-1，根据系统情况来调度
```

10、NSTimer、CADisplayLink、dispatch_source_t 的优劣

```
1、当定时器添加到 runloop 中的时候，runloop 会持有定时器，定时器会持有 target，就会导致 target 不释放。
    block，在内部使用 weakSelf
    NSProxy，使用消息转发

2、NSTimer 与 CADisplayLink 依赖与 RunLoop，如果 RunLoop 的任务过于繁重，可能导致 NSTimer 不准时

3、使用 GCD 的定时器，避免强引用以及精度高、不依赖runloop

```
[https://juejin.im/post/5b5c51d0e51d45199358c80d](https://juejin.im/post/5b5c51d0e51d45199358c80d)


---
---


# 视图&图像相关

1、AutoLayout的原理，性能如何
[https://www.dazhuanlan.com/2019/10/05/5d97e8f15427d/](https://www.dazhuanlan.com/2019/10/05/5d97e8f15427d/)

```
Layout Engine 会从上到下调用 layoutSubviews() ，通过 Cassowary 算法计算各
个子视图的位置，算出来后将子视图的 frame 从 Layout Engine 里拷贝出来。在
这之后的处理，就和手写布局的绘制、渲染过程一样了。所以，使用 Auto 
Layout 和手写布局的区别，就是多了布局上的这个计算过程
```

2、UIView & CALayer的区别
[http://www.cocoachina.com/articles/13244](http://www.cocoachina.com/articles/13244)


```
UIView能接收并处理事件，CALayer不行
UIView的内容是通过CALayer来填充，包括frame的获取
```

3、事件响应链

```

```

4、drawrect & layoutsubviews调用时机

```
layoutSubViews调用时机

1、init初始化不会调用layoutSubviews方法
2、addSubview时会调用
3、改变一个UIView的frame时会调用
4、滚动一个UIScrollView导致UIView重新布局时会调用
5、旋转Screen会触发父UIView上的事件
6、手动调用setNeedsLayout或者layoutIfNeeded

drawRect调用时机

1、如果在UIView初始化时没有设置frame，会导致drawRect不被自动调用
2、sizeToFit后会调用。这时候可以先用sizeToFit中计算出size，然后系统自动调用drawRect方法
3、通过设置contentMode为.redraw时，那么在每次设置或更改frame的时候自动调用drawRect
4、直接调用setNeedsDisplay，或者setNeedsDisplayInRect会触发drawRect
```

5、UI的刷新原理

```
每一个view的变化的修改并不是立刻变化，相反的会在当前run loop的结束或者休眠的时候统一进行重绘，这样设计的目的是为了能够在一个runloop里面处理好所有需要变化的view，包括resize、hide、reposition等等，所有view的改变都能在同一时间生效，这样能够更高效的处理绘制，这个机制被称为绘图循环（View Drawing Cycle)
```

6、隐式动画 & 显示动画区别

```
显式动画是指用户自己通过beginAnimations:context:和commitAnimations创建
的动画。
隐式动画是指通过UIView的animateWithDuration:animations:方法创建的动
画。

隐式动画是系统框架自动完成的。

Core Animation在每个runloop周期中自动开始一次新的事务，即使你不显式的
用[CATransaction begin]开始一次事务，任何在一次runloop循环中属性的改变
都会被集中起来，然后做一次0.25秒的动画。在iOS4中，苹果对UIView添加了一
种基于block的动画方法：+animateWithDuration:animations:。这样写对做一堆
的属性动画在语法上会更加简单，但实质上它们都是在做同样的事情。
CATransaction的+begin和+commit方法在+animateWithDuration:animations:
内部自动调用，这样block中所有属性的改变都会被事务所包含。
```

7、什么是离屏渲染

```
指的是在GPU在当前屏幕缓冲区以外开辟一个缓冲区进行渲染操作.

性能消耗主要在上下文的切换
```

8、imageName &  imageWithContentsOfFile区别

```
imageName 
1、通过图片名来获取image
2、系统内部 有缓存，多张相同图片加载性能好，一般用于小图加载

imageWithContentsOfFile
1、通过路径来获取image
2、系统内部 不缓存，一般用于大图加载，使用地方较少


```

9、多个相同的图片，会重复加载吗

```
使用imageName来读取的话，则不会重复加载；其他加载方法内部没有缓存机制，会重复加载
```

10、图片是什么时候解码的，如何优化
[http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/](http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/)

```
1、假设我们使用 +imageWithContentsOfFile: 方法从磁盘中加载一张图片，这个时候的图片并没有解压缩；
2、然后将生成的 UIImage 赋值给 UIImageView ；
3、接着一个隐式的 CATransaction 捕获到了 UIImageView 图层树的变化；
4、在主线程的下一个 run loop 到来时，Core Animation 提交了这个隐式的 
transaction ，这个过程可能会对图片进行 copy 操作，而受图片是否字节对齐等
因素的影响，这个 copy 操作可能会涉及以下部分或全部步骤：

①分配内存缓冲区用于管理文件 IO 和解压缩操作；
②将文件数据从磁盘读到内存中；
③将压缩的图片数据解码成未压缩的位图形式，这是一个非常耗时的 CPU 操作；
④最后 Core Animation 使用未压缩的位图数据渲染 UIImageView 的图层。

优化：

1、缓存解码后的图片数据，比如SDWebImage，内部可设置是否缓存

缺点：解码后的图片数据很大，会容易引起内存问题。

2、提前解压缩

可通过CGBitmapContextCreate重新绘制生成新的解压缩后的位图。
YYKit和SDWebImage都有实现

```
11、图片渲染怎么优化
[https://swift.gg/2019/11/01/image-resizing/](https://swift.gg/2019/11/01/image-resizing/)

```
当image明显大于 UIImageView 显示尺寸的时候，优化才有意义

1.  [绘制到 UIGraphicsImageRenderer 上](https://swift.gg/2019/11/01/image-resizing/#technique-1-drawing-to-a-uigraphicsimagerenderer)
2.  [绘制到 Core Graphics Context 上](https://swift.gg/2019/11/01/image-resizing/#technique-2-drawing-to-a-core-graphics-context)
3.  [使用 Image I/O 创建缩略图像](https://swift.gg/2019/11/01/image-resizing/#technique-3-creating-a-thumbnail-with-image-io)
4.  [使用 Core Image 进行 Lanczos 重采样](https://swift.gg/2019/11/01/image-resizing/#technique-4-lanczos-resampling-with-core-image)
5.  [使用 vImage 优化图片渲染](https://swift.gg/2019/11/01/image-resizing/#technique-5-image-scaling-with-vimage)

```

12、如果GPU的刷新率超过了iOS屏幕60Hz刷新率是什么现象，怎么解决
[https://lzhenhong.github.io/2018/01/06/V-Sync/](https://lzhenhong.github.io/2018/01/06/V-Sync/)

```
屏幕撕裂
原因是：上一帧图像还没显示完全，下一帧就绘制渲染完成，加载显示

解决方案：
1、多个缓存区绘制处理，显示器依次读取（只能缓解，不能根治）

2、垂直同步 V-Sync

工作原理大概是：显示器在处理完当前图像帧，会向 GPU 发送 VSync 信号，当 
GPU 接收到 VSync 信号之后，就会进行帧缓冲区的更新和新图像帧的绘制。这
样就能保证 GPU 绘制的图像帧不会出现被覆盖的情况，也就解决了屏幕撕裂。
```

--- 

# 性能优化

1、如何做启动优化，如何监控

2、如何做卡顿优化，如何监控

3、如何做耗电优化，如何监控

4、如何做网络优化，如何监控

---

# 架构设计

1、手动埋点、自动化埋点、可视化埋点
2、MVC、MVP、MVVM设计模式
3、常见的设计模式
4、单例的弊端
5、常见的路由方案，以及优缺点对比
6、如果保证项目的稳定性
7、设计一个图片缓存框架(LRU)
8、如何设计一个git diff
9、设计一个线程池？画出你的架构图
10、你的app架构是什么，有什么优缺点、为什么这么做、怎么改进
