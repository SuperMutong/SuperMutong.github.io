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

MRC 的原则是谁用谁 `retain`/`alloc`/`new`, 谁不用谁release

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

## ARC

ARC: Automatic Reference Counting 自动引用计数

自动维护引用计数, 并释放当前堆空间. 不需要程序员去关心内存释放问题.

iOS5 推出的功能, ARC 所做的无非就是在合适的时机帮我们添加,`retain`,`release`或者`autoRelease`, 无需程序员手动敲这些代码, 极大的节省了时间, 并且避免了因忘记释放造成的内存泄露, 但是从本质上讲, 它与 MRC 还是无异的

在 ARC 中,所有实例变量和局部变量默认情况下都是`strong`类型, 类似于 MRC 中的`retain`,只要某个对象被任一`strong`指针指向, 那么它都不会被销毁, 相反的, 如果对象没有被任何` Strong`类型的指针指向, 那么就将被销毁. 与`strong`相对是`weak`, `weak`类型的指针可以指向对象, 但是不会只有该对象, 当对象被销毁, `weak`会自动指向 nil, weak 类型的指针一般用在两个对象之间存在包含关系时, 常见的就是 `delegate`模式

ARC 创建的对象默认都是'storng', 如果你在创建对象的时候用的是'weak', 则创建完就消失了

```objc
__weak NSObject *p = [NSObject alloc]init];
// 这个时候打印 p 直接就是 nil
```

即使有了 ARC 也不能解决所有问题, ARC 只适用 OC 对象(还得去除 NSDate NSNumber 部分 NSString), 对于底层的东西, 例如 `Core Foundation` 就不在 ARC 的管辖范围内, 为了做好两者的链接, 需要使用三个关键字, __bridge __bridge_retained __bridge_transfer:

* __bridge 
    只做类型转换, 不修改内存管理权
* __bridge_retained
    将 OC 的对象转换成 CF 对象, 同时将对象内存的管理权交给`CFRelease`
* __bridge_transfer
    将 CF 对象转换为 OC 对象, 同时将管理权交给 ARC


## 自动释放池

### autorelease

先说下 `autorelease`, `autorelease`是一种支持引用计数的内存管理方式, 只要给对象发送一条` autorelease`消息, 会将对象放到一个自动释放池中, 当自动释放池被销毁时, 会对池子里的所有对象发送`release`操作, 如果当前引用计数为0, 就会调用当前对象的`dealloc`方法

#### autorelease 好处
* 不用关心对象释放的时间
* 不用关心什么时候调用`release`

#### autorelease 原理实质

`autorelease` 实际上只是把对`release`的调用延迟了, 当对一个对象添加`autorelease`, 只是把该对象放入了当前的`autorelease pool`, 当该`pool`释放时, 该`pool`中的所有对象都会被调用`release`



### AutoReleasePool

#### 原理

存储在自动释放池的对象, 在自动释放池销毁时, 会自动调用该对象的`release`方法, 如果引用计数为0, 会自动调用`dealloc`方法, 不需要用户自己写 `release`

#### 自动释放池嵌套使用

* 自动释放池以栈的形式存在
* 由于栈只有一个入口, 所以调用`autorelease`会将对象放到栈顶的自动释放池
* 自动释放池中不宜放占用内存比较大的对象

#### ARC 下的应用

我当前测试的系统是 ios11, 使用数组的block版本的枚举或者使用普通的for循环, 系统内部都会自动添加一个`autoreleasePool`:
```objc
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    // 这里被一个局部@autoreleasepool包围着
}];
for (int i = 0; i<10;i++){
        // 这里被一个局部@autoreleasepool包围着
}
```




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