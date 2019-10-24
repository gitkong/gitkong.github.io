---
layout:     post
title:      ARC下，Dealloc还需要注意什么？
subtitle:    帮你避免掉坑
date:       2019-10-24
author:     gitKong
header-img: img/post-bg-miui6.jpg
catalog: true
tags:
    - iOS
---

# 前言：

本次分享会先介绍Dealloc，对常见的用法分析，然后初步深入了解Dealloc的机制。

## 一、Dealloc 是什么
> Deallocates the memory occupied by the receiver.

摘自[官方文档](https://developer.apple.com/documentation/objectivec/nsobject/1571947-dealloc)，文档描述，dealloc 其实就是NSObject的一个**方法**，当对象被销毁的时候，系统就会回调这个方法，用来释放内存占用。
![摘自官方文档](https://img-blog.csdnimg.cn/20191022162145964.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzODc2OTk=,size_16,color_FFFFFF,t_70)
当然当iOS开发的应该都知道，这个方法默认是没有写的，因为系统会自动处理了，至于处理了什么，以及怎么处理，这个后文会继续分析。

## 二、Dealloc 怎么写
> 推荐的代码组织方式是将dealloc方法放在实现文件的最前面（直接在@synthesize以及@dynamic之后）

摘自《**禅与Objective-C编程艺术**》一书，书中提到了dealloc的方法实现建议放的位置。

顺便提一下，`@synthesize` 的语义是如果你没有手动实现 setter 方法和 getter 方法，那么编译器会自动为你加上这两个方法，当然定义一个属性的时候，系统会默认编写了，而 `@dynamic` 的用法则相反，属性的 setter 与 getter 方法由用户自己实现，不自动生成。

那dealloc方法里面，我们要处理什么？

**在dealloc方法中通常需要做的有移除通知或监听操作，或对于一些非Objective-C对象也需要手动清空，比如CoreFoundation中的对象。**

MRC下就要手动释放、置空变量等操作后还需要调用父类的dealloc，而ARC下除了CoreFoundation的对象需要手动释放以及KVO监听移除外（NSNotification 在iOS9之后也不需要手动移除了），基本就没了，当然ARC的内存销毁具有一定的滞后性，也可将一些变量手动置空，也就是告诉系统这些变量已经使用完毕可以释放了，当然也可以不做任何操作，**系统会自动释放这些成员变量或者属性**。

## 三、Dealloc 容易犯的错误
前面简单介绍了dealloc的定义以及用法，那我们平时开发的时候，会容易出现什么样的错误呢？下面会列举四种错误用法分析：

##### 1、dealloc在什么线程中被调用
很多人都认为是在主线程调用的，其实并不是，而是取决于最后在什么线程中release后触发，这个稍后结合第四个错误点来分析会比较清楚。

从[Runtime 源码](https://opensource.apple.com/tarballs/objc4/) objc-object.h 源码中可以得到上述结论。

```
ALWAYS_INLINE bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) 

...

if (performDealloc) {
((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
}
```

##### 2、dealloc中置空操作
常见的操作：

```
- (void)dealloc {
NSLog(@"BaseModel dealloc");
self.baseName = nil;
}
```

这个表面看上去没什么问题，在《**Effective Objective-C 2.0**》一书中第7条提到：
> 在对象内部尽量直接访问实例变量，而在初始化方法和dealloc方法中，总是应该直接通过实例变量来读写数据。

除了文中所说的加快访问速度之外，但是如果用法不巧当的话，会出现不必要的崩溃问题。下面举个简单的例子分析一下：

**定义了一个BaseModel 基类，基类中演示了使用`self.baseName = nil`：**

```
@interface BaseModel : NSObject

@property (nonatomic, copy) NSString * _Nullable baseName;

@end

@implementation BaseModel

- (void)dealloc {
NSLog(@"BaseModel dealloc");
self.baseName = nil;
}

- (void)setBaseName:(NSString *)baseName {
_baseName = baseName;
NSLog(@"BaseModel setBaseName:%@", baseName);
}

@end
```

**同时定义了一个子类SubModel继承自BaseModel，子类中重写了baseName 的setter方法，并获取baseName进行其他操作**

```
@implementation SubModel// 继承自BaseModel

- (void)dealloc {
NSLog(@"SubModel dealloc");
}

- (void)setBaseName:(NSString *)baseName {
[super setBaseName:baseName];
NSLog(@"SubModel setBaseName:%@", [NSString stringWithString:baseName]);
}

@end
```

当SubModel作为一个临时变量生成后赋值baseName，变量使用完后系统会自动回收，此时大家可以想想会发什么什么问题？

不难想到，此时会出现崩溃现象，原因是`[NSString stringWithString:baseName]` 这里，baseName是nil，而这个方法是不允许传nil参数的，当然，这个业务处理上肯定需要一个判空操作，我们先来分析一下为什么会是nil

**子类SubModel被释放会调用子类的dealloc方法，然后会调用父类BaseModel的dealloc方法，此时父类中通过setter方法来赋值nil，而子类SubModel重写了，子类拿到nil来处理导致崩溃问题**

究竟属性是否需要手动置空释放？实际上来说，是不需要手动释放的，因为dealloc中`.cxx_destruct`会处理。当然因为执行是有一定延迟性，为了节省资源，在确保属性没利用价值的时候可以手动清空，这个后文会分析dealloc的处理逻辑。

##### 3、dealloc中使用__weak
举个简单例子来模拟实际复杂业务场景：

```
- (void)dealloc {
NSLog(@"SubModel dealloc");
[self performSelectorWhenDealloc];
}

- (void)performSelectorWhenDealloc {
__weak typeof(self) weakSelf = self;
// 模拟复杂的block结构，需要弱引用解除循环引用
void (^block)(void) = ^ {
[weakSelf test];
};
block();
}

```

当SubModel这个类被释放，调用dealloc的时候会出现崩溃，崩溃信息如下：`Cannot form weak reference to instance (0x2813c4d90) of class xxx. It is possible that this object was over-released, or is in the process of deallocation.`

![堆栈截图](https://img-blog.csdnimg.cn/20191022181748456.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzODc2OTk=,size_16,color_FFFFFF,t_70)

崩溃原因我们来分析一下：

先了解一下`__weak` 到底做了什么操作，通过clang 转换的代码是这样的` __attribute__((objc_ownership(weak))) typeof(self) weakSelf = self; `这样还是看不出问题，我们看回堆栈，堆栈崩在`objc_initWeak` 函数中，我们可以看看[Runtime 源码](https://opensource.apple.com/tarballs/objc4/)  `objc_initWeak` 函数的定义是怎么样的：

```
id
objc_initWeak(id *location, id newObj)
{
if (!newObj) {
*location = nil;
return nil;
}

return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
(location, (objc_object*)newObj);
}
```

可以留意到，内部调用了 `storeWeak` 函数，其中有个模板名称是`DontCrashIfDeallocating` 不难猜到，当调用到了 `storeWeak` 函数的时候，如果释放过程中存储，那就会crash，函数最终会调用register函数 `id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
id *referrer_id, bool crashIfDeallocating)`

```
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
id *referrer_id, bool crashIfDeallocating)
{
...
if (deallocating) {
if (crashIfDeallocating) {
_objc_fatal("Cannot form weak reference to instance (%p) of "
"class %s. It is possible that this object was "
"over-released, or is in the process of deallocation.",
(void*)referent, object_getClassName((id)referent));
} else {
return nil;
}
}
...
}
```

在这就找到了崩溃时打印出信息了。通过上面的分析，我们也知道，`__weak` 其实就是会最终调用`objc_initWeak` 函数进行注册。抱着求学的态度，可以在[clang 8.7](https://opensource.apple.com/source/clang/clang-211.10.1/src/tools/clang/docs/AutomaticReferenceCounting.html#runtime.objc_initWeak) 对`objc_initWeak`函数描述中找到答案：

> object is a valid pointer which has not been registered as a __weak object. value is null or a pointer to a valid object.
> If value is a null pointer or the object to which it points has begun deallocation, object is zero-initialized. Otherwise, object is registered as a __weak object pointing to value. Equivalent to the following code:

##### 4、dealloc中使用GCD
GCD相信大家平时用得不少，但在Dealloc方法里面使用GCD大家有没有注意呢，先来举个简单例子，我们在主线程中创建一个定时器，然后类被释放的时候销毁定时器，相关代码如下。

```
- (void)dealloc {
[self invalidateTimer];
}

- (void)fireTimer {
__weak typeof(self) weakSelf = self;
dispatch_async(dispatch_get_main_queue(), ^{
if (!weakSelf.timer) {
weakSelf.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
NSLog(@"TestDeallocModel timer:%p", timer);
}];
[[NSRunLoop currentRunLoop] addTimer:weakSelf.timer forMode:NSRunLoopCommonModes];
}
});
}

- (void)invalidateTimer {
dispatch_async(dispatch_get_main_queue(), ^{
if (self.timer) {
NSLog(@"TestDeallocModel invalidateTimer:%p model:%p", self->_timer, self);
[self.timer invalidate];
self.timer = nil;
}
});
}

```

补充说明一下，定时器的释放和创建必须在同一个线程，这个也是比较容易犯的错误点，官方描述如下：
> Stops the timer from ever firing again and requests its removal from its run loop.


> This method is the only way to remove a timer from an NSRunLoop object. The NSRunLoop object removes its strong reference to the timer, either just before the invalidate method returns or at some later point.
> If it was configured with target and user info objects, the receiver removes its strong references to those objects as well.

简单解释一下，当前的定时器销毁只能从启动定时器的Runloop中移除，然后Runloop和线程是一一对应的，因此需要确保销毁和创建在同一个线程中处理，否则可能会出现释放不了的情况。

说回正题，当定时器所在类被释放后，此时调用`invalidateTimer` 方法去销毁定时器的时候就会出现崩溃情况。

崩溃报错：`Thread 1: EXC_BAD_ACCESS (code=EXC_I386_GPFLT)` 

出现访问野指针问题了，原因其实不难想到程序代码默认是在主线程主队列中执行，而dealloc中异步执行主队列中释放定时器释放，GCD会强引用`self`，此时dealloc已经执行完成了，那么`self` 其实已经被free释放掉了，此时销毁内部再调用`self`就会访问野指针。

我们来继续分析一下，GCD为啥会强引用`self`，以及简单分析一下GCD的调用时机问题。

强引用问题，我们可以通过Clang去查看一下底层源码实现，简单转换如下代码：

```
- (void)dealloc {
dispatch_async(dispatch_queue_create("Kong", 0), ^{
[self test];
});
}
```

转换后如下：

```

struct __TestModel__dealloc_block_impl_0 {
struct __block_impl impl;
struct __TestModel__dealloc_block_desc_0* Desc;
TestModel *const __strong self;
__TestModel__dealloc_block_impl_0(void *fp, struct __TestModel__dealloc_block_desc_0 *desc, TestModel *const __strong _self, int flags=0) : self(_self) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};

static void __TestModel__dealloc_block_func_0(struct __TestModel__dealloc_block_impl_0 *__cself) {
TestModel *const __strong self = __cself->self; // bound by copy

((void (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("test"));
}


static void __TestModel__dealloc_block_copy_0(struct __TestModel__dealloc_block_impl_0*dst, struct __TestModel__dealloc_block_impl_0*src) {_Block_object_assign((void*)&dst->self, (void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}



static void __TestModel__dealloc_block_dispose_0(struct __TestModel__dealloc_block_impl_0*src) {_Block_object_dispose((void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}



static struct __TestModel__dealloc_block_desc_0 {
size_t reserved;
size_t Block_size;
void (*copy)(struct __TestModel__dealloc_block_impl_0*, struct __TestModel__dealloc_block_impl_0*);
void (*dispose)(struct __TestModel__dealloc_block_impl_0*);
} __TestModel__dealloc_block_desc_0_DATA = { 0, sizeof(struct __TestModel__dealloc_block_impl_0), __TestModel__dealloc_block_copy_0, __TestModel__dealloc_block_dispose_0};

static void _I_TestModel_dealloc(TestModel * self, SEL _cmd) {
dispatch_async(dispatch_queue_create("Kong", 0), ((void (*)())&__TestModel__dealloc_block_impl_0((void *)__TestModel__dealloc_block_func_0, &__TestModel__dealloc_block_desc_0_DATA, self, 570425344)));
}

```

转换后的代码量有点多，我们可以只抓重点来查看一下，具体实现细节本文就不过多分析。

可以发现，`dealloc` 方法 转换成 `static void _I_TestModel_dealloc(TestModel * self, SEL _cmd)` 内部调用的`dispatch` 方法变化不大，主要是看Block的传递，可以留意到`__TestModel__dealloc_block_impl_0` 这个结构体地址参数，看其代码实现可以发现，`TestModel *const __strong self;` 就是这个`__strong` 使得Block 会对`self` 进行强引用。顺带说一下，结构体中有个`struct __block_impl impl;` 成员变量，而这个结构体内部有个`FuncPtr` 成员，Block的调用实际上就是通过`FuncPtr` 来实现的。

以上就通过一个简单的例子，解释了dealloc 中使用GCD中出现的问题，实际上，GCD还会出现很多种搭配情况，这里简单画了一个图：

![GCD使用](https://img-blog.csdnimg.cn/20191023215027361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzODc2OTk=,size_16,color_FFFFFF,t_70)

原理是一样的，简单总结一下：**GCD任务底层通过链表管理，队列任务遵循FIFO模式，那么任务执行肯定就会有延迟性，同一时刻只能执行一个任务，只要dealloc任务执行先，那么此时block使用self就会访问野指针，因为dealloc内会有free操作。**


## 四、Dealloc 源码分析
上文解答了dealloc 中的几种使用情况，具体源码实现还没分析，下面我们来分析一下源码到底是如何实现的。本文的分析源码版本是[objc4-756.2.tar.gz](https://opensource.apple.com/tarballs/objc4/ ) 

我们通过问题来阅读源码：

###### 1、为什么dealloc 中使用GCD 会容易访问野指针？
出现野指针访问，那肯定就有free操作，我们查一下这个free是在哪一步执行:

*`dealloc` 在NSObject.mm 文件中的实现*
```
// Replaced by NSZombies
- (void)dealloc {
_objc_rootDealloc(self);
}
```
*最终调用在 objc-object.h 的 `inline void
objc_object::rootDealloc()` 函数中*

```
inline void
objc_object::rootDealloc()
{
if (isTaggedPointer()) return;  // fixme necessary?

if (fastpath(isa.nonpointer  &&  // 是否是优化过的isa
!isa.weakly_referenced  &&  // 不包含或者不曾经包含weak指针
!isa.has_assoc  &&  // 没有关联对象
!isa.has_cxx_dtor  &&  // 没有c++析构方法
!isa.has_sidetable_rc))// 引用计数没有超出上限的时候可以快速释放,rootRetain(bool tryRetain, bool handleOverflow) 中设置为true
{
assert(!sidetable_present());
free(this);
} 
else {
object_dispose((id)this);
}
}
```
*可以看到满足一定条件下，对象指针会直接free释放掉，实际很多情况下都会走`object_dispose((id)this)` 函数，这个函数是在objc-runtime-new.mm文件，下面继续分析这个函数实现*

```
id 
object_dispose(id obj)
{
if (!obj) return nil;

objc_destructInstance(obj);
/// 释放内存
free(obj);

return nil;
}
```

**终于看到了大概调用结构了，`objc_destructInstance` 函数后面分析，可以发现`dealloc`内部最终都会走`free` 操作，而这个操作就会导致野指针访问问题**


###### 2、为什么属性或者说成员变量会自动释放？
开篇说了系统会自动释放属性或者成员变量，其实就是`objc_destructInstance` 函数的处理，其定义如下：

```
void *objc_destructInstance(id obj) 
{
if (obj) {
// Read all of the flags at once for performance.
bool cxx = obj->hasCxxDtor();
bool assoc = obj->hasAssociatedObjects();

// This order is important.
// 对象拥有成员变量时编译器会自动插入.cxx_desctruct方法用于自动释放,可打印方法名证明;
if (cxx) object_cxxDestruct(obj);
// 移除关联对象
if (assoc) _object_remove_assocations(obj);
// weak->nil
obj->clearDeallocating();
}

return obj;
}
```

*代码上我已经部分注释了，我们直接看`object_cxxDestruct` 函数实现，后面`_object_remove_assocations` 是移除关联对象，就是我们分类中通过`objc_setAssociatedObject` 函数新增的就是关联对象，而`clearDeallocating` 则是把对应weak哈希表的置空*

```
void object_cxxDestruct(id obj)
{
if (!obj) return;
// 如果是isTaggedPointer，不处理

///为了节省内存和提高执行效率，苹果提出了Tagged Pointer的概念，避免32位机器迁移到64位机器内存翻倍 https://blog.devtang.com/2014/05/30/understand-tagged-pointer
if (obj->isTaggedPointer()) return;
object_cxxDestructFromClass(obj, obj->ISA());
}
```

```
static void object_cxxDestructFromClass(id obj, Class cls)
{
void (*dtor)(id);

// Call cls's dtor first, then superclasses's dtors.
// 按继承链释放
for ( ; cls; cls = cls->superclass) {
if (!cls->hasCxxDtor()) return;
/// .cxx_destruct是编译器生成的代码,在.cxx_destruct进行形如objc_storeStrong(&ivar, null)的调用后，对应的实例变量就被release和设置成nil了
dtor = (void(*)(id))
lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct);
// 进行过动态方法解析后会标记IMP为_objc_msgForward_impcache，进行缓存后会进行消息分发
if (dtor != (void(*)(id))_objc_msgForward_impcache) {
if (PrintCxxCtors) {
_objc_inform("CXX: calling C++ destructors for class %s", 
cls->nameForLogging());
}
(*dtor)(obj);
}
}
}

```

**看最终实现可以发现，内部就是通过继承链遍历调用`lookupMethodInClassAndLoadCache` 函数来进行后续释放，实际上是通过`SEL_cxx_destruct` 来执行C++的析构方法**

我们可以通过`watchpoint` 来监控对象属性的释放，监控堆栈如下：
![watchpoint监听堆栈](https://img-blog.csdnimg.cn/20191023222846223.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzODc2OTk=,size_16,color_FFFFFF,t_70)
可以看出，最终实际上就是调用了`objc_storeStrong` 函数来做释放操作，可以查看源码如下

```
void
objc_storeStrong(id *location, id obj)
{
id prev = *location;
if (obj == prev) {
return;
}
objc_retain(obj);
*location = obj;
objc_release(prev);
}
```

**简单总结一下dealloc的处理逻辑**

![dealloc的处理逻辑](https://img-blog.csdnimg.cn/20191023223411723.png)

## 五、Dealloc 用法总结
**1、dealloc中尽量直接访问实例变量来置空。**

**2、dealloc中切记不能使用__weak self。**

**3、dealloc中切线程操作尽量避免使用GCD，可利用performSelector，确保线程操作先于dealloc完成。**

## 六、Dealloc 机制应用
**从源码中可以知道，dealloc在对象置nil以及free之前，会进行关联对象释放，那么可以利用关联对象销毁监听dealloc完成，做一些自动释放操作，例如通知监听释放等，实际上网上也是有一些例子了。**

简单的源码演示：

```
@interface TestDeallocAssociatedObject : NSObject

- (instancetype)initWithDeallocBlock:(void (^)(void))block;

@end
```

```
@implementation TestDeallocAssociatedObject {
void(^_block)(void);
}

- (instancetype)initWithDeallocBlock:(void (^)(void))block {

if (self = [super init]) {
self->_block = [block copy];
}
return self;
}

- (void)dealloc {
if (self->_block) {
self->_block();
}
}

@end
```

然后在需要监听的地方创建关联对象，Block内处理即可，此时要注意Block引用的问题。

```
// 添加关联对象
TestDeallocAssociatedObject *object = [[TestDeallocAssociatedObject alloc] initWithDeallocBlock:^{
NSLog(@"TestDeallocAssociatedObject dealloc");
}];
objc_setAssociatedObject(self, &KTestDeallocAssociatedObjectKey, object, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

# 七、结语

**通过几个常见案例，逐步分析dealloc的底层代码实现，本文篇幅有点多，如果有描述错误或者不当的地方，欢迎指正~喜欢的可以点个赞哈，谢谢！**
