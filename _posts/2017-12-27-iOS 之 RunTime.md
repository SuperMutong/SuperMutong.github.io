---
layout:     post
title:      iOS 之 RunTime
subtitle:   RunTime
date:       2017-12-27
author:     Mutong
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - 基础
    - RunTime
---


###  RunTime 学习笔记
#### 		1. 简介
&emsp;&emsp; 因为 Objc 是一门动态语言, 所以他总是想办法把一些决定工作从编译连接推迟到运行时, 这就需要一个运行时系统 `(runtime system)` 来执行编译后的代码, 这就是 Objective-C Runtime 系统存在的意义, 他是整个 Objc 运行框架的一块基石

&emsp;&emsp; 为了动态系统的高效, 所以 Runtime  基本是用 C 和汇编写的,
#### 		2. 核心就一句话-[receiver message]
&emsp;&emsp;编译器会把他转换成 
`bjc_msgSend(receiver, selector)` 
或者
`objc_msgSend(receiver, seletor,arg1,arg2)`

&emsp;&emsp;如果消息的接受者能够找到对应的 selector, 那么就相当于直接执行了接受者这个对象的特定方法, 否则, 消息要么被转发, 或者临时向接受者动态添加这个 selector 对应的实现内容, 要么干脆玩完奔溃

&emsp;&emsp;现在可以看出 [receiver message] 真的不是一个简简单单的方法调用, 因为这只是在编译阶段确定了要向接受者发送 message 这条消息, 而 receiver 将要如果响应这条消息, 那就要看运行时发生的情况来决定了

#### 		3. 与 Runtime 交互
&emsp;&emsp;Objc 从三种不同的层级上与 Runtime 系统进行交互, 分别是通过 OC 源代码, 通过 Foundation 框架的 NSObject 类定义的方法, 通过对 runtime 函数的直接调用
##### 			1. Objective-c 源代码
&emsp;&emsp;大部分情况下你就只管写 OC 代码, Runtime 系统自动在底层处理, 编译器会将 OC 代码转换成运行时代码, 在运行时确定数据结构和函数
##### 			2. 通过 Foundation 框架的 NSObject 类定义的方法
&emsp;&emsp;Cocoa 程序中绝大部分类都是 NSObject 类的子类, 所以都继承了 NSObject 的行为

&emsp;&emsp;一般情况下, NSObject 类仅仅定义了完成某件事情的模板, 并没有提供所需要的代码, 例如 -description 方法, 该方法返回类内容的字符串表示, 该方法主要用来调试程序. NSObject 类并不知道子类的内容, 所以他只是返回类的名字和对象的地址, NSObject的子类可以重新实现

&emsp;&emsp;还有一些 NSObject 的方法可以从 Runtime 系统中获取信息, 允许对象进行自我检查, 例如:

```
-class 返回对象的类
-isKindOfClass: 和 -isMemberOfClass:方法检查对象是否存在于指定的类的集成体系中
```
			
##### 			3. Runtime 的函数
&emsp;&emsp;Runtime 系统是一个由一系列函数和数据结构组成, 具有公共接口的动态共享库, 我们使用时只需要引入 objc/Runtime.h 头文件即可. 许多函数允许你用纯 C 代码来重复实现 Objc 中同样的功能, 你在写 Objc 代码时一般不会直接用到这些函数, 除非写一些 Objc 与其他语言的桥接或者底层的 debug 工作
### Runtime 基础数据结构
#### 	1. 前面提到了 objc_msgSend:方法, 他的真身其实是下面这个
```
id objc_msgSend(id self, SEL op, ...)
```
参数解读

#### 	1. id 他是一个指向类实例的指针
```
typedef struct objc_object *id
struct objc_object {
		private:
			isa_t isa;
		public:
			// ISA() assumes this is NOT a tagged pointer object
			Class ISA();
			// getIsa() allows this to be a tagged pointer object
			Class getIsa();
			 ... 此处省略其他方法声明
}
```
&emsp;&emsp;`objc_object` 结构体包含一个 `isa` 指针, 类型为` isa_t` 联合体, 根据 isa 就可以找到对象所属的类, 但是 `isa` 指针不总是指向实例类对象所属的类, 不能依靠他来确定类型, 而是应该用 class 方法来确认实例对象的类, 因为 `KVO` 的实现机理([面试总结第七条](https://supermutong.github.io/2017/12/27/RunTime-%E9%9D%A2%E8%AF%95%E6%80%BB%E7%BB%93/))就是讲被观察对象的 `isa` 指针指向一个中间类而不是真是的类, 这是一种叫做 `isa-swizzling` 的技术
#### 			2. SEL 
&emsp;&emsp;他是 `selector` 在 Objc 中的表达类型(在 Swift 中是`Selector` 类). `selector` 是方法选择器, 可以理解为区分方法的 ID, 而这个 ID 的数据结构是 `SEL`.  
```
typedef struct objc_selector *SEL
```
&emsp;&emsp;不同类中相同名字的方法对应的方法选择器是相同的, 即使方法名字相同而变量类型不同也会导致他们具有相同的方法选择器, 因为 selector 只记了 method 的 name, 没有参数
#### 			3. ... 
&emsp;&emsp;代表参数, 是变参, 也就是方法携带的参数, 可以没有, 可以有多个
#### 			4. Class  
一个指向` objc_class `结构体的指针
```
struct objc_class:objc_object{
	Class isa  isa指向其所属的元类
    Class super_class  超类
	const char *name  类名
	long version   类的版本信息
	long info   类的详情
	struct objc_ivar_list *ivars  类的成员变量列表
	struct objc_method_list **methodLists 实例方法列表 即 -func
	struct objc_cache *cache  被调用的方法存到 cache 中, 方便下次查找
	struct objc_protocol_list *protocols  该类的协议列表
	class_data_bits_t bits 
	}
```
&emsp;&emsp;这里解释下元类, 先从类对象开始

&emsp;&emsp;类对象: 它是由编译器创建的, 即在编译时所谓的类 比如 `NSString`, `NSArray`, 就是指类对象

&emsp;&emsp;`objc_class` 继承自 `objc_object `, 为了处理类和对象的关系, runtime 库创建了一个叫做元类 (`meta class`) 的东西, 类对象所属类型就叫做元类, 它用来表述类对象本身所具备的元数据, 每个类仅有一个类对象, 而每个类对象仅有一个与之相关的元类, 类方法存储在元类中,

&emsp;&emsp;当你发出类似 `[NSObject alloc]` 的消息时, 你事实上是把这个消息发给了一个类对象(class Object), 这个类对象鄙视是一个元类的实例, 而这个元类同时也是一个根源类(root meta class) 的实例, 所有的元类最终都指向根元类为其超类. 所有的元类的方法列表都有能够响应消息的类方法, 所以当 `[NSObject alloc]` 这条消息发给类对象的时候, `objc_msgSend()` 会去它的元类里面查找能够响应消息的方法, 如果找到了, 然后对这个类对象执行方法调用

&emsp;&emsp;元类就是类对象的类, 类对象是元类的实例

&emsp;&emsp;我们以前调用 "+" 开头的类方法实际是在调用元类的对象方法

&emsp;&emsp;由于每个类有且只有一个, 所以每个类对象都是其对应元类的单例

&emsp;&emsp;我们接触到的大部分 OC 对象都继承自 `NSObject`, 这里直接以 `NSObject` 为例
1. 每个实例对象的类都是类对象, 每个类对象的类都是元类对象, 每个元类对象的类都是根元类
2. 类对象的父类最终继承自根类对象 NSObject, NSObject 的父类为 `nil`
3. 元类对象 (包括根元类) 的父类最终继承自根类对象 NSobject
看下图						
![](https://ws4.sinaimg.cn/large/006tNc79ly1fmh4c8l9koj30e00epaak.jpg)
&emsp;&emsp;上图实线是`superclass` 指针, 虚线是 `ias`指针, 有趣的是根元类的超类时`NSObject`, 而 `isa` 指向了自己, 而 `NSObject `的超类为 `nil`, 也就是他没有超类
&emsp;&emsp;下图是一个具体类的继承关系
![enter image description here](http://blogofzuoyebuluo.qiniudn.com/image_note64271_1.png)

&emsp;&emsp;小知识: 元 (meta) 是什么	
&emsp;&emsp;`meta` 在英语中并不是单字, 通常会和后面的词链接起来, 用来描述后面的词, 表示 "关于...的...", 举个例子, `meta-data `就是关于数据的数据, 即这个数据大小, 类型等用来描述数据的数据信息, 在比如 `meta-language` 就是关于语言的语言, `meta-class `就是关于类的类
&emsp;&emsp; 这里又点绕,最好多啃几遍

&emsp;&emsp;下面解释下 `objc_class` 结构体里面的各个参数
##### 	4.1 cache_t 
```
struct cache_t {
	struct bucket_t *_buckets;
	mask_t _mask;
	mask_t _occupied;
	..... 省略其他方法
}
```
&emsp;&emsp;`_buckets` 存储 `IMP`,` _mask` 和 `_occupied`对应 `vtable`

&emsp;&emsp;`cache` 为方法调用的性能进行优化, 通俗的讲, 每当实例对象接收到一个消息时, 它不会直接在 `isa`指向的类的方法列表中遍历查询找能够响应的方法, 因为这样的效率太低了, 而是优先在 `cache `中查找, Runtime 系统会把调用的方法存储到 `cache` 中 (理论上讲一个方法如果被调用, 那么他有可能今后还会被调用), 下次查找的时候效率更高
			`bucket_t` 中存储了指针与 `IMP` 的键值对
```
	struct bucket_t {
		private:
			cache_key_t _key;
			IMP _imp;
		public:
			inline cache_key_t key() const {return _key;}
			inline IMP imp() const {return (IMP) _imp;}
			inline void setImp(IMP newImp) {_imp = newImp;}
			void set(cache_key_t newKey, IMP newImp);
			}
```
##### 	4.2 class_data_bits_t  
```
struct class_data_bits_t{
	uintptr_t bits;
	class_rw_t* data(){
		return (class_rw_t *)(bits & FAST_DATA_MASK)
		}
		...省略其他方法
}
```
&emsp;&emsp; `objc_class` 中最复杂的是 `bits` , `class_data_bits_t` 结构体包含的信息太多了, 主要包含了 `class_data_bits_t `, `alloc` 等信息, 很多存取方法也是围绕他展开. 
##### 	4.3  class_ro_t 
&emsp;&emsp; `objc_class `包含了 `class_data_bits_t` , `class_data_bits_t `存储了 `class_rw_t `的指针, 而 `class_rw_t `结构体又包含 `class_ro_t` 的指针

&emsp;&emsp;它里面的 `method_list_t` , `ivar_list_t` `property_list_t` 结构体都继承自 `entsize_list_tt<Element, List, FlagMAsk>`

&emsp;&emsp;`entsize_list_tt` 实现了 `non-fragile` 特性的数组结构体, 假如苹果在新版本的 SDK 中向 `NSObject` 类增加了一些内容, `NSObject` 的占据的内存区域会扩大, 开发者以前编译出的二进制中的子类就会与新的 `NSObject` 内存有重叠部分, 于是在编译期会给` instanceStart `和` instanceSize `赋值, 确定好编译时每个类所占内存区域起始偏移量和大小, 这样只需要将子类与基类的这两个变量做对比即可知道子类是否与基类有重叠, 如果有, 也可知道子类需要挪多少偏移量

&emsp;&emsp;`class_ro_t` 存储的大多是类在编译时就已经确定的信息
##### 	4.4 class_rw_t
&emsp;&emsp;`class_rw_t `提供了运行时对类拓展的能力, 但是如上文所说, `class_ro_t` 存储的大多是类在编译时已经确定的信息, 二者都存有类的方法, 属性(成员变量), 协议等信息, 不过存储他们的列表实现方式不同

&emsp;&emsp;`class_rw_t` 中使用的 `method_Array_t` , `property_array_t,` `protocol_Array_t` 都继承自 `list_array_tt<Element, List>`, 他可以不断扩张, 因为他可以存储 list 指针, 内容有三种
1. 空
2. 一个 `entsize_list_tt` 指针
3. ` entsize_list_tt` 指针数组

&emsp;&emsp;`class_rw_t` 的内容是可以在运行时被动态修改的, 可以说运行时对类的拓展大都存储在这里
```
struct class_rw_t{
	uint32_t flags;
	uint32_t version;
	
	const class_ro_t *ro;
	
	method_array_t methods;
	property_array_t properties;
	protocol_array_t protocols;
	... 其他
}
```
##### 4.5 realizeClass			
&emsp;&emsp;在某各类初始化之前, `objc_class->data() `返回的指针指向的其实是个` class_ro_t` 结构体, 等到 `static Class realizeClass((Class cls))`静态方法在类第一次初始化时被调用, 他会开辟` class_rw_t` 的空间, 并将` class_ro_t `指针赋值给 `class_rw_t->ro`, 这种偷天换日的行为是靠` RO_FUTURE` 标志位来记录的

&emsp;&emsp;经过 realizeClass 函数处理的类才是真正的类, 调用它时不能对类做些操作
#### 		5. Category
&emsp;&emsp;Category 为现有的类提供拓展性, 它是 category_t 结构体的指针

&emsp;&emsp;category_t 存储了类别中可以拓展的实例方法. 类方法 . 协议. 实例属性.和类属性
```
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```
&emsp;&emsp;在 App 启动加载镜像文件时, 会在 `_read_images` 函数间接调用到 `attachCategories` 函数, 完成向类中添加 `Category` 的工作, 原理就是向 `class_rw_t `中的 `method_array_t`, `property_array_t`, `protocol_array_t `数组中分别添加 `method_list_t`, `property_list_t` `protocol_list_t` 指针, 之前讲过 `xxx_array_t` 可以储存对应 `xxx_list_t` 的指针数组

&emsp;&emsp;在调用 `attachCategories` 函数之前, 会先使用 `unattachedCategoryesForClass` 函数获取类中还未添加的类别列表. 这个列表类型为 `locstamped_category_list_t,`他封装了 `category_t` 以及对应的 `header_info`, `header_info` 存储了实体在镜像中的加载和初始化状态, 以及一些偏移量, 在加载 Mach-O 文件相关函数中经常用到

&emsp;&emsp;所以更具体来说 `attachCategories` 做的就是将 `locstamped_category_list_t.list` 列表中每个` locstamped_category_t.cat` 中的那些方法, 协议, 和属性分别添加到类的 `class_rw_t` 对应列表中, `header_info `中的信息决定了是否是元类, 从而选择应该是添加实例方法还是类方法, 实例属性还是类属性等

&emsp;&emsp;我这里有一篇文章是学习 Category 的笔记 
[链接](https://supermutong.github.io/2017/12/17/Category%E7%AC%94%E8%AE%B0/)				
	 						 
#### 		6. Method 
&emsp;&emsp;`Method` 是代表类中的某个方法的类型
`Method` 的类型是 `method_t`
```
typedef struct method_t *Method
struct method_t {
	SEL name; 
	const char *types;
	IMP imp;
	}
```
&emsp;&emsp;方法名类型为 SEL, 前面提到过相同名字的方法即使在不同类中定义, 他们的方法选择器也相同.

&emsp;&emsp;方法类型 types 是个 char 指针, 其实存储着方法的参数类型和返回值类型.

&emsp;&emsp;imp 指向了方法的实现, 本质上是一个函数指针
#### 	7. Ivar 
&emsp;&emsp;`Ivar `代表类中实例变量的类型

&emsp;&emsp;类型是`ivar_t `
```
struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```
&emsp;&emsp;class_copyIvarList 函数获取的不仅有实例变量, 还有属性. 但会在原来的属性名前加一个下划线
```
- (void)getAllVars
unsigned int outCount = 0;
    Ivar *vars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar var = vars[i];
        const char *name = ivar_getName(var);
        NSString *key = [NSString stringWithUTF8String:name];
        
        // 注意kvc的特性是，如果能找到key这个属性的setter方法，则调用setter方法
        // 如果找不到setter方法，则查找成员变量key或者成员变量_key，并且为其赋值
        // 所以这里不需要再另外处理成员变量名称的“_”前缀
        id value = [self valueForKey:key]; 
           }
```
#### 8. objc_property_t 
&emsp;&emsp;`@property` 标记了类中的属性, 他是一个指向 `objc_property `结构体的指针

&emsp;&emsp;可以通过 `class_copyPropertyList` 和 `protocol_copyPropertyList `方法来获取类和协议中的属性, 返回类型为指向指针的指针, 因为属性列表是个数组, 每个元素内容都是一个 `objc_property_t `指针, 而这两个函数返回的值都是指向这个数组的指针
#### 	9. IMP  
&emsp;&emsp; `IMP` 在 `objc.h` 中的定义是
`typedef void (*IMP)(void /* id, SEK, ... */);`

&emsp;&emsp;他就是一个函数指针, 是由编译器生成的, 当你发起一个 Objc 消息后, 最终他会执行的那段代码, 就是由这个函数指针指定的, 而 IMP 这个函数指针就指向了这个方法的实现

## 	消息
&emsp;&emsp;Objc 中发送消息是用中括号 [] 把接受者和消息括起来, 而直到运行时才会把消息和方法实现绑定
### 	1.  objc_msgSend 函数
#### 1.1 消息发送步骤
1. 检测这个 `Selector `是不是要忽略的, MAC 开发中有个函数是要忽略的
2. 检测这个 `target` 是不是 `nil `对象, `Objc `的特性是允许对一个 `nil` 对象执行任何一个方法不会 `crash` , 因为被忽略掉了
3. 如果上面两个都过了, 那就开始查找这个类的 `IMP`, 先从 `cache `里面找, 如果找到就跳到对应的函数去执行
4. 如果 `cache `找不到就找一下方法分发表
5. 如果方法分发表中找不到就到超类的分发表去找, 一直找, 直到找到 `NSObject `类为止
6. 如果还找不到就要开始动态方法解析了
如图
![](https://ws3.sinaimg.cn/large/006tKfTcly1fnoddwbp64g30980f4aa3.gif)
&emsp;&emsp;其实编译器会根据情况在 `objc_msgSend`, `objc_msgSend_stret `,`objc_msgSendSuper` 或 `objc_msgSendSuper_strct` 四个方法中选择一个来调用, 如果消息时传递给超类, 那么会调用名字带有 'super'的函数, 如果消息返回值是数据结构而不是简单值是, 那么会调用名字带有'stret'的函数,带 "super" 的消失传递给超类;

&emsp;&emsp;"stret" 可分为"st" + "ret"两部分, 分别代表 "struct" 和 "return"; "fpret" 就是"fp" + "ret", 分别代表 "floating-point" 和 "return"
#### 1.2 方法中的隐藏参数
&emsp;&emsp;我们经常在方法中使用 self 关键字来引用实例本身, 但从没有想过为什么 self 能取到调用当前方法的对象吧, 其实 self 的内容是在方法运行时被偷偷动态传入的

&emsp;&emsp;当 objc_msgSend 找到方法对应的实现是, 他被直接调用该方法实现, 并将消息中所有的参数传递给方法实现, 同时, 他还传递两个隐藏的参数
1. 接受消息的对象 (也就是 `self `指向的内容)
2. 方法选择器(`_cmd` 指向的内容)

&emsp;&emsp;之所以说他们是隐藏的是因为在源代码方法的定义中并没有声明这两个参数, 但是在源代码中我们依然可以引用他们
### 2. 动态解析方法
&emsp;&emsp;`Runtime` 系统在 `Cache` 和方法分发表中(包括超类) 找不到要执行的方法的时候, Runtime 会调用 `resolveInstanceMethod: `或` resolveClassMethod: `来给程序员一次动态添加方法实现的机会, 我们需要用 `class_addMethod `函数完成向特定类添加特定方法实现的操作
1. 理解 `[self class]`  `object_getClass(self)` `object_getClass([self class])`
	1. 当 `self` 为实例对象时,  `[self class]` 与 `object_getClass(self)` 等价, 因为前者会调用后者, `object_getClass([self class])` 得到元类
	2. 当 `self` 为类对象时, `[self class]` 返回值为自身, 还是 `self`, `object_getClass(self)` 与 `object_getClass([self class])` 等价, 返回元类
				
### 	3. 消息转发  
#### 	3.1 重定向
&emsp;&emsp;在消息转发机制执行前, Runtime 系统会在给我们一次偷梁换柱的机会, 即通过重载 - (id)forwardingTargetForSelector:(SEL)aSelector 方法替换消息的接受者为其他对象
```
-(id)forwardingTargetForSelector:(SEL)aSelector{
	if (aSelector == @selector(mysteriousMthod:)){
		return alternateObject;
	}	
	return [super forwardingTargetForSelector:aSelector];
}
```
&emsp;&emsp;毕竟消息转发要耗费更多时间, 抓住这次机会将消息重定向给别人是一个不错的机会.  如果此方法返回 nil 或 self , 则会进入消息转发机制, 否则将向返回的对象重新发送消息

&emsp;&emsp;如果想替换类方法的接受者, 需要覆盖 `+ (id)forwardingTargetForSelector:(SEL)aSelector` 方法, 并返回类对象

&emsp;&emsp;一会咱们用 runtime 替换掉系统的 `forwardingTargetForSelector` 的方法, 来减少 `crash`
				
#### 3.2 转发 
&emsp;&emsp;当动态方法解析不作处理返回 NO 时, 消息转发机制会被触发, 在这时 `forwardInvocation: `方法会被执行, 我们可以重写这个方法来定义我们的转发逻辑
```
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([someOtherObject respondsToSelector:
            [anInvocation selector]])
        [anInvocation invokeWithTarget:someOtherObject];
    else
        [super forwardInvocation:anInvocation];
}
```
&emsp;&emsp;该消息的唯一参数是个 `NSInvocation` 类型的对象, 该对象封装了原始的消息和消息的参数. 我们可以实现 `forwardInvocation:` 方法来对不能处理的消息做一些默认的处理, 页可以将消息妆发给其他对象来处理, 而不抛出错误

&emsp;&emsp;在 `forwardInvocation:` 消息发送前, Runtime 系统会向对象发送 `methodSignatureForSelector `消息, 并取得返回的方法签名用于生成 `NSInvocation`对象, 所以我们在重写` forwardInvocation:`的同时也要重写 `methodSignatureForSelector:` 方法, 否则会抛异常

&emsp;&emsp;`forwardInvocation` 方法就像一个不能识别的消息的分发中心, 将这些消息转发给不同接收对象, 或者他也可以像一个运输站将所有的消息都发送给同一个接收对象, 他可以将一个消息翻译成另外一个消息, 或者简单的"吃掉"某些消息, 因此没有响应也没有错误, `forwardInvocation:` 方法也可以对不同的消息提供相同的响应, 这一切取决于方法的具体实现

&emsp;&emsp;注意:` forwardInvocation `方法只有在消息接收对象中无法响应消息的时候才会被调用, 所以我们希望一个对象将 ` A `消息转发给其他对象, 这个对象不能有` A `方法, 否则, `forwardInvocation` 将不可能被调用.
					
#### 3.3 转发和多继承
&emsp;&emsp;一个对象把消息转发出去, 就好似它把另一个对象的方法借过来或是"继承"过来一样

&emsp;&emsp;消息转发弥补了 Objc 不支持多继承的性质, 也避免了因为多继承导致单个类变得臃肿负责, 它将问题分解的很细, 只针对想要借鉴的方法才转发
#### 3.4 转发和继承
&emsp;&emsp;尽管转发很像继承, 但是` NSObject `类不会将两者混淆,, 像 `respondsToSelector: `和` isKindOfClass: `这类方法只会考虑继承体系, 不会考虑转发链

&emsp;&emsp;如果你想要上面的两个方法响应转发链, 就要重新他们了

					 
#### 3.5  消息转发的大致流程
![](https://ws3.sinaimg.cn/large/006tNc79ly1fmvlh7ubysj30m80h6dg5.jpg)
		
1. 进入`resolveInstanceMethod: `方法(动态方法解析), 指定是否动态添加方法. 若返回 NO, 则进入下一步, 若返回 YES , 则通过 `class_addMethod` 函数动态地添加方法, 消息得到处理, 此流程完毕
2. `resolveInstanceMethod: `方法返回 NO 时, 就会进入` forwardingTargetForSelector:` 方法(重定向), 这是 Runtime 给我们的第二次机会, 用于指定那个对象响应这个 selector, 如果返回 nil, 进入下一步, 返回某个对象, 则会调用该对象的方法
3. 若 `forwaringTargetForSelector:` 返回的是 nil, 则我们首先要通过 `methodSignatureForSelector:` 来指定方法签名, 返回 nil, 表示不处理, 若返回方法签名, 则会进入下一步
4. 当 `methodSignatureForSelector: `方法返回方法签名后, 就会调用 `forwardInvocation:` 方法(转发), 我们可以通过 `anInvocation` 对象做很多处理, 比如修改实现方法, 修改响应对象等
5. 如果到最后, 消息还是没有得到相应, 程序就会 crash, 
				
	
### Method Swizzling
&emsp;&emsp;当我们无法触碰到某各类的源代码, 但是又想更改这个类的某个方法的实现时, 就要用到 `Method Swizzling` 了, 下面举一例子, 这个例子是为了防止系统找不到调用方法 crash 的问题, 比如你添加一个 `UIButton`, 给` btn `添加 `selector`, 但是没有实现对应的 `selector`, 这个时候你点击 `btn`, 就会 crash. 为了防止类似的 crash ,我们用 `Method Swizzling` 重写 `forwardingTargetForSelector` 消息重定向, 下面开始贴代码
		首先, 我们先实现一个转发对象 `FakeForwardTargetObjct `
		
```
fakeIMP(id sender, SEL sel,...){
		    return nil;	
}
@implementation FakeForwardTargetObjct
	-(instancetype)initWithSelector:(SEL)aSelector{
		self = [super init];
			if (self) {
				if (class_addMethod([self class], aSelector, (IMP)fakeIMP, NULL)) 
					{
					     NSLog(@"add Fake Selector:[instance %@]",NSStringFromSelector(aSelector));
						}
				}
				return self;
@end
```	
	
接着实现转发逻辑
	
```
-implementation NSObject (safeSwizzle)
	+ (void)load{
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	        Class aClass = [self class];
	        SEL originalSelector = @selector(forwardingTargetForSelector:);
	        SEL swizzleSelector = @selector(safeForwardingTargetForSelector:);
	        
	        Method originalMethod = class_getInstanceMethod(aClass, originalSelector);
	        Method swizzleMethod = class_getInstanceMethod(aClass, swizzleSelector);
	
	        BOOL didAddMethod = class_addMethod(aClass, originalSelector, method_getImplementation(swizzleMethod), method_getTypeEncoding(swizzleMethod));
	        
	        if (didAddMethod) {
	            class_replaceMethod(aClass, swizzleSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
	        }
	        else{
	            method_exchangeImplementations(originalMethod, swizzleMethod);
	        }
	
	    });
	}
	
- (id)safeForwardingTargetForSelector:(SEL)aSelector{
	 NSMethodSignature *signature = [self methodSignatureForSelector:aSelector];
	  if ([self respondsToSelector:aSelector] || signature) {
	      return [self safeForwardingTargetForSelector:aSelector];
	   }
		 FakeForwardTargetObjct *tmp = [[FakeForwardTargetObjct alloc]initWithSelector:aSelector];
	    return tmp;
}
end
```
### 总结
&emsp;&emsp;终于把 Runtime 相关的文字大概看了一遍, 并形成了自己的笔记, 此博客大部分的内容都是来自[这里](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/) , 看完以后形成了自己的笔记, 加深印象, 在学习的过程中也写了一篇 Runtime 的[面试题](https://supermutong.github.io/2017/12/27/RunTime-%E9%9D%A2%E8%AF%95%E6%80%BB%E7%BB%93/)

&emsp;&emsp;自己写一遍只是为了加深下印象, 写的确实没有原博主们写得好

### 参考博客
[http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)
[http://www.jianshu.com/p/efeb33712445#](http://www.jianshu.com/p/efeb33712445#)
[http://www.jianshu.com/p/d63028fc978f](http://www.jianshu.com/p/d63028fc978f)
[http://blog.csdn.net/njafei/article/details/71172428](http://blog.csdn.net/njafei/article/details/71172428)
[https://njafei.github.io/2017/05/03/Method-SEL-IMP/](https://njafei.github.io/2017/05/03/Method-SEL-IMP/)
[http://www.code4app.com/blog-822715-1562.html](http://www.code4app.com/blog-822715-1562.html)
[https://www.jianshu.com/p/ab966e8a82e2](https://www.jianshu.com/p/ab966e8a82e2)

