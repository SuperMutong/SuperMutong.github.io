---
layout:     post
title:      iOS 之 ARC,MRC 以及 AutoReleasePool
subtitle:   RunTime
date:       2017-9-16
author:     Mutong
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - 基础
    - ARC
    - MRC
---


## iOS 之 ARC,MRC 以及 AutoReleasePool

## 前言

这篇文章我猜测篇幅会很长, 因为涉及到了ARC, MRC, AutoReleasePool 以及内存分配

## iOS 程序中的内存分配
首先先讲下内存分区, 无论是 ARC 还是 MRC 都是对内存的管理, 你需要先了解下内存分区, 知道哪些分区可以人工管理,哪些分区是系统管理

不同的类型的数据, 保存的内存分区不同, 可以分为五部分: 栈区, 堆区, 全局区, 常量区, 代码区

#### 栈区
栈区是由编译器自动分配并释放, 存放函数的参数值, 局部变量等. 栈是系统数据结构, 对应的线程/进程是唯一的
    
优点:快速高效, 缺点:数据不灵活

#### 堆区
堆区是有程序员分配和释放, 如果程序员不释放, 程序结束时, 可能会由操作系统统一回收, 比如 `alloc` 出来的对象放在堆中, 但是对象的指针放在栈中, OC 的内存管理主要是对`堆区中的 OC 对象`进行管理(并不是所有的对象都存在堆区, 后面会详细解说)

优点: 灵活方便, 数据适应面广泛, 但是效率有一定降低

#### 全局区/静态区
全局变量和静态变量的存储是放在一起的, 初始化的全局变量和静态变量存放在一块区域, 未初始化的全局变量和静态变量在相邻的另一块区域, 程序结束后由系统释放

注意, 全局区可分为未初始化全局区: .bss段 和 初始化全局区: data 段

举例:

```objc
    int a = 10; //全局初始化区
    char *p; //全局为初始化区
    main {
        int b; //栈区
        char s[] = "abc"; //栈区
        char *p1; //栈区
        char *p2 = "1231"; // p2 在栈上, 1231 在常量区
        static int c = 0; //全局(静态)初始化区
        NSObject *p = [[NSObject alloc]init]; // p 在栈上, 后面的那个对象在堆上

        //这个字符串老复杂了, 根据字符串长度编译器会自动优化成三种类型,根据这三种类型在判断存储常量区还是栈上, 看最下面博客参考[NSString]
        NSString *str = @"123";

        //苹果在2013年引入了`Tagged Pointer`对 NSNumber 和 NSDate 做了优化,不受引用计数影响,可以看下最下方唐巧的博客关于`Tagged Pointer`的介绍
        NSNumber *number = [NSNumber alloc]init];
    }
```

#### 常量区
存放常量字符串, 程序结束后由系统释放

#### 代码区
存放程序的二进制代码

内存分配图

![](https://ws3.sinaimg.cn/large/006tNc79ly1fovy4thp7yj307w0baa9y.jpg)

## MRC
OC内存管理有两种模式: ARC 和 MRC, 咱们先从 MRC 开始

MRC:Manual Reference Counting 手动引用计数

在2011年, iOS5 以前, OC 的开发只支持 MRC 模式, 需要程序员自己去管理内存, 可能一不小心就会内存泄露, 造成僵尸对象和野指针

```
关于内存释放的本质

当一块内存释放的时候, 本质上只是给这部分字节打了标签, 并没有把字节里的二进制数据全部清成0或者1,所以当一块内存说是释放了, 但如果没有其他的二进制数据去填充它, 那么它的内部的数据是一直存在的. 内存释放, 数据仍然存在, 就会出现一种僵尸对象的问题.

僵尸对象:只要一个对象被释放了, 我们就称这个对象为僵尸对象(不能再次被当前指针使用的对象) 堆空间已经被标记清空, 能别其他数据使用, 但此时此刻, 新的二进制数据还没有进来, 这个堆空间就是僵尸对象

野指针: 当一个指针指向一个僵尸对象, 我们就称这个指针为野指针, 只要给一个野指针发送消息就会报错( EXC_BAD_ACCESS 错误)

为了避免给野指针发送消息报错, 一般情况下, 当一个对象被释放后我们将这个对象的指针设置为空指针

空指针: 没有指向存储空间的指针(里面存的是 nil), 向空指针发消息是没有任何反应的
```

MRC 的原则是谁用谁 `retain`/`alloc`/`new`, 谁不用谁`release`

上段代码

```objc
NSObject *p = [NSObject alloc]init]; //alloc 以后, p的引用计数已经变为1了, 默认就是1
...
//p 不用了, 需要 release
[p release];

//如果这个时候你调用 p , 就会报错 `EXC_BAD_ACCESS` 野指针错误
NSLog(p.class);

//当一块堆空间被标记可以删除后, 即使暂时没有新的数据填充这块空间, 原来空间的数据也不能"复活"过来重新使用
//也会报野指针的错误
[p retain];

```

### retain
既然聊到了`retain`和`release`, 那么就讲下`retain`和`release`的实现细节, 从`retain`引用计数加一开始

下面是`retain`方法的调用栈

```objc
- [NSObject retain]
└── id objc_object::rootRetain()
    └── id objc_object::rootRetain(bool tryRetain, bool handleOverflow)
        ├── uintptr_t LoadExclusive(uintptr_t *src)
        ├── uintptr_t addc(uintptr_t lhs, uintptr_t rhs, uintptr_t carryin, uintptr_t *carryout)
        ├── uintptr_t bits
        │   └── uintptr_t has_sidetable_rc  
        ├── bool StoreExclusive(uintptr_t *dst, uintptr_t oldvalue, uintptr_t value)
        └── bool objc_object::sidetable_addExtraRC_nolock(size_t delta_rc)                
            └── uintptr_t addc(uintptr_t lhs, uintptr_t rhs, uintptr_t carryin, uintptr_t *carryout)
```

这里面的`id objc_object::rootRetain(bool tryRetain, bool handleOverflow)`方法是调用栈中最重要的方法, 其原理就是将`isa`结构体中的`extra_rc`的值加1

`extra_rc` 用于保存<kbd>额外</kbd>的自动引用计数的标志位, 下面是`isa`结构体中的结构, `isa`是在`objc_object`这个结构体, 在学习`Runtime`的时候学习过

下面是`isa_t`结构体的结构:
![](https://img.draveness.me/2016-05-27-objc-rr-isa-struct.png-1000width)

接下来我们会分三种情况对`rootRetain`进行分析
* 正常的rootRetain
* 有进位版的rootRetain
* 有进位版本的rootRetain(处理移除)

#### 正常的 rootRetain
这个是简化后的`rootRetain`方式的实现, 其中只处理一般情况的代码

```objc
id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);
    } while (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits));

    return (id)this;
}
```

这种case是加色`extra_rc`的位数足以存储`retainCount`

- 使用`LoadExclusive`加载`isa`的值
- 调用`addc(newisa.bits,RC_ONE,0,&carry)`方法将`isa`的值加一
- 调用`StoreExclusive(&isa.bits,oldisa.bits,newisa.bits)`更新`isa`的值
- 返回当前对象

#### 有进位版本的rootRetain

在这里调用`addc`方法为`extra_rc`加一时, 8位的`extra_rc`可能不足以保存引用计数.
```objc
id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    transcribeToSideTable = false;
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;

    uintptr_t carry;
    newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);

    if (carry && !handleOverflow)
        return rootRetain_overflow(tryRetain);
}
```
当方法传入的`handleOverflow == false`时(这个也是通常情况), 我们会调用`rootRetain_overflow`方法:
```objc
id objc_object::rootRetain_overflow(bool tryRetain) {
    return rootRetain(tryRetain, true);
}
```
这个方法其实就是重新执行`rootRetain`方法, 并传入`handleOverflow = true`

#### 有进位版本的 rootRetain (处理溢出)

当传入的handleOverflow = true 时, 我们就会在`rootRetain`方法中处理引用计数的溢出
```objc
id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    bool sideTableLocked = false;

    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);

        if (carry) {
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits));

    sidetable_addExtraRC_nolock(RC_HALF);

    return (id)this;
}
```

当调用这个方法, 并且`handleOverflow = true`时, 我们就可以确定`carry`一定是存在的了, 因为`extra_rc`已经溢出了, 所以要更新他的值为`RC_HALF`
```objc
#define RC_HALF (1ULL<<7)
```
然后设置`has_sidetable_rc`为真, 存储新的`isa`的值以后, 调用`sidetable_addExtraRC_noLock`方法

```objc
bool objc_object::sidetable_addExtraRC_nolock(size_t delta_rc) {
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt =
        addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage = SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    } else {
        refcntStorage = newRefcnt;
        return false;
    }
}
```
这里我们将溢出的一位`RC_HALF`添加到`oldRefcnt`中, 其中的各种`SIDE_TABLE`宏定义如下:
```objc
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)
#define SIDE_TABLE_RC_ONE            (1UL<<2)
#define SIDE_TABLE_RC_PINNED         (1UL<<(WORD_BITS-1))

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)
```

因为`refcnts`中的64位的最低两位是有意义的标志位, 所以在使用`addc`时要将`delta_rc`左移两位,获得一个新的引用计数`newRefcnt`

如果这时出现了溢出, 那么就会撤销这次的行为, 否则, 会将新的引用计数存储到`refcntStorage`指针中

也就是说, 在 iOS的内存管理中, 我们使用了`isa`结构体中的`extra_rc`和`SideTable`来存储某个对象的自动引用计数

另外`extra_rc`保存的是额外的引用计数, 当一个对象的引用计数为1, 那么`extra_rc`实际上是0, 苹果这么做可能是能够减少很多不必要的函数调用吧

### release

先看下方法简化后的调用栈

```objc
- [NSObject release]
└── id objc_object::rootRelease()
    └── id objc_object::rootRetain(bool performDealloc, bool handleUnderflow)
```

`retain`有三种case, 理论上`release`也会有三种对应的方法

先看下最简单的

#### 正常的release

```objc
bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);
    } while (!StoreReleaseExclusive(&isa.bits, oldisa.bits, newisa.bits));

    return false;
}
```

- 使用`loadExclusive`获取`isa`内容
- 将`isa`中的引用计数减一
- 调用`StoreReleaseExclusice`方法保存新的`isa`

#### 从 SideTable 借位

```objc
bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);
        if (carry) goto underflow;
    } while (!StoreReleaseExclusive(&isa.bits, oldisa.bits, newisa.bits));

    ...

 underflow:
    newisa = oldisa;

    if (newisa.has_sidetable_rc) {
        if (!handleUnderflow) {
            return rootRelease_underflow(performDealloc);
        }

        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF);

        if (borrowed > 0) {
            newisa.extra_rc = borrowed - 1;
            bool stored = StoreExclusive(&isa.bits, oldisa.bits, newisa.bits);

            return false;
        }
    }
}
```
这里也有一个`handleUnderflow`, 与`retain`中的相同, 如果发生了`underflow`, 会重新调用该`rootRelease`方法, 并传入`handleUnderFlow = true`

在调用`sideTable_subExtraRC_nolock`成功借位之后, 我们会重新设置`newisa`的值`newisa.extra_rc = borrowed-1`并更新`isa`

#### dealloc

如果在`SideTable`中也没有获取到借位的话, 就说明没有任何的变量引用了当前对象(即retainCount = 0), 就需要向她发送`dealloc`消息了

```objc
bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    isa_t oldisa;
    isa_t newisa;

 retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);
        if (carry) goto underflow;
    } while (!StoreReleaseExclusive(&isa.bits, oldisa.bits, newisa.bits));

    ...

 underflow:
    newisa = oldisa;

    if (newisa.deallocating) {
        return overrelease_error();
    }
    newisa.deallocating = true;
    StoreExclusive(&isa.bits, oldisa.bits, newisa.bits);

    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;
}
```
上述代码会直接调用`objc_msgSend`向当前对象发送`dealloc`消息

不过为了确保消息只会发送一次, 系统使用了`deallocating`标记位

#### 获取自动引用计数

我们来看下`retainCount`方法的实现
```objc
- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}

inline uintptr_t objc_object::rootRetainCount() {
    isa_t bits = LoadExclusive(&isa.bits);
    uintptr_t rc = 1 + bits.extra_rc;
    if (bits.has_sidetable_rc) {
        rc += sidetable_getExtraRC_nolock();
    }
    return rc;
}
```
看实现可以发现, retainCount有三部分组成
- 1
- extra_rc 中存储的值
- sideTable_getExtraRC_nolock返回的值

这也就证明了我们之前得到的结论


## ARC

ARC: Automatic Reference Counting 自动引用计数

自动维护引用计数, 并释放当前堆空间. 不需要程序员去关心内存释放问题.

iOS5 推出的功能, ARC 所做的无非就是在合适的时机(应该是编译期的某个时机,我还没有发现)帮我们添加,`retain`,`release`或者`autoRelease`, 无需程序员手动敲这些代码, 极大的节省了时间, 并且避免了因忘记释放造成的内存泄露, 但是从本质上讲, 它与 MRC 还是无异的

### 释放原则

在 ARC 中,所有实例变量和局部变量默认情况下都是`strong`类型, 类似于 MRC 中的`retain`,只要某个对象被任一`strong`指针指向, 那么它都不会被销毁, 相反的, 如果对象没有被任何`strong`类型的指针指向, 那么就将被销毁. 与`strong`相对是`weak`, `weak`类型的指针可以指向对象, 但是不会只有该对象, 当对象被销毁, `weak`会自动指向 nil, weak 类型的指针一般用在两个对象之间存在包含关系时, 常见的就是 `delegate`模式.

ARC 判断一个对象是否需要释放不是通过引用计数来判断的, 而是通过`强指针`来进行判断

* 强指针
    - 默认所有对象的指针都是强指针
    - 被 `__strong` 修饰的指针
* 弱指针
    - 被 `__weak` 修饰的指针

ARC 创建的对象默认都是`storng`, 如果你在创建对象的时候用的是`weak`, 则创建完就消失了

```objc
//这里举例用的是 NSObject , 最好别用其他的变量, 因为NSObjct对象肯定是保存在堆上, 完全受ARC影响, 如果用 NSString 和 NSNumber, 有的时候运行结果和猜测不一样, 具体原因请看最上面的内存分配
__weak NSObject *p = [NSObject alloc]init];
// p会立即释放 这个时候打印 p 直接就是 nil
NSLog(p)
```


即使有了 ARC 也不能解决所有问题, ARC 只适用 OC 对象(还得去除 NSDate NSNumber 部分 NSString), 对于底层的东西, 例如 `Core Foundation` 就不在 ARC 的管辖范围内, 为了做好两者的链接, 需要使用三个关键字, __bridge __bridge_retained __bridge_transfer:

* __bridge 
    只做类型转换, 不修改内存管理权
* __bridge_retained
    将 OC 的对象转换成 CF 对象, 同时将对象内存的管理权交给`CFRelease`
* __bridge_transfer
    将 CF 对象转换为 OC 对象, 同时将管理权交给 ARC


## 自动释放池

### AutoReleasePool

#### 用途

存储在自动释放池的对象, 在自动释放池销毁时, 会自动调用该对象的`release`方法, 如果引用计数为0, 会自动调用`dealloc`方法, 不需要用户自己写 `release`

#### 原理

自动释放池是由多个`AutoreleasePoolPage`组成的`双向链表`, 其中主要通过`push`和`pop`操作来管理

#### 底层实现

* 我们先用`clang` 把`main.m`转成`c++`文件, 里面可以看到`autoreleasePool`的源码

首先我们看到`@autoreleasePool`这个block 转换成了下面的这行代码`__AtAutoreleasePool __autoreleasepool;`
```objc
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```

那`AtAureleasePool` 又是什么呢, 一步步找,找到了下方方法, `__AtAutoreleasePool` 是一个结构体, 其中包含一个构造函数以及一个析构函数

```objc
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);

struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

这个结构的构造函数会调用`objc_autoreleasePoolPush()` 并返回一个`atautoreleasepoolobj`对象, 并其析构函数, 会将`atautoreleasepoolobj`对象作为`objc_autoreleasePoolPop()`的入参.

其实`main.m`中的方法其实就是这样
```objc
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();

        //insert code

        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

这两个函数的实现如下, 后面我们会详细解析这这两个核心函数

```objc
void *objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
```

```objc
void objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt)
}
```

上面的两个方法分别调用的是`AutoreleasePoolPage`的`push`和`pop`这两个静态方法

在此往下挖, `AutoreleasePoolPage`是啥?


#### AutoreleasePoolPage

先看下`NSObject.mm`中官方解释

```
Autorelease pool implementation
A thread's autorelease pool is a stack of pointers. 
Each pointer is either an object to release, or POOL_BOUNDARY which is 
 an autorelease pool boundary.
A pool token is a pointer to the POOL_BOUNDARY for that pool. When 
 the pool is popped, every object hotter than the sentinel is released.
The stack is divided into a doubly-linked list of pages. Pages are added 
 and deleted as necessary. 
Thread-local storage points to the hot page, where newly autoreleased 
 objects are stored.
```

汉化版:

* 每个线程的`autorelease pool`是一个指针的堆栈
* 每个指针不是指向一个需要`release`的对象, 就是指向一个`POOL_BOUNDARY`(哨兵对象, 表示一个`autorelease pool`的边界, 一会会详细解释)
* 一个 `pool token`指向这个`POOL_BOUNDARY`, 当这个`pool`被`pop`的时候, 这个哨兵对象后面添加的那些结点都会被`release`
* 这个堆栈(即autorelease pool)是一个以`page`为结点的双向链表, 这个`page`会在必要的时候增加或删除
* Thread-local stroage (TLS,即线程局部存储)指向 `hot page`, 这个`hot page`是指最新添加的`autorelease`对象所在的那个`page` 

下面说一下 `AutoreleasePoolPage`这个类的成员变量部分代码

```objc
class AutoreleasePoolPage 
{
    // EMPTY_POOL_PLACEHOLDER is stored in TLS when exactly one pool is 
    // pushed and it has never contained any objects. This saves memory 
    // when the top level (i.e. libdispatch) pushes and pops pools but 
    // never uses them.
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)
#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
}
```

画个图明显点, 一会我们会一一介绍里面的每个变量

![](https://ws1.sinaimg.cn/large/006tKfTcly1fp1lxrke4nj30aj0c3dfy.jpg)

* `magic` 用来检验`AutoreleasePoolPage`的结构是否完整
* `next`指向最新添加的`autoreleased`对象的下一个位置, 初始化时指向`begin()`;
* `thread`指向当前页所在的线程
* `parent`指向父节点, 第一个节点的`parent`值为`nil`
* `child`指向子节点, 最后一个结点的`child`值为`nil`
* `depth`代表深度, 从`0`开始, 往后递增`1`
* `hiwat`代表`high water mark`, 表示入栈最多时候的指针个数

通过上图的定义可以看到, 一个`page`会开辟`PAGE_MAX_SIZE`的内存(以前的版本是`4096 bytes`, 现在可能会根据不同设备及系统分配不同的内存), 除了`AutoreleasePoolPage`的成员变量所占空间(共`56 bytes`), 其余空间将会用来存储加入到自动释放池的对象

初始的`next == begin()`, 新加入自动释放池的一个对象, 会存放在当前`next`指向的位置, 当对象存放完成后, `next`指针会指向下一个为空的地址

当`next == end()`时, 表示当前`page`已经满了, 如果在加入一个对象需要创建`child page`

#### objc_autoreleasePoolPush

先上一张图(红色部分表示push后会变化的东西), 接下来在详细说下其流程
![](https://ws2.sinaimg.cn/large/006tKfTcly1fp20or98jxj30sj0kfwet.jpg)

`objc_autoreleasePoolPush`的函数定义如下:
```c++
void *objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
```

`push`的定义如下

```c++
static inline void *push() 
{
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
```

这里会调用`autoreleaseFase(POOL_BOUNDARY)`操作, 定义如下

```c++
static inline id *autoreleaseFast(id obj)
{
    //获取当前正在使用的page
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```

这里有三种情况

* `hotPage` 存在并且还没有满
    - 调用`page-add(obj)`方法将对象加入该`hotPage`中
* `hotPage`满了
    - 调用`autoreleaseFullPage(obj,page)`方法, 该方法会先查找`hotPage`的`child`, 如果有则将`child page`设置为`hotPage`, 如果没有则将创建一个新的`hotPage`,之后在这个新的`hotPage`上执行`page->add(obj)`操作
* hotPage 不存在
    - 调用`autoReleaseNoPage(obj)`方法, 改方法会创建一个`hotPage`, 然后执行`page->add(obj)`操作

再来看看`add`方法的定义

```c++
id *add(id obj)
{
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}
```

`add`会把`obj`存放在原本`next`所在的位置, 然后`next`指针++移动到下一个位置

在看下当前`hotPage`已满

`autoreleaseFullPage`会在当前的`hotPage`已满的时候调用

```
static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page) {
    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```
他会从传入的`page`开始遍历整个双向链表, 直到:
* 查找到一个未满的`AutoreleasePoolPage`
* 使用析构器传入`parent`创建一个新的`AutoreleasePoolPage`

在查到到一个可以使用的`AutoreleasePoolPage`之后, 会将改页面标记成`hotPage`, 然后调用上面分析过的`page->add`方法添加对象

如果当前没有`hotPage`

如果当前内存中不存在`hotpage`,就会调用`autoreleaseNoPage`方法初始化一个`AutoreleasePoolPage`

```
static id *autoreleaseNoPage(id obj) {
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);

    if (obj != POOL_SENTINEL) {
        page->add(POOL_SENTINEL);
    }

    return page->add(obj);
}
```
既然当前内存中不存在`AutoreleasePoolPage`, 就要从头开始构建这个自动释放池的双向链表, 也就是说, 新的`AutoreleasePoolPage`是没有`parent`指针的

初始化之后, 将当前页标记为`hotPage`, 然后会先向这个`page`中添加一个`POOL_BOUNDARY`对象, 在`pop`的时候回用到

最后, 将`obj`添加到自动释放池中

接着看`autorelease`方法, 同样也是会调用'autoreleaseFast(obj)'方法

```c++
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
```

总结: 调用`objc_autoreleasePoolPush`方法时, 这个方法首先在当前`next`指向的位置`add`一个`POOL_BOUNDARY`, 然后当向一个对象方法`autorelease`消息,就会把该对象`add`进`page`里`POOL_BOUNDARY`后面, 之后`next`指向刚插入的位置的下一个内存地址. 当当前`page`快满的时候(即`next`即将指向栈顶`end()`位置), 说明这一页`page`快满了, 如果这个时候再加入一个对象, 会先建立下一页`page`, 双向链表建立完成后, 新的`page`的`next`指向该页的栈底`begin()`位置, 然后继续向栈顶添加新的指针, 周而复始.

#### POOL_BOUNDARY (哨兵对象)

前面提到了`POOL_BOUNDARY`, 那它到底是什么, 以及为什么在栈中

其实`POOL_BOUNDARY`只是`nil`的别名, 请看源代码
```objc
#define POOL_SENTINEL nil
```

在每个自动释放池初始化调用`objc_autoreleasePoolPush`的时候, 都会把一个`POOL_BOUNDARY` push 到自动释放池的栈顶, 并返回这个`POOL_BOUNDARY`哨兵对象, 

在讲解`main.m`函数的时候, 调用`objc_autoreleasePoolPush`会返回一个`atautoreleasepoolObj`对象, 这个对象就是一个`POOL_BOUNDARY`, 在调用`objc_autoreleasePoolPop`的时候要把这个哨兵对象传入, 下面详细解释下`objc_autoreleasePoolPop`

#### objc_autoreleasePoolPop

方法定义:
```c++
void objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

静态方法`pop(ctxt)`(`ctxt`一般是前面`push`后返回的哨兵对象, 也可以传入任何一个指针, ), 接着看下 `pop`方法
```c++
static inline void pop(void *token) 
{
    AutoreleasePoolPage  *page = pageForPointer(token);
    id *stop = (id *)token;
    
    page->releaseUntil(stop);
    
    if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        } else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

其中`pageForPointer(tojen)`会获取哨兵对象所在`page`

```c++
static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
{
    AutoreleasePoolPage *result;
    uintptr_t offset = p % SIZE;
    assert(offset >= sizeof(AutoreleasePoolPage));
    result = (AutoreleasePoolPage *)(p - offset);
    result->fastcheck();
    return result;
}
```

主要是通过指针与`page`大小取余得到其偏移量(因为所有的`AutoreleasePoolPage`在内存中都是对其的), 最后通过`fastCheck()`方法检测得到的是不是一个`AutoreleasePoolPage`

之后调用`releaseUntil`循环释放对象, 其定义如下:

```c++
void releaseUntil(id *stop) 
{
    while (this->next != stop) {
        // Restart from hotPage() every time, in case -release 
        // autoreleased more objects
        AutoreleasePoolPage *page = hotPage();
        // fixme I think this `while` can be `if`, but I can't prove it
        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }
        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();
        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }
    setHotPage(this);
}
```

`releaseUntil`方法会先把`next`指针向前移动, 取到将要释放的一个指针, 之后调用`memset`擦除该指针所占内存, 在调用`objc_release`方法释放该指针指向的对象, 这样通过`next`指针循环往前查找要释放对象, 期间可往前跨越多个`page`, 直到找到传进来的哨兵对象为止.

当有嵌套的`autoreleasePool`时, 会清除一层后在清除另一层, 因为`pop`是会释放到上次`push`的位置为止, 和剥洋葱一张, 每层一次, 互不影响

最后如果传入的哨兵对象所在的`page`有`child`, 有两种情况
* 当前`page`使用不满一半, 从`child page`开始将后面所有的`page`删除
* 当前`page`使用超过一半, 从`child page`的`child page`(即孙辈)开始将后面所有的`page`删除

```c++
if (page->child) {
    // hysteresis: keep one empty child if page is more than half full
    if (page->lessThanHalfFull()) {
        page->child->kill();
    } else if (page->child->child) {
        page->child->child->kill();
    }
}
```

至于为什么会分这两种情况, 有人猜测可能是空间换时间吧, 当使用超过一半时, 当前`page`可能很快就用完了, 所以将`child page`留着, 减少创建新`page`的开销

`kill()`方法会将后面所有的`page`都删除
```c++
void kill() 
{
    // Not recursive: we don't want to blow out the stack 
    // if a thread accumulates a stupendous amount of garbage
    AutoreleasePoolPage *page = this;
    while (page->child) page = page->child;
    AutoreleasePoolPage *deathptr;
    do {
        deathptr = page;
        page = page->parent;
        if (page) {
            page->unprotect();
            page->child = nil;
            page->protect();
        }
        delete deathptr;
    } while (deathptr != this);
}
```

先释放指针,在删除`page`

总结: 调用完前面说的`objc_autoreleasePoolPush`后, 会返回一个`POOL_BOUNDARY`的地址, 当释放池要释放的时候, 会调用`objc_autoreleasePoolPop`函数, 将`POOL_BOUNDARY`作为其入参, 然后会执行如下操作
* 根据传入的`POOL_BOUNDARY`找到其所在的`page`
* 从`hotPage`的`next`指针开始往前查找, 向找到的每个指针调用`memset`方法以擦除指针所占内存, 在调用`objc_release`方法释放该指针指向的对象, 直到前一步所找到的`page`的`POOL_BOUNDARY`位置,期间可能跨越多个`page`, 并且在释放前, `next`指针也会往回指向正确的位置
* 如果有`子page`会`kill`掉`子page`

当有嵌套的`autoreleasePool`时, 会清除一层厚在清除另一层, 因为`pop`是会释放到上次`push`的位置位置, 和剥洋葱类似, 一层一层


#### 自动释放池嵌套使用

* 自动释放池以栈的形式存在
* 由于栈只有一个入口, 所以调用`autorelease`会将对象放到栈顶的自动释放池
* 自动释放池中不宜放占用内存比较大的对象

#### 自动释放池和线程的关系

每个线程(包括主线程)包含一个他自己的自动释放池的栈, 作为一个新的被创建的池子, 他们被添加到栈的顶部. 当池子被释放的时候, 他们从栈中被移除. 自动释放的对象被放在当前线程的自动释放池的顶部(自动释放池也是个栈) 当一个线程终止的时候, 它自动清空与他关联的所有的自动释放池

#### 自动释放池的应用

* 当我们的应用有需要创建大量的临时变量的时候, 可以使用`@autoreleasepoll`来减少内存峰值

下面这个`for`循环, 加和不加`autoreleasepool`的差距非常大, 加了内存稳定以后就是一个均值, 没有加的话内存会一直攀升

```objc
    for (int i =0 ; i<NSIntegerMax; i++) {
//        @autoreleasepool{
        NSString *Str = [NSString stringWithFormat:@"自动释放池"];
//        }
    }
```

* 有些变量是系统自动给我们加了`aotureleasepool`

我当前测试的系统是 ios11, 使用数组的block版本的枚举,系统内部都会自动添加一个`autoreleasePool`:
```objc
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    // 这里被一个局部@autoreleasepool包围着
}];
```


* 自己创建辅助线程

暂时还没有遇到这种case


### autorelease

理解了自动释放池, 说下`autorelease`, `autorelease`是一种支持引用计数的内存管理方式, 只要给对象发送一条` autorelease`消息, 会将对象放到一个自动释放池中, 当自动释放池被销毁时, 会对池子里的所有对象发送`release`操作, 如果当前引用计数为0, 就会调用当前对象的`dealloc`方法

#### autorelease 好处
* 不用关心对象释放的时间
* 不用关心什么时候调用`release`

#### autorelease 原理实质

`autorelease` 实际上只是把对`release`的调用延迟了, 当对一个对象添加`autorelease`, 只是把该对象放入了当前的`autorelease pool`, 当该`pool`释放时, 该`pool`中的所有对象都会被调用`release`

#### autorelease 方法的实现

这里面很多方法下面都会详细解释
```
- [NSObject autorelease]
└── id objc_object::rootAutorelease()
    └── id objc_object::rootAutorelease2()
        └── static id AutoreleasePoolPage::autorelease(id obj)
            └── static id AutoreleasePoolPage::autoreleaseFast(id obj)
                ├── id *add(id obj)
                ├── static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
                │   ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                │   └── id *add(id obj)
                └── static id *autoreleaseNoPage(id obj)
                    ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                    └── id *add(id obj)
```

在`autorelease`方法的调用栈中, 最终都会调用`autoreleaseFast`方法, 将当前对象加到`AutoreleasePoolpage`中


#### autorelease对象在什么时候释放

有无显示的创建`autoreleasePool`为准分为两种
* 显示使用`@autoreleasePool`, 会在大括号结束的时候释放
* 没有显示的创建`@autoreleasePool`, 由系统自己创建,系统会自动释放, 释放实际在当前`runloop`准备进入休眠(`BeforeWaiting`)的时候调用`pop`和`push`来释放旧的池子并创建新的池子


## 参考博客
* [http://aevit.xyz/2017/03/12/iOS-autorelease/](http://aevit.xyz/2017/03/12/iOS-autorelease/)
* [https://draveness.me/rr](https://draveness.me/rr)
* [https://draveness.me/autoreleasepool](https://draveness.me/autoreleasepool)
* [https://www.jianshu.com/p/5559bc15490d](https://www.jianshu.com/p/5559bc15490d)
* [http://blog.sunnyxx.com/2014/10/15/behind-autorelease/](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
* [http://tutuge.me/2015/03/17/what-is-autoreleasepool/](http://tutuge.me/2015/03/17/what-is-autoreleasepool/)
* [https://www.jianshu.com/p/554c9fe0f041](https://www.jianshu.com/p/554c9fe0f041)
* [https://bujige.net/blog/iOS-Memory-management.html](https://bujige.net/blog/iOS-Memory-management.html)
* [http://blog.csdn.net/liangge013/article/details/45150625](http://blog.csdn.net/liangge013/article/details/45150625)
* [https://www.jianshu.com/p/f3c1b920e8eb](https://www.jianshu.com/p/f3c1b920e8eb)
* [tagged-pointer](http://blog.devtang.com/2014/05/30/understand-tagged-pointer/)
* [NSString](http://blog.csdn.net/Lotheve/article/details/52035477)