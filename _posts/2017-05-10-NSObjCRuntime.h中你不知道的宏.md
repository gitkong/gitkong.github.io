---
layout:     post
title:      NSObjCRuntime.h中你不知道的宏
subtitle:    让你的API设计更加完美
date:       2017-05-10
author:     gitKong
header-img: img/post-bg-miui6.jpg
catalog: true
tags:
    - 音频
---

### 前言

**通过阅读别人的优秀源码，你会发现别人的开源API设计中，有一些宏你是经常忽略的，或者你不知道的。通过这些宏，可以让你的设计的API更加完善，当然看上去也会更加高端~举个栗子：**

![这些宏，你知道吗？](http://upload-images.jianshu.io/upload_images/1085031-d4c3cabea30c186a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其实这些宏，大部分都在`NSObjCRuntime.h`中，下面听我细细分析，当然，文章篇幅过长，如果你有精力有恒心一口气看完，给你个赞；更多的，本文可做参考文档，用作查询，已同步到 [个人博客](https://gitkong.github.io/)

---

- ### FOUNDATION_EXTERN

```
#if defined(__cplusplus)
#define FOUNDATION_EXTERN extern "C"
#else
#define FOUNDATION_EXTERN extern
#endif
```

> 表示 `extern` 全局变量，此时并没有分配内存，需要在.m文件中实现，此时为了支持C和C++混编（`__cplusplus` 是C++编译器内部定义的宏，在C++中，需要加
`extern"C"` 或包含在 `extern "C"` 块中），注意，此时外界是可以修改这个值，详细 `extern` 用法可自行查询相关资料，本文不详谈。

**用法如下：**

```
FOUNDATION_EXTERN NSString *name;// h文件
const NSString *name = @"gitKong";// m文件
```

---

- ### FOUNDATION_EXPORT 、FOUNDATION_IMPORT

```
#if TARGET_OS_WIN32

#if defined(NSBUILDINGFOUNDATION)
#define FOUNDATION_EXPORT FOUNDATION_EXTERN __declspec(dllexport)
#else
#define FOUNDATION_EXPORT FOUNDATION_EXTERN __declspec(dllimport)
#endif

#define FOUNDATION_IMPORT FOUNDATION_EXTERN __declspec(dllimport)

#else
#define FOUNDATION_EXPORT  FOUNDATION_EXTERN
#define FOUNDATION_IMPORT FOUNDATION_EXTERN
#endif
```

> 两个宏也是` extern` 只是为了兼容 `win32` 同时适配C++编译器（其中`__declspec(dllexport)` 和 `__declspec(dllimport)` 是C++定义的宏，可在 [Mrcrosoft Developer 文档](https://msdn.microsoft.com/zh-cn/library/3y1sfaz2.aspx) 查询）

**用法如下：**

用法跟 `FOUNDATION_EXTERN ` 一样，不再举例。

---

- ### NS_INLINE

```
#if !defined(NS_INLINE)
#if defined(__GNUC__)
#define NS_INLINE static __inline__ __attribute__((always_inline))
#elif defined(__MWERKS__) || defined(__cplusplus)
#define NS_INLINE static inline
#elif defined(_MSC_VER)
#define NS_INLINE static __inline
#elif TARGET_OS_WIN32
#define NS_INLINE static __inline__
#endif
#endif
```

> 内联函数 inline static修饰，针对当前文件，兼容 win32，适配多个编译器环境（ `__GNUC__ ` 是GCC编译器定义的一个宏；`__MWERKS__` 是Metrowerks C/C++编译器标识宏；`_MSC_VER` 是Microsoft Visual C++编译器标识宏,其中：`__inline__` 、 `inline` 、 `__inline` 、`__attribute__((always_inline))` 只是不同编译器对应的keyword而已；GNU C的一大特色就是 `__attribute__` 机制。`__attribute__` 可以设置 [函数属性](https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Function-Attributes.html#Function-Attributes)、[变量属性](https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Variable-Attributes.html) 和 [类型属性](https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Type-Attributes.html);例如：函数声明中的 `__attribute__((noreturn))` ，就是告诉编译器这个函数不会返回给调用者，以便编译器在优化时去掉不必要的函数返回代码）

用法如下：

```
NS_INLINE int add(){
// do something
}
```

---

- ### FOUNDATION_STATIC_INLINE

```
#if !defined(FOUNDATION_STATIC_INLINE)
#define FOUNDATION_STATIC_INLINE static __inline__
#endif
```

> 表示static内联函数，可以看出，Xcode 的LLVM编译器（前身clang）使用
`__inline__`（Xcode 4之前是GCC）

用法跟 `NS_INLINE ` 一样，不再举例。

---

- ### FOUNDATION_EXTERN_INLINE

```
#if !defined(FOUNDATION_EXTERN_INLINE)
#define FOUNDATION_EXTERN_INLINE extern __inline__
#endif
```

> 表示全局内联函数

用法跟 `NS_INLINE ` 一样，不再举例。

---

- ### NS_REQUIRES_NIL_TERMINATION

```
#if !defined(NS_REQUIRES_NIL_TERMINATION)
#if TARGET_OS_WIN32
#define NS_REQUIRES_NIL_TERMINATION
#else
#if defined(__APPLE_CC__) && (__APPLE_CC__ >= 5549)
#define NS_REQUIRES_NIL_TERMINATION __attribute__((sentinel(0,1)))
#else
#define NS_REQUIRES_NIL_TERMINATION __attribute__((sentinel))
#endif
#endif
#endif
```

> 表示可变函数最后一个参数需要添加 `NULL` 或 `nil` ,适配 `win32` , `__APPLE_CC__ ` 表示 Apple GCC编译器，值为6000，至于定义中为啥判断5549，笔者也找不到说法，`GCC` 在 `Xcode4`
之后已经改用 `LLVM` 了，可参考 [ Clang 中 `__APPLE_CC__ ` 宏解释](http://clang-developers.42468.n3.nabble.com/APPLE-CC-macro-td4046240.html#a4046243)；sentinel 是哨兵的意思，`__attribute__((sentinel))` 就是说，可变参数函数最后需要一个  `NULL` 或 `nil` 作为参数，`sentinel(0,0)` 第一个参数0表示需要一个 `NULL` 或 `nil`，1表示需要两个，依次类增，第二个参数只能是0或1（execle 的时候就是1）。可查看 [GCC 中 Function-Attributes](https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Function-Attributes.html#Function-Attributes) 详细解释

截图如下：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1085031-ab8abed1f8e089f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**用法如下：** 注意，如果使用C函数，`NS_REQUIRES_NIL_TERMINATION` 需要修饰声明，修饰函数定义会报警告

```
// __attribute__((sentinel)) 用于声明，用于实现会警告，此时就提示最后一个参数需要添加NULL
void mutableParams(NSString *first, ...) NS_REQUIRES_NIL_TERMINATION;
void mutableParams(NSString *first, ...){
va_list args;
va_start(args, first);
NSString *next;
while ((next = va_arg(args, NSString *))) {
NSLog(@"%@",next);
}
va_end(args);
}
```


![缺少NULL报警告](http://upload-images.jianshu.io/upload_images/1085031-653663481a6e324f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### NS_BLOCKS_AVAILABLE

```
#if !defined(NS_BLOCKS_AVAILABLE)
#if __BLOCKS__ && (MAC_OS_X_VERSION_10_6 <= MAC_OS_X_VERSION_MAX_ALLOWED || __IPHONE_4_0 <= __IPHONE_OS_VERSION_MAX_ALLOWED)
#define NS_BLOCKS_AVAILABLE 1
#else
#define NS_BLOCKS_AVAILABLE 0
#endif
#endif
```

> 表示当前的 block 是否有效，iOS 4.0之后都适用的（因为block 是 iOS 4.0 之后才有的）

**用法如下：**

```
#if NS_BLOCKS_AVAILABLE
if(self.completionBlock){
self.completionBlock();
}
#else
// do something
#endif
```

---

- ### NS_NONATOMIC_IOSONLY

```
// Marks APIs whose iOS versions are nonatomic, that is cannot be set/get from multiple threads safely without additional synchronization
#if !defined(NS_NONATOMIC_IOSONLY)
#if TARGET_OS_IPHONE
#define NS_NONATOMIC_IOSONLY nonatomic
#else
#if __has_feature(objc_property_explicit_atomic)
#define NS_NONATOMIC_IOSONLY atomic
#else
#define NS_NONATOMIC_IOSONLY
#endif
#endif
#endif
```

> 表示iOS 上是 `nonatomic`，适配了非iOS，其中 `__has_feature` 表示当前新特性是否被Clang编译器支持和符合当前语言标准（[可参考](https://clang.llvm.org/docs/LanguageExtensions.html)，其中 `__has_feature` 的参数可在 [PPMacroExpansion_8cpp_so](http://docs.huihoo.com/doxygen/clang/r222231/PPMacroExpansion_8cpp_source.html) 这查看可知，`objc_property_explicit_atomic` 参数表示Clang编译器是否支持 `atomic` 关键字，因为OSX中不支持 `nonatomic`，因此OSX只有`atomic`）

**用法如下：**

```
@property (NS_NONATOMIC_IOSONLY,copy) NSString *className;
```

---

- ### NS_NONATOMIC_IPHONEONLY

```
// Use NS_NONATOMIC_IOSONLY instead of this older macro
#if !defined(NS_NONATOMIC_IPHONEONLY)
#define NS_NONATOMIC_IPHONEONLY NS_NONATOMIC_IOSONLY
#endif
```

> 历史原因，等同 `NS_NONATOMIC_IOSONLY`，苹果建议使用`NS_NONATOMIC_IOSONLY`

用法和`NS_NONATOMIC_IOSONLY` 一样

---

- ### NS_FORMAT_FUNCTION(F,A)

```
// Marks APIs which format strings by taking a format string and optional varargs as arguments
#if !defined(NS_FORMAT_FUNCTION)
#if (__GNUC__*10+__GNUC_MINOR__ >= 42) && (TARGET_OS_MAC || TARGET_OS_EMBEDDED)
#define NS_FORMAT_FUNCTION(F,A) __attribute__((format(__NSString__, F, A)))
#else
#define NS_FORMAT_FUNCTION(F,A)
#endif
#endif
```

> - 表示是 `Format string` 方案来实现不定参数（不同于哨兵方案`NS_REQUIRES_NIL_TERMINATION `（va_list）） **GCC 3.2.1 will define `__GNUC__` to 3, `__GNUC_MINOR__` to 2**  [可参考](https://gcc.gnu.org/onlinedocs/gcc-3.1.1/cpp/Common-Predefined-Macros.html#Common%20Predefined%20Macros)
-  `__attribute__((format))` 标示一个可变参函数使用了 `Format string` ，从而在编译时对其进行检查,其定义为 `format (archetype, string-index, first-to-check)`，其中`archetype`代表 `format string` 的类型，它可以是 `printf`，`scanf`，`strftime`或者`strfmon`，`Cocoa`开发者还可以使用`__NSString__`来指定其使用和 `[NSString stringWithFormat:]` 与 `NSLog()` 一致的 `Format string` 规则。`string-index` 代表 `Format string`是第几个参数，`first-to-check` 则代表了可变参数列表从第几个参数开始
- F 就是 `Format string` 是第F个参数，A 就是代表可变参数列表是从第A个参数开始

**用法如下：**

```
- (void)append:(NSString *)format,...NS_FORMAT_FUNCTION(1,2){
// do something
}
```

```
[self append:@"%@%@",@"hello",@"gitKong"];
```

**如何使用Format string 方式实现系统的 `stringWithFormat` ，而使用 `Sentinel value` (哨兵)方式**,很简单：

```
- (NSString *)append:(NSString *)format,...NS_FORMAT_FUNCTION(1,2){
va_list ap;
va_start(ap, format);
NSString *information = [[NSString alloc] initWithFormat:format arguments:ap];
va_end(ap);
return information;
}
```

---

- ### NS_FORMAT_ARGUMENT(A)

```
// Marks APIs which are often used to process (take and return) format strings, so they can be used in place of a constant format string parameter in APIs
#if !defined(NS_FORMAT_ARGUMENT)
#if defined(__clang__)
#define NS_FORMAT_ARGUMENT(A) __attribute__ ((format_arg(A)))
#else
#define NS_FORMAT_ARGUMENT(A)
#endif
#endif
```

> 表示函数参数个数，A 表示约束的参数下标，一般默认从1开始，不能超过真实参数个数（[可查](https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Function-Attributes.html#Function-Attributes)），被约束的参数必须是 `String type`，`__clang__` 代表 Clang编译器，只限于此编译器才有效，注意：
- 使用此宏后修饰的API必须有返回值，而且返回值必须是 `String type`。
- 如果修饰的是C函数，最好在函数声明中修饰。
- 如果参数被约束，那么参数必须是一个 `String type` 

**用法如下：**

```
NSString * testArgument(NSString *name,NSString *age,NSString *sex) NS_FORMAT_ARGUMENT(3);
NSString * testArgument(NSString *name,NSString *age,NSString *sex){
return @"gitKong";
}

- (NSString *)testArg:(NSString *)name age:(NSString *)age NS_FORMAT_ARGUMENT(1){
return [NSString stringWithFormat:@"hello %@,i am %@ years old",name,age];
}
```


![修饰C函数实现报警告](http://upload-images.jianshu.io/upload_images/1085031-196e3aa8cb0cc698.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![参数不是String类型](http://upload-images.jianshu.io/upload_images/1085031-c495d47dca213b79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![只约束了第一个参数，因此第二个参数不受限制](http://upload-images.jianshu.io/upload_images/1085031-d78d2e9bdb30ae16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![必须要返回`String type`](http://upload-images.jianshu.io/upload_images/1085031-90e5a0c28d943061.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### __has_attribute、__has_extension、__has_feature

```
// Some compilers provide the capability to test if certain features are available. This macro provides a compatibility path for other compilers.
#ifndef __has_feature
#define __has_feature(x) 0
#endif

#ifndef __has_extension
#define __has_extension(x) 0
#endif

// Some compilers provide the capability to test if certain attributes are available. This macro provides a compatibility path for other compilers.
#ifndef __has_attribute
#define __has_attribute(x) 0
#endif
```

> 如同GCC编译器, Clang也支持 `__attribute__` , 而且对它进行的一些扩展,如果要检查指定属性的可用性，你可以使用 `__has_attribute`指令，可在[LLVM
的 LanguageExtensions](https://clang.llvm.org/docs/LanguageExtensions.html)  中查询；`ns_returns_not_retained` 和 `objc_returns_inner_pointer` 可查询 [LLVM
的 AutomaticReferenceCounting](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)

> 摘自文档（针对 `__attribute__`）：**This function-like macro takes a single identifier argument that is the name of a GNU-style attribute. It evaluates to 1 if the attribute is supported by the current compilation target, or 0 if not.**

> - 其中 `__has_attribute(x)`、`__has_feature(x)` 为 0，表示 Compatibility with non-clang compilers，意思就是没有Clang编码
> -  `__has_extension ` 为 0，表示 Compatibility with pre-3.0 compilers，意思就是当前的feature 在当前语音中不被Clang 编译器支持

> 摘自文档（针对 `__has_feature(x)`、`__has_extension ` ）：**These function-like macros take a single identifier argument that is the name of a feature. `__has_feature` evaluates to 1 if the feature is both supported by Clang and standardized in the current language standard or 0 if not (but see [below](https://clang.llvm.org/docs/LanguageExtensions.html#langext-has-feature-back-compat)), while `__has_extension`
 evaluates to 1 if the feature is supported by Clang in the current language (either as a language extension or a standard language feature) or 0 if not.**

**用法如下：**

`__has_attribute(x)` 其实跟 GCC编译器的 `__attribute__ ` 类似，`__attribute__ ` 的 key 可查询 [Attributes Type](https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Type-Attributes.html#Type-Attributes)

`__has_feature(x)` 、 `__has_extension `中的 key 可查询 [Clang API Documentation 的PPMacroExpansion_8cpp_source](http://docs.huihoo.com/doxygen/clang/r222231/PPMacroExpansion_8cpp_source.html) 在 `00865` 行开始

---

- ### NS_RETURNS_RETAINED、NS_RETURNS_NOT_RETAINED

```
// Marks methods and functions which return an object that needs to be released by the caller but whose names are not consistent with Cocoa naming rules. The recommended fix to this is to rename the methods or functions, but this macro can be used to let the clang static analyzer know of any exceptions that cannot be fixed.
// This macro is ONLY to be used in exceptional circumstances, not to annotate functions which conform to the Cocoa naming rules.
#if __has_feature(attribute_ns_returns_retained)
#define NS_RETURNS_RETAINED __attribute__((ns_returns_retained))
#else
#define NS_RETURNS_RETAINED
#endif
```

```
// Marks methods and functions which return an object that may need to be retained by the caller but whose names are not consistent with Cocoa naming rules. The recommended fix to this is to rename the methods or functions, but this macro can be used to let the clang static analyzer know of any exceptions that cannot be fixed.
// This macro is ONLY to be used in exceptional circumstances, not to annotate functions which conform to the Cocoa naming rules.
#if __has_feature(attribute_ns_returns_not_retained)
#define NS_RETURNS_NOT_RETAINED __attribute__((ns_returns_not_retained))
#else
#define NS_RETURNS_NOT_RETAINED
#endif
```

> 表示函数返回值是一个是否需要被release的对象，官方解释了，这个宏除非特殊情况，不要自己使用，对于准守Cocoa 的命名规范的，这里有篇 [sunnyxx 的文章](http://blog.sunnyxx.com/2014/03/15/objc_arc_secret/) 说明:"编译器约定，对于`alloc`,`init`,`copy`,`mutableCopy`,`new`这几个家族的方法，后面默认加`NS_RETURNS_RETAINED`标识；而其他不指名标识的family的方法默认添加`NS_RETURNS_NOT_RETAINED`标识" ，这个也可在[LLVM ARC-Method families](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-method-families) 中查询到

**用法如下：**（摘自[sunnyxx 的文章](http://blog.sunnyxx.com/2014/03/15/objc_arc_secret/)）

```
@interface Sark : NSObject
+ (instancetype)sarkWithMark:(NSString *)mark NS_RETURNS_NOT_RETAINED; // 1
- (instancetype)initWithMark:(NSString *)mark NS_RETURNS_RETAINED; // 2
@end
```

---

- ### NS_RETURNS_INNER_POINTER

```
#ifndef NS_RETURNS_INNER_POINTER
#if __has_attribute(objc_returns_inner_pointer)
#define NS_RETURNS_INNER_POINTER __attribute__((objc_returns_inner_pointer))
#else
#define NS_RETURNS_INNER_POINTER
#endif
#endif
```

> 表示返回一个内部C指针，`objc_returns_inner_pointer` 可在[LLVM ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html) 中查询

> 摘自文档：An Objective-C method returning a non-retainable pointer may be annotated with the `objc_returns_inner_pointer` attribute to indicate that it returns a handle to the internal data of an object, and that this reference will be invalidated if the object is destroyed. When such a message is sent to an object, the object’s lifetime will be extended until at least the earliest of:
- the last use of the returned pointer, or any pointer derived from it, in the calling function or
- the autorelease pool is restored to a previous state.

**用法如下：**（摘自系统API）

```
@property (nullable, readonly) const char *UTF8String NS_RETURNS_INNER_POINTER;	// Convenience to return null-terminated UTF8 representation
```

---

- ### NS_AUTOMATED_REFCOUNT_UNAVAILABLE

```
// Marks methods and functions which cannot be used when compiling in automatic reference counting mode.
#if __has_feature(objc_arc)
#define NS_AUTOMATED_REFCOUNT_UNAVAILABLE __attribute__((unavailable("not available in automatic reference counting mode")))
#else
#define NS_AUTOMATED_REFCOUNT_UNAVAILABLE
#endif
```

> 表示函数method在ARC中无效, `objc_arc` 表示是ARC环境， 可在[Clang的PPMacroExpansion_8cpp_source](http://docs.huihoo.com/doxygen/clang/r222231/PPMacroExpansion_8cpp_source.html)中查询 `.Case("objc_arc", LangOpts.ObjCAutoRefCount) `

**用法如下：**

```
void c_method_test() NS_AUTOMATED_REFCOUNT_UNAVAILABLE{

}

- (void)oc_method_test NS_AUTOMATED_REFCOUNT_UNAVAILABLE{

}
```

![无法在ARC中使用](http://upload-images.jianshu.io/upload_images/1085031-f047085980e006f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE

```
// Marks classes which cannot participate in the ARC weak reference feature.
#if __has_attribute(objc_arc_weak_reference_unavailable)
#define NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE __attribute__((objc_arc_weak_reference_unavailable))
#else
#define NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE
#endif
```

> 表示标记的类在ARC中无法进行弱引用，`__has_attribute` 是 Clang编译器对GCC编译器的拓展，表示详细描述，`objc_arc_weak_reference_unavailable` 可在 [LLVM 的ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html) 查询

**用法如下：**

```
NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE @interface MacrosIntroduction : NSObject

@end
```

![弱引用就报错](http://upload-images.jianshu.io/upload_images/1085031-e7ca84023cb0d47b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### NS_REQUIRES_PROPERTY_DEFINITIONS

```
// Marks classes that must specify @dynamic or @synthesize for properties in their @implementation (property getters & setters will not be synthesized unless the @synthesize directive is used)
#if __has_attribute(objc_requires_property_definitions)
#define NS_REQUIRES_PROPERTY_DEFINITIONS __attribute__((objc_requires_property_definitions)) 
#else
#define NS_REQUIRES_PROPERTY_DEFINITIONS
#endif
```

> 表示标记的类必须在 `@implementation` 中为其属性实现 `@dynamic` or `@synthesize`，其中 `objc_requires_property_definitions` 可在[ConceptClang API Documentation -- Extended 中的AttrParsedAttrKinds.inc](https://www.crest.iu.edu/projects/conceptcpp/docs/html-ext/AttrParsedAttrKinds_8inc_source.html) 查询，如果不实现，使用属性的`setter` or `getter` 方法的时候就会crash：`Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[MacrosIntroduction setClassName:]: unrecognized selector sent to instance 0x60000001d2e0'`

**用法如下：**

```
NS_REQUIRES_PROPERTY_DEFINITIONS @interface MacrosIntroduction : NSObject

@property (NS_NONATOMIC_IPHONEONLY,copy) NSString *className;

@end
```

```
@implementation MacrosIntroduction

@synthesize className;

@end
```


![没有实现 `@dynamic` or `@synthesize`](http://upload-images.jianshu.io/upload_images/1085031-19cce3d8909400dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### NS_REPLACES_RECEIVER

```
// Decorates methods in which the receiver may be replaced with the result of the method. 
#if __has_feature(attribute_ns_consumes_self)
#define NS_REPLACES_RECEIVER __attribute__((ns_consumes_self)) NS_RETURNS_RETAINED
#else
#define NS_REPLACES_RECEIVER
#endif
```

> 替换接收者，就是替换成另一个对象，类似 `self = [super init]` 一样，可在
[Clang 中的annotations](http://clang-analyzer.llvm.org/annotations.html#attr_ns_consumes_self) 查询

**用法如下：**

```
- (instancetype)hello NS_REPLACES_RECEIVER;// 此时方法命名不规范，只是作演示效果
```

```
- (instancetype)hello{
return self = [super init];
}
```

![没有使用 `NS_REPLACES_RECEIVER` 修饰](http://upload-images.jianshu.io/upload_images/1085031-2706422a2036ea08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**系统例子：**（`NSObject (NSCoderMethods)`）

```
- (nullable id)awakeAfterUsingCoder:(NSCoder *)aDecoder NS_REPLACES_RECEIVER;
```

上面的方法在 [sunnyxx 的 xib的动态桥接](http://blog.sunnyxx.com/2014/07/01/ios_ib_bridge/) 一文中有说明，也举了个好例

---

- ### NS_RELEASES_ARGUMENT

```
#if __has_feature(attribute_ns_consumed)
#define NS_RELEASES_ARGUMENT __attribute__((ns_consumed))
#else
#define NS_RELEASES_ARGUMENT
#endif
```

> 其实在ARC 下默认的参数都会准守这个，当函数或者方法执行完毕后，就会调用对象参数的 `release` 方法，其中 `ns_consumed` 可查询 [clang-analyzer annotations](http://clang-analyzer.llvm.org/annotations.html#attr_ns_consumes_self) 

> 摘自文档：The 'ns_consumed' attribute can be placed on a specific parameter in either the declaration of a function or an Objective-C method. It indicates to the static analyzer that a release message is implicitly sent to the parameter upon completion of the call to the given function or method. The Foundation framework defines a macro `NS_RELEASES_ARGUMENT` that is functionally equivalent to the `NS_CONSUMED` macro shown below.

**用法如下：**

```
- (void)oc_method_test1:(NSString *)name age:(NS_RELEASES_ARGUMENT NSString *)age{
NSLog(@"age = %@",age);
}
```

---

- ### NS_VALID_UNTIL_END_OF_SCOPE

```
// Mark local variables of type 'id' or pointer-to-ObjC-object-type so that values stored into that local variable are not aggressively released by the compiler during optimization, but are held until either the variable is assigned to again, or the end of the scope (such as a compound statement, or method definition) of the local variable.
#ifndef NS_VALID_UNTIL_END_OF_SCOPE
#if __has_attribute(objc_precise_lifetime)
#define NS_VALID_UNTIL_END_OF_SCOPE __attribute__((objc_precise_lifetime))
#else
#define NS_VALID_UNTIL_END_OF_SCOPE
#endif
#endif
```

> 表明存储在某些局部变量中的值在优化时不应该被编译器强制释放，翻译官方：局部变量标记为id类型或者是指向ObjC对象类型的指针，以便存储在这些局部变量中的值在优化时不会被编译器强制释放。相反，这些值会在变量再次被赋值之前或者局部变量的作用域结束之前都会被保存。其中 `objc_precise_lifetime` 可在 [LLVM ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html) 可查
- 至于具体的应用场景也说不出，反正遇到了就知道有这个东西可以防止提前释放。

**用法如下：**

```
- (void)oc_method_test {
NS_VALID_UNTIL_END_OF_SCOPE id className = @"xxx";
}
```

---

- ### NS_ROOT_CLASS

```
// Annotate classes which are root classes as really being root classes
#ifndef NS_ROOT_CLASS
#if __has_attribute(objc_root_class)
#define NS_ROOT_CLASS __attribute__((objc_root_class))
#else
#define NS_ROOT_CLASS
#endif
#endif
```

> 表示当前修饰的类就是根类

**用法如下：**（NSObject 类就是 `OBJC_ROOT_CLASS` 跟 `NS_ROOT_CLASS` 一样，都是 ` __attribute__((objc_root_class))`）

```
NS_ROOT_CLASS
@interface MacrosIntroduction

@end
```

此时可以实现类似 `NSObject` 、`NSProxy` 一样，注意，此时去掉 `NS_ROOT_CLASS` 就会编译失败


![编译失败，需要添加父类](http://upload-images.jianshu.io/upload_images/1085031-cd5d140a88b8cdc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 拓展：
```
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-root-class"
```
同样可以避免编译失败，但此时的作用是，忽略编译器对根类的警告，意味着这样做不安全，但是 `__attribute__((objc_root_class))` 在以前的 `GCC Objective-C extension` 编译器是无法识别

```
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-root-class"
@interface MacrosOtherClass

@end
```

那么此时就好玩了，我完全可以仿造一个类似 `NSObject` 的类，至于对象的实例化，其实就是需要一个 `isa` 指针，指向起元类，看 `NSObject` 类定义就发现也是有这么一个 `isa` 指针,这里有篇 [文章](https://uranusjr.com/blog/post/53/objective-c-class-without-nsobject/),就是讲解这个东西，对象的创建没有那么简单，可以参考霜大佬的 [Objc 对象的今生今世](http://www.jianshu.com/p/f725d2828a2f)，大家有兴趣可以尝试尝试，笔者试了，发现并没有想象中那么简单，待研究，有机会再开新文章介绍。

---

- ### NS_REQUIRES_SUPER

```
#ifndef NS_REQUIRES_SUPER
#if __has_attribute(objc_requires_super)
#define NS_REQUIRES_SUPER __attribute__((objc_requires_super))
#else
#define NS_REQUIRES_SUPER
#endif
#endif
```

> 表示标志子类继承这个方法时需要调用 `super`，否则给出编译警告，其中 `objc_requires_super` 可在 [LLVM 的 AttributeReference](https://clang.llvm.org/docs/AttributeReference.html) 中查询，注意 `protocol` 中无效

> 摘自文档：
- Some Objective-C classes allow a subclass to override a particular method in a parent class but expect that the overriding method also calls the overridden method in the parent class. For these cases, we provide an attribute to designate that a method requires a “call to super” in the overriding method in the subclass.
- This attribute can only be applied the method declarations within a class, and not a protocol. Currently this attribute does not enforce any placement of where the call occurs in the overriding method (such as in the case of -dealloc where the call must appear at the end). It checks only that it exists.

**用法如下：**

```
- (void)oc_method_mustCallSuper NS_REQUIRES_SUPER;// 父类中声明
```

```
- (void)oc_method_mustCallSuper{
[super oc_method_mustCallSuper];// 子类中实现需要调用super
}
```


![不调用super，编译器警告](http://upload-images.jianshu.io/upload_images/1085031-f157f2e9d97107d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![protocol 中无效](http://upload-images.jianshu.io/upload_images/1085031-fc7855aac503d8ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- ### NS_DESIGNATED_INITIALIZER

```
#ifndef NS_DESIGNATED_INITIALIZER
#if __has_attribute(objc_designated_initializer)
#define NS_DESIGNATED_INITIALIZER __attribute__((objc_designated_initializer))
#else
#define NS_DESIGNATED_INITIALIZER
#endif
#endif
```

> 指定类的初始化方法。指定初识化方法并不是对使用者。而是对内部的现实，其实就是必须调用父类的 `designated initializer` 方法，至于的作用，[这里](https://gist.github.com/steipete/9482253) 有说明

> 摘自原文：
To clarify the distinction between designated and secondary initializers clear, you can add the NS_DESIGNATED_INITIALIZER macro to any method in the init family, denoting it a designated initializer. Using this macro introduces a few restrictions:
- The implementation of a designated initializer must chain to a superclass init method (with [super init...]) that is a designated initializer for the superclass.
- The implementation of a secondary initializer (an initializer not marked as a designated initializer within a class that has at least one initializer marked as a designated initializer) must delegate to another initializer (with [self init...]).
- If a class provides one or more designated initializers, it must implement all of the designated initializers of its superclass.

**用法如下：**

```
- (instancetype)initWithClassName:(NSString *)name NS_DESIGNATED_INITIALIZER;// 方法声明

- (instancetype)initWithClassName:(NSString *)name{// 方法实现
if (self = [super init]) {
self.className = name;
}
return self;
}
```

> 此时编译器就会出现警告，原因很简单，本类中实现了 `NS_DESIGNATED_INITIALIZER` 那么必须实现父类的 `NS_DESIGNATED_INITIALIZER` 方法


![必须实现父类的 `NS_DESIGNATED_INITIALIZER` 方法](http://upload-images.jianshu.io/upload_images/1085031-de104c8a862acaab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**实现父类的 `init` 方法，因为 `init` 也是`NS_DESIGNATED_INITIALIZER` 修饰**

```
- (instancetype)init{
self.className = @"";
return self;
}
```

> 当然，此时毫无疑问会出现编译警告，原因很简单，本类中实现了 `NS_DESIGNATED_INITIALIZER` ，那么构造方法中必须调用`NS_DESIGNATED_INITIALIZER` 方法，因此这里就有个大坑，操作不慎容易造成循环调用，下面会解释


![编译警告：没调用`NS_DESIGNATED_INITIALIZER` 方法](http://upload-images.jianshu.io/upload_images/1085031-d01f111de468fd05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**实现父类的 `init` 方法并调用`NS_DESIGNATED_INITIALIZER` 修饰的方法**

```
- (instancetype)init{
return [self initWithClassName:@"gitKong"];
}

- (instancetype)initWithClassName:(NSString *)name{
if (self = [super init]) {
self.className = name;
}
return self;
}
```
> 此时编译警告就没了，也能正常使用，不过注意 `initWithClassName` 方法不要调用 `[self init]` ，当然此时编译器很智能提示你，`NS_DESIGNATED_INITIALIZER` 修饰的方法只能使用 `super` 调用其他`NS_DESIGNATED_INITIALIZER` 修饰的方法，如果你忽略这个警告，那么就会出现循环调用了，更多具体的说明可参考 [Clang 拾遗之objc_designated_initializer](https://yq.aliyun.com/articles/5847)

![编译警告：必须使用super](http://upload-images.jianshu.io/upload_images/1085031-7f330c955a87caa0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### NS_PROTOCOL_REQUIRES_EXPLICIT_IMPLEMENTATION

```
#ifndef NS_PROTOCOL_REQUIRES_EXPLICIT_IMPLEMENTATION
#if __has_attribute(objc_protocol_requires_explicit_implementation)
#define NS_PROTOCOL_REQUIRES_EXPLICIT_IMPLEMENTATION __attribute__((objc_protocol_requires_explicit_implementation))
#else
#define NS_PROTOCOL_REQUIRES_EXPLICIT_IMPLEMENTATION
#endif
#endif
```

> 准守使用 `NS_PROTOCOL_REQUIRES_EXPLICIT_IMPLEMENTATION`
修饰的协议后，当前类必须实现协议方法，如果此时当前类的父类已经准守协议并实现了协议方法，当前类则无须实现，编译器不会报警告。当然如果协议方法是 `@optional` 修饰，就不需要实现，编译器不会报警告。另外可参考 [这里](http://lists.llvm.org/pipermail/cfe-commits/Week-of-Mon-20140303/100492.html)，还有这里有比较详细的 [例子](https://github.com/llvm-mirror/clang/blob/master/test/SemaObjC/protocols-suppress-conformance.m)。

> 摘自原文：
- Per more discussion, 'objc_protocol_requires_explicit_implementation' is
refinement that it mainly adds that requirement that a protocol must be
explicitly satisfied at the moment the first class in the class hierarchy
conforms to it.  Any subclasses that also conform to that protocol,
either directly or via conforming to a protocol that inherits that protocol,
do not need to re-implement that protocol.
- Doing this requires first doing a pass on the super class hierarchy,
gathering the set of protocols conformed to by the super classes,
and then culling those out when determining conformance.  This
two-pass algorithm could be generalized for all protocol checking,
and could possibly be a performance win in some cases.  For now
we restrict this change to protocols with this attribute to isolate
the change in logic (especially as the design continues to evolve).

**用法如下：**

```
NS_PROTOCOL_REQUIRES_EXPLICIT_IMPLEMENTATION
@protocol MacrosIntroductionProtocol <NSObject>
@optional
- (void)sayHello;

@required
- (void)sayHi;

@end
```

```
NS_REQUIRES_PROPERTY_DEFINITIONS @interface MacrosIntroduction:NSObject<MacrosIntroductionProtocol>// 准守协议
```


![编译警告：`@required` 修饰的协议方法没有实现](http://upload-images.jianshu.io/upload_images/1085031-3957446fc20c786d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### NS_NO_TAIL_CALL

```
#if __has_attribute(not_tail_called)
#define NS_NO_TAIL_CALL __attribute__((not_tail_called))
#else
#define NS_NO_TAIL_CALL
#endif
```

> 直译过来就是不是最后调用，对于间接调用(下面介绍)的方法是无效的，而且不能使用来修饰虚函数、OC的方法、被 `always_inline ` 修饰的函数，可参考 [LLVM 的 AttributeReference](https://clang.llvm.org/docs/AttributeReference.html#not-tail-called-clang-not-tail-called)，至于详细的应该

> 摘自原文：The not_tail_called attribute prevents tail-call optimization on statically bound calls. It has no effect on indirect calls. Virtual functions, objective-c methods, and functions marked as always_inline cannot be marked as not_tail_called.

**用法如下：**

```
void NS_NO_TAIL_CALL print(){
printf("---\n");
}
```

系统的 `NSLog` 其实也是用 `NS_NO_TAIL_CALL` 修饰，至于具体用法和效果，笔者也没找到相关详细介绍

```
FOUNDATION_EXPORT void NSLog(NSString *format, ...) NS_FORMAT_FUNCTION(1,2) NS_NO_TAIL_CALL;
```

**什么是间接调用，其实就是用变量存储起来，OC里面用闭包，因为函数不是一等公民，swift的话就是一等公民了**

```
void hi(){
void (*func)() = &print;
(*func)();// === func() === print()
}
```

---

- ### NS_UNAVAILABLE、UNAVAILABLE_ATTRIBUTE

```
#if !defined(NS_UNAVAILABLE)
#define NS_UNAVAILABLE UNAVAILABLE_ATTRIBUTE
#endif
```

```
/*
* only certain compilers support __attribute__((unavailable))
*/
#if defined(__GNUC__) && ((__GNUC__ >= 4) || ((__GNUC__ == 3) && (__GNUC_MINOR__ >= 1)))
#define UNAVAILABLE_ATTRIBUTE __attribute__((unavailable))
#else
#define UNAVAILABLE_ATTRIBUTE
#endif
```

> 两个宏代表同一个东西，可以修饰任何东西(包括类、方法、属性、变量等等)，表示当前编译器并不支持，或者说当前平台不支持或者无效，其中 `__attribute__ ` 上文也有介绍，是 `GCC` 编译器特有的，用作描述详细信息，`unavailable` 语义上就是表示无效的，可以在 [LLVM 的 AttributeReference](https://clang.llvm.org/docs/AttributeReference.html#not-tail-called-clang-not-tail-called) 中查询。当然，苹果也解释了，只有某些编译器支持 `__attribute__((unavailable))`

> 摘自原文：This declaration is never available on this platform.

**用法如下：**

```
- (void)sayHi NS_UNAVAILABLE;
```


![此时使用就出现编译错误](http://upload-images.jianshu.io/upload_images/1085031-f747c67bd7130fb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

- ### NS_AVAILABLE、NS_DEPRECATED 等

```
#include <CoreFoundation/CFAvailability.h>

#define NS_AVAILABLE(_mac, _ios) CF_AVAILABLE(_mac, _ios)
#define NS_AVAILABLE_MAC(_mac) CF_AVAILABLE_MAC(_mac)
#define NS_AVAILABLE_IOS(_ios) CF_AVAILABLE_IOS(_ios)

#define NS_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, ...) CF_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, __VA_ARGS__)
#define NS_DEPRECATED_MAC(_macIntro, _macDep, ...) CF_DEPRECATED_MAC(_macIntro, _macDep, __VA_ARGS__)
#define NS_DEPRECATED_IOS(_iosIntro, _iosDep, ...) CF_DEPRECATED_IOS(_iosIntro, _iosDep, __VA_ARGS__)

#define NS_DEPRECATED_WITH_REPLACEMENT_MAC(_rep, _macIntroduced, _macDeprecated) API_DEPRECATED_WITH_REPLACEMENT(_rep, macosx(_macIntroduced, _macDeprecated)) API_UNAVAILABLE(ios, watchos, tvos)

#define NS_ENUM_AVAILABLE(_mac, _ios) CF_ENUM_AVAILABLE(_mac, _ios)
#define NS_ENUM_AVAILABLE_MAC(_mac) CF_ENUM_AVAILABLE_MAC(_mac)
#define NS_ENUM_AVAILABLE_IOS(_ios) CF_ENUM_AVAILABLE_IOS(_ios)

#define NS_ENUM_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, ...) CF_ENUM_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, __VA_ARGS__)
#define NS_ENUM_DEPRECATED_MAC(_macIntro, _macDep, ...) CF_ENUM_DEPRECATED_MAC(_macIntro, _macDep, __VA_ARGS__)
#define NS_ENUM_DEPRECATED_IOS(_iosIntro, _iosDep, ...) CF_ENUM_DEPRECATED_IOS(_iosIntro, _iosDep, __VA_ARGS__)

#define NS_AVAILABLE_IPHONE(_ios) CF_AVAILABLE_IOS(_ios)
#define NS_DEPRECATED_IPHONE(_iosIntro, _iosDep) CF_DEPRECATED_IOS(_iosIntro, _iosDep)

...
```

> 其中很多涉及到都是`AVAILABLE`和`DEPRECATED`，其实都表示当前API在不同的操作系统（这里只谈苹果的操作系统）的可用版本、废弃版本、淘汰版本，其中关键就是 `availability `,在 [LLVM 的 AttributeReference](https://clang.llvm.org/docs/AttributeReference.html#not-tail-called-clang-not-tail-called) 中有详细参数说明以及其用法举例，因此本文就不再啰嗦。还有一个 `visibility` ，表示编译符号可见性，提供两个值 `default` 和 `hidden` ，这里有文章：[控制符号的可见性](http://my.huhoo.net/archives/2010/03/post-52.html) 作了详细的说明，文中说了， `visibility` 可见性控制只能应用到代码中的C或者C++子集，不能应用到Objective-C的类和方法上。

---

- ### NS_ENUM、NS_OPTIONS

```
/* NS_ENUM supports the use of one or two arguments. The first argument is always the integer type used for the values of the enum. The second argument is an optional type name for the macro. When specifying a type name, you must precede the macro with 'typedef' like so:

typedef NS_ENUM(NSInteger, NSComparisonResult) {
...
};

If you do not specify a type name, do not use 'typedef'. For example:

NS_ENUM(NSInteger) {
...
};
*/
#define NS_ENUM(...) CF_ENUM(__VA_ARGS__)
#define NS_OPTIONS(_type, _name) CF_OPTIONS(_type, _name)
```

> 具体的定义在 `CFAvaliability.h` ，都是表示枚举，区别在于 `NS_ENUM` 是通用枚举，而 `NS_OPTIONS` 则是位移枚举，位移枚举的值就是用位移值表示，其中内部对编译器做了适配， `__cplusplus` 表示支持C++编译器，`__has_feature(objc_fixed_enum)` 表示当前编译器是否支持固定的没定义类型，参考 [LLVM 的 LanguageExtensions](https://clang.llvm.org/docs/LanguageExtensions.html) 

> 摘自原文：Use `__has_feature(objc_fixed_enum)` to determine whether support for fixed underlying types is available in Objective-C.

用法就不再举例，相信大家都用烂了~

---

- ### CF_STRING_ENUM、CF_EXTENSIBLE_STRING_ENUM

```
#ifndef CF_STRING_ENUM
#if __has_attribute(swift_wrapper)
#define _CF_TYPED_ENUM __attribute__((swift_wrapper(enum)))
#else
#define _CF_TYPED_ENUM
#endif

#define CF_STRING_ENUM _CF_TYPED_ENUM
#endif

#ifndef CF_EXTENSIBLE_STRING_ENUM
#if __has_attribute(swift_wrapper)
#define _CF_TYPED_EXTENSIBLE_ENUM __attribute__((swift_wrapper(struct)))
#else
#define _CF_TYPED_EXTENSIBLE_ENUM
#endif

#define CF_EXTENSIBLE_STRING_ENUM _CF_TYPED_EXTENSIBLE_ENUM
#endif
/* */

#define _NS_TYPED_ENUM _CF_TYPED_ENUM
#define _NS_TYPED_EXTENSIBLE_ENUM _CF_TYPED_EXTENSIBLE_ENUM

#define NS_STRING_ENUM _NS_TYPED_ENUM
#define NS_EXTENSIBLE_STRING_ENUM _NS_TYPED_EXTENSIBLE_ENUM
```

> 这两个宏是iOS 10引入的，可查 [CoreFoundation Changes for Objective-C](https://developer.apple.com/library/content/releasenotes/General/iOS10APIDiffs/Objective-C/CoreFoundation.html),可惜官方没有提供具体说明，不过从语义上看应该能猜到，这两个宏就是为了兼容 `swift` 的，其中 `swift_wrapper(enum)` 和 `swift_wrapper(struct)` 笔者暂时没查阅到具体介绍，不过很容易就猜出，一个是表示枚举，一个表示结构体


![Not Found](http://upload-images.jianshu.io/upload_images/1085031-e916185e8e7a7af5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**用法如下：**

```
typedef CF_STRING_ENUM NS_ENUM(NSInteger ,FLSystemEnum){
FLSystemEnumOne,
FLSystemEnumTwo = 1
} ;
```

---

- ### NS_ASSUME_NONNULL_BEGIN、NS_ASSUME_NONNULL_END

```
#define NS_ASSUME_NONNULL_BEGIN _Pragma("clang assume_nonnull begin")
#define NS_ASSUME_NONNULL_END   _Pragma("clang assume_nonnull end")
```

> 表示当前包在里面的属性都标记为 `nonnull`,如果需要其他处理，则单独使用其他（例如：`nullable`）标记，`_Pragma("string")` 与 `#pragma` 字符串行为完全相同,在C/C++标准中，`#pragma`是一条预处理的指令（preprocessor directive）,在C++11中，标准定义了与预处理指令`#pragma`功能相同的操作符`_Pragma`。简单地说，`#pragma`是用来向编译器传达语言标准以外的一些信息,例如我们平时经常使用的 `#pragma mark - <#name#>`,可参考 [C/C++ 预处理器参考](https://msdn.microsoft.com/zh-cn/library/d9x1s805.aspx#),那么上面的其实就是等同：

```
#pragma clang assume_nonnull begin
#pragma clang assume_nonnull end
```

**用法如下：**

```
//NS_ASSUME_NONNULL_BEGIN
#pragma clang assume_nonnull begin
@interface MacrosIntroduction:NSObject
@property (nonatomic,assign) FLSystemEnum systemEnums;
@property (nullable, nonatomic,copy) NSString *xxx;
@end
#pragma clang assume_nonnull end
//NS_ASSUME_NONNULL_END
```

---

- ### NS_SWIFT_NAME

```
#define NS_SWIFT_NAME(_name) CF_SWIFT_NAME(_name)
```

> 利用这个宏，可间接使用C函数，具体可参考 [苹果官方 Guides and Sample Code](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html)，里面有具体例子讲解

> 摘自原文：
C APIs, such as the Core Foundation framework, often provide functions that create, access, or modify C structures. You can use the CF_SWIFT_NAME macro in your own code to have Swift import C functions as members of the imported structure type. 

用法官方文档有详细例子，此处不再举例

---

还有一些针对 swift 的宏，例如 `NS_SWIFT_UNAVAILABLE`（表示在`swift`中无效）、`NS_NOESCAPE`（`swift`中有逃逸概念，默认闭包是noescape）、`NS_SWIFT_NOTHROW`（意思就是在`swift`中没有错误抛出） ，从语义上就能看什么作用，这里就不再做详细分析。

`NSObjCRuntime.h` 最后还有一下基本运算的宏，例如大家熟悉的 `YES` or `NO`定义、`Min` or `MAX`、`ABS`等

---

- ###最后

-  花了不少时间，资料总算整理好了，通过整理这份资料，也了解了不少编译器方面的知识，希望能帮到大家。

-  如果文中有不对的地方，或者有什么建议，请务必提出喔，喜欢我的文章，可以点个like，加个关注，我会不定时更新文章。

---

### 参考资料：

- https://developer.apple.com

- http://clang.llvm.org/docs/AttributeReference.html

- https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Type-Attributes.html#Type-Attributes

- https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Variable-Attributes.html#Variable-Attributes

- https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Function-Attributes.html#Function-Attributes

- https://clang.llvm.org/docs/AutomaticReferenceCounting.html

- https://clang.llvm.org/docs/LanguageExtensions.html

- https://www.crest.iu.edu/projects/conceptcpp/docs/html-ext/AttrParsedAttrKinds_8inc_source.html

- https://gcc.gnu.org/onlinedocs/gcc-3.1.1/cpp/Common-Predefined-Macros.html#Common%20Predefined%20Macros

- http://docs.huihoo.com/doxygen/clang/r222231/PPMacroExpansion_8cpp_source.html

- http://gracelancy.com/blog/2014/05/05/variable-argument-lists/

- http://clang-developers.42468.n3.nabble.com/APPLE-CC-macro-td4046240.html#a4046243

- https://uranusjr.com/blog/post/53/objective-c-class-without-nsobject/

- http://blog.sunnyxx.com/2014/07/01/ios_ib_bridge/

- https://gist.github.com/steipete/9482253
